# Architecture and developer runbook

pdfGPT is a Next.js 15 application for asking questions about one uploaded
document at a time. Document parsing and retrieval run on the server. Document
chunks and chat messages live in browser state.

## Runtime flow

```text
Browser UI (src/app/page.tsx)
  |
  | multipart/form-data: file
  v
POST /api/upload (src/app/api/upload/route.ts)
  |
  v
parseDocument() (src/lib/document-processor.ts)
  |
  | JSON: document metadata + chunks
  v
Browser state stores chunks
  |
  | JSON: question + chunks + modelId + optional apiKey + mode
  v
POST /api/chat (src/app/api/chat/route.ts)
  |
  v
searchChunks() top 8 TF-IDF matches (src/lib/search.ts)
  |
  v
Prompt template + provider client
  |
  v
Vercel AI SDK data stream back to browser
```

There is no vector database, embedding service, document table, or server-side
conversation store. A page refresh clears the uploaded chunks and messages.

## Major subsystems

### Client UI

Source: `src/app/page.tsx`

- Accepts `.pdf`, `.docx`, `.doc`, `.xlsx`, `.xls`, `.csv`, `.pptx`, `.ppt`,
  `.txt`, and `.md` files.
- Stores parsed `DocumentChunk[]`, document metadata, messages, selected model,
  optional API key, and analysis mode in React state.
- Sends the full chunk array with every chat request.
- Handles JSON API errors separately from streaming responses by checking the
  response `content-type`.
- Manually parses Vercel AI SDK data stream lines that begin with `0:`.

### Upload and parsing

Sources:

- `src/app/api/upload/route.ts`
- `src/lib/document-processor.ts`

`POST /api/upload` converts the uploaded `File` into a `Buffer`, calls
`parseDocument(buffer, file.name)`, and returns parsed metadata plus chunks.
Parser selection is extension-based.

Chunking defaults:

| Constant | Value | Meaning |
| --- | --- | --- |
| `CHUNK_SIZE` | `300` | Maximum words per chunk. |
| `CHUNK_OVERLAP` | `50` | Words repeated between adjacent chunks. |

Page values are best-effort:

- PDF pages come from `pdf-parse` metadata and extracted text sections.
- DOCX pages are paragraph indexes, not physical document pages.
- XLSX pages are sheet indexes.
- TXT and Markdown are one-page documents.

PPT/PPTX extraction is intentionally limited in the current implementation. If
no text-like spreadsheet content is found, the parser returns one chunk advising
the user to convert the presentation to PDF.

### Retrieval

Source: `src/lib/search.ts`

`searchChunks(query, chunks, topK = 5)` tokenizes lowercase alphanumeric terms
longer than two characters, computes TF-IDF vectors over the chunks plus the
query, and ranks chunks by cosine similarity.

The chat route overrides the default with `topK = 8`. When all scores are zero,
the function returns the first chunks with small synthetic scores. This keeps the
chat route from producing an empty prompt for vocabulary mismatches, but it also
means some answers may be based on early document content rather than true
matches.

### Chat and model providers

Sources:

- `src/app/api/chat/route.ts`
- `src/lib/models.ts`

The model registry is `MODEL_LIST`. The route expects IDs in
`provider:model-name` format and constructs a provider client from the prefix.

| Provider prefix | Client | API key source |
| --- | --- | --- |
| `google` | `@ai-sdk/google` | Request `apiKey`, then `GOOGLE_GENERATIVE_AI_API_KEY` |
| `groq` | OpenAI-compatible `@ai-sdk/openai` client | Request `apiKey`, then `GROQ_API_KEY` |
| `openai` | `@ai-sdk/openai` | Request `apiKey`, then `OPENAI_API_KEY` |
| `anthropic` | `@ai-sdk/anthropic` | Request `apiKey`, then `ANTHROPIC_API_KEY` |

RAG and agentic modes use the same retrieval results. The only difference is the
prompt template:

- RAG mode asks for a direct answer grounded only in the excerpts.
- Agentic mode asks the model to analyze, extract, synthesize, evaluate, and
  respond with citations.

## Setup

```bash
npm install
npm run dev
```

At least one provider key must be available either in `.env.local` or entered in
the UI for the selected model:

```dotenv
OPENAI_API_KEY=...
GOOGLE_GENERATIVE_AI_API_KEY=...
GROQ_API_KEY=...
ANTHROPIC_API_KEY=...
```

Run the production build with:

```bash
npm run build
```

The `next.config.ts` setting `serverExternalPackages: ["pdf-parse"]` is required
because `pdf-parse` is loaded dynamically on the server.

## Troubleshooting

### Upload fails with `Unsupported file type`

`parseDocument()` selects parsers from the filename extension. Confirm the file
has one of the accepted extensions and that the extension matches the actual file
format.

### DOC, XLS, or PPT files parse poorly

The code routes `.doc` through the DOCX parser, `.xls` through the XLSX parser,
and `.ppt` through the PPTX parser. Older binary Office formats may fail or
return weak text. Convert them to PDF, DOCX, XLSX, or text when possible.

### Presentation uploads return a conversion message

PPTX parsing is best-effort. Convert the deck to PDF to preserve slide text and
page citations.

### Chat returns JSON instead of a stream

The chat route can return JSON for validation errors, no-context responses, or
provider failures. Clients should check `content-type` before reading the body as
a stream.

### Answers ignore the most relevant part of a large document

Retrieval is lexical TF-IDF over the chunk text, not semantic embeddings. Try a
question that uses terms present in the document, or adjust `topK` in
`src/app/api/chat/route.ts` and validate prompt size for the target model.

### Large documents feel slow or produce large requests

The browser stores all chunks and sends them to `/api/chat` on every question.
There is no server-side document cache. Reducing document size, improving
chunking, or adding server-side persistence are the main design options.

### Docker commands do not start the Next.js app

`Dockerfile`, `docker-compose.yaml`, and `requirements.txt` are legacy
Python/Gradio artifacts. The supported development path for the current app is
`npm install` and `npm run dev`.

## Common changes

### Add or update a model

1. Add the display entry in `MODEL_LIST` in `src/lib/models.ts`.
2. Ensure the ID uses an existing provider prefix, or add a provider branch in
   `getProvider()` in `src/app/api/chat/route.ts`.
3. Document the needed environment variable in `README.md` and `docs/API.md`.

### Add or update a parser

1. Implement the parser in `src/lib/document-processor.ts`.
2. Add the extension branch in `parseDocument()`.
3. Return `ParsedDocument` with stable `DocumentChunk` fields.
4. Update `docs/API.md` with supported extensions and limitations.

### Tune retrieval

1. Change chunk size or overlap in `src/lib/document-processor.ts` when upload
   output needs different granularity.
2. Change `topK` in `src/app/api/chat/route.ts` when prompts need more or fewer
   excerpts.
3. Change tokenization or zero-score fallback behavior in `src/lib/search.ts`
   when ranking quality needs improvement.
4. Test with representative short and large documents because changes affect
   request size, prompt size, and citation quality.

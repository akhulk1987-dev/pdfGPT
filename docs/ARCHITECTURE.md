# Architecture and developer runbook

pdfGPT is a single Next.js application that parses a document, keeps the parsed
chunks in browser state, and sends those chunks with each chat request. It does
not use a database, vector store, or embedding API.

## Request flow

```text
Browser UI
  |
  | multipart upload
  v
/api/upload
  |
  | parseDocument(buffer, filename)
  v
DocumentChunk[] returned to browser state
  |
  | JSON question + chunks + modelId + mode
  v
/api/chat
  |
  | searchChunks(question, chunks, 8)
  v
prompt with [Page N] excerpts
  |
  | streamText(...)
  v
Vercel AI SDK data stream consumed by the UI
```

Key source files:

| Area | File |
| --- | --- |
| Main React UI, upload state, chat stream reader | `src/app/page.tsx` |
| Upload route | `src/app/api/upload/route.ts` |
| Chat route and prompt templates | `src/app/api/chat/route.ts` |
| File parsing and chunking | `src/lib/document-processor.ts` |
| TF-IDF retrieval | `src/lib/search.ts` |
| Model registry | `src/lib/models.ts` |

## Document processing model

`parseDocument` selects a parser from the uploaded filename extension. Every parser
returns:

```ts
interface ParsedDocument {
  name: string;
  type: string;
  pageCount: number;
  chunks: DocumentChunk[];
}

interface DocumentChunk {
  text: string;
  page: number;
  source: string;
  index: number;
}
```

Chunking is word-based with a 300-word size and 50-word overlap. The `page` value
means different things by file type:

- PDF: page number from extracted PDF text.
- DOC/DOCX: paragraph block number.
- XLS/XLSX/CSV: workbook sheet number.
- TXT/Markdown: always page 1.
- PPT/PPTX: best-effort fallback; users may need to convert presentations to PDF
  for reliable extraction.

The PDF parser is dynamically imported, and `next.config.ts` lists `pdf-parse` in
`serverExternalPackages`. Keep that setting when changing PDF parsing behavior.

## Retrieval and answer generation

`searchChunks` tokenizes text to lowercase alphanumeric terms longer than two
characters, computes TF-IDF vectors for all chunks plus the query, and ranks chunks
by cosine similarity. `/api/chat` requests the top 8 chunks.

If every similarity score is zero, retrieval falls back to the first chunks so the
LLM still receives context. This makes generic questions less likely to fail, but
it also means answers can depend on document order when query terms do not match.

RAG and Agentic modes share the same retrieval path:

- RAG mode uses a constrained prompt that tells the model to answer only from the
  retrieved excerpts and cite `[Page N]`.
- Agentic mode asks the model to structure the response around Analyze, Extract,
  Synthesize, Evaluate, and Respond steps.

## Model routing

The UI model dropdown is built from `MODEL_LIST` in `src/lib/models.ts`.
`/api/chat` routes each request by splitting the selected model ID as
`provider:model-name`.

Provider keys can come from the request body `apiKey` field or from environment
variables:

- `GOOGLE_GENERATIVE_AI_API_KEY`
- `GROQ_API_KEY`
- `OPENAI_API_KEY`
- `ANTHROPIC_API_KEY`

To add a model for an existing provider, add an entry to `MODEL_LIST` using the
same provider prefix. To add a new provider, also add a case to `getProvider` in
`src/app/api/chat/route.ts`.

## Local development checks

```bash
npm install
npm run dev
npm run build
```

The `lint` script currently maps to `next lint`; verify it still works with the
installed Next.js version before relying on it in CI.

## Common troubleshooting

### Upload returns `Unsupported file type`

The parser is selected only from the filename extension. Confirm the uploaded file
has one of the accepted extensions in the file input and `parseDocument` switch.

### PDF parsing fails in production builds

Confirm `next.config.ts` still includes:

```ts
serverExternalPackages: ["pdf-parse"]
```

### Chat returns a JSON error instead of streaming

The UI treats `application/json` responses as errors. Check for:

- missing `question`
- empty or malformed `chunks`
- unknown `modelId` provider prefix
- missing or invalid provider key

### Answers cite unexpected page numbers

Only PDFs have real page numbers. Other formats use section-like units such as
paragraph blocks or sheets, so citations should be understood as source units.

### Retrieval misses obvious text

TF-IDF search is lexical. It does not understand synonyms unless those terms also
appear in the chunks. Rephrase with document vocabulary or improve retrieval before
changing prompt behavior.

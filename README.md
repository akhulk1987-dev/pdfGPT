# pdfGPT - Smart Document Analysis

Chat with documents using AI. The current app is a Next.js 15 web application that parses uploaded files server-side, keeps document chunks in browser state, retrieves relevant chunks with TF-IDF, and streams an LLM answer back to the UI.

**Live Demo**: [pdfgpt.vercel.app](https://pdfgpt.vercel.app) *(deploy your own instance - see below)*

## Features

- **Multi-format support**: PDF, DOCX, XLSX, CSV, PPTX, TXT, and Markdown uploads
- **RAG Mode**: Fast retrieval-augmented answers over the top matching chunks
- **Agentic Mode**: A deeper prompt that asks the model to analyze, extract, synthesize, evaluate, and respond
- **Multiple LLM providers**: Google Gemini, Groq, OpenAI, and Anthropic via the Vercel AI SDK
- **Page-style citations**: Chat prompts require answers to cite retrieved chunks with `[Page N]`
- **Streaming responses**: `/api/chat` returns a Vercel AI SDK data stream consumed by the React client
- **Vercel-ready**: Next.js app router deployment with `pdf-parse` configured as a server external package

## Quick Start

```bash
git clone https://github.com/akhulk1987-dev/pdfGPT.git
cd pdfGPT
npm install

# Set at least one provider key. The selected model determines which one is used.
echo "OPENAI_API_KEY=your-key-here" > .env.local

npm run dev
```

Open [http://localhost:3000](http://localhost:3000).

Useful scripts:

```bash
npm run dev    # local development server
npm run build  # production build
npm run start  # serve a production build
npm run lint   # lint script defined by package.json
```

## Environment Variables and Models

The model registry lives in `src/lib/models.ts`. Model IDs use the form `provider:model-name`; `/api/chat` splits the ID at `:` and routes to the matching provider SDK.

| Provider | Model IDs | Environment variable |
|----------|-----------|----------------------|
| Google | `google:gemini-2.0-flash`, `google:gemini-1.5-flash` | `GOOGLE_GENERATIVE_AI_API_KEY` |
| Groq | `groq:llama-3.1-8b-instant`, `groq:llama-3.3-70b-versatile`, `groq:mixtral-8x7b-32768` | `GROQ_API_KEY` |
| OpenAI | `openai:gpt-4o`, `openai:gpt-4o-mini` | `OPENAI_API_KEY` |
| Anthropic | `anthropic:claude-sonnet-4-20250514`, `anthropic:claude-3-5-haiku-20241022` | `ANTHROPIC_API_KEY` |

Users can also enter an API key in the UI. When present, that key is sent in the `/api/chat` request and takes precedence over the server environment variable for the selected provider.

## Architecture

```text
Next.js 15 App Router
  |
  +-- src/app/page.tsx
  |     React client UI, file picker, model picker, API-key field,
  |     RAG/Agentic mode toggle, streaming chat renderer
  |
  +-- src/app/api/upload/route.ts
  |     Multipart file upload -> parseDocument(...) -> ParsedDocument JSON
  |
  +-- src/app/api/chat/route.ts
  |     Question + chunks + model ID -> TF-IDF search -> prompt -> data stream
  |
  +-- src/lib/document-processor.ts
  |     File parsing and 300-word chunk generation with 50-word overlap
  |
  +-- src/lib/search.ts
        In-process TF-IDF scoring; no vector database or embedding API
```

Document content is not persisted by the server. After upload, `page.tsx` stores `DocumentChunk[]` in React state and sends those chunks with each chat request. Refreshing the page or removing the document clears the working set.

## Upload API

`POST /api/upload` accepts a multipart form with one `file` field.

```bash
curl -X POST http://localhost:3000/api/upload \
  -F "file=@example.pdf"
```

Successful responses mirror the `ParsedDocument` shape from `src/lib/document-processor.ts`:

```json
{
  "name": "example.pdf",
  "type": "pdf",
  "pageCount": 12,
  "chunkCount": 34,
  "chunks": [
    {
      "text": "Chunk text...",
      "page": 1,
      "source": "example.pdf",
      "index": 0
    }
  ]
}
```

Errors:

- Missing `file`: `400` with `{ "error": "No file provided" }`
- Unsupported extension: `500` with `{ "error": "Unsupported file type: .ext" }`
- Parser failures: `500` with the parser error message

### Parser behavior and constraints

| Extension | Parser path | `page` / `pageCount` meaning | Notes |
|-----------|-------------|------------------------------|-------|
| `.pdf` | `pdf-parse` | PDF page number / reported PDF page count | Text is split on form-feed page separators from `pdf-parse`. |
| `.docx`, `.doc` | `mammoth.extractRawText` | Paragraph group index / paragraph group count | Styling and rich document structure are not preserved. |
| `.xlsx`, `.xls`, `.csv` | `xlsx` | Sheet index / sheet count | Sheets are converted to CSV text before chunking. |
| `.txt`, `.md` | UTF-8 buffer text | Always page `1` / page count `1` | No front matter or Markdown structure parsing is performed. |
| `.pptx`, `.ppt` | Lightweight `xlsx` fallback | Slide-like fallback index | Extraction is limited; if no text is found, the response contains a fallback chunk recommending PDF conversion. |

All parser paths use the same chunker:

- `CHUNK_SIZE = 300` words
- `CHUNK_OVERLAP = 50` words
- Step size is 250 words
- Non-initial chunks shorter than 15 words are skipped

## Chat API

`POST /api/chat` accepts JSON from the client:

```json
{
  "question": "What are the renewal terms?",
  "chunks": [
    { "text": "Contract excerpt...", "page": 4, "source": "contract.pdf", "index": 7 }
  ],
  "modelId": "openai:gpt-4o-mini",
  "apiKey": "optional-user-supplied-key",
  "mode": "rag"
}
```

Required fields:

- `question`: user prompt. Missing values return `400` with `{ "error": "No question" }`.
- `chunks`: document chunks from `/api/upload`. Empty or missing chunks produce no search context.
- `modelId`: one of the IDs in `MODEL_LIST`.

Optional fields:

- `apiKey`: overrides the matching provider environment variable for this request.
- `mode`: `rag` or `agentic`; defaults to `rag`.

The route searches the supplied chunks with `searchChunks(question, chunks, 8)`, builds a prompt from the top eight results, then calls `streamText` with `temperature: 0.3` and `maxTokens: 2048`. If no context can be built, it returns `200` JSON with `{ "error": "No relevant content found. Try a different question." }`; otherwise it returns a Vercel AI SDK data stream.

### Retrieval behavior

`src/lib/search.ts` implements local TF-IDF search:

1. Lowercase text, remove non-alphanumeric characters, split on whitespace, and ignore tokens with length `<= 2`.
2. Build TF-IDF vectors for every chunk plus the query.
3. Score chunks with cosine similarity against the query vector.
4. Return the top `K` chunks sorted by score.
5. If every score is zero but chunks exist, return the first `K` chunks with small fallback scores.

This keeps retrieval fast and self-contained, but it is keyword-oriented. Semantic matches that do not share terms with the question may rank poorly.

## Client Workflow

1. The user uploads a file from `src/app/page.tsx`.
2. The client posts the file to `/api/upload`.
3. Parsed chunks and document metadata are stored in React state.
4. The user selects RAG or Agentic mode and a model.
5. Each question posts the full chunk array, selected `modelId`, optional UI API key, and mode to `/api/chat`.
6. The client reads the AI SDK data stream and appends `0:` text chunks to the assistant message as they arrive.

Because chunks live only in client state, a browser refresh or document removal requires uploading the file again before asking more questions.

## Deploy to Vercel

[![Deploy with Vercel](https://vercel.com/button)](https://vercel.com/new/clone?repository-url=https://github.com/akhulk1987-dev/pdfGPT)

Set provider API keys as environment variables in the Vercel dashboard. `next.config.ts` includes:

```ts
serverExternalPackages: ["pdf-parse"]
```

Keep that setting when changing the Next.js configuration; `pdf-parse` is loaded dynamically on the server by `src/lib/document-processor.ts`.

## Troubleshooting

- **`Unsupported file type`**: confirm the extension is one of `pdf`, `docx`, `doc`, `xlsx`, `xls`, `csv`, `pptx`, `ppt`, `txt`, or `md`.
- **No answer after upload**: the chat route needs non-empty `chunks`. Check the `/api/upload` response for `chunkCount`.
- **No relevant content found**: TF-IDF found no context. Rephrase the question with terms used in the document or verify the parser extracted text.
- **Provider authentication errors**: set the environment variable for the selected model provider, or enter a key in the UI. The UI key is sent to the server for that request.
- **Weak PPTX extraction**: convert the presentation to PDF when the fallback chunk says text could not be extracted.
- **Unexpected citation labels**: `[Page N]` means the parser's page-like index. For DOCX it maps to paragraph groups; for spreadsheets it maps to sheet indexes.
- **Legacy Python files**: `Dockerfile`, `docker-compose.yaml`, and `requirements.txt` describe the previous Python/Gradio app and are not the supported Next.js run path.

## License

MIT License. See [LICENSE.txt](LICENSE.txt).

## Citation

```bibtex
@misc{pdfgpt2023,
  author = {Bhaskar Tripathi},
  title = {pdfGPT},
  year = {2023},
  publisher = {GitHub},
  journal = {GitHub Repository},
  howpublished = {\url{https://github.com/bhaskatripathi/pdfGPT}}
}
```

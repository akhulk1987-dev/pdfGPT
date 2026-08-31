# pdfGPT - Smart Document Analysis

pdfGPT is a Next.js app for chatting with uploaded documents. It parses a
document into server-generated text chunks, keeps those chunks in browser state,
retrieves the most relevant chunks with TF-IDF search, and streams an LLM answer
with page citations.

## Features

- **Multi-format uploads**: PDF, DOC/DOCX, XLS/XLSX/CSV, PPT/PPTX, TXT, and Markdown.
- **Two analysis modes**:
  - **RAG** for concise retrieval-augmented answers over the top matching chunks.
  - **Agentic** for a structured Analyze, Extract, Synthesize, Evaluate, Respond prompt.
- **Multiple LLM providers**: Google Gemini, Groq, OpenAI, and Anthropic through the Vercel AI SDK.
- **No vector database required**: document search is local TF-IDF over uploaded chunks.
- **Streaming chat UI**: responses are streamed from `/api/chat` into the React client.
- **Vercel-ready**: the app uses the Next.js App Router and serverless route handlers.

## Quick start

```bash
git clone https://github.com/akhulk1987-dev/pdfGPT.git
cd pdfGPT
npm install

# Configure at least one provider used by the selected model.
echo "OPENAI_API_KEY=your-key-here" > .env.local

npm run dev
```

Open [http://localhost:3000](http://localhost:3000), upload a document, choose a
model and mode, then ask a question. The sidebar API key field can override the
server environment key for a single request.

### Scripts

| Command | Purpose |
| --- | --- |
| `npm run dev` | Start the Next.js development server. |
| `npm run build` | Create a production build. |
| `npm run start` | Serve the production build. |
| `npm run lint` | Run the configured lint script. |

## Required environment variables

The selected model determines which key is required. Keys can be set in
`.env.local`, in Vercel project settings, or supplied from the UI for an
individual chat request.

| Provider | Model ID prefix | Environment variable |
| --- | --- | --- |
| Google | `google:` | `GOOGLE_GENERATIVE_AI_API_KEY` |
| Groq | `groq:` | `GROQ_API_KEY` |
| OpenAI | `openai:` | `OPENAI_API_KEY` |
| Anthropic | `anthropic:` | `ANTHROPIC_API_KEY` |

Model options live in `src/lib/models.ts`. When adding a new model, add it to
`MODEL_LIST`; when adding a new provider prefix, also update `getProvider()` in
`src/app/api/chat/route.ts`.

## Architecture

```text
src/app/page.tsx
  React client UI: upload control, model/mode selection, API-key override,
  in-memory document chunks, and Vercel data-stream parsing.

src/app/api/upload/route.ts
  POST multipart/form-data file -> parseDocument() -> ParsedDocument JSON.

src/lib/document-processor.ts
  Multi-format parsing and 300-word chunking with 50-word overlap.

src/lib/search.ts
  Tokenization, TF-IDF vectors, cosine similarity, and top-K retrieval.

src/app/api/chat/route.ts
  POST question + chunks + modelId + mode -> search -> prompt -> streamText().
```

### Request flow

1. The client uploads one file to `/api/upload` as `multipart/form-data` under
   the `file` field.
2. The upload route parses the file and returns `ParsedDocument` metadata plus
   `DocumentChunk[]`.
3. The client stores chunks in component state. Chunks are not persisted and
   are sent back with every chat request.
4. `/api/chat` runs `searchChunks(question, chunks, 8)` and builds context lines
   as `[Page N] chunk text`.
5. The route chooses the prompt template from `mode` (`rag` or `agentic`) and
   streams the LLM response with `streamText().toDataStreamResponse()`.

## API contracts

### `POST /api/upload`

Request:

```text
Content-Type: multipart/form-data
file=<PDF, Office document, CSV, text, or markdown file>
```

Success response:

```ts
{
  name: string;
  type: string;
  pageCount: number;
  chunkCount: number;
  chunks: DocumentChunk[];
}
```

Errors:

- `400 { "error": "No file provided" }` when the `file` field is missing.
- `500 { "error": string }` when parsing fails or the extension is unsupported.

### `POST /api/chat`

Request body:

```ts
{
  question: string;
  chunks: DocumentChunk[];
  modelId: "provider:model-name";
  apiKey?: string;
  mode?: "rag" | "agentic"; // defaults to "rag"
}
```

Responses:

- A Vercel AI SDK data stream on success.
- `400 { "error": "No question" }` when `question` is empty.
- `200 { "error": "No relevant content found. Try a different question." }`
  when no context can be built.
- `500 { "error": string }` for provider, streaming, or server failures.

### Shared document types

```ts
interface DocumentChunk {
  text: string;
  page: number;
  source: string;
  index: number;
}

interface ParsedDocument {
  name: string;
  type: string;
  pageCount: number;
  chunks: DocumentChunk[];
}
```

## Parser behavior and constraints

- Chunks are up to 300 words with a 50-word overlap. Tiny trailing chunks under
  15 words are skipped after the first chunk.
- PDF parsing uses `pdf-parse`; `next.config.ts` marks it as a
  `serverExternalPackages` entry for Next.js server execution.
- DOC and DOCX route through Mammoth raw-text extraction. Validate legacy binary
  `.doc` files before relying on them in production workflows.
- XLS, XLSX, and CSV route through `xlsx` and are converted sheet-by-sheet to
  CSV-like text. Sheet number is used as the chunk page number.
- PPT and PPTX currently use a lightweight `xlsx`-based fallback. If text cannot
  be extracted, the parser returns a placeholder chunk that asks the user to
  convert the presentation to PDF first.
- TXT and Markdown files are read as UTF-8 text and treated as one page.

## Troubleshooting

- **"Upload a document first"**: the client has no chunks in state. Upload a
  supported file before asking a question.
- **Unsupported file type**: `parseDocument()` chooses the parser from the file
  extension. Add a case there before accepting a new extension in the UI.
- **Provider authentication errors**: confirm the selected model prefix matches
  a configured key, or supply the key in the sidebar for that request.
- **No relevant content found**: the search route received chunks but could not
  build usable context. Try a more specific question or confirm parsing returned
  non-empty chunks.
- **PPT/PPTX extraction is weak**: convert presentations to PDF for more
  reliable page text and citations.

## Deployment

Deploy with Vercel or any host that supports Next.js 15. Configure provider API
keys as environment variables in the deployment target. No external search
service, vector database, or embedding API is required.

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

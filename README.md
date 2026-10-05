# pdfGPT - Smart Document Analysis

Chat with documents using AI. Upload a PDF, Word, Excel, CSV, PowerPoint, text, or Markdown file, then ask cited questions against its contents.

**Live Demo**: [pdfgpt.vercel.app](https://pdfgpt.vercel.app) *(deploy your own instance - see below)*

## Features

- **Multi-format upload**: PDF, DOCX, XLSX/XLS, CSV, PPTX/PPT, TXT, and Markdown.
- **RAG mode**: Fast retrieval-augmented answers from the most relevant excerpts.
- **Agentic mode**: Deeper multi-step analysis using an Analyze, Extract, Synthesize, Evaluate, Respond prompt.
- **Multiple LLM providers**: OpenAI, Anthropic, Google Gemini, and Groq via the Vercel AI SDK.
- **Free model options**: Google Gemini and Groq models are listed separately in the UI.
- **Page citations**: Prompts require `[Page N]` citations for document-backed claims.
- **Streaming responses**: Answers stream back to the React client as Vercel AI SDK data stream chunks.
- **Vercel-ready**: Next.js 15 App Router app with no database or vector store dependency.

## Quick start

```bash
# Clone and install
git clone https://github.com/bhaskatripathi/pdfGPT.git
cd pdfGPT
npm install

# Set at least one provider key for server-side fallback
echo "OPENAI_API_KEY=your-key-here" > .env.local

# Run the Next.js dev server
npm run dev
```

Open [http://localhost:3000](http://localhost:3000).

### Useful scripts

```bash
npm run dev    # Local development server on :3000
npm run build  # Production Next.js build
npm run start  # Serve a production build
npm run lint   # Next.js ESLint config
```

## Configuration

The chat API accepts an optional per-request API key from the UI. If a user does not enter one, the server falls back to the matching environment variable.

| Provider | Model IDs | Environment variable |
| --- | --- | --- |
| Google | `google:gemini-2.0-flash`, `google:gemini-1.5-flash` | `GOOGLE_GENERATIVE_AI_API_KEY` |
| Groq | `groq:llama-3.1-8b-instant`, `groq:llama-3.3-70b-versatile`, `groq:mixtral-8x7b-32768` | `GROQ_API_KEY` |
| OpenAI | `openai:gpt-4o`, `openai:gpt-4o-mini` | `OPENAI_API_KEY` |
| Anthropic | `anthropic:claude-sonnet-4-20250514`, `anthropic:claude-3-5-haiku-20241022` | `ANTHROPIC_API_KEY` |

When adding models, update both `src/lib/models.ts` for the UI registry and `src/app/api/chat/route.ts` if the provider routing changes.

## Architecture

```text
Next.js 15 App Router
  |
  +-- src/app/page.tsx
  |     React UI, file picker, model selector, mode selector,
  |     local document chunk state, streamed chat rendering
  |
  +-- POST /api/upload
  |     Multipart file upload -> parseDocument -> JSON chunks
  |
  +-- POST /api/chat
        question + chunks + modelId + optional apiKey + mode
        -> TF-IDF search -> prompt -> streamText()
```

There is no persisted document store. The browser keeps parsed chunks in React state after upload and sends those chunks with every chat request. Refreshing the page, clearing the document, or uploading a new file discards the current chunk set and chat history.

## Document processing

`src/lib/document-processor.ts` normalizes uploaded files into this shared shape:

```ts
interface DocumentChunk {
  text: string;
  page: number;
  source: string;
  index: number;
}
```

Chunks are word-based:

- `CHUNK_SIZE` is 300 words.
- `CHUNK_OVERLAP` is 50 words.
- Trailing chunks under 15 words are skipped unless they are the first chunk for that source segment.

Supported extensions and parser behavior:

| Extension | Parser path | Notes |
| --- | --- | --- |
| `.pdf` | `pdf-parse` | Text is split on form-feed page separators; `pageCount` comes from the PDF metadata. |
| `.docx`, `.doc` | `mammoth.extractRawText` | Best with DOCX files. Legacy binary `.doc` files may fail and should be converted to DOCX or PDF. |
| `.xlsx`, `.xls`, `.csv` | `xlsx` | Each sheet is converted to CSV text and treated as a page-like unit. CSV files follow the same workbook path. |
| `.pptx`, `.ppt` | Best-effort `xlsx` fallback | Presentation extraction is limited. If no text is extracted, the API returns a fallback chunk asking the user to convert to PDF. |
| `.txt`, `.md` | UTF-8 text | Treated as a single page. |

`pdf-parse` is listed in `next.config.ts` under `serverExternalPackages`; keep that setting when changing the PDF parser path.

## Chat workflow

1. The user uploads a document through the sidebar file input in `src/app/page.tsx`.
2. `POST /api/upload` receives `multipart/form-data` with a `file` field and returns:

   ```json
   {
     "name": "example.pdf",
     "type": "pdf",
     "pageCount": 12,
     "chunkCount": 34,
     "chunks": []
   }
   ```

3. The client stores `chunks` and document metadata in React state.
4. The user asks a question. The client posts this payload to `POST /api/chat`:

   ```json
   {
     "question": "What are the renewal terms?",
     "chunks": [],
     "modelId": "openai:gpt-4o-mini",
     "apiKey": "optional-user-supplied-key",
     "mode": "rag"
   }
   ```

5. `/api/chat` runs `searchChunks(question, chunks, 8)` and formats excerpts as `[Page N] text`.
6. RAG mode uses a concise grounded-answer prompt. Agentic mode uses the multi-step analysis prompt.
7. `streamText()` returns a Vercel AI SDK data stream. The client reads text chunks prefixed with `0:` and appends them to the assistant message.

If TF-IDF scoring returns all zeroes, `searchChunks` falls back to the first `topK` chunks with small decreasing scores so the model still has document context.

## API reference

### `POST /api/upload`

- **Request**: `multipart/form-data` with required `file`.
- **Success**: JSON document metadata and chunks.
- **Errors**:
  - `400` with `{ "error": "No file provided" }` when the file field is missing.
  - `500` with the thrown parser message for unsupported extensions or parse failures.

### `POST /api/chat`

- **Request**: JSON with `question`, `chunks`, `modelId`, optional `apiKey`, and optional `mode`.
- **Modes**: `rag` by default, or `agentic`.
- **Success**: Vercel AI SDK streaming response.
- **Errors**:
  - `400` with `{ "error": "No question" }` when the question is empty.
  - `200` with `{ "error": "No relevant content found. Try a different question." }` when no context is available.
  - `500` with `{ "error": "..." }` for provider, routing, or streaming setup errors.

## Search constraints

`src/lib/search.ts` implements local TF-IDF search. It lowercases text, removes non-alphanumeric characters, drops tokens shorter than three characters, computes cosine similarity, and sorts descending by score. This avoids external embedding APIs and vector databases, but it also means:

- exact wording matters more than semantic similarity;
- very short identifiers or punctuation-heavy values may be ignored;
- search quality depends on the chunks sent by the client;
- larger documents increase request payload size because chunks are reposted for each chat turn.

## Deployment

[![Deploy with Vercel](https://vercel.com/button)](https://vercel.com/new/clone?repository-url=https://github.com/bhaskatripathi/pdfGPT)

Set the provider API keys you want to support in the Vercel dashboard. The app does not require a database, background worker, object storage bucket, vector store, or embedding service.

## Troubleshooting

- **"Unknown provider"**: The `modelId` prefix before `:` is not handled in `getProvider()` in `src/app/api/chat/route.ts`.
- **Provider authentication failures**: Enter a key in the UI or set the provider's environment variable on the server.
- **PDF parser errors in Next.js**: Confirm `pdf-parse` remains in `serverExternalPackages` in `next.config.ts`.
- **Poor answers for a valid document**: Try RAG first for focused questions, Agentic mode for broad analysis, and rephrase using terms that appear in the document.
- **PowerPoint content missing**: Convert the presentation to PDF and upload the PDF.
- **Docker commands fail**: `Dockerfile`, `docker-compose.yaml`, and `requirements.txt` are legacy Python/Gradio artifacts. Use the npm scripts above for the supported Next.js app.

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

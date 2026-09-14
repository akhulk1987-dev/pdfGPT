# pdfGPT - Smart Document Analysis

Chat with your documents using AI. Upload a PDF, Word, Excel, PowerPoint, text,
CSV, or Markdown file and ask questions grounded in the extracted content.

**Live Demo**: [pdfgpt.vercel.app](https://pdfgpt.vercel.app) *(deploy your own instance - see below)*

## Features

- **Multi-format upload**: PDF, DOCX/DOC, XLSX/XLS/CSV, PPTX/PPT, TXT, and Markdown
- **RAG mode**: Fast retrieval-augmented generation for focused answers
- **Agentic mode**: Deeper analysis prompt with Analyze, Extract, Synthesize, Evaluate, and Respond steps
- **Multiple LLM providers**: OpenAI, Anthropic, Google Gemini, and Groq through the Vercel AI SDK
- **Free model options**: Google Gemini and Groq model entries are available in the default selector
- **Source citations**: Chat prompts require `[Page N]` citations for claims grounded in retrieved chunks
- **Streaming responses**: `/api/chat` streams AI SDK data-stream responses to the browser
- **Modern UI**: Dark-themed responsive Next.js 15, React 19, and Tailwind CSS 4 app
- **Vercel-ready**: Deploy as a standard Next.js app

## Quick Start

```bash
# Clone and install
git clone https://github.com/akhulk1987-dev/pdfGPT.git
cd pdfGPT
npm install

# Set at least one provider key for the model you will use
echo "OPENAI_API_KEY=your-key-here" > .env.local

# Run the development server
npm run dev
```

Open [http://localhost:3000](http://localhost:3000).

Useful scripts:

```bash
npm run dev    # Next.js dev server on :3000
npm run build  # Production build
npm run start  # Serve a production build
npm run lint   # Next.js lint command
```

## Configuration

Set provider keys in `.env.local`, Vercel environment variables, or paste a key
into the UI for a single request. The chat API uses the request key first and
falls back to the matching environment variable.

| Provider | Model IDs | Environment variable |
|----------|-----------|----------------------|
| Google | `google:gemini-2.0-flash`, `google:gemini-1.5-flash` | `GOOGLE_GENERATIVE_AI_API_KEY` |
| Groq | `groq:llama-3.1-8b-instant`, `groq:llama-3.3-70b-versatile`, `groq:mixtral-8x7b-32768` | `GROQ_API_KEY` |
| OpenAI | `openai:gpt-4o`, `openai:gpt-4o-mini` | `OPENAI_API_KEY` |
| Anthropic | `anthropic:claude-sonnet-4-20250514`, `anthropic:claude-3-5-haiku-20241022` | `ANTHROPIC_API_KEY` |

## Architecture

```
Next.js 15 App Router
  |
  +-- src/app/page.tsx              Browser UI, upload flow, chat streaming
  +-- src/app/api/upload/route.ts   File upload endpoint
  +-- src/app/api/chat/route.ts     Retrieval, prompt construction, LLM streaming
  +-- src/lib/document-processor.ts Multi-format parsing and chunking
  +-- src/lib/search.ts             In-process TF-IDF search
  +-- src/lib/models.ts             Model registry used by the UI
```

Workflow:

1. The browser uploads one file to `POST /api/upload` as multipart form data.
2. The server parses the file into `DocumentChunk` objects of roughly 300 words
   with a 50-word overlap.
3. The browser stores the returned chunks in React state. Chunks are not written
   to a database by this app.
4. Each question is sent to `POST /api/chat` with the selected `modelId`, mode,
   optional per-request `apiKey`, and the current chunks.
5. `/api/chat` runs TF-IDF search over the supplied chunks, keeps the top 8
   matches, builds either the RAG or Agentic prompt, and returns a streamed AI
   SDK data response.

## API Contracts

### `POST /api/upload`

Accepts multipart form data with a single `file` field.

Example:

```bash
curl -F "file=@sample.pdf" http://localhost:3000/api/upload
```

Response:

```json
{
  "name": "sample.pdf",
  "type": "pdf",
  "pageCount": 12,
  "chunkCount": 18,
  "chunks": [
    {
      "text": "Extracted document text...",
      "page": 1,
      "source": "sample.pdf",
      "index": 0
    }
  ]
}
```

Errors:

- `400` with `{ "error": "No file provided" }` when the form has no file.
- `500` with `{ "error": "Unsupported file type: .ext" }` or parser failures.

### `POST /api/chat`

Accepts JSON:

```json
{
  "question": "What are the payment terms?",
  "chunks": [
    { "text": "Net 30 payment terms...", "page": 2, "source": "contract.pdf", "index": 3 }
  ],
  "modelId": "openai:gpt-4o-mini",
  "mode": "rag",
  "apiKey": "optional-request-key"
}
```

Fields:

- `question` is required.
- `chunks` should be the array returned by `/api/upload`.
- `modelId` must use the `provider:model-name` format recognized by
  `src/app/api/chat/route.ts`.
- `mode` is optional and defaults to `rag`; pass `agentic` for the deeper
  analysis prompt.
- `apiKey` is optional and overrides the configured environment variable for
  that request.

Response behavior:

- Successful answers are streamed with `streamText().toDataStreamResponse()`.
- Missing `question` returns `400` with `{ "error": "No question" }`.
- If no chunks are supplied, the endpoint returns `200` with
  `{ "error": "No relevant content found. Try a different question." }`.
- Unknown providers or LLM/provider errors return `500` JSON errors.

## Parsing and Search Constraints

- PDF parsing uses `pdf-parse`; `next.config.ts` lists `pdf-parse` in
  `serverExternalPackages`, which is required for the Next.js server bundle.
- DOC/DOCX parsing uses Mammoth raw text extraction.
- XLS/XLSX/CSV parsing uses `xlsx` and converts each sheet to CSV text.
- PPT/PPTX support is best-effort. The current parser attempts a lightweight
  spreadsheet-like extraction and otherwise returns a chunk explaining that text
  could not be extracted; converting presentations to PDF is more reliable.
- Plain text and Markdown are read as UTF-8 and treated as one page.
- Search is lexical TF-IDF in `src/lib/search.ts`. It lowercases text, removes
  non-alphanumeric characters, ignores tokens shorter than 3 characters, and
  falls back to the first chunks when every cosine similarity score is zero.
- Uploaded chunks live in browser state. Refreshing the page or removing the
  document clears the current document context.

## Troubleshooting

- **"Upload a document first"**: The chat form is disabled until `/api/upload`
  returns chunks for the selected file.
- **"No relevant content found"**: The chat endpoint received no chunks. Upload
  a document again or confirm the client is sending the `chunks` array.
- **Provider authentication errors**: Set the provider environment variable for
  the selected `modelId`, or paste the provider key into the UI before sending.
- **PDF parser build/runtime issues**: Confirm `next.config.ts` includes
  `serverExternalPackages: ["pdf-parse"]`.
- **Weak PowerPoint extraction**: Convert the presentation to PDF before upload
  when slide text is important.

## Deploy to Vercel

[![Deploy with Vercel](https://vercel.com/button)](https://vercel.com/new/clone?repository-url=https://github.com/akhulk1987-dev/pdfGPT)

Set the provider API keys you need as environment variables in the Vercel
dashboard. No database or vector store is required for the current workflow.

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

# pdfGPT - Smart Document Analysis

Chat with your documents using AI. Upload a PDF, Word, Excel, PowerPoint, CSV,
text, or Markdown file, then ask questions using either fast RAG retrieval or a
deeper multi-step analysis prompt.

**Live Demo**: [pdfgpt.vercel.app](https://pdfgpt.vercel.app) *(deploy your own instance - see below)*

## Features

- **Multi-format support**: PDF, DOCX/DOC, XLSX/XLS/CSV, PPTX/PPT, TXT, Markdown
- **RAG mode**: Fast retrieval-augmented answers over the most relevant chunks
- **Agentic mode**: A structured Analyze, Extract, Synthesize, Evaluate, Respond prompt
- **Multiple LLM providers**: Google Gemini, Groq, OpenAI, and Anthropic via the Vercel AI SDK
- **Client-managed documents**: Parsed chunks stay in React state and are sent with each chat request
- **Streaming responses**: Real-time token streaming from `/api/chat`
- **Modern UI**: Dark-themed, responsive interface built with Next.js and Tailwind CSS
- **Vercel-ready**: Next.js deployment with no vector database or embedding service required

## Quick Start

```bash
# Clone and install
git clone https://github.com/akhulk1987-dev/pdfGPT.git
cd pdfGPT
npm install

# Set up at least one provider key
echo "OPENAI_API_KEY=your-key-here" > .env.local

# Run the Next.js app
npm run dev
```

Open [http://localhost:3000](http://localhost:3000).

## Configuration

Set one or more provider API keys in `.env.local` for local development, or in
the Vercel dashboard for deployments.

| Provider | Environment variable | Models in the UI |
|----------|----------------------|------------------|
| Google | `GOOGLE_GENERATIVE_AI_API_KEY` | `google:gemini-2.0-flash`, `google:gemini-1.5-flash` |
| Groq | `GROQ_API_KEY` | `groq:llama-3.1-8b-instant`, `groq:llama-3.3-70b-versatile`, `groq:mixtral-8x7b-32768` |
| OpenAI | `OPENAI_API_KEY` | `openai:gpt-4o`, `openai:gpt-4o-mini` |
| Anthropic | `ANTHROPIC_API_KEY` | `anthropic:claude-sonnet-4-20250514`, `anthropic:claude-3-5-haiku-20241022` |

The chat request can also include an `apiKey` value from the UI. When present,
that key is used for the selected request instead of the matching environment
variable.

## How the App Works

```
Next.js 15 (App Router)
  |
  +-- src/app/page.tsx          -> Upload, model selection, mode toggle, streaming chat UI
  +-- src/app/api/upload/route.ts
  |     parses one file and returns document chunks
  +-- src/app/api/chat/route.ts
  |     searches chunks, builds a RAG or agentic prompt, streams model output
  |
  +-- src/lib/document-processor.ts -> Format-specific parsing and chunking
  +-- src/lib/search.ts             -> In-process TF-IDF search
  +-- src/lib/models.ts             -> Model registry used by the UI
```

There is no persisted document store. The browser uploads a file to
`/api/upload`, keeps the returned chunks in component state, and sends those
chunks back to `/api/chat` with each question.

## API Contracts

### `POST /api/upload`

Send multipart form data with one field:

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `file` | File | Yes | Extension determines the parser |

Successful responses include:

```json
{
  "name": "example.pdf",
  "type": "pdf",
  "pageCount": 3,
  "chunkCount": 12,
  "chunks": [
    {
      "text": "chunk text",
      "page": 1,
      "source": "example.pdf",
      "index": 0
    }
  ]
}
```

### `POST /api/chat`

Send JSON:

```json
{
  "question": "What are the renewal terms?",
  "chunks": [
    {
      "text": "chunk text",
      "page": 1,
      "source": "example.pdf",
      "index": 0
    }
  ],
  "modelId": "openai:gpt-4o-mini",
  "mode": "rag",
  "apiKey": "optional-request-scoped-key"
}
```

`mode` can be `rag` or `agentic`; omitted values default to `rag`. The route
retrieves the top 8 chunks with TF-IDF search, formats them as `[Page N]`
excerpts, then streams the LLM response. If no content is available, it returns
a JSON error instead of a stream.

## Document Processing Notes

- Chunks are word-based: 300 words per chunk with a 50-word overlap.
- PDF parsing uses `pdf-parse`; `next.config.ts` keeps it in
  `serverExternalPackages`.
- DOCX/DOC files are parsed with `mammoth.extractRawText`. The reported
  `pageCount` is based on paragraph groups, not physical pages.
- XLSX/XLS/CSV files are read with `xlsx` and converted to CSV text per sheet.
  The reported `page` value maps to the sheet number.
- PPTX/PPT extraction is best-effort. If text cannot be extracted, the app
  returns a single fallback chunk that asks the user to convert the presentation
  to PDF first.
- TXT and Markdown files are treated as UTF-8 text with `pageCount: 1`.

## Troubleshooting

- **No response stream**: `/api/chat` only streams successful LLM responses. JSON
  responses indicate validation, retrieval, or provider errors.
- **`Unknown provider` errors**: Ensure `modelId` uses the
  `provider:model-name` format and matches an entry in `src/lib/models.ts`.
- **Missing citations**: The prompt asks the model to cite `[Page N]`, but page
  numbers are parser-specific for non-PDF files as described above.
- **PPT content is missing**: Convert the presentation to PDF and upload the PDF.
- **Docker or Python commands fail**: The supported app is the Next.js project
  run with `npm`. `Dockerfile`, `docker-compose.yaml`, and `requirements.txt`
  are historical Python/Gradio artifacts and are not the current run path.

## Deploy to Vercel

[![Deploy with Vercel](https://vercel.com/button)](https://vercel.com/new/clone?repository-url=https://github.com/akhulk1987-dev/pdfGPT)

Set your API keys as environment variables in the Vercel dashboard. The app does
not require a database, vector store, or background worker.

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

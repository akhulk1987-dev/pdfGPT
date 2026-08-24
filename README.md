# pdfGPT - Smart Document Analysis

Chat with your documents using AI. Upload a PDF, Word, Excel, or PowerPoint file and ask questions about it.

**Live Demo**: [pdfgpt.vercel.app](https://pdfgpt.vercel.app) *(deploy your own instance - see below)*

## Features

- **Multi-format support**: PDF, DOCX, XLSX, PPTX, CSV, TXT, Markdown
- **RAG Mode**: Fast retrieval-augmented generation for quick answers
- **Agentic Mode**: Deep multi-step analysis with structured reasoning (Analyze, Extract, Synthesize, Evaluate, Respond)
- **Multiple LLM providers**: OpenAI, Anthropic, Google Gemini, Groq, and more
- **Free model options**: Google Gemini and Groq offer free API tiers
- **Page citations**: Every answer references source pages from the document
- **Streaming responses**: Real-time token streaming for instant feedback
- **Modern UI**: Dark-themed, responsive interface built with Next.js and Tailwind CSS
- **Vercel-ready**: One-click deployment to Vercel

## Quick Start

```bash
# Clone and install
git clone https://github.com/bhaskatripathi/pdfGPT.git
cd pdfGPT
npm install

# Set up at least one API key
echo "OPENAI_API_KEY=your-key-here" > .env.local

# Run
npm run dev
```

Open [http://localhost:3000](http://localhost:3000).

## Supported Models

| Provider | Models | Cost | API Key |
|----------|--------|------|---------|
| **Google** | Gemini 2.0 Flash, Gemini 1.5 Flash | Free tier | [Get free key](https://aistudio.google.com) |
| **Groq** | Llama 3.1 8B/70B, Mixtral 8x7B | Free tier | [Get free key](https://console.groq.com) |
| **OpenAI** | GPT-4o, GPT-4o Mini | Paid | [Get key](https://platform.openai.com) |
| **Anthropic** | Claude Sonnet 4, Claude 3.5 Haiku | Paid | [Get key](https://console.anthropic.com) |

## Deploy to Vercel

[![Deploy with Vercel](https://vercel.com/button)](https://vercel.com/new/clone?repository-url=https://github.com/bhaskatripathi/pdfGPT)

Set your API keys as environment variables in the Vercel dashboard.

## Architecture

```
Next.js 15 (App Router)
  |
  +-- /api/upload   -> Parse PDF/DOCX/XLSX/PPTX into text chunks
  +-- /api/chat     -> TF-IDF search + LLM streaming (Vercel AI SDK)
  |
  +-- Client        -> React UI, stores document chunks in state
```

### Document workflow

1. The browser uploads one file to `POST /api/upload` as multipart form data with a
   `file` field.
2. `src/lib/document-processor.ts` parses the file and splits extracted text into
   overlapping chunks of about 300 words with 50-word overlap.
3. The upload response returns all chunks to the browser. The app keeps them in React
   state; there is no server-side document store, vector database, or embedding API.
4. Each question sends the current chunks to `POST /api/chat`. The server runs TF-IDF
   search over those chunks, selects the top 8 matches, builds a prompt, and streams
   the LLM response back to the UI.

`DocumentChunk` objects have this shape:

```ts
{
  text: string;
  page: number;   // PDF page, DOCX paragraph number, sheet number, or 1 for text files
  source: string; // original filename
  index: number;  // chunk index within the parsed document
}
```

### API contracts

#### `POST /api/upload`

Request:

```bash
curl -F "file=@example.pdf" http://localhost:3000/api/upload
```

Response:

```json
{
  "name": "example.pdf",
  "type": "pdf",
  "pageCount": 12,
  "chunkCount": 34,
  "chunks": [
    {
      "text": "Extracted document text...",
      "page": 1,
      "source": "example.pdf",
      "index": 0
    }
  ]
}
```

Supported extensions are `pdf`, `doc`, `docx`, `xls`, `xlsx`, `csv`, `ppt`, `pptx`,
`txt`, and `md`. Unsupported extensions return an error. PPT/PPTX parsing is best
effort; if text cannot be extracted, the parser returns a fallback chunk recommending
PDF conversion.

#### `POST /api/chat`

Request:

```json
{
  "question": "What are the main findings?",
  "chunks": [
    {
      "text": "Extracted document text...",
      "page": 1,
      "source": "example.pdf",
      "index": 0
    }
  ],
  "modelId": "openai:gpt-4o-mini",
  "mode": "rag",
  "apiKey": "optional-provider-key"
}
```

- `mode` can be `rag` or `agentic`; omitted values default to `rag`.
- `modelId` must use the `provider:model-name` IDs from `src/lib/models.ts`.
- `apiKey` is optional when the matching provider environment variable is set on the
  server. The UI sends a typed key only for the current request.
- Responses use the Vercel AI SDK data stream format. JSON responses indicate request,
  retrieval, provider, or server errors.

### Model configuration

Models are registered in `src/lib/models.ts`. To add or remove a model, update
`MODEL_LIST` and ensure `src/app/api/chat/route.ts` can resolve the provider prefix:

| Prefix | Provider | Environment variable |
|--------|----------|----------------------|
| `google` | Google Gemini | `GOOGLE_GENERATIVE_AI_API_KEY` |
| `groq` | Groq OpenAI-compatible API | `GROQ_API_KEY` |
| `openai` | OpenAI | `OPENAI_API_KEY` |
| `anthropic` | Anthropic | `ANTHROPIC_API_KEY` |

### Troubleshooting

- **`No file provided`**: submit multipart form data with a `file` field.
- **`Unsupported file type`**: convert the document to one of the supported extensions
  listed above.
- **`No relevant content found`**: the chat request did not include searchable chunks.
  Re-upload the document; for presentations, try converting to PDF.
- **Provider authentication errors**: set the provider key in `.env.local` or enter it
  in the UI before asking a question.
- **`pdf-parse` build/runtime issues**: keep `pdf-parse` listed in
  `serverExternalPackages` in `next.config.ts`.

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

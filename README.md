# pdfGPT - Smart Document Analysis

Chat with your documents using AI. Upload a PDF, Word, Excel, PowerPoint, text,
or Markdown file and ask questions about it.

**Live Demo**: [pdfgpt.vercel.app](https://pdfgpt.vercel.app) *(deploy your own instance - see below)*

## Features

- **Multi-format support**: PDF, DOCX, XLSX, CSV, PPTX, TXT, and Markdown
- **RAG Mode**: Fast retrieval-augmented generation for quick answers
- **Agentic Mode**: Deeper prompt workflow for analyze, extract, synthesize, evaluate, and respond
- **Multiple LLM providers**: OpenAI, Anthropic, Google Gemini, and Groq
- **Free model options**: Google Gemini and Groq offer free API tiers
- **Page citations**: Answers are instructed to cite source excerpts with `[Page N]`
- **Streaming responses**: Real-time token streaming through the Vercel AI SDK
- **Modern UI**: Dark-themed, responsive interface built with Next.js and Tailwind CSS
- **Vercel-ready**: Deploy as a standard Next.js application

## Documentation

- [API reference](docs/API.md) - request and response shapes for `/api/upload`
  and `/api/chat`, model IDs, streaming behavior, and parser constraints.
- [Architecture and developer runbook](docs/ARCHITECTURE.md) - upload to chat
  data flow, retrieval details, setup, troubleshooting, and common extension
  points.

## Quick start

```bash
# Clone and install
git clone https://github.com/akhulk1987-dev/pdfGPT.git
cd pdfGPT
npm install

# Set up at least one provider key
echo "OPENAI_API_KEY=your-key-here" > .env.local

# Run the Next.js dev server
npm run dev
```

Open [http://localhost:3000](http://localhost:3000).

The app can also use a client-supplied API key entered in the UI for a single
request. Server environment variables remain the recommended deployment path.

## Supported models

Model IDs use `provider:model-name` format and are defined in
`src/lib/models.ts`.

| Provider | Models | Cost | Environment variable |
|----------|--------|------|----------------------|
| **Google** | Gemini 2.0 Flash, Gemini 1.5 Flash | Free tier | `GOOGLE_GENERATIVE_AI_API_KEY` |
| **Groq** | Llama 3.1 8B/70B, Mixtral 8x7B | Free tier | `GROQ_API_KEY` |
| **OpenAI** | GPT-4o, GPT-4o Mini | Paid | `OPENAI_API_KEY` |
| **Anthropic** | Claude Sonnet 4, Claude 3.5 Haiku | Paid | `ANTHROPIC_API_KEY` |

## Supported documents

| Type | Extensions | Notes |
|------|------------|-------|
| PDF | `.pdf` | Parsed with `pdf-parse`; requires `serverExternalPackages: ["pdf-parse"]` in `next.config.ts`. |
| Word | `.docx`, `.doc` | DOCX is the intended path; older binary `.doc` files may fail. |
| Spreadsheet | `.xlsx`, `.xls`, `.csv` | Sheets are converted to CSV text before chunking. |
| Presentation | `.pptx`, `.ppt` | Best-effort only; convert to PDF when extraction is weak. |
| Text | `.txt`, `.md` | Parsed as UTF-8 text. |

Large uploads can create large chat requests because the browser sends all
document chunks to `/api/chat` on every question.

## Deploy to Vercel

[![Deploy with Vercel](https://vercel.com/button)](https://vercel.com/new/clone?repository-url=https://github.com/akhulk1987-dev/pdfGPT)

Set your API keys as environment variables in the Vercel dashboard.

## Architecture

```text
Next.js 15 (App Router)
  |
  +-- /api/upload   -> Parse files into text chunks
  +-- /api/chat     -> TF-IDF search + LLM streaming (Vercel AI SDK)
  |
  +-- Client        -> React UI, stores document chunks and messages in state
```

The current app does not use a vector database or server-side document storage.
Legacy Python/Gradio files (`Dockerfile`, `docker-compose.yaml`, and
`requirements.txt`) are not the supported path for running the Next.js app.

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

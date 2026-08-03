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

## Developer Docs

- [API contracts](docs/API.md) - request/response shapes for upload and chat, model IDs, streaming behavior, and parser constraints.
- [Architecture and runbook](docs/ARCHITECTURE.md) - document-processing flow, retrieval behavior, extension points, and troubleshooting.

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

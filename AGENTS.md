# AGENTS.md

## Cursor Cloud specific instructions

### Project Overview

pdfGPT is a modern Next.js web application for document analysis using AI.
Supports PDF, Word, Excel, PowerPoint, and text files.
Offers both RAG (fast retrieval) and Agentic (deep analysis) modes.

### Tech Stack

- **Frontend**: Next.js 15, React 19, TypeScript, Tailwind CSS 4
- **LLM**: Vercel AI SDK with multi-provider support (OpenAI, Anthropic, Google, Groq)
- **Search**: TF-IDF based document search (runs entirely server-side, no external API)
- **File parsing**: pdf-parse, mammoth (DOCX), xlsx (Excel)

### Running Locally

```bash
npm install
npm run dev      # Dev server on :3000
npm run build    # Production build
npm run lint     # ESLint
```

### Environment Variables

Set in `.env.local` or as Cursor Cloud secrets:
- `OPENAI_API_KEY` - For OpenAI models (GPT-4o, GPT-4o Mini)
- `GOOGLE_GENERATIVE_AI_API_KEY` - For Google Gemini models (free tier)
- `GROQ_API_KEY` - For Groq models (Llama, Mixtral - free tier)
- `ANTHROPIC_API_KEY` - For Anthropic Claude models

### Project Structure

```
src/app/page.tsx              - Main UI (React client component)
src/app/layout.tsx            - Root layout
src/app/api/chat/route.ts     - Chat API (streaming LLM responses)
src/app/api/upload/route.ts   - File upload and parsing API
src/lib/models.ts             - LLM model registry
src/lib/document-processor.ts - Multi-format file parsing
src/lib/search.ts             - TF-IDF document search
Dockerfile, docker-compose.yaml, requirements.txt - Historical Python/Gradio artifacts, not the primary Next.js run path
```

### Deployment

Deploys to Vercel via `vercel deploy` or GitHub integration. No special config needed.

### Key Notes

- The app uses Vercel AI SDK's `streamText` for streaming responses.
- Document chunks are stored in client state (React) and sent with each chat request.
- TF-IDF search runs server-side in the API route; no vector DB or embedding API needed.
- The `pdf-parse` package requires `serverExternalPackages` in next.config.ts.

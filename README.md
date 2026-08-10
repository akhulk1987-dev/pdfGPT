# pdfGPT - Smart Document Analysis

Chat with your documents using AI. Upload a document, choose an LLM provider, and ask questions against retrieved document excerpts.

**Live Demo**: [pdfgpt.vercel.app](https://pdfgpt.vercel.app) *(deploy your own instance - see below)*

## Features

- **Multi-format support**: PDF, DOCX, XLSX, PPTX, CSV, TXT, Markdown
- **RAG Mode**: Fast retrieval-augmented generation for quick answers
- **Agentic Mode**: Deep multi-step analysis with structured reasoning (Analyze, Extract, Synthesize, Evaluate, Respond)
- **Multiple LLM providers**: OpenAI, Anthropic, Google Gemini, and Groq
- **Free model options**: Google Gemini and Groq offer free API tiers
- **Page citations**: Every answer references source pages from the document
- **Streaming responses**: Real-time token streaming for instant feedback
- **Modern UI**: Dark-themed, responsive interface built with Next.js and Tailwind CSS
- **Vercel-ready**: One-click deployment to Vercel

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

Set provider keys in `.env.local`, Vercel environment variables, or enter a key in the UI for the current browser session.

| Variable | Used for |
|----------|----------|
| `OPENAI_API_KEY` | OpenAI models such as GPT-4o and GPT-4o Mini |
| `GOOGLE_GENERATIVE_AI_API_KEY` | Google Gemini models |
| `GROQ_API_KEY` | Groq-hosted OpenAI-compatible models |
| `ANTHROPIC_API_KEY` | Anthropic Claude models |

When the UI sends an `apiKey` with a chat request, that request key takes precedence over the matching environment variable. The browser stores uploaded chunks and the typed API key in React state only; there is no database or server-side document store in this app.

## Supported Models

| Provider | Models | Cost | API Key |
|----------|--------|------|---------|
| **Google** | Gemini 2.0 Flash, Gemini 1.5 Flash | Free tier | [Get free key](https://aistudio.google.com) |
| **Groq** | Llama 3.1 8B/70B, Mixtral 8x7B | Free tier | [Get free key](https://console.groq.com) |
| **OpenAI** | GPT-4o, GPT-4o Mini | Paid | [Get key](https://platform.openai.com) |
| **Anthropic** | Claude Sonnet 4, Claude 3.5 Haiku | Paid | [Get key](https://console.anthropic.com) |

## Deploy to Vercel

[![Deploy with Vercel](https://vercel.com/button)](https://vercel.com/new/clone?repository-url=https://github.com/akhulk1987-dev/pdfGPT)

Set your API keys as environment variables in the Vercel dashboard.

## Architecture

```
Next.js 15 (App Router)
  |
  +-- src/app/page.tsx          -> Client UI, document state, streaming chat renderer
  +-- src/app/api/upload/route  -> Multipart upload, file parsing, chunk response
  +-- src/app/api/chat/route    -> TF-IDF retrieval, prompt construction, LLM stream
  |
  +-- src/lib/document-processor.ts -> Format-specific text extraction and chunking
  +-- src/lib/search.ts             -> In-process TF-IDF search
  +-- src/lib/models.ts             -> Model registry and provider metadata
```

### Request flow

1. The user uploads one file from the sidebar.
2. `POST /api/upload` parses the file into `DocumentChunk[]` and returns the chunks to the client.
3. The client keeps those chunks in component state and clears any previous chat messages.
4. For every question, the client posts the question, chunks, selected model ID, optional API key, and analysis mode to `POST /api/chat`.
5. The chat API retrieves the top 8 chunks with TF-IDF, formats them as `[Page N] ...` excerpts, builds either the RAG or Agentic prompt, and streams the response with the Vercel AI SDK.

No vector database, embedding service, queue, or persistent file storage is involved.

## API contracts

### `POST /api/upload`

Request: `multipart/form-data` with a single `file` field.

Successful response:

```json
{
  "name": "report.pdf",
  "type": "pdf",
  "pageCount": 12,
  "chunkCount": 34,
  "chunks": [
    {
      "text": "Extracted document text...",
      "page": 1,
      "source": "report.pdf",
      "index": 0
    }
  ]
}
```

Errors:

- `400` with `{ "error": "No file provided" }` when the multipart field is missing.
- `500` with the parser error message for unsupported extensions or extraction failures.

### `POST /api/chat`

Request body:

```json
{
  "question": "What are the renewal terms?",
  "chunks": [],
  "modelId": "openai:gpt-4o-mini",
  "apiKey": "optional request-scoped provider key",
  "mode": "rag"
}
```

Fields:

- `question` is required.
- `chunks` should be the exact array returned by `/api/upload`.
- `modelId` uses the `provider:model-name` format from `src/lib/models.ts`.
- `apiKey` is optional and overrides the server environment variable for that request.
- `mode` is `rag` by default; use `agentic` for the multi-step analysis prompt.

Responses:

- A Vercel AI SDK data stream for successful answers.
- `400` with `{ "error": "No question" }` when `question` is empty.
- `200` with `{ "error": "No relevant content found. Try a different question." }` when there is no context to send to the model.
- `500` with `{ "error": "..." }` for provider, model, or streaming errors.

## Document processing and retrieval

All parsing runs server-side in `src/lib/document-processor.ts`.

| Extension | Extraction behavior | Page/source numbering |
|-----------|---------------------|-----------------------|
| `.pdf` | `pdf-parse` extracts text, split on form-feed page separators when present. | Uses PDF page count from `pdf-parse`; chunks are assigned page indexes from split text. |
| `.doc`, `.docx` | `mammoth.extractRawText` extracts raw paragraphs. | Paragraph groups are treated as page-like sections. |
| `.xls`, `.xlsx`, `.csv` | `xlsx` converts each sheet to CSV text. | Each sheet is treated as one page-like section. |
| `.ppt`, `.pptx` | Best-effort extraction via the current `xlsx`-based fallback. | If no text can be extracted, the API returns one fallback chunk asking the user to convert the deck to PDF first. |
| `.txt`, `.md` | UTF-8 buffer text. | The whole file is treated as page 1. |

Chunking constraints:

- Chunks are 300 words with a 50-word overlap.
- Trailing chunks shorter than 15 words are skipped unless they are the first chunk for that section.
- Chunk objects contain `{ text, page, source, index }` and are safe to send back to `/api/chat`.

Retrieval constraints:

- `src/lib/search.ts` tokenizes lowercase ASCII letters and numbers, ignores tokens of 2 characters or fewer, and computes TF-IDF vectors in memory.
- The query is included in the TF-IDF corpus, then compared with document chunks by cosine similarity.
- `/api/chat` requests the top 8 chunks.
- If every score is zero but chunks exist, search falls back to the first chunks so the model still receives document context.

## Adding or changing models

Model choices live in `src/lib/models.ts`. To add a model:

1. Add a `ModelOption` with an ID in `provider:model-name` format.
2. Ensure `provider` is one of the providers handled in `getProvider` in `src/app/api/chat/route.ts` (`google`, `groq`, `openai`, or `anthropic`).
3. Set the correct environment variable name in `keyEnvVar`.
4. Verify the model name matches the provider API exactly.

Unknown provider prefixes throw `Unknown provider: ...` from the chat API.

## Developer runbook

Common commands:

```bash
npm install
npm run dev
npm run build
npm run lint
```

Operational notes:

- `next.config.ts` keeps `pdf-parse` in `serverExternalPackages`; keep that setting when changing PDF parsing.
- The current production path is the Next.js app. `Dockerfile`, `docker-compose.yaml`, and `requirements.txt` are legacy Python/Gradio artifacts and are not used by `npm run dev`, `npm run build`, or Vercel deployment.
- Provider authentication errors usually mean the selected model's API key is missing, invalid, or entered for the wrong provider.
- Empty answers after upload often indicate extraction produced little useful text. For PowerPoint files, convert to PDF if the fallback chunk appears.
- Because chunks are kept in browser state, refreshing the page or clearing the selected document removes the active chat context.

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

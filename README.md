# pdfGPT - Smart Document Analysis

Chat with documents using AI. Upload a PDF, Word, Excel, PowerPoint, text, or
Markdown file, then ask questions using a fast RAG prompt or a deeper agentic
analysis prompt.

**Live Demo**: [pdfgpt.vercel.app](https://pdfgpt.vercel.app) *(deploy your own
instance - see below)*

## Features

- **Multi-format support**: PDF, DOC/DOCX, XLS/XLSX, CSV, PPT/PPTX, TXT, Markdown
- **RAG mode**: Fast retrieval-augmented generation for quick answers
- **Agentic mode**: Multi-step analysis prompt with Analyze, Extract, Synthesize,
  Evaluate, and Respond phases
- **Multiple LLM providers**: Google Gemini, Groq, OpenAI, and Anthropic via the
  Vercel AI SDK
- **Source citations**: Retrieved chunks are passed to the model with `[Page N]`
  labels and the prompts require page citations
- **No vector database**: Retrieval uses an in-process TF-IDF search over chunks
  sent with each chat request
- **Vercel-ready**: Next.js App Router app with server-side parsing APIs

## Quick start

```bash
git clone https://github.com/akhulk1987-dev/pdfGPT.git
cd pdfGPT
npm install

# Add at least one provider key.
cat > .env.local <<'EOF'
OPENAI_API_KEY=your-openai-key
# GOOGLE_GENERATIVE_AI_API_KEY=your-google-key
# GROQ_API_KEY=your-groq-key
# ANTHROPIC_API_KEY=your-anthropic-key
EOF

npm run dev
```

Open [http://localhost:3000](http://localhost:3000).

### Scripts

| Command | Purpose |
| --- | --- |
| `npm run dev` | Start the local Next.js dev server. |
| `npm run build` | Build the production app. |
| `npm run start` | Serve a production build. |
| `npm run lint` | Run the configured Next.js lint script. |

## Configuration

Provider keys can be supplied in two ways:

1. Server environment variables in `.env.local` or the hosting platform.
2. The API key field in the UI, which is sent with the chat request and takes
   precedence for that request.

| Provider | Model IDs | Environment variable |
| --- | --- | --- |
| Google | `google:gemini-2.0-flash`, `google:gemini-1.5-flash` | `GOOGLE_GENERATIVE_AI_API_KEY` |
| Groq | `groq:llama-3.1-8b-instant`, `groq:llama-3.3-70b-versatile`, `groq:mixtral-8x7b-32768` | `GROQ_API_KEY` |
| OpenAI | `openai:gpt-4o`, `openai:gpt-4o-mini` | `OPENAI_API_KEY` |
| Anthropic | `anthropic:claude-sonnet-4-20250514`, `anthropic:claude-3-5-haiku-20241022` | `ANTHROPIC_API_KEY` |

Model options are defined in [`src/lib/models.ts`](src/lib/models.ts). The chat
API expects model IDs in `provider:model-name` format; unknown providers throw an
error.

## Architecture

```text
Next.js 15 App Router
  |
  +-- src/app/page.tsx
  |     React client UI, file upload, model selection, mode selection,
  |     client-side document chunk state, streaming chat rendering
  |
  +-- src/app/api/upload/route.ts
  |     Multipart upload endpoint that parses one file into DocumentChunk[]
  |
  +-- src/app/api/chat/route.ts
  |     TF-IDF chunk search, prompt construction, provider selection,
  |     Vercel AI SDK streaming response
  |
  +-- src/lib/document-processor.ts
  |     File-type parsers and word chunking
  |
  +-- src/lib/search.ts
        In-process tokenization, TF-IDF, cosine similarity, fallback retrieval
```

Documents are not written to a database or object store. The upload response
returns chunks to the browser, and the browser includes those chunks in each
`/api/chat` request.

## Document processing workflow

1. The browser sends a single selected file to `POST /api/upload` as multipart
   form data with the field name `file`.
2. The API route reads the file into a `Buffer` and calls `parseDocument`.
3. The parser chooses an implementation from the file extension.
4. Extracted text is split into word chunks of about 300 words with a 50-word
   overlap. Very short trailing chunks are skipped after the first chunk.
5. The response returns document metadata and `DocumentChunk[]`; the client keeps
   those chunks in React state until the document is cleared or replaced.

### Upload response shape

```json
{
  "name": "report.pdf",
  "type": "pdf",
  "pageCount": 12,
  "chunkCount": 24,
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

### Parser behavior and constraints

| Extension | Parser path | Page/source labels |
| --- | --- | --- |
| `.pdf` | `pdf-parse`; text is split on form-feed page separators when present | PDF page numbers from parsed text, with total page count from `data.numpages` |
| `.doc`, `.docx` | `mammoth.extractRawText` | Paragraph blocks are treated as page-like units |
| `.xls`, `.xlsx`, `.csv` | `xlsx`; each sheet is converted to CSV text | Sheet index is used as the page number |
| `.ppt`, `.pptx` | Best-effort spreadsheet-like extraction fallback | May return a single fallback chunk asking the user to convert to PDF |
| `.txt`, `.md` | UTF-8 text from the file buffer | Single page |

Operational notes:

- `pdf-parse` is listed in `serverExternalPackages` in
  [`next.config.ts`](next.config.ts); keep this when changing the Next.js config.
- Legacy binary `.doc` and presentation files can fail or extract little text.
  Convert those files to PDF or DOCX when reliable extraction matters.
- There is no explicit upload size limit in the app code. Hosting platform limits
  and memory usage are the practical constraints.

## Chat and retrieval workflow

The client sends `POST /api/chat` with JSON:

```json
{
  "question": "What are the renewal terms?",
  "chunks": [
    {
      "text": "Extracted document text...",
      "page": 3,
      "source": "contract.pdf",
      "index": 7
    }
  ],
  "modelId": "openai:gpt-4o-mini",
  "apiKey": "optional-request-scoped-provider-key",
  "mode": "rag"
}
```

Server behavior:

- `question` is required; missing questions return `400` with `{ "error": "No question" }`.
- `chunks` defaults to an empty array if omitted. If retrieval finds no context,
  the route returns a JSON error message with status `200`.
- TF-IDF retrieval scores chunks with lowercase alphanumeric tokens longer than
  two characters. Punctuation is ignored.
- `/api/chat` retrieves the top 8 chunks. If every score is zero, it falls back
  to the first chunks so the model can still answer from available content.
- RAG mode builds a concise citation-focused prompt. Agentic mode builds a
  structured multi-step prompt. Both prompts instruct the model to use only the
  supplied excerpts.
- Responses stream through `streamText(...).toDataStreamResponse()` with
  `temperature: 0.3` and `maxTokens: 2048`.

## Adding or changing models

1. Add or update the option in `MODEL_LIST` in [`src/lib/models.ts`](src/lib/models.ts).
2. Ensure the provider prefix is handled in `getProvider` in
   [`src/app/api/chat/route.ts`](src/app/api/chat/route.ts).
3. Add the required environment variable to this README and to the deployment
   environment.
4. Test both a server-side env key and the request-scoped UI key path.

## Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `Unknown provider: ...` | `modelId` has a provider prefix not handled by `/api/chat` | Add a provider case or use an existing model ID. |
| Chat returns `No relevant content found` | No chunks were sent or parsing produced no searchable text | Re-upload the document and check the upload response chunk count. |
| Answers cite unexpected pages for DOCX/XLSX | Non-PDF page labels are derived from paragraphs or sheets, not real pages | Convert to PDF when exact page citations are required. |
| PPT/PPTX uploads produce a conversion message | Presentation extraction is best-effort | Convert the presentation to PDF before uploading. |
| PDF parsing fails during build or runtime | `pdf-parse` must stay server-side | Keep `serverExternalPackages: ["pdf-parse"]` in `next.config.ts`. |
| Provider authentication fails | Missing env var or wrong request-scoped API key | Set the provider key in `.env.local` or paste a valid key in the UI. |

## Deploy to Vercel

[![Deploy with Vercel](https://vercel.com/button)](https://vercel.com/new/clone?repository-url=https://github.com/akhulk1987-dev/pdfGPT)

Set provider API keys as environment variables in the Vercel dashboard. The app
does not require a database, vector store, background worker, or embedding API.

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

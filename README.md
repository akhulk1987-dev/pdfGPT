# pdfGPT - Smart Document Analysis

Chat with your documents using AI. Upload a PDF, Word, Excel, PowerPoint,
plain text, or Markdown file and ask questions about its contents.

## Features

- **Multi-format upload**: PDF, DOC/DOCX, XLS/XLSX, CSV, PPT/PPTX, TXT, and Markdown.
- **RAG mode**: fast retrieval-augmented answers from the most relevant chunks.
- **Agentic mode**: the same retrieval step with a deeper analysis prompt that asks the
  model to analyze, extract, synthesize, evaluate, and respond.
- **Multiple LLM providers**: Google Gemini, Groq, OpenAI, and Anthropic through the
  Vercel AI SDK.
- **Streaming responses**: chat answers are streamed back to the browser.
- **No vector database**: TF-IDF search runs server-side for each chat request.
- **Vercel-ready**: deploy as a standard Next.js app.

## Quick start

```bash
git clone https://github.com/akhulk1987-dev/pdfGPT.git
cd pdfGPT
npm install

# Configure at least one provider key for the model you plan to use.
echo "OPENAI_API_KEY=your-key-here" > .env.local

npm run dev
```

Open [http://localhost:3000](http://localhost:3000).

### Environment variables

The chat route can use API keys from environment variables or a per-request key
entered in the UI. Prefer environment variables for shared deployments.

| Provider | Environment variable |
| --- | --- |
| Google | `GOOGLE_GENERATIVE_AI_API_KEY` |
| Groq | `GROQ_API_KEY` |
| OpenAI | `OPENAI_API_KEY` |
| Anthropic | `ANTHROPIC_API_KEY` |

Client-entered keys are kept in React state and sent only with the `/api/chat`
request body. They are not persisted by the browser code.

## Supported models

Model options are defined in `src/lib/models.ts` and use `provider:model`
identifiers in API requests.

| Provider | Model IDs |
| --- | --- |
| Google | `google:gemini-2.0-flash`, `google:gemini-1.5-flash` |
| Groq | `groq:llama-3.1-8b-instant`, `groq:llama-3.3-70b-versatile`, `groq:mixtral-8x7b-32768` |
| OpenAI | `openai:gpt-4o`, `openai:gpt-4o-mini` |
| Anthropic | `anthropic:claude-sonnet-4-20250514`, `anthropic:claude-3-5-haiku-20241022` |

## Architecture

```
Next.js 15 App Router
  |
  +-- src/app/page.tsx             React chat UI and document state
  +-- src/app/api/upload/route.ts  File upload endpoint
  +-- src/app/api/chat/route.ts    Retrieval, prompt construction, streaming LLM response
  |
  +-- src/lib/document-processor.ts  Format parsers and chunking
  +-- src/lib/search.ts              Request-time TF-IDF search
  +-- src/lib/models.ts              Supported model registry
```

The app is intentionally stateless on the server after upload. Parsed
`DocumentChunk` objects are stored in the client component state and posted back
with every chat request. There is no database, object storage, embedding index, or
background job queue in the current implementation.

## Document ingestion workflow

1. The user chooses a file in `src/app/page.tsx`.
2. The browser posts `multipart/form-data` to `POST /api/upload` with a `file`
   field.
3. `src/lib/document-processor.ts` parses the file into chunks of about 300 words
   with 50 words of overlap.
4. The upload route returns document metadata and all chunks to the browser.
5. The browser keeps those chunks in state until the user uploads a new document,
   clears the current document, or reloads the page.

`DocumentChunk` shape:

```ts
interface DocumentChunk {
  text: string;
  page: number;
  source: string;
  index: number;
}
```

Parser notes:

- PDF parsing uses `pdf-parse`; `next.config.ts` lists `pdf-parse` in
  `serverExternalPackages`.
- DOC/DOCX parsing uses Mammoth raw text extraction. The `page` value represents
  paragraph position, not a physical document page.
- XLS/XLSX/CSV parsing uses `xlsx` and treats each sheet as a page.
- PPT/PPTX files are accepted, but text extraction is limited. If extraction fails,
  the app returns a fallback chunk asking the user to convert the presentation to
  PDF first.
- TXT and Markdown files are treated as one-page UTF-8 text documents.

## Chat and retrieval workflow

`POST /api/chat` expects JSON:

```json
{
  "question": "What are the main risks?",
  "chunks": [
    { "text": "Document text...", "page": 1, "source": "report.pdf", "index": 0 }
  ],
  "modelId": "openai:gpt-4o-mini",
  "apiKey": "optional-request-scoped-provider-key",
  "mode": "rag"
}
```

Runtime behavior:

- `mode` may be `rag` or `agentic`; it changes only the prompt template.
- The chat route searches the provided chunks with TF-IDF and sends the top 8
  chunks to the selected model.
- If every TF-IDF score is zero, search falls back to the first chunks so the model
  can still attempt an answer.
- Prompts instruct the model to cite source locations with `[Page N]`.
- Responses stream through `streamText(...).toDataStreamResponse()` from the
  Vercel AI SDK.
- If no relevant context is available, the route returns a JSON error instead of a
  stream.

## Development commands

```bash
npm run dev    # local dev server on port 3000
npm run build  # production build
npm run lint   # configured lint command
```

## Common pitfalls

- Uploading a second document replaces the current chunks and clears chat history.
- Refreshing the page loses uploaded chunks because there is no persisted document
  store.
- Source citations are only as accurate as parser page metadata. For DOC/DOCX,
  spreadsheet, and presentation files, page numbers are logical parser positions.
- Large documents increase request body size because all selected document chunks
  are sent from the browser to `/api/chat`.
- The app does not call embedding APIs and does not require a vector database.

## Deploy to Vercel

[![Deploy with Vercel](https://vercel.com/button)](https://vercel.com/new/clone?repository-url=https://github.com/akhulk1987-dev/pdfGPT)

Set the provider API keys you need as environment variables in the Vercel
dashboard.

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

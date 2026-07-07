# API reference

pdfGPT exposes two application routes under the Next.js App Router. The browser
client in `src/app/page.tsx` calls both routes directly; there is no separate
backend service or persisted document session.

## `POST /api/upload`

Parses one uploaded document and returns normalized text chunks for later chat
requests.

### Request

Send `multipart/form-data` with one field:

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `file` | File | Yes | The route uses the original filename extension to select a parser. |

Example:

```bash
curl -X POST http://localhost:3000/api/upload \
  -F "file=@/path/to/report.pdf"
```

### Success response

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

`chunks` are `DocumentChunk` objects from `src/lib/document-processor.ts`.
Chunks are about 300 words with 50 words of overlap. Short trailing chunks under
15 words are skipped after the first chunk for a page or section.

### Errors

| Status | Body | Cause |
| --- | --- | --- |
| `400` | `{ "error": "No file provided" }` | The multipart body did not include `file`. |
| `500` | `{ "error": "..." }` | Parsing failed or the extension is unsupported. |

### Supported extensions and constraints

| Extension | Parser path | Notes |
| --- | --- | --- |
| `.pdf` | `parsePDF` via `pdf-parse` | Page text is split on form-feed markers when available. |
| `.docx`, `.doc` | `parseDOCX` via `mammoth` | Intended for DOCX content; binary `.doc` files may fail. |
| `.xlsx`, `.xls`, `.csv` | `parseXLSX` via `xlsx` | Each sheet is converted to CSV text and treated like a page. |
| `.pptx`, `.ppt` | `parsePPTX` | Best-effort only; currently falls back to a single guidance chunk when text cannot be extracted. |
| `.txt`, `.md` | UTF-8 text | Treated as a one-page document. |

The upload route does not currently enforce a file-size limit. Large files create
large chunk arrays, and the client sends the full chunk array again with each
chat request.

## `POST /api/chat`

Searches the uploaded chunks for relevant context, builds a prompt, and streams
the LLM answer with the Vercel AI SDK.

### Request

Send JSON:

```json
{
  "question": "What are the renewal terms?",
  "chunks": [
    {
      "text": "Contract excerpt...",
      "page": 4,
      "source": "contract.pdf",
      "index": 12
    }
  ],
  "modelId": "google:gemini-2.0-flash",
  "apiKey": "optional-client-supplied-key",
  "mode": "rag"
}
```

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `question` | string | Yes | Empty values return `400`. |
| `chunks` | `DocumentChunk[]` | Yes | Usually the exact `chunks` array returned by `/api/upload`. |
| `modelId` | string | Yes | Format is `provider:model-name`, for example `openai:gpt-4o-mini`. |
| `apiKey` | string | No | Overrides the matching server environment variable for this request. |
| `mode` | `"rag"` or `"agentic"` | No | Defaults to `"rag"`. The retrieval pipeline is the same; the prompt template changes. |

### Model IDs and environment variables

Model IDs are defined in `src/lib/models.ts` and resolved in
`src/app/api/chat/route.ts`.

| Provider prefix | Example model ID | Environment variable |
| --- | --- | --- |
| `google` | `google:gemini-2.0-flash` | `GOOGLE_GENERATIVE_AI_API_KEY` |
| `groq` | `groq:llama-3.1-8b-instant` | `GROQ_API_KEY` |
| `openai` | `openai:gpt-4o-mini` | `OPENAI_API_KEY` |
| `anthropic` | `anthropic:claude-sonnet-4-20250514` | `ANTHROPIC_API_KEY` |

Groq uses the OpenAI-compatible client with `https://api.groq.com/openai/v1` as
its base URL. If `apiKey` is present in the request body it is used before the
server environment variable.

### Retrieval and prompting

`/api/chat` calls `searchChunks(question, chunks, 8)`, so each prompt receives up
to eight chunks. Search is local TF-IDF with cosine similarity in
`src/lib/search.ts`. If every score is zero, search falls back to the first
chunks with small synthetic scores so the model still receives document context.

The model call uses:

- `temperature: 0.3`
- `maxTokens: 2048`
- one user message containing either the RAG prompt or agentic prompt

RAG mode instructs the model to answer only from the excerpts with `[Page N]`
citations. Agentic mode asks the model to analyze, extract, synthesize, evaluate,
and respond with citations.

### Responses

Successful model responses use the Vercel AI SDK data stream returned by
`result.toDataStreamResponse()`. The current browser client manually reads stream
lines starting with `0:` and appends the decoded text to the assistant message.

JSON error responses:

| Status | Body | Cause |
| --- | --- | --- |
| `400` | `{ "error": "No question" }` | `question` is missing or empty. |
| `200` | `{ "error": "No relevant content found. Try a different question." }` | No context was produced, usually because no chunks were supplied. |
| `500` | `{ "error": "..." }` | Unknown provider, provider/client failure, or another chat error. |

The `200` JSON error is intentional current behavior, so clients should check
`content-type` before assuming a response is a stream.

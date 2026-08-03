# API contracts

pdfGPT exposes two server routes under the Next.js App Router. The browser UI calls
both routes directly; there is no persistent server-side document store.

## `POST /api/upload`

Parses one uploaded file into normalized document chunks.

### Request

`multipart/form-data` with a single field:

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `file` | `File` | Yes | Filename extension selects the parser. |

Example:

```bash
curl -X POST http://localhost:3000/api/upload \
  -F "file=@sample.pdf"
```

### Response

```json
{
  "name": "sample.pdf",
  "type": "pdf",
  "pageCount": 12,
  "chunkCount": 34,
  "chunks": [
    {
      "text": "Chunk text...",
      "page": 1,
      "source": "sample.pdf",
      "index": 0
    }
  ]
}
```

`chunks` are generated with a 300-word target size and 50-word overlap. Very short
trailing chunks are dropped after the first chunk for a source page or section.

### Supported extensions

| Extension | Parser path | Page/source unit |
| --- | --- | --- |
| `.pdf` | `pdf-parse` | PDF page text split on form-feed markers. |
| `.doc`, `.docx` | `mammoth.extractRawText` | Paragraph blocks are treated as page numbers. |
| `.xls`, `.xlsx`, `.csv` | `xlsx` | Workbook sheets are treated as page numbers. |
| `.ppt`, `.pptx` | Best-effort `xlsx` fallback | May return a fallback chunk recommending PDF conversion. |
| `.txt`, `.md` | UTF-8 buffer text | Single page. |

Unsupported extensions return HTTP 500 with:

```json
{ "error": "Unsupported file type: .ext" }
```

Missing files return HTTP 400:

```json
{ "error": "No file provided" }
```

## `POST /api/chat`

Searches uploaded chunks, builds a prompt, and streams an LLM response.

### Request

`application/json`:

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `question` | `string` | Yes | User question. Empty values return HTTP 400. |
| `chunks` | `DocumentChunk[]` | Yes | Usually the exact `chunks` returned by `/api/upload`. |
| `modelId` | `string` | Yes | Format is `provider:model-name`, for example `openai:gpt-4o-mini`. |
| `apiKey` | `string` | No | Overrides the provider environment variable for this request. |
| `mode` | `"rag"` or `"agentic"` | No | Defaults to `"rag"`. |

Example:

```bash
curl -X POST http://localhost:3000/api/chat \
  -H "Content-Type: application/json" \
  -d '{
    "question": "What are the renewal terms?",
    "chunks": [
      { "text": "The agreement renews annually...", "page": 3, "source": "contract.pdf", "index": 4 }
    ],
    "modelId": "openai:gpt-4o-mini",
    "mode": "rag"
  }'
```

### Retrieval and prompts

`/api/chat` calls `searchChunks(question, chunks, 8)` before invoking the model.
Search is TF-IDF with cosine similarity and runs in-process on the request payload.
If all chunks score zero, the search helper falls back to the first `topK` chunks
with small synthetic scores so the model still receives context.

The route formats retrieved chunks as:

```text
[Page 3] The agreement renews annually...
```

RAG mode asks the model to answer only from excerpts and cite `[Page N]`.
Agentic mode uses the same retrieved context but asks for explicit Analyze,
Extract, Synthesize, Evaluate, and Respond steps.

### Streaming response

Successful model calls return the Vercel AI SDK data stream produced by
`streamText(...).toDataStreamResponse()`. The current UI reads text deltas from
lines starting with `0:`.

JSON error responses can still be returned:

| Status | Body | Cause |
| --- | --- | --- |
| `400` | `{ "error": "No question" }` | Missing `question`. |
| `200` | `{ "error": "No relevant content found. Try a different question." }` | No searchable context was built. |
| `500` | `{ "error": "Unknown provider: ..." }` or provider error | Bad model provider, missing/invalid provider key, or upstream failure. |

## Model IDs and provider keys

Model options are registered in `src/lib/models.ts`. Runtime provider selection is
implemented in `src/app/api/chat/route.ts` by splitting `modelId` at the first
colon.

| Provider prefix | Environment fallback |
| --- | --- |
| `google` | `GOOGLE_GENERATIVE_AI_API_KEY` |
| `groq` | `GROQ_API_KEY` |
| `openai` | `OPENAI_API_KEY` |
| `anthropic` | `ANTHROPIC_API_KEY` |

When adding a model, update both `MODEL_LIST` for the UI and `getProvider` if the
provider prefix is new.

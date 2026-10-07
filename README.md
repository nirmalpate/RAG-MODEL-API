# PDF RAG API

A production-style **REST API** that answers questions about your PDF documents using **Retrieval-Augmented Generation (RAG)**. Built with **FastAPI**, with hybrid search, a reranker, API-key authentication, Slack integration and Docker support.

## Features

- **Hybrid retrieval**: BM25 keyword search + vector search (ChromaDB)
- **Reranking**: cross-encoder (`bge-reranker-base`) selects the best chunks
- **Source citations**: every answer includes file name and page number
- **Flexible LLM**: hosted Groq model (fast, no GPU) or a local Hugging Face model
- **API-key authentication** on the `/ask` endpoint
- **Input and output validation** with Pydantic models
- **Slack integration**: answers are posted to a channel (optional, non-blocking)
- **Logging, error handling and health check**
- **Docker-ready**

## Architecture

```
Client (curl / app / Slack)
        |  POST /ask  (x-api-key header, JSON body)
        v
+--------------------------------------------------+
| FastAPI  (app/main.py)                           |
|   1. Validate input        (Pydantic: Question)  |
|   2. Check API key         (dependency)          |
|   3. Call RAG chain                              |
|   4. Build response        (Pydantic: Answer)    |
|   5. Post to Slack         (background task)     |
+--------------------------------------------------+
        |
        v
+--------------------------------------------------+
| RAG layer  (app/rag.py)                          |
|   PDFs -> chunks -> embeddings -> Chroma + BM25  |
|   Question -> hybrid search -> reranker -> LLM   |
+--------------------------------------------------+
```

The RAG chain is built **once at startup** (FastAPI lifespan), so requests stay fast.

## Tech Stack

| Part | Tool |
|---|---|
| API framework | FastAPI + Uvicorn |
| Validation | Pydantic |
| RAG framework | LangChain |
| Vector database | ChromaDB |
| Keyword search | BM25 (`rank_bm25`) |
| Embeddings | `sentence-transformers/all-MiniLM-L6-v2` |
| Reranker | `BAAI/bge-reranker-base` |
| LLM | Groq (`llama-3.1-8b-instant`) or local `Qwen/Qwen2.5-1.5B-Instruct` |
| Integration | Slack Incoming Webhooks (`httpx`) |
| Packaging | Docker |

## Project Structure

```
rag-api/
├── app/
│   ├── __init__.py
│   ├── main.py          # API layer: endpoints, models, auth, Slack
│   └── rag.py           # AI layer: loading, retrieval, reranking, LLM
├── pdf_data/            # put your PDFs here
├── test_api.py          # quick manual tests
├── requirements.txt
├── Dockerfile
└── README.md
```

## Getting Started

### Prerequisites
- Python 3.12 (PyTorch support for newer versions can be unreliable)
- At least one PDF in `pdf_data/`
- A free [Groq API key](https://console.groq.com) (recommended if you have no GPU)

### Option 1: Run locally

```bash
git clone https://github.com/<your-username>/rag-api.git
cd rag-api

python -m venv venv
source venv/bin/activate          # Windows: .\venv\Scripts\activate
python -m pip install -r requirements.txt
```

Set environment variables and start the server:

```bash
export API_KEY=my-secret-key
export GROQ_API_KEY=your_groq_key
python -m uvicorn app.main:app --port 8000
```

Windows PowerShell:

```powershell
$env:API_KEY="my-secret-key"
$env:GROQ_API_KEY="your_groq_key"
python -m uvicorn app.main:app --port 8000
```

Wait for `RAG chain ready` in the logs. The first start downloads the models.

### Option 2: GitHub Codespaces
Click **Code -> Codespaces -> Create codespace**, then run the same commands as Option 1 in the terminal.

### Option 3: Docker

```bash
docker build -t rag-api .
docker run -p 8000:8000 \
  -e API_KEY=my-secret-key \
  -e GROQ_API_KEY=your_groq_key \
  rag-api
```

## Configuration

| Variable | Required | Description | Default |
|---|---|---|---|
| `API_KEY` | Yes | Key clients must send in the `x-api-key` header | none (all requests denied) |
| `GROQ_API_KEY` | No | Use Groq's hosted LLM instead of a local model | local model |
| `SLACK_WEBHOOK_URL` | No | Post each Q&A to Slack | disabled |
| `PDF_DIR` | No | Folder containing PDFs | `pdf_data` |
| `GROQ_MODEL` | No | Groq model name | `llama-3.1-8b-instant` |
| `LOCAL_LLM_NAME` | No | Local Hugging Face model | `Qwen/Qwen2.5-1.5B-Instruct` |
| `EMBEDDING_MODEL` | No | Embedding model | `all-MiniLM-L6-v2` |
| `RERANKER_MODEL` | No | Reranker model | `BAAI/bge-reranker-base` |

Never commit secrets to Git. Use environment variables.

## API Reference

Interactive docs are available at `http://localhost:8000/docs`.

### `GET /health`
Checks that the service is alive.

```json
{ "status": "ok", "model_loaded": true }
```

### `POST /ask`
Ask a question about the indexed PDFs.

**Headers**
- `x-api-key: <your key>`
- `Content-Type: application/json`

**Body**
```json
{ "question": "What is this document about?" }
```
`question` must be 3 to 1000 characters.

**Example**
```bash
curl -X POST http://localhost:8000/ask \
  -H "Content-Type: application/json" \
  -H "x-api-key: my-secret-key" \
  -d '{"question": "What is this document about?"}'
```

**Response `200`**
```json
{
  "answer": "The document describes ...",
  "sources": [
    { "file": "report.pdf", "page": 3 },
    { "file": "report.pdf", "page": 7 }
  ]
}
```

**Errors**

| Code | Meaning |
|---|---|
| `401` | Invalid or missing API key |
| `422` | Invalid input (question too short or long, or header missing) |
| `500` | RAG pipeline failed (details are in the server logs) |

## Testing

With the server running, in a second terminal:

```bash
python test_api.py
```

Expected: health `ok`, a valid answer (`200`), wrong key (`401`), short question (`422`).

## Slack Integration

1. Create an app at [api.slack.com/apps](https://api.slack.com/apps) and enable **Incoming Webhooks**.
2. Add a webhook to a channel and copy the URL.
3. Set `SLACK_WEBHOOK_URL`.

Each answered question is posted to the channel **after** the response is returned. A Slack failure never affects the API.

## Design Decisions

- **Two layers**: `main.py` handles web concerns, `rag.py` handles AI. Either can change without touching the other.
- **Load once at startup**: building the chain takes a minute, so it happens in the FastAPI lifespan and not per request.
- **Constant-time key comparison**: `secrets.compare_digest` avoids timing attacks.
- **Safe default**: if `API_KEY` is not set, every request is denied.
- **Non-blocking integration**: Slack runs as a background task with a timeout and error handling.
- **Clean errors**: clients get a generic `500`, while the full trace goes to the logs.

## Limitations

- The index is built in memory at startup, so adding PDFs requires a restart.
- Scanned (image-only) PDFs need OCR, which is not included.
- Small local models give weaker answers than hosted ones.
- A single API key is shared by all clients.

## Roadmap

- [ ] Upload PDFs through the API
- [ ] Persistent vector store
- [ ] Conversation memory and query rewriting
- [ ] Evaluation with RAGAS
- [ ] Slack slash command (`/ask`) for two-way integration
- [ ] Rate limiting and per-client API keys
- [ ] CI pipeline with automated tests
- [ ] Agentic RAG with LangGraph

## License

MIT

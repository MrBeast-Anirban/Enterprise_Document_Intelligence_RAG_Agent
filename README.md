# Enterprise Document Intelligence RAG Agent

A production-style Retrieval-Augmented Generation (RAG) application that lets you ingest PDF documents, embed and store them in a vector database, and ask natural-language questions that are answered using the retrieved context. Built with **FastAPI**, **Inngest** (for durable, event-driven workflows), **Qdrant** (vector database), **OpenAI** (embeddings + LLM), and **Streamlit** (optional UI).

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Setup](#setup)
  - [1. Clone the repo](#1-clone-the-repo)
  - [2. Install dependencies](#2-install-dependencies)
  - [3. Configure environment variables](#3-configure-environment-variables)
  - [4. Start Qdrant (vector database)](#4-start-qdrant-vector-database)
  - [5. Start the Inngest dev server](#5-start-the-inngest-dev-server)
  - [6. Start the FastAPI app](#6-start-the-fastapi-app)
- [Usage](#usage)
  - [Ingesting a PDF](#ingesting-a-pdf)
  - [Querying the document](#querying-the-document)
  - [Using the Streamlit UI](#using-the-streamlit-ui)
- [Events Reference](#events-reference)
- [Configuration Notes](#configuration-notes)
- [Troubleshooting](#troubleshooting)
- [Roadmap Ideas](#roadmap-ideas)
- [License](#license)

---

## Overview

This project demonstrates a **production-grade RAG pipeline** using durable, event-driven workflows instead of a simple linear script. Rather than calling functions directly, the app emits **events** (`rag/ingest_pdf`, `rag/query_pdf_ai`) that trigger **Inngest functions**. Each function is broken into discrete, retryable **steps** (`ctx.step.run(...)`), which gives you:

- Automatic retries on failure for each step independently
- Full observability of every run via the Inngest dashboard
- Built-in throttling / concurrency control
- A clean separation between ingestion and querying workflows

At a high level, the app does two things:

1. **Ingest** — Load a PDF, split it into chunks, embed each chunk with OpenAI embeddings, and upsert the vectors + metadata into a Qdrant collection.
2. **Query** — Embed a user's question, perform a similarity search against Qdrant to retrieve the most relevant chunks, and pass that context to an LLM (via Inngest's AI step) to generate a grounded answer.

---

## Architecture

```
                    ┌──────────────────────┐
                    │   PDF Document(s)    │
                    └──────────┬───────────┘
                               │
                     rag/ingest_pdf event
                               │
                               ▼
                 ┌──────────────────────────────┐
                 │   Inngest: rag_ingest_pdf    │
                 │  1. load_and_chunk (step)    │
                 │  2. embed_and_upsert (step)  │
                 └──────────────┬───────────────┘
                                │
                                ▼
                     ┌───────────────────┐
                     │   Qdrant (vector  │
                     │   database)       │
                     └─────────┬─────────┘
                                │
                     rag/query_pdf_ai event
                                │
                                ▼
                 ┌──────────────────────────────┐
                 │  Inngest: rag_query_pdf_ai   │
                 │  1. embed-and-search (step)  │
                 │  2. llm-answer (AI step)     │
                 └──────────────┬───────────────┘
                                │
                                ▼
                     ┌───────────────────┐
                     │  Answer + Sources │
                     └───────────────────┘
```

- **FastAPI** exposes the Inngest webhook endpoint (`/api/inngest`) that the Inngest dev server (or Inngest Cloud in production) calls to invoke functions.
- **Inngest** orchestrates the workflow, handling retries, step memoization, and event delivery.
- **Qdrant** stores document chunk embeddings and their metadata (source file, raw text).
- **OpenAI** provides both the text embedding model and the chat completion model used to generate answers.
- **Streamlit** (optional) provides a simple web UI for uploading PDFs and asking questions without manually firing events.

---

## Tech Stack

| Component        | Purpose                                              |
|-------------------|-------------------------------------------------------|
| **FastAPI**       | Web server hosting the Inngest webhook endpoint       |
| **Inngest**       | Durable function orchestration / event-driven workflow|
| **Qdrant**        | Vector database for storing & searching embeddings    |
| **OpenAI API**    | Embeddings (`text-embedding-3-large` or similar) + chat completions (`gpt-4o-mini`) |
| **LlamaIndex Core / Readers** | PDF loading and chunking utilities         |
| **Streamlit**     | Optional front-end UI                                 |
| **uv**            | Python package & environment manager                  |
| **Docker**        | Running Qdrant locally as a container                 |

---

## Project Structure

```
RAGProductionApp/
├── main.py               # FastAPI app + Inngest function definitions (ingest & query)
├── vector_db.py           # QdrantStorage class: upsert & search logic
├── data_loader.py         # PDF loading, chunking, and embedding helpers
├── custom_types.py        # Pydantic models used across steps for type-safe I/O
├── streamlit_app.py       # Optional Streamlit UI for ingest/query
├── pyproject.toml         # Project metadata & dependencies (uv/PEP 621)
├── uv.lock                # Locked, reproducible dependency versions
├── qdrant_storage/        # Local Qdrant data volume (Docker mount)
├── .env                   # Environment variables (not committed)
├── .gitignore
└── README.md
```

---

## Prerequisites

Make sure you have the following installed:

- **Python 3.13+**
- **[uv](https://docs.astral.sh/uv/)** — fast Python package/dependency manager
- **[Docker](https://www.docker.com/)** — to run Qdrant locally
- **Node.js + npm** — to run the Inngest CLI dev server (`npx`)
- An **OpenAI API key**

---

## Setup

### 1. Clone the repo

```bash
git clone https://github.com/MrBeast-Anirban/Enterprise-Document-Intelligence-RAG-Agent.git
cd Enterprise-Document-Intelligence-RAG-Agent
```

### 2. Install dependencies

Using `uv` (recommended — reads `pyproject.toml` / `uv.lock`):

```bash
uv sync
```

This creates a `.venv` and installs all dependencies at their locked versions.

<details>
<summary>Alternative: using pip + requirements.txt</summary>

```bash
python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```
</details>

### 3. Configure environment variables

Create a `.env` file in the project root:

```env
OPENAI_API_KEY=sk-your-openai-key-here
```

> **Never commit `.env`** — it's already covered in `.gitignore`.

### 4. Start Qdrant (vector database)

Qdrant runs locally via Docker and persists data to `./qdrant_storage`.

**macOS / Linux:**
```bash
docker run -d --name qdrant -p 6333:6333 -v "./qdrant_storage:/qdrant/storage" qdrant/qdrant
```

**Windows (PowerShell):**
```powershell
docker run -d --name qdrant -p 6333:6333 -v "$(pwd)/qdrant_storage:/qdrant/storage" qdrant/qdrant
```

Verify it's running by visiting: [http://localhost:6333/dashboard](http://localhost:6333/dashboard)

### 5. Start the Inngest dev server

In a separate terminal:

```bash
npx inngest-cli@latest dev -u http://127.0.0.1:8000/api/inngest --no-discovery
```

This starts the local Inngest development server and points it at your FastAPI app's webhook endpoint. Open the Inngest dashboard in your browser:

```
http://localhost:8288
```

From here you can manually trigger events, inspect step-by-step run logs, and see retries in real time.

### 6. Start the FastAPI app

In another terminal:

```bash
uv run uvicorn main:app --reload
```

`--reload` is recommended during development so code changes are picked up automatically without manually restarting the server.

By default, this runs on `http://127.0.0.1:8000`.

---

## Usage

The app is event-driven — you don't call ingestion/query functions directly. Instead, you send an **event** to Inngest, and it invokes the corresponding function.

### Ingesting a PDF

Send a `rag/ingest_pdf` event with the path to your PDF:

```python
import inngest

inngest_client = inngest.Inngest(app_id="rag_app")

inngest_client.send_sync(
    inngest.Event(
        name="rag/ingest_pdf",
        data={
            "pdf_path": "/absolute/path/to/document.pdf",
            "source_id": "document.pdf"   # optional, defaults to pdf_path
        }
    )
)
```

You can also trigger this manually from the **Inngest Dev Server dashboard** (`http://localhost:8288`) under the "Functions" tab by selecting `RAG: Ingest PDF` and providing a matching JSON payload.

**What happens under the hood:**
1. `load_and_chunk` step — loads the PDF and splits it into text chunks.
2. `embed_and_upsert` step — embeds each chunk via OpenAI and upserts vectors + metadata (`source`, `text`) into the Qdrant `docs` collection.

### Querying the document

Send a `rag/query_pdf_ai` event with your question:

```python
inngest_client.send_sync(
    inngest.Event(
        name="rag/query_pdf_ai",
        data={
            "question": "What is the candidate's most recent job title?",
            "top_k": 5   # optional, defaults to 5
        }
    )
)
```

**What happens under the hood:**
1. `embed-and-search` step — embeds the question and performs a similarity search in Qdrant to retrieve the top-k most relevant chunks.
2. `llm-answer` step (AI step) — sends the retrieved context + question to the LLM (`gpt-4o-mini`) and returns a grounded answer.

**Response shape:**
```json
{
  "answer": "The candidate's most recent title is ...",
  "sources": ["document.pdf"],
  "num_contexts": 5
}
```

### Using the Streamlit UI

If you'd rather not manually send events, use the included Streamlit app:

```bash
uv run streamlit run streamlit_app.py
```

This provides a simple browser UI to upload a PDF (triggering ingestion) and ask questions (triggering the query workflow), without needing to interact with the Inngest dashboard directly.

---

## Events Reference

| Event name           | Trigger                    | Payload                                                  |
|------------------------|-----------------------------|-----------------------------------------------------------|
| `rag/ingest_pdf`      | Ingest a new PDF document   | `pdf_path` (str, required), `source_id` (str, optional)  |
| `rag/query_pdf_ai`    | Ask a question over ingested documents | `question` (str, required), `top_k` (int, optional, default `5`) |

---

## Configuration Notes

- **Vector dimensions**: `QdrantStorage` defaults to `dim=3072`, matching OpenAI's `text-embedding-3-large`. If you switch embedding models, update this to match the new model's output dimension (e.g. `1536` for `text-embedding-3-small`).
- **Distance metric**: Cosine similarity (`Distance.COSINE`) is used for the Qdrant collection.
- **Throttling**: Functions can be configured with `inngest.Throttle(limit=..., period=...)` to control how frequently they run (e.g. to stay within OpenAI rate limits).
- **Collection name**: Defaults to `"docs"` in `QdrantStorage`. All ingested PDFs share this single collection unless you customize it.
- **IDs**: Each chunk gets a deterministic UUID via `uuid.uuid5(uuid.NAMESPACE_URL, f"{source_id}:{i}")`, so re-ingesting the same `source_id` will overwrite (not duplicate) existing chunks.

---

## Troubleshooting

**`AttributeError: 'QdrantClient' object has no attribute 'search'`**
Your installed `qdrant-client` version has removed the legacy `.search()` method. Use `.query_points()` instead (already reflected in `vector_db.py` in this project, requires `qdrant-client>=1.10`).

**`pydantic_core._pydantic_core.ValidationError` on `RAGSearchResult`**
Make sure `contexts` and `sources` in `custom_types.py` are typed as `list[str]`, not `list[set]`.

**`inngest._internal.errors.FunctionConfigInvalidError: limit: Field required`**
`inngest.Throttle` expects a `limit` argument, not `count`:
```python
throttle=inngest.Throttle(limit=2, period=datetime.timedelta(minutes=1))
```

**Qdrant connection refused**
Make sure the Docker container is running:
```bash
docker ps
```
If it's not listed, re-run the `docker run ...` command from [step 4](#4-start-qdrant-vector-database).

**Inngest functions not appearing in the dashboard**
Ensure both the FastAPI app (`uvicorn`) and the Inngest dev server are running simultaneously, and that the `-u` URL in the `inngest-cli dev` command matches where your FastAPI app is actually served.

**Code changes not taking effect**
- Restart `uvicorn` if not using `--reload`.
- Restart the Inngest dev server after changing function signatures/config (e.g. `Throttle`).
- Clear stale bytecode: `find . -name "__pycache__" -type d -exec rm -rf {} +`

---

## Roadmap Ideas

- [ ] Support multiple file types (Word, Markdown, HTML) via additional LlamaIndex readers
- [ ] Add authentication for the API
- [ ] Add per-source deletion/re-indexing endpoint
- [ ] Add citation highlighting in the Streamlit UI
- [ ] Add automated tests (pytest) for ingestion and query pipelines
- [ ] Deploy to a production Inngest environment + hosted Qdrant instance

---

## License

This project is licensed under the Apache License 2.0 — see the [LICENSE](LICENSE) file for details.

Copyright 2026 Anirban Maitra

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.

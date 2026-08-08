<div align="center">

<img src="https://capsule-render.vercel.app/api?type=waving&color=0:0078D4,100:F2C811&height=200&section=header&text=RAG%20API&fontSize=48&fontColor=ffffff&fontAlignY=35&desc=Production-Ready%20Retrieval-Augmented%20Generation%20Service&descAlignY=55&descSize=16&animation=fadeIn" width="100%" />

![Python](https://img.shields.io/badge/Python-3776AB?style=for-the-badge&logo=python&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-009688?style=for-the-badge&logo=fastapi&logoColor=white)
![LlamaIndex](https://img.shields.io/badge/LlamaIndex-000000?style=for-the-badge&logoColor=white)
![Qdrant](https://img.shields.io/badge/Qdrant-DC244C?style=for-the-badge&logoColor=white)
![AWS](https://img.shields.io/badge/AWS-232F3E?style=for-the-badge&logo=amazon-aws&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-green?style=for-the-badge)

<img src="https://readme-typing-svg.demolab.com/?font=Fira+Code&pause=1000&color=0078D4&center=true&vCenter=true&width=650&lines=Ask+questions%2C+get+cited+answers;Hybrid+search+%2B+reranking+%2B+streaming;Powered+by+Claude%2C+GPT-4o%2C+or+Llama+3.1" alt="Typing SVG" />

</div>

---

## 📖 Overview

**RAG API** is a FastAPI service that answers user queries by retrieving relevant context from a document knowledge base and generating grounded, cited responses with an LLM. It's built for multi-domain deployments (e.g. Finance, Oil & Gas, Healthcare) and streams answers back over Server-Sent Events, so responses appear token-by-token instead of waiting for the full generation.

Under the hood it combines **LlamaIndex** for orchestration, **Qdrant** for hybrid vector + sparse search, an optional **Cohere reranker** for precision, and pluggable LLM backends — **Anthropic Claude**, **OpenAI GPT-4o**, or a **local Ollama model**.

---

## ✨ Features

- 🔎 **Hybrid Retrieval** — dense + sparse (BM42) search on Qdrant for better recall than vector search alone
- 🎯 **Reranking** — Cohere reranker narrows top-K candidates down to the most relevant chunks before generation
- 💬 **Streaming Chat Endpoint** — `/v1/chat` streams the answer as it's generated via `StreamingResponse`
- 🧵 **Conversation Memory** — per-`chat_id` history is persisted and summarized so follow-up questions stay in context
- 📎 **Cited Answers** — responses reference the source chunks they were generated from, not just free-form text
- 🧭 **Query Rewriting** — incoming queries are rephrased for better retrieval before hitting the vector store
- 🛡️ **Input Validation & Safety** — profanity filtering, category validation, and chat-ID sanitization via Pydantic validators
- 🔌 **Multi-LLM Support** — swap between Anthropic, OpenAI, or Ollama by editing one config value
- ☁️ **AWS-Native Ops** — S3 for persistence/chat logs, Secrets Manager for credentials, CloudWatch for logging
- 🧩 **Prompt Library** — modular `.prompt` templates for greeting detection, query rephrasing, citation formatting, history summarization, and more
- ♻️ **Retry-Safe Calls** — `tenacity`-backed retries around LLM/storage calls for resilience

---

## 🏗️ Architecture

```
Client
  │  POST /v1/chat  { chat_id, query, category, ... }
  ▼
FastAPI (app.py)
  ├─ Validate request (chat_id, query, category)
  ├─ Rewrite query via LLM
  ├─ Load chat history + persisted index (StorageManager → S3)
  ▼
Generate (generate.py)
  ├─ Build query engine over Qdrant (hybrid: dense + BM42 sparse)
  ├─ Rerank candidates (Cohere)
  ├─ Generate streamed, cited answer (Claude / GPT-4o / Llama)
  ▼
StreamingResponse → Client (text/event-stream)
  + background task: summarize & persist chat history to S3
```

---

## 🚀 Getting Started

### Prerequisites
- Python 3.8+
- A running [Qdrant](https://qdrant.tech/) instance
- AWS account (S3 for persistence/logs, Secrets Manager for API keys)
- API key for at least one LLM provider (Anthropic / OpenAI) or a local Ollama install

### Installation

```bash
git clone https://github.com/swasti9004/rag-api.git
cd rag-api
pip install -r requirements.txt
```

### Configuration

All runtime settings live in `config/config.yaml` — embedding model, reranker, LLM provider/model, Qdrant hybrid-search flags, S3 buckets, and CloudWatch logging targets. Secrets (API keys, AWS credentials) are pulled at runtime via `secrets_manager.py` from AWS Secrets Manager.

Key settings you'll typically adjust:

| Setting | Purpose |
|---|---|
| `LLM_MODEL_TYPE` | `ANTHROPIC`, `OPENAI`, or `OLLAMA` |
| `HF_EMBED` | HuggingFace embedding model used for indexing |
| `COHERE_RERANKER` | Reranker model applied post-retrieval |
| `QDRANT_ENABLE_HYBRID` | Toggle dense + sparse hybrid search |
| `RAG_SIMILARITY_TOP_K` / `RAG_RERANKED_TOP_N` | Retrieval → rerank funnel size |
| `RAG_STREAMING` | Enable/disable token streaming |

### Run

```bash
uvicorn app:app --host 0.0.0.0 --port 8000 --reload
```

---

## 📡 API Usage

**Health check**
```bash
curl http://localhost:8000/health
```

**Ask a question**
```bash
curl -X POST http://localhost:8000/v1/chat \
  -H "Content-Type: application/json" \
  -d '{
        "chat_id": "demo-session-01",
        "query": "What were the key production trends last quarter?",
        "category": "finance",
        "collection_name": "rag_llm"
      }'
```

The response streams back as `text/event-stream`, so the answer appears incrementally as it's generated.

---

## 📁 Project Structure

```
rag-api/
├── app.py                 # FastAPI app, request models, /health & /v1/chat endpoints
├── generate.py             # Core RAG pipeline: storage, retrieval, reranking, generation
├── secrets_manager.py       # AWS Secrets Manager integration
├── config/
│   └── config.yaml          # Central runtime configuration
├── prompts/                 # Modular prompt templates (rephrasing, citation, greeting, etc.)
├── requirements.txt
└── Makefile                 # lint (ruff) / format (black)
```

---

## 🧰 Tech Stack

`FastAPI` · `LlamaIndex` · `Qdrant` (hybrid dense + BM42 sparse search) · `FastEmbed` · `Cohere Rerank` · `Anthropic Claude` / `OpenAI GPT-4o` / `Ollama` · `AWS S3 / Secrets Manager / CloudWatch` · `Tenacity` · `Ruff` + `Black`

---

## ⚠️ Notes & Limitations

- Retrieval quality depends on how the source documents were chunked and indexed — tune `RAG_CITATION_CHUNK_SIZE` / `RAG_SIMILARITY_TOP_K` for your corpus.
- Requires valid AWS credentials and a reachable Qdrant instance to run end-to-end.
- Like any RAG system, answers are only as accurate as the retrieved context — always review citations for high-stakes use cases.

---

<div align="center">

Built by **[Swasti Narayan Mohapatra](https://github.com/swasti9004)**

<a href="https://linkedin.com/in/swastinarayan"><img src="https://img.shields.io/badge/LinkedIn-0077B5?style=for-the-badge&logo=linkedin&logoColor=white" /></a>
<a href="mailto:chinuswasti8@gmail.com"><img src="https://img.shields.io/badge/Email-D14836?style=for-the-badge&logo=gmail&logoColor=white" /></a>

<img src="https://capsule-render.vercel.app/api?type=waving&color=0:F2C811,100:0078D4&height=90&section=footer" width="100%" />

</div>

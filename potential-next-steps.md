# From POC to Production: Next Steps After AutoRAG

## Executive Summary

The AutoRAG POC proved which RAG configuration works best for the customer's data. The next phase takes those validated parameters and stands up a production service: index the full document set, expose the Llama Stack Responses API as a secured endpoint, and build an automated ingestion pipeline so the knowledge base stays current.

Teams can then integrate this endpoint into any application — a chat interface, a helpdesk bot, or an internal search tool — with a single API call.

---

## What the POC Delivered

From the winning pattern (Pattern4), we have validated production-ready parameters:

| Parameter | Validated Value |
|-----------|----------------|
| Chunk size | 2048 characters |
| Chunk overlap | 128 characters |
| Embedding model | nomic-embed-text-v1.5 (768 dimensions) |
| Retrieval count | 10 chunks per query |
| Search mode | Hybrid (vector + keyword) with RRF ranking |
| Foundation model | Llama 3.2 3B Instruct (recommend upgrading to 8B+ for production) |
| Faithfulness score | 0.844 |
| Context correctness | 0.906 |

These parameters eliminate the guesswork — we know empirically that this configuration delivers the best faithfulness for this document set.

---

## Production Architecture

```
┌─────────────────────────────────────────────────────┐
│  Users                                               │
│  (Chat UI / API Clients / App Integration)           │
├─────────────────────────────────────────────────────┤
│  OpenShift Route + OAuth Proxy                       │
│  (authentication, TLS termination, rate limiting)    │
├─────────────────────────────────────────────────────┤
│  Llama Stack Responses API (port 8321)              │
│  ├── file_search tool → Milvus (vector retrieval)   │
│  ├── Embedding → sentence-transformers (inline)     │
│  └── Generation → vLLM (GPU)                        │
├─────────────────────────────────────────────────────┤
│  Milvus (production cluster, replicated, backed up) │
├─────────────────────────────────────────────────────┤
│  Document Ingestion Pipeline (KFP, scheduled)       │
│  └── S3 watch → chunk (2048/128) → embed → upsert  │
└─────────────────────────────────────────────────────┘
```

---

## Implementation Phases

### Phase 1: Index Full Document Corpus (Week 1)

Take the winning pattern's chunking + embedding configuration and index the customer's complete document set into Milvus.

**Approach options:**
- **One-time manual run**: Execute the indexing notebook against the full S3 bucket
- **Automated pipeline**: Build a KFP pipeline that chunks, embeds, and upserts documents on schedule or on new uploads

**Key decisions:**
- How many documents in the full corpus?
- How often do documents change?
- Do we need incremental updates or full re-indexing?

---

### Phase 2: Enable the Responses API (Week 1-2)

Llama Stack already includes a built-in **Responses API** — an agentic layer that orchestrates the full RAG flow in a single API call.

#### Shortcut: Use the pipeline-generated `v1_responses_body.json`

AutoRAG's pipeline generates a ready-to-use Responses API request body for each winning pattern at:

```
s3://pipelines/documents-rag-optimization-pipeline/<run-id>/prepare-responses-api-requests/<artifact-id>/responses_api_artifacts/<PatternX>/v1_responses_body.json
```

This file contains the exact API call parameters (model, vector store ID, search config, ranking options) pre-configured from the winning pattern. No manual parameter extraction needed — just use it directly:

```bash
# Download the pre-built request body for the winning pattern
oc exec <minio-pod> -n <ns> -- mc cat \
  local/pipelines/documents-rag-optimization-pipeline/<run-id>/prepare-responses-api-requests/<artifact-id>/responses_api_artifacts/Pattern4/v1_responses_body.json \
  > v1_responses_body.json
```

Then call the Responses API with it:

```bash
curl -X POST http://llama-stack-service:8321/v1/responses \
  -H "Content-Type: application/json" \
  -d @v1_responses_body.json
```

#### Manual approach: Build the API call yourself

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://llama-stack-service:8321/v1",
    api_key="<api-key>"
)

# Single API call = embed query + search Milvus + generate answer
response = client.responses.create(
    model="llama-32-3b-instruct",
    input="What is the return policy?",
    tools=[{
        "type": "file_search",
        "vector_store_ids": ["<vector-store-id>"],
        "max_num_results": 10,
        "ranking_options": {
            "ranker": "rrf",
            "k": 60,
            "alpha": 1.0
        }
    }],
    tool_choice={"type": "file_search"}
)

print(response.output_text)
# → "Items can be returned within 30 days of purchase..."
```

No custom RAG orchestration code needed. The API handles:
- Query embedding
- Vector store search (hybrid + RRF)
- Context assembly
- LLM generation with grounding
- Response formatting

---

### Phase 3: Expose as a Production Service (Week 2-3)

| Option | Effort | Best for |
|--------|--------|----------|
| **Direct Llama Stack Route** | 1-2 days | Internal teams comfortable with API calls |
| **Custom FastAPI wrapper** | 3-5 days | Adding auth, logging, custom prompts, rate limiting |
| **Chat UI (Streamlit/Gradio)** | 1 week | Business user demo, helpdesk agents, non-technical users |
| **Embed in existing app** | Varies | CRM, helpdesk, internal portal integration |

#### Example: Streamlit Chatbot on OpenShift

A minimal production chatbot needs:

| Resource | Purpose |
|----------|---------|
| ConfigMap | Stores INFERENCE_MODEL, LLAMA_STACK_HOST, LLAMA_STACK_PORT, VECTOR_STORE_ID |
| Deployment | Streamlit app container calling Llama Stack Responses API |
| Service | ClusterIP on port 8501 |
| Route | TLS edge-terminated for external access |

The app container connects to Llama Stack using the vector store ID from the winning AutoRAG pattern and calls `/v1/responses` with `file_search` tool for every user query. The Responses API handles retrieval + generation internally.

Key environment variables for the chatbot:

```
INFERENCE_MODEL=vllm-inference-1/llama-32-3b-instruct
LLAMA_STACK_HOST=lsd-genai-playground-service
LLAMA_STACK_PORT=8321
VECTOR_STORE_ID=<from winning pattern's v1_responses_body.json>
```

**Minimum production wrapper adds:**
- Authentication (OAuth proxy / API key validation)
- Request logging and audit trail
- Rate limiting per user/team
- Custom system prompts per use case
- Error handling and fallback responses

---

### Phase 4: Automated Document Ingestion (Week 3-4)

Build a pipeline that keeps the knowledge base current:

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  S3 Bucket   │────▶│  Chunk docs  │────▶│  Embed chunks│────▶│  Upsert to   │
│  (new docs)  │     │  (2048/128)  │     │  (nomic-v1.5)│     │  Milvus      │
└──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
```

**Options:**
- **Scheduled**: KFP pipeline runs nightly, re-indexes changed documents
- **Event-driven**: S3 notification triggers pipeline on upload
- **Manual**: Team runs indexing notebook when documents are updated

---

### Phase 5: Operational Hardening (Week 4+)

| Concern | Solution |
|---------|----------|
| **Scaling** | HPA on vLLM deployment for concurrent users; Milvus cluster mode for large datasets |
| **Monitoring** | Track response latency, token usage, faithfulness drift via OpenShift monitoring + Prometheus |
| **Model upgrades** | Re-run AutoRAG when upgrading foundation model to validate new config |
| **Guardrails** | Llama Stack safety providers for content filtering, PII detection |
| **Backup** | Milvus snapshots + S3 bucket versioning for disaster recovery |
| **Multi-tenancy** | Separate vector stores per team/department, RBAC via OAuth proxy |
| **Evaluation** | Periodic re-evaluation against test data to detect quality drift |

---

## Model Upgrade Recommendations

### Foundation Model (LLM)

For production, consider upgrading from the POC's 3B model:

| Model | Params | Use case | GPU requirement |
|-------|--------|----------|-----------------|
| Llama 3.2 3B (POC) | 3B | Validation, dev/test | ~8GB VRAM |
| Llama 3.1 8B | 8B | Production (standard) | ~16GB VRAM |
| Llama 3.1 70B | 70B | Production (high quality) | ~140GB VRAM (multi-GPU) |

A larger model improves both answer quality AND evaluation reliability (since the same model acts as the LLM-as-judge).

### Embedding Model

The POC used `nomic-embed-text-v1.5` (auto-discovered by the sentence-transformers provider). For production, consider purpose-built models validated by Red Hat:

| Model | Params | Embedding Dim | Context Length | Best for |
|-------|--------|---------------|----------------|----------|
| nomic-embed-text-v1.5 (POC) | 137M | 768 | 8192 | General purpose, good baseline |
| BAAI/bge-m3 | 568M | 1024 | 8192 | Multilingual (100+ languages), dense + sparse retrieval. Red Hat recommended. |
| RedHatAI/granite-embedding-english-r2 | 149M | 768 | 8192 | Enterprise-optimized, best MTEB scores in its class, Apache 2.0 |
| RedHatAI/granite-embedding-small-english-r2 | 47M | 384 | 8192 | Lightweight, fastest throughput, ideal for high-volume production |

**Recommendation**: For English-only production workloads, `RedHatAI/granite-embedding-english-r2` gives the best accuracy-to-size ratio. For multilingual needs, `BAAI/bge-m3` is the Red Hat documented recommendation.

**Important**: Changing the embedding model requires re-indexing the entire document corpus (different dimensions = incompatible vectors). Re-run AutoRAG after changing the embedding model to validate the full configuration still holds.

### After any model upgrade

Re-run AutoRAG with the new model(s) to validate that the chunking/retrieval parameters remain optimal. A larger foundation model may perform better with different chunk sizes or retrieval counts.

---

## Engagement Proposal

| Phase | Duration | Deliverable |
|-------|----------|-------------|
| Document indexing + API enablement | 1 week | Full corpus indexed, Responses API verified |
| Production service + auth | 1-2 weeks | Secured endpoint with monitoring |
| Ingestion automation | 1 week | Scheduled re-indexing pipeline |
| Hardening + handoff | 1 week | Runbooks, monitoring dashboards, knowledge transfer |
| **Total** | **3-4 weeks** | **Production RAG agent on RHOAI** |

---

## Key Customer Talking Points

1. **"The hard part is done"** — AutoRAG already proved which configuration works. Production is now engineering, not research.

2. **"Single API call"** — The Llama Stack Responses API means no custom RAG orchestration code. One endpoint, one call, grounded answers.

3. **"Everything stays private"** — Same air-gapped, localized architecture from the POC scales directly to production. No data leaves the cluster.

4. **"Knowledge stays current"** — Automated ingestion pipeline means the RAG system updates as documents change, without manual intervention.

5. **"Re-evaluate anytime"** — When models improve or documents change significantly, re-run AutoRAG to validate the configuration still holds.

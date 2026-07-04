# AutoRAG POC - Red Hat OpenShift AI

Sample documents, test data, and implementation guide for running an AutoRAG optimization on Red Hat OpenShift AI (RHOAI 3.4+).

## What's Included

### Documents (`documents/`)

17 Markdown files representing a fictional company's (NovaTech Solutions) internal knowledge base:

- Employee handbook, PTO, remote work, expenses, travel
- Benefits, compensation, parental leave
- IT security, data privacy, engineering standards
- Performance reviews, learning & development
- Code of conduct, onboarding, workplace safety, termination

### Test Data (`test_data.json`)

20 question/answer pairs covering all documents. Format follows the AutoRAG evaluation dataset specification:

```json
{
  "question": "...",
  "correct_answers": ["...", "..."],
  "correct_answer_document_ids": ["filename.md"]
}
```

### Cheat Sheet (`CHEAT-SHEET.md`)

Complete step-by-step implementation guide covering all 9 phases from zero to a working AutoRAG optimization — including every gotcha, workaround, and configuration detail discovered during hands-on validation.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  RHOAI Dashboard → AutoRAG Wizard                            │
├─────────────────────────────────────────────────────────────┤
│  Data Science Pipelines (KFP v2 + Argo Workflows)           │
├─────────────────────────────────────────────────────────────┤
│  Llama Stack / OGX (port 8321)                               │
│  ├── /v1/chat/completions → vLLM (GPU)                      │
│  ├── /v1/embeddings → sentence-transformers (CPU, inline)    │
│  ├── /v1/vector_stores → Remote Milvus (:19530)             │
│  └── scoring → inline::llm-as-judge                          │
├─────────────────────────────────────────────────────────────┤
│  MinIO (S3) │ Milvus (vectors) │ MariaDB (metadata)         │
└─────────────────────────────────────────────────────────────┘
```

## Prerequisites

- OpenShift 4.20+ with RHOAI 3.4 operator installed
- GPU node(s) for vLLM model serving
- Llama Stack / OGX instance with:
  - At least 1 foundation model (LLM) — used for both generation AND evaluation (LLM-as-judge)
  - At least 1 embedding model — runs inline via sentence-transformers (no GPU needed)
- Remote Milvus vector database (inline Milvus is NOT supported)
- Data Science Pipelines server (DSPA) configured with S3 storage
- S3-compatible storage with HTTPS route (MinIO or equivalent)

## Quick Start

1. Upload all files from `documents/` to a **single folder** in your S3 bucket
2. Import the [fixed pipeline YAML](https://github.com/red-hat-data-services/pipelines-components/blob/rhoai-3.4-fixed/pipelines/training/autorag/documents_rag_optimization_pipeline/pipeline.yaml) (required for RHOAI 3.4 — see RHOAIENG-64768)
3. Create an AutoRAG optimization run in the RHOAI dashboard, uploading `test_data.json` as the evaluation dataset
4. Review the leaderboard and download the winning pattern's inference notebook
5. Test in a Workbench (Jupyter | Data Science | CPU | Python 3.12) with Llama Stack + S3 connections attached

See **[CHEAT-SHEET.md](CHEAT-SHEET.md)** for the full detailed walkthrough.

## Model Recommendations

| Role | Minimum (POC) | Recommended | Notes |
|------|---------------|-------------|-------|
| Foundation (LLM) | Llama 3.2 3B | Llama 3.1 8B+ | Larger model = better generation AND evaluation quality |
| Embedding | granite-embedding-125m-english | BAAI/bge-m3 | Must include `context_length` in metadata |

## Known Issues (RHOAI 3.4)

| Issue | Impact | Workaround |
|-------|--------|------------|
| RHOAIENG-64768: Pipeline image pull errors | Run fails immediately | Import fixed pipeline YAML from `rhoai-3.4-fixed` branch |
| Dashboard "Failed to list templates optimization directory" | Can't view results in UI | Download artifacts from S3 directly (see cheat sheet) |
| Llama Stack operator reverts resource changes | Embedding timeouts | Scale down operator before editing deployment |
| Default pod resources too low (250m CPU) | All patterns timeout | Increase to 2+ CPU, 4Gi memory, LLS_WORKERS=4 |

All issues above are resolved in RHOAI 3.5.

## POC Results (Sample)

Full results from a successful 8-pattern run are in [`sample-results/`](sample-results/) — including an interactive HTML leaderboard you can open in your browser.

| Rank | Pattern | Faithfulness | Correctness | Context | Config |
|------|---------|-------------|-------------|---------|--------|
| 1 | **Pattern4** | **0.844** | 0.837 | 0.906 | chunk:2048, overlap:128, 10 chunks, hybrid/RRF, nomic-embed |
| 2 | Pattern3 | 0.817 | 0.842 | 0.913 | chunk:2048, overlap:128, 5 chunks, hybrid/RRF, nomic-embed |
| 3 | Pattern2 | 0.811 | 0.896 | 0.888 | chunk:1024, overlap:128, 10 chunks, vector, granite-125m |
| 4 | Pattern5 | 0.801 | 0.805 | 0.900 | chunk:2048, overlap:128, 3 chunks, hybrid/RRF, nomic-embed |
| ... | ... | ... | ... | ... | (see full leaderboard in sample-results/) |

**Key findings:**
- Larger chunks (2048) + hybrid search + RRF ranking consistently outperformed smaller chunks with pure vector search
- Retrieving more chunks (10) improved faithfulness by giving the LLM more context for grounding
- nomic-embed-text-v1.5 slightly outperformed granite-embedding-125m

## License

Sample documents are fictional and provided for testing purposes only.

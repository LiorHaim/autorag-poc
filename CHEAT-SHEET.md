# AutoRAG POC Cheat Sheet

**Red Hat OpenShift AI 3.4** — Complete implementation guide from zero to running optimization. This guide is specific to RHOAI 3.4. In RHOAI 3.5, Llama Stack is renamed to OGX, the pipeline import workaround is no longer needed, and the dashboard rendering bug is fixed.

> This guide assumes a fresh OpenShift 4.20+ cluster with RHOAI 3.4 operator installed, GPU nodes available, and cluster-admin access. Follow each phase in order.

---

## Phase 0: Environment Prerequisites

### Required Operators

| Operator | Purpose | Namespace |
|----------|---------|-----------|
| Red Hat OpenShift AI Operator | Core platform operator | redhat-ods-operator |
| Cert-manager Operator | TLS certificate management | cert-manager |
| Red Hat OpenShift Serverless | KServe dependency (Knative) | openshift-serverless |
| Red Hat OpenShift Service Mesh | Networking for model serving | istio-system |
| NFD Operator | Node feature discovery for GPUs | openshift-nfd |
| NVIDIA GPU Operator | GPU device plugin + drivers | nvidia-gpu-operator |

### Optional Operators

| Operator | Purpose | When needed |
|----------|---------|-------------|
| Red Hat build of Kueue | Workload queue management | Multi-tenant clusters only |
| Red Hat Leader Worker Set | Distributed training | Not needed for AutoRAG |

---

## Phase 1: Deploy Foundation Model (vLLM)

Deploy a Llama model using KServe with the vLLM runtime. This is your foundation LLM for both answer generation and evaluation (LLM-as-judge).

### 1.1 Create Data Science Project

From RHOAI Dashboard: **Data Science Projects → Create project** → e.g. "my-first-model"

Or via CLI: `oc new-project my-first-model`

### 1.2 Deploy Llama model via KServe

In RHOAI Dashboard → Data Science Projects → your project → Models → Deploy model:

| Setting | Value |
|---------|-------|
| Model name | llama-32-3b-instruct |
| Serving runtime | vLLM ServingRuntime for KServe |
| Model framework | vLLM |
| Model source | Modelcar or S3 bucket with model weights |
| Accelerator | NVIDIA GPU |
| Additional args | `--dtype=half --max-model-len=20000 --gpu-memory-utilization=0.95` |

Wait for the model pod to show **2/2 Ready** before proceeding.

---

## Phase 2: Deploy S3 Storage (MinIO)

AutoRAG needs S3 for both pipeline artifacts and document storage. Deploy MinIO and create an HTTPS Route (required — dashboard rejects HTTP endpoints).

### 2.1 Resources to create

| Resource | Name | Key config |
|----------|------|------------|
| PersistentVolumeClaim | minio-pvc | 20Gi, ReadWriteOnce |
| Secret | minio-secret | MINIO_ROOT_USER=minio, MINIO_ROOT_PASSWORD=minio123 |
| Deployment | minio | image: quay.io/minio/minio:latest, args: server /data --console-address :9001 |
| Service | minio | Ports: 9000 (api), 9001 (console) |
| Route (TLS edge) | minio-s3 | targetPort: api, termination: edge |

> **WARNING**: The RHOAI dashboard requires HTTPS for S3 endpoints. You MUST create an edge-terminated Route. Internal `http://` URLs are rejected with "endpoint URL must use HTTPS scheme."

### 2.2 Create buckets

Exec into the MinIO pod:
```bash
mc alias set local http://localhost:9000 minio minio123
mc mb local/pipelines
mc mb local/documents
```

| Bucket | Purpose |
|--------|---------|
| pipelines | Data Science Pipelines artifacts (DSPA) |
| documents | Your RAG source documents |

---

## Phase 3: Deploy Remote Milvus Vector Database

AutoRAG requires a **REMOTE** Milvus instance. Inline Milvus is NOT supported.

### 3.1 Resources to create

| Resource | Name | Key config |
|----------|------|------------|
| ConfigMap | milvus-etcd-config | Embedded etcd config (listen on :2379) |
| PersistentVolumeClaim | milvus-data | 20Gi, ReadWriteOnce |
| Deployment | milvus-standalone | image: milvusdb/milvus:v2.5.8, command: milvus run standalone |
| Service | milvus | Ports: 19530 (grpc), 9091 (metrics) |

> **DANGER**: Do NOT set securityContext with explicit runAsUser/runAsGroup on OpenShift. The SCC will reject it. Let OpenShift assign a UID from the namespace range automatically.

Required env vars: `ETCD_USE_EMBED=true`, `ETCD_DATA_DIR=/var/lib/milvus/etcd`, `COMMON_STORAGETYPE=local`

---

## Phase 4: Configure Llama Stack

Llama Stack is the unified AI runtime connecting LLM, embeddings, and vector DB. Managed by the Llama Stack Operator via `LlamaStackDistribution` CR. In RHOAI 3.5 this is renamed to "OGX."

### 4.1 Key ConfigMap settings (llama-stack-config)

**Providers to configure:**

| Provider | Type | Config |
|----------|------|--------|
| sentence-transformers | inline::sentence-transformers | config: {} (embedding model runs in-pod on CPU) |
| vllm-inference-1 | remote::vllm | base_url: http://\<model\>-predictor.\<ns\>.svc.cluster.local:8080/v1 |
| milvus | remote::milvus | uri: http://milvus.\<ns\>.svc.cluster.local:19530, persistence.backend: kv_default |
| llm-as-judge | inline::llm-as-judge | config: {} (uses foundation model for evaluation scoring) |

**Registered models:**

| Model ID | Provider | Type | Critical metadata |
|----------|----------|------|-------------------|
| sentence-transformers/ibm-granite/granite-embedding-125m-english | sentence-transformers | embedding | embedding_dimension: 768, context_length: 512 |
| llama-32-3b-instruct | vllm-inference-1 | llm | display_name: llama-32-3b-instruct |

> **WARNING**: You MUST include `context_length` in embedding model metadata. Without it, AutoRAG fails with: "RuntimeError: Failed to auto-detect context_length for model."

### 4.2 Resource Sizing (CRITICAL)

> **DANGER**: The default Llama Stack pod resources (250m CPU, 500Mi memory, 1 worker) are COMPLETELY INADEQUATE for AutoRAG batch embedding. The pipeline WILL timeout with these defaults.

| Setting | Default (broken) | Required for AutoRAG |
|---------|-----------------|---------------------|
| CPU request | 250m | 2+ cores |
| CPU limit | 2 cores | 4-8 cores |
| Memory request | 500Mi | 4Gi |
| Memory limit | 12Gi | 12Gi (keep) |
| LLS_WORKERS env var | 1 | 4 |

**How to override (operator blocks direct edits):**

1. Scale down the Llama Stack operator:
   ```bash
   oc scale deployment llama-stack-operator-controller-manager -n redhat-ods-applications --replicas=0
   ```
2. Edit the deployment directly:
   ```bash
   oc edit deployment lsd-genai-playground -n <your-namespace>
   ```
3. After POC completes, scale operator back up:
   ```bash
   oc scale deployment llama-stack-operator-controller-manager -n redhat-ods-applications --replicas=1
   ```

> **WARNING**: The Llama Stack Operator aggressively reconciles the deployment and reverts manual resource changes. You MUST scale it down first or your edits will be overwritten within seconds.

### 4.3 Critical Gotchas (learned from experience)

| Issue | Symptom | Fix |
|-------|---------|-----|
| Insufficient CPU for embedding | APITimeoutError on /v1/embeddings, all patterns fail | Increase CPU to 2-4 cores + set LLS_WORKERS=4 (see 4.2) |
| Missing context_length | RuntimeError: Failed to auto-detect context_length | Add context_length to embedding model metadata |
| Using inline::milvus | AutoRAG rejects the run | Must use remote::milvus with URI to Milvus service |
| Missing persistence field | Pydantic ValidationError on Llama Stack startup | Add persistence.backend + namespace under vector_io config |
| Embedding model OOM / 500 errors | Model excluded from search space, SearchSpaceValueError | Use only 1 embedding model or increase pod memory |
| Model not discoverable | SearchSpaceValueError: model not available | Verify model responds at /v1/models before running AutoRAG |
| Auto-discovered nomic models | Extra models appear in wizard you didn't register | Deselect unwanted models; keep max 2 embedding models |
| Too many documents for available compute | Embedding timeouts even with good resources | Reduce to 5-10 docs for POC, or use smaller docs |

---

## Phase 5: Create Llama Stack Connection Secret

This Secret tells the RHOAI dashboard how to reach your Llama Stack instance.

| Field | Value |
|-------|-------|
| metadata.name | llama-stack-connection |
| metadata.labels.opendatahub.io/dashboard | "true" |
| metadata.labels.opendatahub.io/managed | "true" |
| metadata.annotations.opendatahub.io/connection-type | llama-stack |
| metadata.annotations.openshift.io/display-name | Llama Stack Connection |
| stringData.LLAMA_STACK_BASE_URL | http://lsd-genai-playground-service.\<ns\>.svc.cluster.local:8321 |
| stringData.LLAMA_STACK_API_KEY | fake |
| type | Opaque |

---

## Phase 6: Configure Data Science Pipelines (DSPA)

AutoRAG runs as a Kubeflow Pipeline (v2 + Argo Workflows). You need a pipeline server in your project.

### 6.1 Configure pipeline server via Dashboard

RHOAI Dashboard → Data Science Projects → your project → Pipelines → Configure pipeline server:

| Field | Value |
|-------|-------|
| S3 Endpoint | http://minio.\<ns\>.svc:9000 |
| Access Key | minio |
| Secret Key | minio123 |
| Bucket | pipelines |
| Region | us-east-1 (or leave default) |

Wait for all `ds-pipeline-*` pods to reach Running state before proceeding.

### 6.2 Import Updated AutoRAG Pipeline (CRITICAL)

> **DANGER** (RHOAI 3.4 only): Known issue RHOAIENG-64768 — The first AutoRAG run FAILS with image pull errors unless you import the updated pipeline definition. This is fixed in RHOAI 3.5.

1. Download the fixed pipeline YAML from: `https://github.com/red-hat-data-services/pipelines-components/blob/rhoai-3.4-fixed/pipelines/training/autorag/documents_rag_optimization_pipeline/pipeline.yaml`
2. Go to **Pipelines → Import pipeline**
3. Pipeline name: `documents-rag-optimization-pipeline`
4. Version name: `documents-rag-optimization-pipeline-3.4.0`
5. Upload the YAML file

### 6.3 Fix AutoRAG Results Dashboard (RHOAI 3.4)

The AutoRAG results page shows "Failed to list templates optimization directory" because the dashboard BFF rejects HTTP S3 endpoints. Fix by changing the DSPA's S3 endpoint to HTTPS:

```bash
oc edit dspa dspa -n <your-namespace>
```

Change `objectStorage.externalStorage`:
- `scheme: http` → `scheme: https`
- `host: minio.<ns>.svc:9000` → `host: <your-minio-https-route-hostname>`

This makes the dashboard BFF accept the S3 connection and render AutoRAG results natively. Pipeline runs continue to work via the HTTPS route.

---

## Phase 7: Prepare Documents and Test Data

### 7.1 Upload documents to MinIO

- All documents must be in a **SINGLE FOLDER** in the bucket
- Supported formats: PDF, DOCX, PPTX, Markdown, HTML, TXT
- Max 32 MiB per file when uploading via dashboard (no limit via S3 API)

```bash
mc cp ./documents/*.md mycluster/documents/my-docs/
```

### 7.2 Test data JSON format

```json
[
  {
    "question": "What is the return policy?",
    "correct_answers": ["Items can be returned within 30 days"],
    "correct_answer_document_ids": ["policies.pdf"]
  }
]
```

| Field | Type | Notes |
|-------|------|-------|
| question | string | The query AutoRAG will test against each RAG pattern |
| correct_answers | array of strings | Multiple phrasings improve scoring accuracy |
| correct_answer_document_ids | array of strings | Base filenames ONLY — no folder paths (e.g. 'policy.pdf' not 'docs/policy.pdf') |

Aim for 20-50 questions covering different documents. Include both simple factual and multi-section reasoning questions. Save as `.json` file.

---

## Phase 8: Run AutoRAG Optimization

### 8.1 Create optimization run

**Dashboard → Data Science Projects → your project → AutoRAG → Create optimization run**

**Page 1 — Connections:**

| Field | Value |
|-------|-------|
| Llama Stack connection | Select your llama-stack-connection secret |
| Vector database | Auto-discovered from Llama Stack config |
| Documents source | S3: browse bucket → select the FOLDER (not individual files) |
| S3 Endpoint | https://\<minio-route-hostname\> (MUST be HTTPS) |
| Evaluation dataset | Upload your test_data.json file |

**Page 2 — Optimization settings:**

| Setting | Recommended for POC |
|---------|-------------------|
| Optimization metric | Answer faithfulness (default) |
| Maximum RAG patterns | 4 (fast POC) to 8 (thorough) |
| Foundation models | Keep your LLM checked (max 3 total) |
| Embedding models | Keep 1 working model only (max 2 total, but be careful of OOM) |

> **WARNING**: When selecting files from S3, you must select a FOLDER, not individual files. If your files are at the bucket root, create a subfolder first and move them there.

### 8.2 What AutoRAG tests (built-in, not configurable)

| Parameter | Values tested |
|-----------|--------------|
| Chunking method | Recursive (only option) |
| Chunk sizes | 1024, 2048 |
| Chunk overlaps | 128, 256 |
| Retrieval chunks | 3, 5, 10 |
| Search mode | Vector, Hybrid |
| + your selected models | Foundation model(s) x Embedding model(s) |

A "RAG pattern" = one specific combination from this grid. AutoRAG picks which combinations to test up to your maximum.

---

## Phase 9: Evaluate Results and Use Output

### 9.1 The leaderboard

| Metric (0-1) | What it measures | Optimize when... |
|--------------|-----------------|------------------|
| Answer faithfulness | Is the answer grounded in retrieved context? | Accuracy and traceability are critical (compliance, legal) |
| Answer correctness | Does the answer match your ground-truth? | You have well-defined expected answers |
| Context correctness | Did retrieval fetch the right docs? | Retrieval quality matters more than answer phrasing |

### 9.2 Output: Jupyter Notebooks

From the best pattern's actions menu, download two notebooks:

| Notebook | Contains | When to use |
|----------|----------|-------------|
| Indexing notebook | Code to chunk docs, embed them, store in Milvus with the winning config | Only if adding MORE documents (vector DB is already populated from the run) |
| Inference notebook | Code to query the RAG system — retrieves chunks + generates grounded answer | Immediately — this is your 'try it now' tool |

> If the dashboard results page shows "Failed to list templates optimization directory" (known TP bug in 3.4, fixed in 3.5), download artifacts directly from S3:
> ```bash
> oc exec <minio-pod> -n <ns> -- mc cp --recursive local/pipelines/documents-rag-optimization-pipeline/<run-id>/rag-templates-optimization/<artifact-id>/rag_patterns/Pattern4/ /tmp/Pattern4/
> oc exec <minio-pod> -n <ns> -- cat /tmp/Pattern4/inference.ipynb > ~/Desktop/inference.ipynb
> oc exec <minio-pod> -n <ns> -- cat /tmp/Pattern4/indexing.ipynb > ~/Desktop/indexing.ipynb
> ```
> Also download the leaderboard HTML:
> ```bash
> oc exec <minio-pod> -n <ns> -- mc cat local/pipelines/documents-rag-optimization-pipeline/<run-id>/leaderboard-evaluation/<artifact-id>/html_artifact > ~/Desktop/leaderboard.html
> ```

### 9.3 Testing the winning pattern in a Workbench

**Create a Workbench:**

1. RHOAI Dashboard → Data Science Projects → your project → Workbenches → Create workbench
2. Configure:

| Setting | Value |
|---------|-------|
| Name | autorag-test |
| Image | Jupyter \| Data Science \| CPU \| Python 3.12 |
| Size | Small (CPU only — inference is handled remotely by Llama Stack/vLLM) |
| Cluster storage | Default (20Gi) |
| Connections | Attach: **Llama Stack Connection** + **minio-https** (S3) |

3. Wait for the workbench pod to reach Running status

**Run the inference notebook:**

1. Click the workbench link to open JupyterLab
2. Upload `inference.ipynb` (the Pattern4 inference notebook)
3. Run all cells in order. When prompted for connection details, use:

| Prompt | Value |
|--------|-------|
| Llama Stack base URL | http://lsd-genai-playground-service.\<ns\>.svc.cluster.local:8321 |
| Llama Stack API key | fake |
| Milvus endpoint (if asked) | http://milvus.\<ns\>.svc.cluster.local:19530 |

4. When prompted, enter a test question — the notebook will:
   - Embed your question using the winning embedding model
   - Search Milvus for the most relevant document chunks
   - Send chunks + question to the LLM via Llama Stack
   - Return a grounded answer

**Example test questions:**
- "How many PTO days does a new employee get?"
- "What is the minimum password length?"
- "How much parental leave does a birth parent receive?"
- "What is the severance package for a layoff?"

> The vector database is already populated from the optimization run. You do NOT need to run the indexing notebook first unless you want to add new documents.

> These notebooks are the RECIPE — the proven-best config as runnable code. To productionize, extract the parameters and build your own application/service around them.

---

## Quick Reference: Service URLs

| Service | Internal URL | Port |
|---------|-------------|------|
| Llama Stack API | http://lsd-genai-playground-service.\<ns\>.svc.cluster.local | 8321 |
| MinIO S3 (internal) | http://minio.\<ns\>.svc | 9000 |
| MinIO S3 (HTTPS Route) | https://minio-s3-\<ns\>.apps.\<cluster-domain\> | 443 |
| MinIO Console | http://minio.\<ns\>.svc | 9001 |
| Milvus gRPC | http://milvus.\<ns\>.svc.cluster.local | 19530 |
| vLLM Model endpoint | http://\<model\>-predictor.\<ns\>.svc.cluster.local | 8080 |

---

## Architecture

| Layer | Component | Role |
|-------|-----------|------|
| UI | RHOAI Dashboard → AutoRAG Wizard | Configuration + monitoring |
| Orchestration | Data Science Pipelines (KFP v2 + Argo) | Executes pipeline steps as pods |
| Runtime | Llama Stack (port 8321) | Unified API for inference, embeddings, vector IO, scoring |
| Inference | vLLM on GPU | Foundation model serving (generation + judging) |
| Embeddings | sentence-transformers (inline in Llama Stack pod) | Vector embedding (CPU, no GPU needed) |
| Vector DB | Milvus standalone (port 19530) | Stores and searches document embeddings |
| Storage | MinIO (port 9000) | S3 for documents + pipeline artifacts |
| Metadata | MariaDB (auto-deployed by DSPA) | Pipeline run history |

```
┌─────────────────────────────────────────────────────────────┐
│  RHOAI Dashboard                                             │
│  └── AutoRAG Wizard → creates Pipeline Run                   │
├─────────────────────────────────────────────────────────────┤
│  Data Science Pipelines (KFP v2 + Argo Workflows)           │
│  └── Pipeline steps:                                         │
│      ├── text-extraction (parse docs from S3)                │
│      ├── search-space-preparation (validate models)          │
│      ├── rag-templates-optimization (test each pattern)      │
│      ├── leaderboard-evaluation (rank and score)             │
│      └── prepare-responses-api-requests (generate notebooks) │
├─────────────────────────────────────────────────────────────┤
│  Llama Stack (port 8321)                                     │
│  ├── /v1/chat/completions → vLLM (GPU, foundation model)    │
│  ├── /v1/embeddings → sentence-transformers (CPU, inline)    │
│  ├── /v1/vector_stores → Remote Milvus (:19530)             │
│  └── scoring → inline::llm-as-judge                          │
├─────────────────────────────────────────────────────────────┤
│  Infrastructure                                              │
│  ├── MinIO — S3 storage (docs + pipeline artifacts)          │
│  ├── Milvus — Vector database (embeddings)                   │
│  ├── MariaDB — Pipeline run metadata                         │
│  └── vLLM on GPU — Model inference                           │
└─────────────────────────────────────────────────────────────┘
```

---

*Based on Red Hat OpenShift AI 3.4 documentation and hands-on validation. AutoRAG is a Technology Preview feature — not for production use without GA release alignment. Several issues documented here (pipeline import, dashboard rendering, operator resource defaults) are fixed in RHOAI 3.5.*

# AutoRAG POC - Red Hat OpenShift AI

Sample documents and test data for running an AutoRAG optimization run on Red Hat OpenShift AI (RHOAI 3.4+).

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

## Prerequisites

- RHOAI 3.4+ with OpenShift AI operator installed
- Llama Stack / OGX instance with:
  - At least 1 foundation model (LLM) for generation + judging
  - At least 1 embedding model for vectorization
- Remote Milvus vector database registered with Llama Stack
- Data Science Pipelines server configured
- S3-compatible storage (MinIO or equivalent)

## Usage

1. Upload all files from `documents/` to your S3 bucket (single folder)
2. Upload `test_data.json` when creating the AutoRAG optimization run
3. Configure the run in the RHOAI dashboard under AutoRAG

## Model Recommendations

| Role | Minimum (POC) | Recommended |
|------|---------------|-------------|
| Foundation (LLM) | Llama 3.2 3B | Llama 3.1 8B or 70B |
| Embedding | granite-embedding-125m | BAAI/bge-m3 |

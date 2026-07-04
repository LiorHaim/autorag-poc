# Sample Results

These are actual outputs from a successful AutoRAG optimization run on RHOAI 3.4.

## Run Configuration

- **Run name**: poc-NovaTech-test-v2-8patterns
- **Maximum patterns**: 8
- **Optimization metric**: Answer faithfulness
- **Foundation model**: llama-32-3b-instruct (vLLM)
- **Embedding models**: nomic-embed-text-v1.5, ibm-granite/granite-embedding-125m-english
- **Documents**: 17 Markdown files (NovaTech company policies)
- **Test questions**: 20

## Files

| File | Description |
|------|-------------|
| `leaderboard.html` | Self-contained HTML leaderboard — open in browser to view all 8 patterns ranked by faithfulness |
| `winning-pattern-config.json` | The winning pattern's full configuration and scores |

## Leaderboard Summary

| Rank | Pattern | Faithfulness | Correctness | Context | Config |
|------|---------|-------------|-------------|---------|--------|
| 1 | **Pattern4** | **0.844** | 0.837 | 0.906 | chunk:2048, overlap:128, 10 chunks, hybrid/RRF, nomic-embed |
| 2 | Pattern3 | 0.817 | 0.842 | 0.913 | chunk:2048, overlap:128, 5 chunks, hybrid/RRF, nomic-embed |
| 3 | Pattern2 | 0.811 | 0.896 | 0.888 | chunk:1024, overlap:128, 10 chunks, vector, granite-125m |
| 4 | Pattern5 | 0.801 | 0.805 | 0.900 | chunk:2048, overlap:128, 3 chunks, hybrid/RRF, nomic-embed |
| 5 | Pattern6 | 0.788 | 0.853 | 0.958 | chunk:1024, overlap:128, 10 chunks, hybrid/RRF, nomic-embed |
| 6 | Pattern7 | 0.788 | 0.841 | 0.963 | chunk:1024, overlap:128, 5 chunks, hybrid/RRF, nomic-embed |
| 7 | Pattern1 | 0.770 | 0.849 | 0.935 | chunk:1024, overlap:256, 5 chunks, hybrid/weighted, nomic-embed |
| 8 | Pattern8 | 0.767 | 0.795 | 0.942 | chunk:1024, overlap:128, 3 chunks, hybrid/RRF, nomic-embed |

## Key Findings

1. **Larger chunks (2048) outperform smaller (1024)** for faithfulness on these documents
2. **Hybrid search with RRF ranking** consistently beats pure vector search and weighted ranking
3. **More retrieved chunks (10) improves faithfulness** — giving the LLM more context to ground answers
4. **nomic-embed-text-v1.5** slightly outperformed granite-embedding-125m across most patterns
5. All patterns scored above 0.76 faithfulness — the documents are well-structured for RAG

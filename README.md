# Retrieval Strategies for Generative AI-Assisted Code Comprehension

Code, benchmark and results accompanying the paper *Retrieval Strategies for Generative AI-Assisted Code Comprehension*.

The study evaluates the retrieval stage of a retrieval-augmented generation (RAG) system for source code comprehension. Twelve retrieval configurations (seven dense embedding models, a BM25 lexical baseline and four hybrid fusions) are compared on a single open-source Java/Spring Boot repository, using 37 natural-language queries with manually annotated relevance labels. The generation stage is out of scope.

## Contents

- [Study Overview](#study-overview)
- [Repository Structure](#repository-structure)
- [Requirements](#requirements)
- [Setup](#setup)
- [Running the Analysis](#running-the-analysis)
- [Outputs](#outputs)
- [Mapping to the Paper](#mapping-to-the-paper)
- [Reproducibility Notes](#reproducibility-notes)
- [Limitations](#limitations)
- [Citation](#citation)
- [License](#license)

## Study Overview

**Research question.** Which retrieval strategy (dense, lexical or hybrid) best retrieves relevant code fragments in a Java microservices repository, and what factors determine its performance?

**Repository under study.** [`internet-banking-concept-microservices`](https://github.com/JavatoDev-com/internet-banking-concept-microservices) (MIT license), commit `efb42d6a6656ec128552ba3f3f93c5ce96f039ff`, snapshot of 2026-09-04. It is a concept and demonstration project, not a production system.

**Benchmark.**

| Item | Value |
| --- | --- |
| Chunks in the repository | 200 (146 Java, 54 configuration) |
| Retrieval corpus | 94 Java fragments, selected purposively (18 distractors) |
| Queries | 37, in four intent categories |
| Query categories | T1 Location (10), T2 Behavior (9), T3 Cross-cutting (9), T4 Tests (9) |
| Relevance | Binary, single annotator |

**Configurations.**

| Family | Configurations |
| --- | --- |
| Dense | `all-MiniLM-L6-v2`, `all-mpnet-base-v2`, `paraphrase-MiniLM-L3-v2` (control), `unixcoder-base`, `jina-code-embeddings-0.5b`, `gemini-embedding-001` with `RETRIEVAL_QUERY` and with `CODE_RETRIEVAL_QUERY` |
| Lexical | Okapi BM25 with code-aware tokenization ($k_1 = 1.5$, $b = 0.75$) |
| Hybrid | Reciprocal Rank Fusion ($k = 60$) over Gemini, MPNet and JinaCode; weighted sum ($\alpha = 0.5$) over Gemini |

**Evaluation.** nDCG@5 (primary), R-precision, MRR, Recall@5, Recall@10 and Precision@5, computed per query. Uncertainty is reported with 95% percentile bootstrap intervals (10,000 resamples). Paired comparisons use Wilcoxon signed-rank tests with Holm correction. Query intent is analyzed with linear mixed-effects models (query as random intercept) and permutation tests. Sensitivity checks cover input truncation, the full 146-fragment corpus, the RRF constant and chunk granularity.

## Repository Structure

```
.
├── code-retrieval-exp.ipynb   # Full analysis: corpus, embeddings, metrics, tests, figures
├── eval_dataset.json          # Single source of truth: corpus selection, queries, labels, taxonomy
├── chunks.jsonl               # All 200 chunks of the study repository, with metadata
├── requirements.txt           # Python dependencies
├── .env.example               # Template for the Google API key
├── cache/                     # Cached embeddings (.npy), created on first run
├── docs/
│   ├── study_repo_tree.txt    # File listing of the pinned commit, used to verify the corpus
│   ├── benchmark.json         # Queries, categories, relevance labels and selected fragments
│   └── queries.csv            # Queries in tabular form
└── results/
    ├── claims.json            # Registry of every value reported in the paper
    ├── metrics.json           # Full numeric record of the analysis
    ├── provenance.json        # Library versions, model revisions, API access date
    ├── *.csv                  # Summary tables and per-query scores
    └── *.png, *.pdf           # Figures
```

## Requirements

- Python 3.11 or later (the reported results were produced with Python 3.14.2 on macOS, Apple Silicon)
- About 3 GB of free disk space for the Hugging Face model weights
- A Google API key with access to the Gemini API, required to generate the Gemini embeddings (see [Setup](#setup))
- Internet access on the first run, to download the open-weight models from Hugging Face

A GPU is not required. All open-weight models run on CPU, although the first run is slower.

## Setup

### 1. Clone the repository and create an environment

```bash
git clone https://github.com/<user>/<repository>.git
cd <repository>
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

### 2. Obtain a Google API key

The two Gemini configurations call the `gemini-embedding-001` model through the Gemini API, which requires a personal API key.

1. Sign in to [Google AI Studio](https://aistudio.google.com/) with a Google account.
2. Open the API keys page and select **Create API key**.
3. Copy the generated key.

Usage of the Gemini API is subject to Google's terms, quotas and pricing. Consult the [Gemini API documentation](https://ai.google.dev/gemini-api/docs) for current limits before running the analysis.

### 3. Configure the key

Create a `.env` file in the repository root (the directory that contains `.gitignore`):

```bash
cp .env.example .env
```

Then edit `.env`:

```
GOOGLE_API_KEY=your-api-key-here
```

The notebook loads this file automatically through `python-dotenv`. The `.env` file is listed in `.gitignore` and must never be committed.

## Running the Analysis

Open and run the notebook from top to bottom:

```bash
jupyter lab code-retrieval-exp.ipynb
```

Or execute it headlessly:

```bash
jupyter nbconvert --to notebook --execute --inplace code-retrieval-exp.ipynb
```

On the first run, the notebook downloads the open-weight models, calls the Gemini API once per corpus and per query set, and stores every embedding matrix in `cache/`. Subsequent runs read the cache, make no API calls and produce identical numbers.

To force all embeddings to be regenerated, delete the cache directory:

```bash
rm -rf cache/
```

## Outputs

All outputs are written to `results/` and `docs/` at the end of the notebook.

| File | Content |
| --- | --- |
| `results/claims.json` | Every value cited in the paper, keyed by a stable identifier |
| `results/metrics.json` | Complete numeric record: metrics, intervals, tests, sensitivity analyses |
| `results/provenance.json` | Pinned commit, library versions, model revisions, Gemini access date, seed |
| `results/summary.csv` | Mean of every metric per configuration |
| `results/bootstrap_ci.csv` | Bootstrap confidence intervals |
| `results/pairwise_wilcoxon.csv` | Paired tests with Holm-adjusted and minimum attainable p-values |
| `results/category_by_config.csv` | nDCG@5 by configuration and query category |
| `results/relevance_separability.csv` | ROC-AUC and score overlap for relevant versus non-relevant fragments |
| `results/sensitivity_corpus.csv` | Comparison on the 94-fragment and the full 146-fragment corpus |
| `results/sensitivity_rrf_k.csv` | RRF results across values of `k` |
| `results/input_budget.csv` | Truncation per model and cost of a reduced input budget |
| `results/per_query_*.csv` | Per-query scores for each configuration |
| `docs/benchmark.json` | The benchmark: queries, categories, labels and selected fragments |

## Mapping to the Paper

| Paper element | Source |
| --- | --- |
| Table I (query taxonomy) | `eval_dataset.json`, notebook Section 2.4 |
| Table II (retrieval effectiveness) | `results/summary.csv`, `results/bootstrap_ci.csv` |
| Table III (paired comparisons) | `results/pairwise_wilcoxon.csv` |
| Fig. 1 (nDCG@5 by category) | `results/03_category_by_config.pdf` |
| Fig. 2 (score separability) | `results/05_relevance_separability.pdf` |
| Mixed-effects and permutation tests | `results/metrics.json`, notebook Section 8 |
| Robustness checks | `results/sensitivity_*.csv`, `results/input_budget.csv`, notebook Sections 6 and 10 |

Any number in the paper can be traced to its identifier in `results/claims.json` and, from there, to the notebook cell that computed it.

## Reproducibility Notes

- **Deterministic reruns.** With the provided cache, every number is reproduced exactly. The random seed (42) and the number of bootstrap resamples (10,000) are fixed in the notebook.
- **Gemini embeddings.** The reported results were produced with `gemini-embedding-001` on 2026-09-04, with the output dimensionality reduced to 768. A managed model may change over time, so embeddings regenerated from the API on a later date may differ slightly from the cached ones.
- **Open-weight models.** Model revisions are recorded in `results/provenance.json`. `jina-code-embeddings-0.5b` requires `trust_remote_code=True`, which the notebook sets explicitly.
- **Hugging Face rate limits.** Unauthenticated downloads work but may be rate-limited. Setting an `HF_TOKEN` environment variable avoids this.
- **Corpus verification.** The notebook checks every chunk against `docs/study_repo_tree.txt`, the file listing of the pinned commit, before any score is computed.

## Limitations

The results describe a single demonstration repository and should not be generalized beyond it. The retrieval corpus was selected purposively, queries were written after the corpus was fixed, relevance was labeled by a single annotator, and 37 queries give limited statistical power, particularly within categories. Only the retrieval stage is evaluated. Section 11 of the notebook and Section VI of the paper discuss each limitation and what it prevents the study from concluding.

## Citation

If you use this code or benchmark, please cite the paper:

```bibtex
@inproceedings{<key>,
  title     = {Retrieval Strategies for Generative AI-Assisted Code Comprehension},
  author    = {<Author One> and <Author Two> and <Author Three>},
  booktitle = {<Conference Name>},
  year      = {2026}
}
```

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.

The study repository, `internet-banking-concept-microservices`, is also distributed under the MIT License by its authors. This repository includes only chunks derived from it for research purposes.
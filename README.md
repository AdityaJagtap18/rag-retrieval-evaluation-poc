# RAG Retrieval Evaluation & Optimisation

> A Forward Deployed Engineering (FDE) proof of concept. This repository contains both the **pre-transformation** (before) and **post-transformation** (after) states of the solution, so the impact of the work can be seen side by side.

---

## 1. Overview

| Item | Details |
|------|---------|
| **POC Name** | RAG Retrieval Evaluation & Optimisation |
| **Scenario / Customer** | Illustrative scenario: a policy-question assistant answering from a small corpus (two versions of one policy, one real claim, ten distractor claims) |
| **Owner** | adiraj2503@gmail.com |
| **Status** | Completed |
| **Program** | FDE Program · Sprint 1 · Day 13 Lab (Assignment 4) |

**One-line summary:** Replacing a deterministic, rule-based evaluation with a real model judge, and measuring cost per query, selects a retrieval setup that answers 9 of 12 questions correctly with about half the tokens, and shows how much of the earlier scoring was an artefact of the evaluator.

---

## 2. Problem Statement

A retrieval pipeline for policy questions had to be tuned across chunking, retrieval mode, re-ranking and query decomposition. The first round of evaluation relied on a simulated answer generator and rule-based metrics, so the numbers looked precise but did not reflect how a real model would answer.

- Evaluation scores came from a simulated generator, so faithfulness (about 0.5) and answer relevancy (0.51 on every configuration) did not separate one setup from another.
- The "best" configuration was chosen on recall and correctness alone, at a high token cost (701 tokens per query).
- The two policy versions and the "answer not in the corpus" questions exposed weaknesses that the aggregate averages hid.
- There was no measure of how noisy a model-based judge is, so any small gap could have been over-read.

---

## 3. Pre-Transformation (Before)

The starting state scored 48 configurations (4 chunkers × 3 retrieval modes × re-ranking × decomposition) on 12 questions with a **deterministic `mock` judge**: a simulated generator and rule-based metrics, retrieving the top 4 passages.

| Aspect | Current State |
|--------|---------------|
| Process | Full grid of 48 configurations scored by rules in `ragas_lab/metrics.py`; no real model in the loop |
| Model / Tooling | Simulated answer generator, rule-based RAGAS-style metrics, TF-IDF "vector" retriever |
| Cost | Free to evaluate; best configuration cost **$0.105 per 1k queries** (701 tokens/query) |
| Latency / Performance | Seconds to run the whole grid |
| Accuracy / Quality | Best result **9/12 correct**, precision 0.65, recall 0.96, faithfulness 0.59, relevancy 0.51 |
| Pain points | Scores identical on every run, so they say nothing about real-model behaviour; "correct" is optimistic because it checks strings anywhere in the joined context; relevancy flat at 0.51 across all 48 rows; negative questions scored 0.00 everywhere |

**Location in repo:** [Pre-Transformation/](Pre-Transformation/)

---

## 4. Post-Transformation (After)

The end state generates answers with a real Azure OpenAI deployment (`gpt-4.1-mini`) and scores them with the same model using the RAGAS decomposition. It evaluates 40 configurations with the top 5 passages, and measures the judge's own noise floor so that small gaps are reported as ties.

| Aspect | New State |
|--------|-----------|
| Process | Real generated answers scored by a model judge; answers and judgements cached so reruns are nearly free; run stops at a call budget |
| Model / Tooling | `gpt-4.1-mini` as generator and judge; `--variance` mode to measure judge noise; `ragas_lab/judge.py` added |
| Cost | Selected configuration costs **$0.050 per 1k queries** (332 tokens/query). Judging the full run cost about $0.18 (400 API calls) |
| Latency / Performance | About 885 s of model time for the run; cached re-runs are fast |
| Accuracy / Quality | Best result **9/12 correct**, precision 0.67, recall 0.90, faithfulness 1.00, relevancy 0.83 |
| Benefits | Scores reflect real answers; the structure-aware chunker reaches the best correctness at roughly half the tokens; a documented noise floor tells you which differences count |

**Location in repo:** [Post-Transformation/](Post-Transformation/)

---

## 5. Before vs. After

| Metric | Pre-Transformation | Post-Transformation | Change |
|--------|--------------------|---------------------|--------|
| Best configuration | fixed_512 + keyword | structure + keyword + decomposition | Different winner |
| Correct answers | 9/12 | 9/12 | No change |
| Tokens per query | 701 | 332 | −52.6% |
| Cost per 1k queries | $0.105 | $0.050 | −52.4% |
| Context precision | 0.65 | 0.67 | +0.02 |
| Context recall | 0.96 | 0.90 | −0.06 |
| Faithfulness | 0.59 | 1.00 | +0.41 (see caveat) |
| Answer relevancy | 0.51 | 0.83 | +0.32 (see caveat) |

Detailed results: [Pre-Transformation/out/ragas_report.md](Pre-Transformation/out/ragas_report.md) and [Post-Transformation/out/ragas_report.md](Post-Transformation/out/ragas_report.md).

**Judge noise floor (3 identical runs, `structure + hybrid`, temperature 0.0):** precision spread 0.000, recall 0.000, faithfulness 0.000, relevancy 0.002 ([judge_variance.md](Post-Transformation/out/judge_variance.md)). An earlier measurement in the pre-transformation folder showed larger spreads (precision 0.083, faithfulness 0.021), so gaps near those sizes should be treated as ties.

---

## 6. Approach

1. **Discover** – Reviewed the policy corpus, the 12-question evaluation set and the five question types (single fact, clause lookup, multi-part, version-sensitive, negative).
2. **Baseline** – Ran all 48 configurations with the deterministic judge and recorded the pre-transformation metrics.
3. **Evaluate** – Compared chunking, keyword/vector/hybrid retrieval, re-ranking and decomposition against correctness, recall, precision and token cost.
4. **Build** – Added a model-based judge (`judge.py`), a response cache, a call budget and a judge-variance mode.
5. **Measure** – Re-ran the grid with real generated answers and recorded post-transformation metrics and the judge's noise floor.
6. **Hand off** – Documented the results, the limitations, and a guide to reading the output (`Interpreting_RAGAS_Results.docx`).

### Decision Criteria

| Criterion | Weight | Why |
|---|---|---|
| Correctness (answers right out of 12) | High | The outcome users actually see |
| Context recall | High | If the answer was never retrieved, nothing downstream can fix it |
| Token cost per query | Medium | Drives the cost line at production volume |
| Context precision | Medium | Noise in the context costs tokens and can mislead the model |
| Faithfulness and relevancy | Medium | Show whether answers stay grounded and on topic |

---

## 7. Repository Structure

```
Assignment4/
├── README.md
├── README_TEMPLATE.md
├── Pre-Transformation/      # "Before" state: deterministic judge, 48 configurations
│   ├── run_lab.py
│   ├── ragas_lab/           # corpus, retrieval, metrics, judge
│   ├── out/                 # report and CSV results
│   └── Interpreting_RAGAS_Results.docx
└── Post-Transformation/     # "After" state: real model judge, 40 configurations
    ├── run_lab.py
    ├── ragas_lab/
    ├── out/                 # report, CSVs, judge cache, judge variance
    ├── .env.example
    └── Interpreting_RAGAS_Results.docx
```

---

## 8. Getting Started

### Prerequisites

- Python 3.10 or later (the offline judge uses the standard library only)
- For the model judge: an Azure OpenAI endpoint, API key and deployment name

### Setup

```powershell
cd Post-Transformation
py -3 -m venv venv
copy .env.example .env
# then fill in the endpoint, key and deployment in .env
```

### Run

```powershell
# Offline, deterministic grid (no key needed)
python run_lab.py --top-k 4

# Measure the judge's noise floor first
python run_lab.py --judge azure --variance 3

# Score the shortlist with the model judge
python run_lab.py --judge azure

# Inspect one question in detail
python run_lab.py --show Q09
```

---

## 9. Results & Learnings

- **Key result:** The structure-aware chunker with keyword retrieval and query decomposition reached 9/12 correct at 332 tokens per query, against 701 tokens for the earlier best, a drop of about 52% in tokens and cost for the same correctness.
- **What worked:** Using a real judge exposed differences the rule-based version could not show. Keyword retrieval did best on clause lookups, and structure-aware chunking used roughly half the tokens of fixed chunking.
- **What didn't:** Recall dropped slightly (0.96 to 0.90). Re-ranking helped some configurations and hurt others, and chunk sizes 512 and 1024 gave identical results because most documents are shorter than 512 tokens.
- **Recommendations:** Choose by correctness by question type and cost, not by the averages. Pin the judge deployment and version, and re-run the evaluation when the judge or the retrieval pipeline changes.

> Caveats: The corpus and the 12 questions are small and illustrative. Faithfulness and relevancy came from two different judges (rule-based before, model-based after), so those two gains are not a like-for-like comparison. Pre used the top 4 passages and Post used the top 5. The "vector" retriever is TF-IDF cosine, not learned embeddings. Under the model judge, answer relevancy is a simplified form of the RAGAS algorithm because it does not use an embedding model.

---

## 10. Next Steps

- [ ] Re-run the shortlist on a larger, real question set
- [ ] Replace the TF-IDF retriever with a learned embedding model and compare
- [ ] Project token cost at the real production query volume
- [ ] Emit context recall and precision per query as monitoring metrics, alerting on recall
- [ ] Define the review trigger for re-running this evaluation (judge upgrade, corpus change)

---

## 11. Contact

adiraj2503@gmail.com

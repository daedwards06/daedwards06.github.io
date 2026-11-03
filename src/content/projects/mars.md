---
title: "MARS — My Anime Recommendation System"
featured: true
date: "2026-10"
links:
  demo: "https://myanimerecommendationsystem-x6rqm6vqjmbr2ij8i8yk3b.streamlit.app"
  repo: "https://github.com/daedwards06/MyAnimeRecommendationSystem"
tags:
  - Recommendations
  - Evaluation
  - NLP
  - Streamlit
  - Python
---

MARS recommends anime from a 13,000-title catalog by blending collaborative filtering (item-kNN + matrix factorization) with synopsis embeddings and metadata, and every card says in plain words why it was picked. Pick a title you like, or load a demo taste profile, and it ranks the whole catalog in a few seconds.

![MARS demo: pick a seed title, read why a card was recommended, switch to a demo profile](/mars-demo.gif)

The [write-up](https://github.com/daedwards06/MyAnimeRecommendationSystem/blob/main/docs/WRITEUP.md) covers how it was built, what went wrong, and how each mistake was caught.

## Results

![Accuracy vs. coverage: NDCG@10 against Coverage@10 for ten model variants, with 95% bootstrap intervals on NDCG](/mars-accuracy-vs-coverage.png)

Offline ranking quality over the **full catalog** on one cohort (1,000 validation users, seed 42, K=10), with collaborative models fit on the train split only. Intervals are bootstrapped over users.

| Model | NDCG@10 | 95% CI | Coverage@10 | vs. popularity |
|:---|---:|:---:|---:|---:|
| Implicit ALS (offline only, not served) | 0.1890 | [0.169, 0.211] | 0.093 | +445% |
| **Hybrid (served)** | **0.0658** | [0.054, 0.080] | 0.103 | **+90%** |
| Item-kNN | 0.0632 | [0.051, 0.078] | 0.117 | +82% |
| Popularity baseline | 0.0347 | [0.026, 0.044] | 0.006 | (baseline) |
| MF (FunkSVD) | 0.0282 | [0.020, 0.038] | 0.067 | −19% |
| Content-only (TF-IDF) | 0.0211 | [0.014, 0.029] | 0.171 | −39% |

The served blend beats popularity (+0.031, CI [+0.017, +0.045]) but is statistically tied with item-kNN alone, and matrix factorization on its own is tied with popularity. Popularity shows everyone the same 0.6% of the catalog. Content similarity spreads wide but ranks poorly. The full ten-variant table, protocol and caveats are in the [evaluation report](https://github.com/daedwards06/MyAnimeRecommendationSystem/blob/main/reports/phase4_evaluation.md) and the [model card](https://github.com/daedwards06/MyAnimeRecommendationSystem/blob/main/docs/MODEL_CARD.md).

## How it works

1. **Offline training:** item-kNN and FunkSVD on 73K users × 13K anime ratings; TF-IDF, SVD and MiniLM sentence embeddings on synopses and metadata.
2. **Stage 0, candidates:** neural neighbors, metadata overlap and a popularity backfill.
3. **Stage 1, shortlist:** semantic admission gates plus type and episode filters.
4. **Stage 2, rerank:** collaborative score (95% item-kNN + 5% MF), genre overlap and synopsis similarity.
5. **Post-processing:** franchise cap, personalization blend, display filters, then cards with plain-language explanations in the Streamlit UI.

Content decides what is eligible, and collaborative filtering helps decide the order, because the two fail in opposite ways: CF knows taste but not new titles, and content knows every title but ranks poorly.

## What was hard / what I learned

- **An evaluation bug hid the best model.** Item-kNN first scored near zero because the offline harness masked "already seen" items using a matrix that contained the held-out item. Fixing it made kNN the strongest single model. Re-tuning the blend on users disjoint from the reporting cohort roughly doubled its NDCG (0.0324 → 0.0658).
- **Matrix factorization learned the wrong thing.** 19% of the ratings are a "watched, not rated" code that cleaning had turned into 0, so FunkSVD learned that much of what people watch is worthless. An implicit-feedback ALS trained on watched / not watched scores 0.1890 against FunkSVD's 0.0282.
- **Learning the weights only paid off with new signals.** A LightGBM LambdaRank reranker ties the tuned blend when it sees only the CF scores. It beats it (0.0931 over three seeds, every paired CI above zero) once it also gets content and metadata affinities. It ships behind a flag, off by default.
- **The blend's edge over item-kNN is an undesigned fallback.** By user segment, the blend beats kNN only for the least active quarter of users and ties it on every user kNN can score. kNN returns no list for 113 of the 1,000 cohort users, and the blend's MF and popularity terms quietly rank for them.
- **There are no real timestamps.** The rating snapshot has no interaction dates, so the "temporal" split is a row-order heuristic and is reported as one.

## Tech Stack

- **Python, pandas, NumPy, SciPy:** data pipeline, plus FunkSVD (SGD) and implicit ALS implemented from scratch
- **scikit-learn:** item-kNN, TF-IDF, truncated SVD
- **sentence-transformers (MiniLM):** synopsis embeddings, built offline
- **LightGBM:** learned LambdaRank reranker
- **Streamlit:** UI and deployment on Streamlit Community Cloud
- **Jikan API / Kaggle MyAnimeList snapshot:** catalog metadata and 7.8M ratings
- **pytest + GitHub Actions + Codecov:** ~290 tests in CI

## Links

- [Live demo](https://myanimerecommendationsystem-x6rqm6vqjmbr2ij8i8yk3b.streamlit.app). If it's asleep, the first load after waking takes longer while it reloads its models.
- [Repository](https://github.com/daedwards06/MyAnimeRecommendationSystem)
- [Write-up](https://github.com/daedwards06/MyAnimeRecommendationSystem/blob/main/docs/WRITEUP.md): what I built, what went wrong, what I learned
- [Model card](https://github.com/daedwards06/MyAnimeRecommendationSystem/blob/main/docs/MODEL_CARD.md) and [ablation table](https://github.com/daedwards06/MyAnimeRecommendationSystem/blob/main/reports/phase4_ablation.md)

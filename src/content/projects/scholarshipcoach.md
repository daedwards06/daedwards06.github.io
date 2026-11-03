---
title: "ScholarshipCoach — Scholarship Decision Support"
featured: false
date: "2026-11"
links:
  repo: "https://github.com/daedwards06/ScholarshipCoach"
tags:
  - Ranking
  - Evaluation
  - Data Quality
  - Streamlit
  - Python
---

ScholarshipCoach is a scholarship and college-planning tool built for one real family: a North Carolina high school student who wants to study computer science, and her parents. It answers four questions: which awards she can actually apply to, when each is due, what each one needs written, and how outside money compares to the institutional and state aid that is usually the larger number.

It started as a four-stage recommender. A product review turned up the real problem: the catalog was the bottleneck, not the ranker. Most of the work since then has been making the data trustworthy and the app usable by people who will never read a metric.

![Find Scholarships: ranked award cards with amount, deadline, and an unverified-eligibility flag](/scholarshipcoach-find.png)

There is no public demo. The app holds a real student's profile, essays and application history, so it runs on a small cloud server that is reachable only over a private Tailscale network. The repository ships a fictional demo profile, so a fresh clone runs end to end. Every screenshot here is from that demo profile.

## Results

Offline, snapshot-based evaluation on the current catalog (133 records), across 9 golden student profiles, K=10:

| Configuration | NDCG@10 (hybrid labels) | NDCG@10 (no-similarity labels) | Coverage@10 | Unique awards |
|:---|---:|---:|---:|---:|
| Baseline weights | 0.720 | 0.545 | 0.389 | 35 |
| **Tuned weights** | **0.803** | **0.624** | **0.456** | **41** |

The relevance labels are proxies, built partly from the same keyword and similarity signals the ranker scores on, so tuning against them is partly self-fulfilling. The second column is the check on that: the same ranking re-scored under a label rule with no similarity term. The tuned weights win under both rules by about the same margin, so the gain is not only the tuner learning the shape of one label rule.

These numbers are lower than an earlier 0.831. The cause is the catalog, not the code: the current code still scores 0.831 on the old snapshot. Records that were corrected, split or dropped moved the number, and the case study reports the catalog that exists. A 44-pair human-labeled set once scored 0.98 for a CS profile against 0.71 for a nursing profile, which showed the catalog and tuning skewed technical. It was labeled on an older catalog whose IDs no longer join, so there is no current human-judged headline, and the repo says so rather than re-keying old labels to make a number reappear.

## How it works

1. **Catalog:** hand-curated JSON records plus a CC BY 4.0 structured feed. Feeds, URL prefill and yearly re-verification write *proposals* into an inbox with a field-level diff. A person confirms each one, and it is schema-validated on the way in.
2. **Stage 1, eligibility:** filters on major, state, county, grade, GPA, test scores, citizenship, need, first-gen, military, membership and more, with 20 reason codes, one per way an award can be ruled out. A blank field passes but is flagged, so a card says "confirm you meet: grade level" rather than claiming a match.
3. **Timeline:** every surviving award lands in *now*, *next cycle*, *senior year*, *needs a date* or *expired*, with the next cycle projected for recurring awards.
4. **Stage 2, semantic fit:** MiniLM sentence embeddings (or TF-IDF), keyword overlap, effort penalty and award size.
5. **Stage 3, rerank:** a weighted composite with a deadline-urgency boost. Weights come from a grid search with relevance, blended and Pareto objectives.

Everything after ingest is a pure function of catalog, profile and date, so every run is reproducible and every result can be explained to the person acting on it.

![Student mode at phone width: My Path toward a first-choice school, and What If tied to those goals](/scholarshipcoach-phone.png)

Around the ranker sits the family product. **Student mode** is her path to college: a checklist of courses, tests and GPA targets for her first-choice school, shared to-dos, summer programs, essays, and a What If page that shows how a GPA or test score change moves her toward those goals. **Parent mode** is the money view: Find, Applications, Colleges & Money and the Timeline. Essays stay private until she shares them.

## What was hard / what I learned

- **The ranker was polishing an empty catalog.** A review found it had collapsed from 166 records to 35 without any alarm, because the scraper reported `succeeded` on zero results. 22 of the 35 were closed, and the major, GPA-minimum and education-level fields were empty in every snapshot, so Stage 1 was really a state-plus-deadline filter. I stopped ranking work and fixed the data first. Now a source that returns nothing where it returned something fails the run visibly.
- **A green checklist hid broken seams.** After the whole product plan was checked off, a second review found the in-app "Rebuild snapshot" button would quietly delete every feed record, and the trust rule was documented but never enforced. Since then every "done" claim gets re-measured against the actual data, not just the tests.
- **Automation proposes, a person confirms.** Automated extraction is wrong often enough that a wrong record costs the family an application. Nothing automated writes to the catalog. Records only count as eligible once a person confirms them or they come from a trusted structured feed. I chose not to use an LLM for extraction for the same reason: deterministic regex and feeds do the job, and an `Extractor` interface leaves room for one later.
- **Good engineering can still not belong in front of a family.** A calibrated win-probability model was trained on labels from a synthetic generator, because no real outcomes exist yet. It provably recovers that generator's coefficients, but showing a student a number that looks like her odds would be worse than showing nothing. It is out of every family-facing view and kept behind an operator toggle until real outcomes replace the synthetic labels.
- **The user was a different person than the one I built for.** The app was designed for a senior applying for awards. The real student is a sophomore, and most awards are closed to her for two more years. Walking every page at phone width turned up "Apply now" on senior-only awards and three taps to change pages. The redesign reorganized the app around her near-term path, with scholarships moved into the parents' view.
- **Privacy shapes the architecture.** The repo is public, and her profile, essays, applications and target school live only in a git-ignored private store. Catalog edits made on the server sync to the public repo nightly, so a fail-closed guard checks each change against a private list of names and holds the commit if any appear.

## Tech Stack

- **Python, pandas, PyArrow:** ingest, versioned Parquet snapshots, change tracking
- **sentence-transformers (MiniLM), scikit-learn:** semantic scoring, TF-IDF, logistic win model with Platt calibration
- **Streamlit:** student and parent modes, a contrast-tested light and dark theme, checked at 400px phone width
- **SQLite:** applications, essays, to-dos, college costs and outcomes, with numbered SQL migrations
- **Playwright:** a screenshot and journey harness that walks each user journey and counts taps against a budget
- **Linux VPS + Tailscale + restic:** private hosting, nightly encrypted backups with a tested restore, deploys gated on green CI
- **pytest + ruff + mypy + GitHub Actions:** 1,027 tests, a coverage gate, and catalog schema validation in CI

## Links

- [Repository](https://github.com/daedwards06/ScholarshipCoach)
- [Decision log](https://github.com/daedwards06/ScholarshipCoach/blob/main/docs/decisions.md): each pivot, with the evidence behind it
- [Evaluation methodology](https://github.com/daedwards06/ScholarshipCoach/blob/main/docs/evaluation.md) and [system design](https://github.com/daedwards06/ScholarshipCoach/blob/main/docs/system_design.md)

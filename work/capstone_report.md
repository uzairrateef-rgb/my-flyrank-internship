# Capstone Report — Refresh / Content Opportunity Scoring

- **Author:** Uzair Rateef
- **Lane:** Refresh / Content Opportunity Scoring
- **Repository:** https://github.com/uzairrateef-rgb/my-flyrank-internship
- **Date:** 2026-10-12
- **Status:** Full-warehouse results generated; metrics and figures committed. Final submission checks are author metadata, deployed-page verification, and the live URL receipt.

## 0. Abstract

This study asks whether monthly search-performance signals can help rank content items likely to see a meaningful decline in the following month. It aggregates the gated FlyRank internship warehouse at client-content-month grain and defines the target as next-month impressions at least 20% below the feature month, with a minimum of 100 impressions and valid Google Search Console coverage. It compares a transparent visibility-plus-momentum baseline with Logistic Regression and Random Forest, using February–April 2026 for model selection and May features → June outcome as the final holdout. On the final May→June 2026 holdout, Random Forest achieved Precision@20 of 0.850 but lagged the transparent baseline at Precision@50 (0.720 versus 0.860) and Precision@100 (0.750 versus 0.880); ROC AUC was 0.541. The decline base rate also shifted from 46.4% on validation to 68.6% on the final holdout, so the result supports at most a narrow top-20 triage hypothesis, not a robust general ranking advantage. The intended output is a ranked editorial review queue for human decision support, not a prediction of Google's algorithm or a causal claim that refreshing content improves traffic.

## 1. Introduction / Problem statement

Editorial teams have more pages to inspect than time available. Prioritizing only by age or current traffic is easy to explain, but it may miss a page whose momentum is deteriorating or elevate a page with high traffic but little evidence of risk.

**Research question:** Can signals available at the end of a month rank pages likely to experience at least a 20% decline in Google Search Console impressions during the following calendar month better than a transparent visibility-plus-momentum rule?

The supported decision is which pages an editor should inspect first during a bounded review cycle. The unit of analysis is a client-content item at a given feature month. A false positive consumes editorial review time; a false negative may leave a meaningful decline unreviewed. ML is useful only if the learned rank improves over the readable baseline on the same future-month test cohort.

This research does not model Google's ranking algorithm and does not measure whether refreshing content causes traffic improvement.

## 2. Data

### Source and grain

The primary capstone notebook uses the gated `FlyRank/internship-warehouse` release on Hugging Face. The source table is `fact_content_daily_performance`, whose daily records are aggregated to one row per pseudonymized client, content item, and calendar month. It scans monthly partitions from May 2025 through June 2026. Candidate feature months are June 2025 through May 2026; the next calendar month is the outcome window.

### Eligibility and label

Only records where `gsc_data_available IS TRUE` are used. False or null availability flags are not treated as zero performance. A training example must have valid GSC records for the previous, feature, and following months, and at least 100 impressions in the feature month. The target is positive when next-month impressions are at most 80% of current-month impressions.

### Exclusions and public-safety contract

The model never uses client/content identifiers as predictors, outcome-month impressions, `trend_direction`, `trend_pct`, raw queries, client names, domains, page URLs, page titles, credentials, or private exports. Pseudonymous IDs are used only to join the same entity across months. The query table, GA4 signals, and AI-referral measures are excluded from this model to keep the target and predictors focused on GSC performance and avoid sparse or unrelated signals.

The eligibility rule favors content that is observed continuously and excludes items without three adjacent months of valid GSC coverage. The resulting population may overrepresent persistent, well-instrumented pages; this is explicitly a limitation.

## 3. Methodology

### Target definition

`decline_20pct_next_month = 1` when next-month impressions are less than or equal to 80% of feature-month impressions. This is a defined operational proxy for potential review need, not an editorial ground-truth label.

### Features

The feature set uses previous- and current-month GSC impressions, clicks, impression-weighted average position, days with impressions, CTR calculated within those months, and month-over-month changes computed only from the previous/current windows. It excludes any value from the outcome month and excludes identifiers and trend-label fields. This means the model can be scored at the end of the feature month before the next-month label is observed.

### Baseline

The transparent baseline combines two percentile-ranked terms:

- 50% visibility: percentile rank of log current-month impressions.
- 50% momentum risk: percentile rank of negative month-over-month impression change, so worse recent momentum receives a higher score.

### Candidate models

Logistic Regression provides a relatively simple classifier; Random Forest can model nonlinear feature interactions. Model selection is based on validation Precision@50, with average precision and ROC AUC as tie-breakers. The notebook uses a fixed seed of 42 and caps fit samples at 250,000 examples for practical Colab runtime; validation and final test cohorts are not sampled.

### Validation design and leakage checks

- **Training:** feature months June 2025–January 2026, with outcomes through February 2026.
- **Validation/model selection:** feature months February–April 2026, with outcomes through May 2026.
- **Final test:** May 2026 features predicting June 2026 outcomes.
- The model is refit on pre-holdout examples only after selection. The June outcome is not used for model selection.
- Baseline and selected model are evaluated on the same final test cohort.
- Assertions check that IDs, target fields, trend labels, and outcome-month data are not included in the feature list.

This is a time-aware test for future-month ranking within an existing portfolio. The same content or client may appear in previous training months and subsequent test months, as it would in a real monitored portfolio. It does not establish performance on unseen clients.

## 4. Results (model versus baseline)

### Primary full-warehouse result — generated by the capstone notebook

The final temporal holdout contains **101,740 eligible content-month examples**, with May 2026 features predicting June 2026 outcomes. The June decline base rate was **68.6%**. Random Forest was selected over Logistic Regression on validation Precision@50 (0.440 versus 0.400), but the final test shows a mixed ranking result:

| Final holdout metric | Random Forest | Transparent baseline |
|---|---:|---:|
| Precision@20 | **0.850** | 0.700 |
| Precision@50 | 0.720 | **0.860** |
| Precision@100 | 0.750 | **0.880** |
| ROC AUC | 0.541 | 0.505 |
| Average precision | 0.713 | 0.710 |
| Decline base rate | 0.686 | 0.686 |

At the very top of the queue, the model found 17 next-month declines among its top 20 items, compared with 14 for the baseline. At the top 50 and top 100, however, the baseline was stronger: 43 versus 36 declines at 50, and 88 versus 75 at 100. ROC AUC is only modestly above chance, and average precision is nearly tied.

The validation decline base rate was 46.4%, versus 68.6% in the final holdout. This large prevalence shift is a warning that ranking performance is sensitive to the time period and cohort. These results should not be described as an overall model win. For a practical editorial workflow, the transparent baseline is currently the stronger default at K=50 and K=100; the Random Forest top-20 result is a narrow finding that should be re-tested on additional future months before operational use. A score is only a request for human review, not an instruction to refresh a page automatically.

The probability-threshold error counts at 0.50 were 34,449 true positives, 14,467 false positives, 35,392 false negatives, and 17,432 true negatives. This thresholded view is supplementary; Precision@K is the primary metric for the ranked review-queue decision.

### Prior starter-snapshot benchmark — audit context only

The repository's existing `outputs/model_report.md` contains an earlier experiment on the bundled 30,000-row starter CSV. It reported a declining-label base rate of 54.2% and the following results:

| Approach | ROC AUC | Average precision | Precision@50 | Recall | F1 |
|---|---:|---:|---:|---:|---:|
| Rule baseline | 0.627 | 0.468 | 0.240 | — | — |
| Decision tree | 0.742 | 0.575 | 0.540 | 0.716 | 0.634 |
| Logistic Regression | 0.700 | 0.522 | 0.400 | 0.567 | 0.566 |
| Random Forest | 0.750 | 0.618 | 0.740 | 0.744 | 0.640 |

These figures are historical pipeline output on a different dataset and labeling setup, not directly comparable with the full-warehouse time-aware result. The starter model used trailing-window aggregates that overlap the trend window used to build its target, so these metrics are retained as audit context and must not be used as the primary headline result.
 
## 5. Limitations & honest framing

1. **Proxy label:** a 20% monthly impression drop is a defined target, not a human-confirmed reason to refresh a page.
2. **Single final outcome month:** May→June 2026 is one temporal holdout, not proof of robustness across all seasons or market environments.
3. **Selection by data coverage:** requiring GSC coverage in three adjacent months and a 100-impression feature-month floor excludes new, removed, and intermittently tracked pages.
4. **Seasonality and demand:** monthly declines can reflect seasonal cycles, changes in search demand, migrations, indexing problems, SERP changes, or measurement changes rather than stale content.
5. **Portfolio scope:** temporal evaluation permits pages/clients to occur in both earlier training months and future test months. It does not measure generalization to unseen clients.
6. **Aggregation:** monthly totals hide daily volatility and short-lived changes.
7. **No causal intervention:** this study does not randomize refreshes or construct a matched intervention comparison. It cannot say that refreshes cause traffic increases.
8. **Interpretability:** feature importance shows how a fitted model used inputs, not whether changing a feature will change search performance.

Use careful terms: *observed*, *measured*, *associated*, *directional*, and *decision-support*. Avoid *proves*, *causes*, *will increase*, and claims about predicting Google's algorithm.

## 6. Ranked recommendations

1. **Inspect high-visibility pages with worsening momentum.** Confirm the decline is not seasonal, caused by a tracking gap, or explained by market-wide demand changes.
2. **Review CTR in context.** Compare only adequate-volume pages with similar intent and observed position; do not diagnose a snippet problem from a low-volume sample.
3. **Investigate position deterioration.** Verify indexing, SERP changes, intent mismatch, internal linking, and competitor changes before rewriting.
4. **Verify factual freshness and task completion.** Update facts, examples, screenshots, references, and internal links only when they no longer serve the user well.
5. **Monitor ambiguous or low-signal pages.** If signals disagree or measurements are sparse, collect more evidence rather than forcing an edit.

A human editor must verify user intent, factual accuracy, seasonality, brand voice, indexing, and internal links. Never automatically delete, redirect, bulk-publish, or keyword-stuff content based on a model score.

## 7. Artifacts the paper embeds

The successful run generated these public-safe artifacts inside `capstone_public_artifacts.zip`; they must be committed to the matching repository paths before publication:

- `work/outputs/capstone_metrics.json` — data windows, selected model, validation and holdout metrics, base rate, aggregate error counts, and feature importance.
- `docs/paper-assets/decline_rate_by_month.svg` — observed future-decline rate by feature month.
- `docs/paper-assets/model_vs_baseline.svg` — Precision@K for model and baseline on the same holdout, with the base rate.
- `docs/paper-assets/feature_importance.svg` — fitted model's feature importance, explicitly non-causal.

The paper HTML displays the full-warehouse metrics and references the three generated SVG figures. The metrics JSON and all three SVG files are now committed at the paths above; verify the deployed page renders each chart before final submission. Do not embed row-level queue previews, pseudonymous IDs, raw CSVs, or client-identifying details.

## 8. Reproducibility

- **Repository:** https://github.com/uzairrateef-rgb/my-flyrank-internship
- **Capstone notebook:** https://github.com/uzairrateef-rgb/my-flyrank-internship/blob/main/work/notebooks/capstone.ipynb
- **Baseline notebook:** https://github.com/uzairrateef-rgb/my-flyrank-internship/blob/main/work/notebooks/w04_baseline_score.ipynb
- **Model notebook:** https://github.com/uzairrateef-rgb/my-flyrank-internship/blob/main/work/notebooks/w05_model.ipynb
- **Validation audit:** https://github.com/uzairrateef-rgb/my-flyrank-internship/blob/main/work/notebooks/w06_validation_audit.ipynb
- **Action playbook:** https://github.com/uzairrateef-rgb/my-flyrank-internship/blob/main/work/notebooks/w07_action_playbook.ipynb

Run from a fresh Google Colab runtime with a read-only Hugging Face token stored in Colab Secrets as `HF_TOKEN`, then use Runtime → Run all. A lightweight March 2026 metadata probe checks the source, after which DuckDB aggregates the monthly Parquet partitions. The aggregate is cached locally at `data/processed/capstone_monthly_gsc_v1.parquet`, a path excluded by `.gitignore`; it contains pseudonymous keys and must never be committed. Subsequent runs in the same workspace reuse that cache rather than rescanning the remote warehouse. The notebook writes an aggregate metrics receipt and SVG charts, then bundles only those public-safe artifacts into `work/outputs/capstone_public_artifacts.zip` for download from Colab. Extract that bundle and commit the JSON and SVG files at their original paths; do not commit the ZIP or local cache. Compare the receipt against the displayed table before publication. Seed: 42.

The metrics JSON and generated figures from the completed run are committed alongside the notebook. A later rerun should replace them only with outputs generated by that exact rerun.

## 9. Acknowledgments & data credit

Built on the [FlyRank ML Internship dataset](https://flyrank.ai).

---

## ML-12 closing deliverables

### Five-minute demo outline

- **0:00–0:45 — Decision:** explain content review prioritization and the cost of wrong calls.
- **0:45–1:30 — Data:** describe the warehouse table, monthly grain, eligibility, and public-safety exclusions.
- **1:30–2:30 — Method:** show the forward label, baseline, candidate models, validation windows, and final holdout.
- **2:30–3:30 — Results:** show the baseline/model comparison next to the test base rate.
- **3:30–4:30 — Action playbook:** demonstrate how editors use the ranking without automatic publishing.
- **4:30–5:00 — Limitations:** discuss proxy labels, one outcome month, coverage bias, seasonality, and the absence of causal evidence.

### Social-post cut (publish after the run)

I built a time-aware content refresh opportunity-scoring workflow using the FlyRank ML Internship warehouse. The capstone compares a transparent visibility-plus-momentum baseline with supervised models on a future-month holdout, reporting Precision@K alongside the decline base rate. The result is a human-reviewed content audit aid—not a promise of ranking gains or evidence that refreshing pages causes traffic growth.

### Employer-facing summary (three sentences)

I built a reproducible content refresh prioritization workflow on FlyRank's gated, pseudonymized daily search-performance warehouse, with month-aligned features and a defined next-month decline proxy. I compared Logistic Regression and Random Forest with a transparent baseline using validation months and a final May→June 2026 holdout. I documented leakage controls, public-data safeguards, the reasoned editorial playbook, and the limits imposed by seasonality, data coverage, and observational evidence.

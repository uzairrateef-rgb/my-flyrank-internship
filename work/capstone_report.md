# Capstone Report — Refresh / Content Opportunity Scoring

- **Author:** Uzair Rateef
- **Lane:** Refresh / Content Opportunity Scoring
- **Repo:** https://github.com/uzairrateef-rgb/my-flyrank-internship
- **Date:** 2026-10-12
- **Status:** Research draft — final claims require a fresh, leakage-audited rerun and matched evaluation.

## 0. Abstract

This study asks whether observable content, search-visibility, freshness, and engagement signals can help editors prioritize pages for refresh review. It begins with the repository’s anonymized 30,000-row content snapshot, where the target proxy marks a page as declining when its derived trend category is `down`, and compares a transparent rule-based score with learned ranking scores. The repository’s current report records a client-holdout evaluation and reports Random Forest Precision@50 of 0.740 against 0.240 for the rule baseline, but conflicting notebook outputs and feature/label time-window overlap mean those values are provisional until the pipeline is rerun and audited. The intended output is a ranked queue with reason codes such as declining demand, stale-but-visible content, low CTR at observed visibility, and possible engagement gaps—not a guarantee that a refresh will improve performance. The work is therefore framed as decision support: it helps decide what to inspect first, while an editor verifies seasonality, intent, data coverage, and content quality before changing a page.

## 1. Introduction / Problem statement

A content team can have thousands of eligible pages but limited time to review them. Reviewing only the oldest pages is easy to explain, but age alone can miss a newer page with weakening visibility or prioritize a stale page that still performs well. This capstone asks:

**Can a transparent refresh score plus a supervised ranking model prioritize pages whose observed trend proxy is declining, while outperforming a simple rule baseline on a held-out client split?**

The decision supported is which pages an editor should inspect first in a bounded review cycle (for example, the first 20 or 50 rows in a queue). The unit of analysis in the starter-snapshot pipeline is one pseudonymized content item. A false positive costs editorial time and may lead to an unnecessary edit; a false negative leaves a potentially deteriorating page unreviewed. ML earns a place only if it improves top-of-queue quality over a readable baseline and generalizes to clients held out of training.

This is not an attempt to reverse-engineer Google's ranking system. The label is a defined proxy in a fixed observational snapshot, not a measured outcome of an editorial refresh.

## 2. Data

### Dataset and grain

The repository includes `data/raw/content_refresh_anonymized.csv`, documented as a 30,000-row × 44-column anonymized starter snapshot covering 32 pseudonymized clients. Each row represents one content item with trailing-90-day search and engagement aggregates, content metadata, and two 30-day comparison windows. The accompanying data dictionary describes the rate columns as percentage points on a 0–100 display scale (for example, `ctr = 0.76` means 0.76%, not 76%).

The feature-preparation script requires `content_id`, `client_id`, `impressions_90d`, `sessions_90d`, `content_age_days`, and `trend_direction`; it converts numeric columns, handles missing values, filters to rows with positive impressions and content age of at least 90 days, and deduplicates by `content_id`. The checked-in report currently describes 30,000 scored rows; this should be reconfirmed from a clean run before publication.

### Public-safety exclusions

No client names, domains, URLs, page titles, keywords, raw queries, or credentials are used in the public paper. Pseudonymous content/client IDs are for joins and group-split validation only and must not be model inputs or published in example rows. The model feature lists are in `scripts/ml_utils.py`; they exclude the target source fields `trend_direction` and `trend_pct`. Provider/model identifiers and other potentially policy-irrelevant fields should remain excluded unless a clear, justified use is documented.

### Time-window caveat

This starter export is a one-time snapshot, not a sequence of archived prediction snapshots. The target is derived from the comparison-window trend category; therefore the current data alone cannot demonstrate genuine future-window forecasting or the causal effect of refreshing. The repository also documents a separate gated warehouse release, but the reported baseline/model metrics here are explicitly from the starter CSV pipeline, not from the full warehouse. Do not describe this study as a full-warehouse experiment unless the notebook is rebuilt to use that release and records its table names and date windows.

## 3. Methodology

### Target and task

The target is `is_declining_label = 1` when `trend_direction == "down"`, otherwise 0. Per the data dictionary, `down` is a rule-derived category based on the difference between recent and prior 30-day impressions; it is a proxy for review need, not an independently observed business outcome. The starter data reports 16,262 positive rows out of 30,000 (54.2%). A majority-class or random-ranking reference should always be displayed beside Precision@K.

### Features

The repository model uses numeric features for search volume, competition, CPC, word/character counts, log-transformed traffic totals, days with impressions/sessions, content age, days since update, CTR, average position, engagement rate, scroll rate, and AI-traffic percentage; categorical inputs include competition level, content type, main intent, age/freshness tiers, word-count tier, impression tier, and position tier. The final feature matrix is one-hot encoded for categoricals. IDs are excluded as model features.

**Important audit gate:** although `trend_direction` and `trend_pct` are not explicit model inputs, several model inputs (including 90-day totals/rates and time-window metrics) overlap with or are derived from periods used to construct the label. That can produce target-window leakage. The reported client-holdout split prevents client overlap but does not by itself solve temporal leakage. Before the paper claims a validated model, run a clean feature audit and compare performance after removing variables that encode the label window; preferably define features strictly before a future label window.

### Baseline and models

The transparent baseline combines normalized visibility, freshness risk, position opportunity, and content-depth gap:

- 40% visibility score
- 30% freshness-risk score
- 25% position-opportunity score
- 5% content-depth-gap score

Its reason codes flag transparent review conditions such as stale-but-visible content, declining-with-demand, thin visible content, low CTR among visible pages, and possible engagement gaps. The model script trains logistic regression, a shallow decision tree, and a Random Forest; it selects the model by Precision@50, then average precision and ROC AUC as tie-breakers.

### Validation and leakage controls

The script attempts an approximately 80/20 client-holdout split using a fixed random seed of 42, with a row-stratified fallback if the holdout lacks both target classes. The repository report states that the selected split was client-holdout. Report train/test row counts and client counts from the actual clean run; assert that no client appears in both partitions. Evaluate all models and the baseline on the exact same held-out rows and score metric.

The present checked-in results should not be considered final because (1) notebook experiments contain inconsistent feature definitions and metrics, (2) one action-playbook cell fits and ranks on the same rows it labels, (3) the primary target is built from a trend category rather than a later observed outcome, and (4) overlapping windows remain a leakage concern. The final notebook must rerun top-to-bottom and save the exact metrics JSON produced by that run.

## 4. Results (vs baseline)

### Current repository report — provisional, not publication-ready

The currently committed `outputs/model_report.md` states the following metrics for the 30,000-row starter-snapshot pipeline and labels its split strategy `client_holdout`. These are transcribed for audit visibility only; they must not be presented as the final capstone result until the clean pipeline reproduces them and the target-window leakage audit is passed.

| Approach | ROC AUC | Average precision | Precision@50 | Recall | F1 |
|---|---:|---:|---:|---:|---:|
| Transparent rule baseline | 0.627 | 0.468 | 0.240 | — | — |
| Decision tree | 0.742 | 0.575 | 0.540 | 0.716 | 0.634 |
| Logistic regression | 0.700 | 0.522 | 0.400 | 0.567 | 0.566 |
| Random Forest | 0.750 | 0.618 | 0.740 | 0.744 | 0.640 |

The reported label prevalence is 54.2%; under random selection, the expected Precision@50 is approximately the base rate, subject to sampling variation. The provisional report's Random Forest Precision@50 of 0.740 is 0.500 absolute points above the rule baseline of 0.240, but this apparent lift is not a claim of production performance because feature/label window overlap and inconsistent notebook experiments remain unresolved.

### What the result can and cannot say

If the reported comparison is reproduced on the same client-held-out rows after the leakage audit, it would suggest that the multivariable score can place more trend-proxy-positive pages near the top of the review queue than the current hand-written rule. It would not show that refreshes cause traffic gains, explain Google's algorithm, or guarantee that pages ranked first will benefit from editing.

**Required before release:** replace this provisional table with the exact metrics from one fresh, reproducible run; report the majority/base rate next to Precision@K; add at least Precision@20 and Precision@50; include ROC AUC/average precision; show the number of held-out clients and test rows; and state results after removing any features that share the label window. If the model does not beat the baseline once features are temporally aligned, report that negative result and use the rule as the current recommended approach.

## 5. Limitations & honest framing

1. **Proxy label:** decline is defined from a trend category; it is not a labeled editorial decision, confirmed content decay, or evidence that a refresh will work.
2. **Snapshot design:** the starter CSV contains a trailing 90-day summary and comparison windows rather than a true rolling deployment dataset with historical prediction cutoffs. It cannot establish future forecasting without a properly aligned time-based label.
3. **Potential window leakage:** excluding the named label columns is necessary but not sufficient if other features overlap the period used to define the label.
4. **Client generalization:** a client holdout is more demanding than a row split, but only 32 clients are represented in the starter slice. Results may be sensitive to which clients are held out and should ideally be repeated across grouped folds.
5. **Non-random missingness:** missing keyword/context fields can be associated with content type or tracking availability. Filling every missing value with zero can imply a real zero or silently encode missingness; include missingness indicators or assess the missingness pattern.
6. **Seasonality and external context:** seasonal cycles, site changes, migrations, changes in measurement coverage, and broader demand shifts can resemble page-level deterioration.
7. **Feature importance is not causality:** importance indicates how a fitted model used inputs; it does not mean changing a feature will change search performance.
8. **No intervention evidence:** the study did not randomize page refreshes or compare refreshed and matched untouched pages. No causal impact claim is warranted.

Language throughout the public paper should use *observed*, *measured*, *associated with*, *directional*, and *decision-support*. Avoid “proves,” “causes,” “will increase,” and “predicts Google's algorithm.”

## 6. Ranked recommendations

This playbook describes how to use the queue, not specific client/page recommendations. Published examples must be aggregate and must not include pseudonymous row IDs or raw queue records.

1. **Review declining pages that still have meaningful visibility.** Prioritize pages with a consistent negative trend proxy and a non-trivial impression volume. Verify that the dip is not seasonal, caused by tracking gaps, or attributable to a market-wide demand change. Reason code: `declining_with_demand`.
2. **Inspect low-CTR pages with usable impressions and known positions.** Compare CTR with an adequately sized, comparable position/content-intent group before changing titles or snippets. Do not treat very small impression counts as strong evidence. Reason code: `low_ctr_visible_page` / `ctr_review_candidate`.
3. **Review stale pages that still attract search demand.** Check whether facts, product details, screenshots, citations, internal links, and search intent are still current. Age alone is not sufficient reason to rewrite. Reason code: `stale_visible_page`.
4. **Check engagement signals only when measurement is adequate.** Inspect pages with sufficient session volume and consistently weak engagement/scroll signals; review intent match, content structure, readability, and page experience. Reason code: `low_engagement_visible_page`.
5. **Expand thin content only when it fails the user task.** Word count is a diagnostic cue, not a target. Add useful coverage or examples only when the article leaves an evident intent gap. Reason code: `thin_visible_page`.
6. **Monitor ambiguous or low-volume rows.** When traffic is sparse, tracking availability is weak, or trend signals disagree, request more observation instead of editing automatically. Reason code: `general_refresh_review` / `monitor`.

A human editor should verify search intent, accuracy, seasonality, brand voice, technical indexing, internal links, and measurement integrity before implementing a change. Never bulk-publish edits, delete or redirect pages solely from a model score, or keyword-stuff content. After any refresh, monitor pre-defined post-change windows; an intervention-aware experiment or a carefully matched comparison is needed to estimate impact.

## 7. Artifacts the paper embeds

The repository currently contains these useful artifacts:

- `outputs/charts/action_mix.svg` — distribution of suggested actions.
- `outputs/charts/confidence_mix.svg` — distribution of queue confidence tiers.
- `outputs/charts/top_reason_codes.svg` — the most frequent reason codes.
- `outputs/charts/top_feature_importance.svg` — top model features.
- `outputs/charts/trend_distribution.svg` — target-proxy distribution.
- `outputs/model_report.md` — pipeline report and comparison table.

All visuals and counts must be regenerated together from the same clean run. Each chart caption should state the takeaway and sample size. Feature-importance charts should clearly say that importance is associational, not causal. Do not embed row-level queue previews, pseudonymous content/client IDs, raw CSVs, or any private identifiers in the deployed page.

## 8. Reproducibility

Repository: https://github.com/uzairrateef-rgb/my-flyrank-internship

Relevant implementation and research artifacts:

- [Capstone notebook](https://github.com/uzairrateef-rgb/my-flyrank-internship/blob/main/work/notebooks/capstone.ipynb)
- [Week 4 baseline notebook](https://github.com/uzairrateef-rgb/my-flyrank-internship/blob/main/work/notebooks/w04_baseline_score.ipynb)
- [Week 5 model notebook](https://github.com/uzairrateef-rgb/my-flyrank-internship/blob/main/work/notebooks/w05_model.ipynb)
- [Week 6 validation audit notebook](https://github.com/uzairrateef-rgb/my-flyrank-internship/blob/main/work/notebooks/w06_validation_audit.ipynb)
- [Week 7 action playbook notebook](https://github.com/uzairrateef-rgb/my-flyrank-internship/blob/main/work/notebooks/w07_action_playbook.ipynb)
- [Feature preparation](https://github.com/uzairrateef-rgb/my-flyrank-internship/blob/main/scripts/01_prepare_features.py)
- [Baseline score](https://github.com/uzairrateef-rgb/my-flyrank-internship/blob/main/scripts/02_baseline_score.py)
- [Model training/evaluation](https://github.com/uzairrateef-rgb/my-flyrank-internship/blob/main/scripts/03_train_model.py)
- [Queue/report export](https://github.com/uzairrateef-rgb/my-flyrank-internship/blob/main/scripts/04_evaluate_and_export.py)

From a fresh clone, install dependencies and run the pipeline:

```bash
python -m pip install -r requirements.txt
python scripts/run_all.py
```

The pipeline fixes the main random seed at 42 in `scripts/03_train_model.py`. The current training script reports the split strategy, test/train row counts, model metrics, baseline metrics, and feature importances to `outputs/model_results.json`; the export stage creates the ranked queue, charts, and Markdown report. The capstone notebook should include assertions for zero client overlap, confirm the feature list contains neither label-source columns nor IDs, record the exact feature/label windows, and write the regenerated metrics plus any holdout-frame builder into `work/outputs/`. Do not call the experiment "sealed" unless the exact holdout construction and metrics receipts are committed and verifiable.

## 9. Acknowledgments & data credit

Built on the [FlyRank ML Internship dataset](https://flyrank.ai).

---

## ML-12: Closing deliverables

### Five-minute demo outline
- **0:00–0:45 — Decision:** explain the editorial prioritization problem, unit of analysis, and cost of false positives/false negatives.
- **0:45–1:30 — Data and safety:** show the anonymized snapshot, label proxy, exclusions, and why IDs are grouping-only.
- **1:30–2:30 — Method:** explain the transparent baseline, Random Forest candidate, client-holdout split, and leakage checks.
- **2:30–3:30 — Results:** present the regenerated baseline-vs-model metrics, base rate, and one chart with a plain-language caption.
- **3:30–4:30 — Action queue:** demonstrate aggregate reason codes and explain how editors verify the top recommendations.
- **4:30–5:00 — Limits:** repeat that the score prioritizes review, does not prove Google mechanisms, and does not establish causal refresh impact.

### Social-post cut (publish after metrics are rerun)
I built a content refresh opportunity-scoring workflow as part of the FlyRank ML Internship, comparing an explainable rules-based baseline with supervised ranking models on anonymized search/content performance data. The key research lesson was that validation design and leakage checks matter as much as the model: a useful score must generalize to held-out clients and use only signals available at decision time. The output is a ranked, reason-coded editorial review queue—not a promise of ranking or traffic gains. [Research repo](https://github.com/uzairrateef-rgb/my-flyrank-internship)

### Employer-facing summary (3 sentences)
I built a reproducible content refresh prioritization pipeline using anonymized search-visibility, freshness, and engagement signals. I compared a transparent heuristic baseline with Logistic Regression, Decision Tree, and Random Forest candidates using a client-holdout evaluation design and top-of-queue ranking metrics. I translated the scores into reason-coded editorial recommendations and documented leakage risks, public-data safeguards, and the limits of observational evidence.

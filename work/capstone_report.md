# Capstone Report — Lane 2: Refresh / Content Opportunity Scoring

- **Author:** Noor Fatima
- **Lane:** Lane 2 — Refresh / Content Opportunity Scoring
- **Repo:** https://github.com/nfatima25seecs/ml-pipeline-ex
- **Date:** August 2026

## 1. Problem framing

This framework supports strategic decisions on content lifecycle management 
by determining when individual published web pages should be updated, left 
alone, or retired. The unit of analysis is a single published page evaluated 
over discrete time windows. The output is a prioritized health score and 
action recommendation per page. A content strategist uses these 
recommendations to execute targeted content refreshes, prune dead-weight 
URLs, or preserve high-performing pages without unnecessary intervention. 
The cost of a wrong call is significant: prematurely updating a thriving 
page risks losing established search equity and ranking stability, while 
failing to refresh a decaying page leads to persistent organic traffic loss 
and wasted domain authority. A hand-written rule based on a single signal 
cannot capture the interaction between age, visibility, CTR, and engagement 
that predicts decay — ML learns those combinations from data.

## 2. Data safety

The starter dataset of 30,000 anonymized pages with 44 columns was used, 
covering impressions, clicks, sessions, CTR, position, age, and engagement 
metrics.

Columns deliberately excluded:

- `trend_direction`, `trend_pct` — label-derived, directly encode the target
- `clicks_last_30d`, `clicks_prev_30d` — used to construct the label
- `sessions_last_30d`, `sessions_prev_30d` — same window as the label
- `client_id` — used for grouping only, never as a model feature

Safe features used: `ctr`, `avg_position`, `impressions_prev_30d`, 
`engagement_rate`, `content_age_days`, `word_count`, `days_since_last_update`

All excluded columns were identified before final model training. No 
client-identifying URLs, names, or raw IDs appear anywhere in `work/`.

## 3. Baseline

A rule-based heuristic was implemented as the baseline: flagging any page 
with `content_age_days > 180` and dropping impression volume as a candidate 
for refresh. This naive rule achieved Precision@50 of 0.72 — meaning 36 out 
of the top 50 flagged pages were genuine click-drop targets on held-out test 
data. While deployable, it treats all aging content identically, failing to 
distinguish between naturally evergreen pages and genuine decay candidates. 
This proves that simple age and impression thresholds cannot reliably capture 
non-linear decay signals across varying page archetypes.

## 4. Model / analysis

To model complex interactions between traffic signals, engagement metrics, 
and page freshness, a Random Forest Classifier was trained alongside Logistic 
Regression as a secondary baseline.

- **Target definition:** A page is labeled as a decay candidate if 
  `clicks_last_30d < clicks_prev_30d` — a drop in clicks from one 30-day 
  window to the next, observable without using label-derived columns.
- **Validation strategy:** Evaluated using GroupKFold cross-validation split 
  strictly on `client_id` to prevent data leakage between domains.
- **Feature processing:** Logarithmic scaling applied to skewed volume metrics 
  (`impressions_prev_30d`); engineered ratio features including 
  `ctr_to_position_ratio` and `engagement_density`.
- **Model training:** Hyperparameter tuning via RandomizedSearchCV using 
  Macro F1 to handle class imbalance. Random Forest captured non-linear 
  decision boundaries between `days_since_last_update`, `avg_position`, 
  and `engagement_rate`.

## 5. Evaluation

The Random Forest model outperformed both the heuristic baseline and Logistic 
Regression across all primary evaluation metrics on held-out grouped 
validation folds:

| Method | Precision@50 | Macro F1 | Accuracy |
|---|---|---|---|
| Baseline rule | 0.72 | ~0.41 | ~56% |
| Logistic Regression | — | ~0.64 | ~71% |
| Random Forest | 0.74 | ~0.83 | ~86% |

The model minimizes costly false positives — flagging performing pages 
unnecessarily — while maintaining high recall on rapidly decaying content. 
False positives were concentrated in high-traffic pages with modest CTR that 
held stable traffic rather than dropping, indicating the model leans on 
impression scale when prioritizing risk.

## 6. Interpretation

Feature importance analysis reveals that decay prediction relies heavily on 
interaction effects rather than isolated single signals:

- **`days_since_last_update` & `content_age_days`:** Primary structural 
  anchors indicating temporal risk.
- **`ctr_to_position_ratio`:** Strongest behavioral indicator; pages dropping 
  in click-through rate relative to their average ranking position show a 
  high probability of imminent traffic loss.
- **`engagement_rate`:** Acts as a stabilizing filter; highly engaged pages 
  remain resilient even as content age increases, protecting high-performing 
  evergreen assets from false positive flags.

## 7. Recommendation

Based on predicted decay scores and reason codes generated by the model, 
content teams should execute the following tiered playbook:

- **Action 1 — Priority Refresh (Action Score > 0.80 / Top 50 Queue):** 
  Immediately update core content sections, refresh meta title/description 
  tags, and check for broken links on pages scoring above 0.80 in predicted 
  decay risk or appearing in the top 50 ranked queue.
- **Action 2 — Protect & Maintain (High Traffic + Action Score < 0.35):** 
  Do not modify URLs, core text, or structure on high-performing evergreen 
  pages with action scores below 0.35 regardless of content age; unnecessary 
  updates risk destabilizing established search equity.
- **Action 3 — Prune or Merge (Low Engagement + Action Score > 0.80 + Low 
  Impressions):** Identify persistent dead-weight URLs that fail to engage 
  users or capture search visibility; consolidate via 301 redirects to 
  stronger canonical pages or prune entirely to preserve crawl budget.

**Confidence and limits:** Results are observational and directional. This 
model does not establish causation between refresh actions and traffic 
recovery. Recommendations are decision-support tools, not automated 
publishing decisions. Performance was measured on one anonymized dataset 
and may not generalize to all client portfolios.

## 8. Reproducibility

- **Repository:** [nfatima25seecs/ml-pipeline-ex](https://github.com/nfatima25seecs/ml-pipeline-ex)
- **Capstone Notebook:** `work/capstone_refresh_scoring.ipynb`
- **Data Acknowledgment:** Built on the FlyRank ML Internship dataset.
- **Random seed used throughout:** 42

**Re-run from fresh clone:**

```bash
cd work/
jupyter nbconvert --to notebook --execute capstone_refresh_scoring.ipynb
```o client-identifying details · numbers in this report
> match a fresh re-run.

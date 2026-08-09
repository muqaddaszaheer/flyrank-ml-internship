# Search Intelligence Content Review

**Author:** Muqaddas Zaheer Ahmad  
**Lane:** Applied Search Intelligence  
**Project:** FlyRank AI Internship Capstone  
**Date:** August 2026  

---

## Abstract

This project asks whether machine learning can help prioritize content pages for human review using observable search, content, and engagement signals. I used the anonymized FlyRank content-refresh dataset and prepared features for classifying pages into the observed declining-performance class. I evaluated Logistic Regression using both a stratified row split and a client-holdout split. On the client-holdout split, the model achieved 0.6606 accuracy, 0.5659 precision, 0.5666 recall, 0.5662 F1, and 0.7003 ROC AUC. The output is intended as directional decision support for human content and SEO review, not as an automatic decision system or proof of future performance.

---

## 1. Introduction / Problem Statement

The goal of this project is to help content and SEO teams decide which pages may deserve attention first.

The unit of analysis is a content page.

The main output is a classification score for the observed declining-performance class, together with a ranked action queue and reason codes.

A human reviewer can use the output to decide which pages should be investigated first instead of reviewing every page in the same order.

A wrong decision can waste review time or cause a page that deserves attention to receive lower priority.

Machine learning is useful here because several observable content, search, and engagement signals can be considered together instead of relying on only one simple rule.

The output is intended as decision support. It does not prove that a particular content change will improve search performance.

---

## 2. Data

The analysis uses the anonymized FlyRank content-refresh dataset:

`data/raw/content_refresh_anonymized.csv`

The modeling workflow contains 30,000 rows and 44 source columns.

The processed feature file used by the validation workflow is:

`data/processed/refresh_feature_vector.csv`

The target is:

`is_declining_label`

The final model uses 52 processed features after numeric processing and categorical encoding.

The features include observable signals such as:

- Search volume
- Competition
- CPC
- Content length
- Impressions
- Clicks
- Sessions
- Content age
- Days since last update
- CTR
- Average position
- Engagement rate
- Scroll rate
- AI traffic percentage
- Selected categorical content and search attributes

### Excluded fields

The following fields were deliberately excluded:

- `content_id` — used as an identifier, not a predictive feature.
- `client_id` — used only for grouped validation and never as a model feature.
- `trend_pct` — excluded because it is derived from trend information used to define the target.
- `trend_direction` — excluded because it is label-related and could create leakage.

No client-identifying information is intentionally included in this public report.

---

## 3. Methodology

The main model is **Logistic Regression**.

I selected Logistic Regression because it is simple, reproducible, and relatively easy to interpret while allowing multiple observable signals to contribute to the classification decision.

The final model uses:

- StandardScaler
- Logistic Regression
- Balanced class weights
- `max_iter=2000`
- `random_state=42`

The final model contains 52 processed features.

The target is `is_declining_label`, representing the observed declining-performance class in the prepared data.

Identifiers and label-derived fields were deliberately left out to reduce leakage risk.

---

## 4. Baseline

The first baseline was a simple and transparent rule using:

- `days_since_last_update`
- `ctr`

A page was marked for review when it met the baseline staleness or CTR condition.

The Week-4 baseline used:

- Stale threshold: 104 days
- Low CTR threshold: 0.0

The baseline provides a simple reference point for the machine-learning model.

### Baseline vs Model

| Method | Accuracy | Precision | Recall | F1 |
| --- | ---: | ---: | ---: | ---: |
| Week-4 baseline | 1.0000 | 1.0000 | 1.0000 | 1.0000 |
| Logistic Regression | 0.8052 | 0.8202 | 0.8930 | 0.8551 |

The perfect baseline result should be interpreted carefully. The Week-5 baseline target was constructed directly from the same staleness and CTR rule used by this baseline. Therefore, the 1.0000 result reflects alignment between the rule and the target definition rather than evidence that the baseline is a stronger independent predictor.

For this reason, the baseline is treated as a development reference rather than an independent benchmark of real-world generalization.

---

## 5. Evaluation

Two validation designs were compared.

### Stratified row split

The first evaluation used a stratified row split.

- Training rows: 24,000
- Test rows: 6,000
- Accuracy: 0.6502
- Precision: 0.6775
- Recall: 0.6765
- F1: 0.6770
- ROC AUC: 0.7107
- Average Precision: 0.7271

### Client-holdout split

The final evaluation used a client-holdout split.

- Training rows: 27,675
- Test rows: 2,325
- Accuracy: 0.6606
- Precision: 0.5659
- Recall: 0.5666
- F1: 0.5662
- ROC AUC: 0.7003
- Average Precision: 0.5215

The client-holdout result is the more important generalization result because clients in the test set were not present in training.

The validation audit confirmed:

**Shared clients: 0**

This means the training and test clients were completely separated.

### Final client-holdout model results

| Metric | Client-holdout result |
| --- | ---: |
| Accuracy | 0.6606 |
| Precision | 0.5659 |
| Recall | 0.5666 |
| F1 | 0.5662 |
| ROC AUC | 0.7003 |
| Average Precision | 0.5215 |

The Average Precision decreased from 0.7271 under the row split to 0.5215 under the client-holdout split. This shows that the row-level evaluation was more optimistic and that performance is harder to maintain on unseen clients.

---

## 6. Interpretation

The model combines multiple observable signals instead of relying on only one rule.

In the earlier two-feature Logistic Regression development experiment:

- `days_since_last_update` had a positive coefficient of approximately `0.068539`.
- `ctr` had a negative coefficient of approximately `-2.809776`.

In that development model, greater time since the last update increased the model's tendency toward the review class, while higher CTR decreased that tendency.

These coefficients describe associations within the model. They should not be interpreted as causal effects.

The error analysis also showed that some pages have signals that point in different directions. For example, a page can look stale but still not belong to the observed declining class, or it can show a declining label even when the simple staleness and CTR rule does not flag it.

The most important negative result is the decrease in Average Precision from 0.7271 with the row split to 0.5215 with the client-holdout split.

This suggests that generalization to unseen clients is more difficult than the simpler row-level evaluation suggests.

---

## 7. Ranked Recommendations

The model should be used to **prioritize human review**, not to make automatic decisions.

### 1. Refresh declining content

Prioritize pages showing evidence of declining performance.

### 2. Review weak engagement

Investigate pages with weaker observed engagement signals.

### 3. Review weak click-through performance

Investigate pages with weaker observed CTR.

### 4. Review older content

Consider older pages as part of a broader content review.

### 5. Monitor pages without a clear action signal

Do not force an action when the available evidence is weak.

A FlyRank editor could use the ranked queue as a starting point for daily or weekly content review.

Before taking action, the reviewer should check the page context, available performance evidence, reason code, and whether the recommendation makes sense for the specific content.

The system should not automatically:

- Publish content
- Delete content
- Redirect URLs
- Make major content changes
- Make high-impact decisions
- Treat a model prediction as proof of future performance

The confidence in individual recommendations is limited because the validation results show weaker performance on unseen clients.

Therefore, the recommendations should remain **directional decision support**.

---

## 8. Limitations

This project has several important limitations:

1. The model is evaluated on observed data and does not prove future search performance.
2. Performance is weaker when tested on unseen clients.
3. The client-holdout Average Precision was 0.5215.
4. The model should not be treated as a causal model.
5. Some pages contain mixed signals that can lead to classification errors.
6. Human review is still required before taking action.
7. Changes in data conditions or feature distributions may reduce model performance over time.

These limitations mean the model is best used as a prioritization and decision-support tool.

---

## 9. Reproducibility

The complete project repository is available here:

[GitHub Repository](https://github.com/muqaddaszaheer/flyrank-ml-internship)

The main capstone notebook is:

`work/notebooks/capstone.ipynb`

Supporting notebooks are stored under:

`work/notebooks/`

The final validation workflow uses:

- `data/raw/content_refresh_anonymized.csv`
- `data/processed/refresh_feature_vector.csv`
- `scripts/01_prepare_features.py`
- `scripts/ml_utils.py`

The final model uses:

- `StandardScaler`
- `LogisticRegression`
- `class_weight="balanced"`
- `max_iter=2000`
- `random_state=42`

The stratified row split uses:

- `test_size=0.20`
- `random_state=42`
- Stratification by the target

The client-holdout validation uses a NumPy random generator with seed `42` and keeps clients completely separate between training and test data.

The analysis should be re-run from a fresh clone before making a final claim about the reported numbers.

---

## 10. Acknowledgments & Data Credit

This project was built on the FlyRank ML Internship dataset.

Data and internship credit: [FlyRank](https://flyrank.ai)

---

## Final Note

This project is presented as an applied machine-learning analysis and decision-support workflow.

The results are observed and measured results from the available evaluation data. They should not be interpreted as causal effects or guaranteed future performance.

The final decision remains with a human reviewer.

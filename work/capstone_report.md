# Capstone Report — Search Intelligence Content Review

- **Author:** Muqaddas Zaheer Ahmad
- **Lane:** Applied Search Intelligence
- **Repo:** https://github.com/muqaddaszaheer/flyrank-ml-internship
- **Date:** August 2026

## 0. Abstract

This project asks whether machine learning can help prioritize content pages for human review using observable search, content, and engagement signals. I used the anonymized FlyRank content-refresh dataset and prepared a feature set for classifying pages into the observed declining-performance class. I evaluated logistic regression using both a stratified row split and a client-holdout split, with the client-holdout design providing the more realistic test of performance on unseen clients. On the client-holdout split, the model measured 0.6606 accuracy, 0.5659 precision, 0.5666 recall, 0.5662 F1, 0.7003 ROC AUC, and 0.5215 average precision. The resulting scores and recommendations are intended as directional decision support for human content and SEO review, not as proof of future performance or a fully automated decision system.

## 1. Problem framing

The goal is to support content and SEO review by helping prioritize which pages may deserve attention first.

The unit of analysis is a content page.

The main output is a classification score for the observed declining-performance class, together with a ranked action queue and reason codes for review.

A human reviewer can use the output to decide which pages should be investigated first instead of reviewing every page in the same order.

A wrong call can result in wasted review time or cause a page that deserves attention to receive lower priority.

Machine learning is useful because several observable content, search, and engagement signals can be considered together instead of relying only on one simple rule.

The output is decision support. It does not prove that a particular content change will improve search performance.

## 2. Data safety

The analysis uses the anonymized FlyRank content-refresh dataset:

`data/raw/content_refresh_anonymized.csv`

The modeling workflow contains 30,000 rows and 44 source columns.

The processed feature file used by the validation workflow is:

`data/processed/refresh_feature_vector.csv`

The target is:

`is_declining_label`

The final model uses 52 features after numeric processing and categorical encoding.

The model uses observable signals including search volume, competition, CPC, content length, impressions, clicks, sessions, content age, days since the last update, CTR, average position, engagement rate, scroll rate, AI traffic percentage, and selected categorical content and search attributes.

The following fields were deliberately excluded from the model:

- `content_id` — used as an identifier, not a predictive feature.
- `client_id` — used only for grouped validation and never as a model feature.
- `trend_pct` — excluded because it is derived from the same trend information used to define the target.
- `trend_direction` — excluded because it is label-related and could create leakage.

The final model feature definitions in `scripts/ml_utils.py` confirm that the model feature lists do not include `content_id`, `client_id`, `trend_pct`, or `trend_direction`.

The validation audit also confirmed:

- Model features: 52
- Declining rate: 0.5421
- Shared clients between training and test: 0
- Train and test clients were separate.

No client-identifying information is intentionally included in this report or in the public work produced for the assignment.

## 3. Baseline

The first baseline was a simple and transparent rule using two observable signals:

- `days_since_last_update`
- `ctr`

A page was marked for review when either it was relatively stale or it had low CTR.

The Week-4 baseline used:

- stale threshold: 104 days
- low CTR threshold: 0.0

The baseline is easy to understand and provides a transparent reference point for the machine-learning model.

On the Week-5 stratified test split, the baseline and logistic regression were evaluated on exactly the same test rows:

| Method | Accuracy | Precision | Recall | F1 |
|---|---:|---:|---:|---:|
| Week-4 baseline | 1.0000 | 1.0000 | 1.0000 | 1.0000 |
| Logistic Regression | 0.8052 | 0.8202 | 0.8930 | 0.8551 |

The perfect baseline result should be interpreted carefully. The Week-5 baseline target was constructed directly from the same staleness and CTR rule, so the baseline is expected to reproduce that rule exactly. It should therefore be treated as a development reference rather than evidence that the baseline is a stronger independent predictor.

## 4. Model / analysis

The main model is Logistic Regression.

Logistic Regression was selected because it is simple, reproducible, and relatively easy to interpret while allowing multiple observable signals to contribute to the classification decision.

The final validation workflow uses:

- StandardScaler
- Logistic Regression
- balanced class weights
- `max_iter=2000`
- `random_state=42`

The final model contains 52 processed features.

The numeric feature set includes signals such as:

- search volume
- competition
- CPC
- word count
- character count
- log-transformed impressions
- log-transformed clicks
- log-transformed sessions
- log-transformed AI sessions
- days with impressions
- days with sessions
- content age
- days since last update
- CTR
- average position
- engagement rate
- scroll rate
- AI traffic percentage

The categorical feature set includes:

- competition level
- content type
- main intent
- age tier
- freshness tier
- word-count tier
- impression tier
- position tier

The target is `is_declining_label`, representing the observed declining-performance class in the prepared data.

The model deliberately leaves out identifiers and label-derived fields to reduce leakage risk.

## 5. Evaluation

Two validation designs were compared.

### Before: stratified row split

The first evaluation used a stratified row split:

- Training rows: 24,000
- Test rows: 6,000
- Accuracy: 0.6502
- Precision: 0.6775
- Recall: 0.6765
- F1: 0.6770
- ROC AUC: 0.7107
- Average precision: 0.7271

### After: client-holdout split

The final validation used a client-holdout split:

- Training rows: 27,675
- Test rows: 2,325
- Accuracy: 0.6606
- Precision: 0.5659
- Recall: 0.5666
- F1: 0.5662
- ROC AUC: 0.7003
- Average precision: 0.5215

The client-holdout result is the more important generalization result because clients in the test set were not present in training.

The validation audit confirmed:

`Shared clients: 0`

This means the training and test clients were separated.

The average precision decreased from 0.7271 under the row split to 0.5215 under the client-holdout split. This shows that the row-level evaluation was more optimistic and that performance is harder to maintain when evaluating on unseen clients.

The earlier model error analysis found 1,169 errors out of 6,000 rows on the Week-5 test split, giving an error rate of 0.1948. The errors occurred where the learned model disagreed with the observed target or the simple baseline rule.

The current validation notebook does not report a separate baseline metric on the client-holdout split, so the final client-holdout numbers are reported as the model's generalization results rather than being presented as a new model-versus-baseline comparison.

## 6. Interpretation

The model combines multiple observable signals instead of relying on only one rule.

The earlier two-feature Logistic Regression analysis provides a simple interpretation of the strongest signals used in that development experiment:

- `days_since_last_update` had a positive coefficient of approximately `0.068539`.
- `ctr` had a negative coefficient of approximately `-2.809776`.

In that development model, greater time since the last update increased the model's tendency toward the review class, while higher CTR decreased that tendency.

These coefficients should be interpreted as associations within the model, not as causal effects.

The error analysis also shows that some pages have signals that point in different directions. For example, a page can look stale but still not belong to the observed declining class, or it can show a declining label even when the simple staleness and CTR rule does not flag it.

The most important negative result is the drop in average precision from 0.7271 with the row split to 0.5215 with the client-holdout split. This suggests that generalization to unseen clients is more difficult than the simpler row-level evaluation suggests.

This is why the client-holdout result should be treated as the more realistic estimate for this use case.

## 7. Recommendation

The model and action queue should be used to prioritize human review.

The recommended order is:

1. **Refresh declining content**
   - Prioritize pages showing evidence of declining performance.

2. **Review weak engagement**
   - Investigate pages with weaker observed engagement signals.

3. **Review weak click-through performance**
   - Investigate pages with weaker observed CTR.

4. **Review older content**
   - Consider older pages as part of a broader content review.

5. **Monitor pages without a clear action signal**
   - Do not force an action when the available evidence is weak.

A FlyRank editor could use the ranked queue as a starting point for daily or weekly content review.

Before taking action, the reviewer should check the page context, available performance evidence, reason code, and whether the recommendation makes sense for the specific content.

The system should never automatically:

- publish content;
- delete content;
- redirect URLs;
- make major content changes;
- make high-impact decisions;
- treat a model prediction as proof of future performance.

The confidence in individual recommendations is limited because the validation results show weaker performance on unseen clients.

The recommendations should therefore remain directional decision support rather than automatic decisions.

Useful monitoring triggers include:

- model performance decreases;
- feature distributions change;
- recommendation patterns change substantially;
- data conditions change;
- error rates increase;
- reason codes become less useful.

These signals should trigger human review and possible retraining rather than automatic retraining.

## 8. Reproducibility

The project repository is:

`https://github.com/muqaddaszaheer/flyrank-ml-internship`

The main capstone notebook is:

`work/notebooks/capstone.ipynb`

The supporting notebooks are stored under:

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
- stratification by the target

The client-holdout validation uses a NumPy random generator with seed `42` and keeps clients completely separate between training and test data.

The main validation command can be reproduced from the repository environment by running the relevant notebook cells in order after the repository and required anonymized data are available.

The analysis should be re-run from a fresh clone before making a final claim about the reported numbers.

The repository should not contain private client data, private queries, or other client-identifying information.

## 9. Acknowledgments & data credit

Built on the FlyRank ML Internship dataset.

Data and internship credit: https://flyrank.ai

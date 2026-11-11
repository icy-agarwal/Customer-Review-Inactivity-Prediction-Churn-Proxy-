# Customer Churn Predictor with a Cost-Sensitive Offer Threshold

Predicts which customers will go quiet for the next 180 days, and chooses who should get a retention offer by expected rupee value instead of a default 0.5 cutoff. Built on Amazon Musical Instruments reviews, with time-based evaluation.

## Results

| | AUC | PR-AUC |
|---|---|---|
| Logistic regression (validation, 8,368 users) | 0.665 | 0.844 |
| XGBoost (validation) | 0.676 | 0.850 |
| **XGBoost (test, 9,203 users)** | **0.674** | **0.852** |

Always predicting churn is already 75.6% accurate on test, so accuracy is not a useful metric here. At the chosen threshold, accuracy is 74.1%, but the policy still earns more money than blanket offers.

Confusion matrix on test at threshold 0.61 (rows: actual, columns: not offered / offered):

| | Not offered | Offered |
|---|---|---|
| Stayed | 738 | 1,503 |
| Churned | 879 | 6,083 |

## The decision (rupee figures are assumptions, not real business data)

Assumptions: offer cost Rs 100, value of a kept customer Rs 800, 20% of would-be churners are kept by an offer. An offer pays off when churn risk is above cost / (save rate x value) = 0.625. The threshold was chosen on validation (0.61) and scored once on test.

| Test (9,203 customers) | Net value |
|---|---|
| Target nobody | Rs 0 |
| Target everyone | Rs 193,620 |
| Model, threshold 0.61 | Rs 214,680 (+10.9%) |

### Sensitivity

| Changed assumption | Best threshold | Model vs target everyone |
|---|---|---|
| Offer cost Rs 50 | 0.30 | +Rs 2,200 (+0.3%) |
| Base case | 0.61 | +Rs 21,060 (+10.9%) |
| Offer cost Rs 200 or Rs 400 | none | offers lose money, send none |
| Customer value Rs 400 | none | offers lose money, send none |
| Customer value Rs 1,600 | 0.30 | +Rs 4,400 (+0.3%) |
| Save rate 10% | none | offers lose money, send none |
| Save rate 30% | 0.38 | +Rs 8,040 (+1.1%) |

The model's advantage over blanket offers is large only when the economics are in a middle band. When offers are cheap or very profitable, almost everyone should get one. When they cost more than they can return, the right policy is to send none.

## Method

- **Definition:** among customers with a review in the 365 days before a cutoff date, churn means no review in the next 180 days. Reviews stand in for purchase activity.
- **Time-based snapshots:** train 2016-01-01, validation 2016-07-01, test 2017-01-01.
- **Features (all computed before the cutoff):** review count, average rating, days since last review, account age, reviews in the last 365 days, reviews in the last 90 days. Four extra behavioral features (average gap, overdue ratio, recent share, last rating) changed AUC by 0.004 and were dropped.
- **Leakage found and fixed:** the dataset's 5-core filter means a user with fewer than 5 earlier reviews must review again later. Their churn rate was 0.47 to 0.50 against 0.74 to 0.75 for everyone else, which inflated XGBoost's validation AUC to 0.72. Only users with 5 or more reviews before the cutoff are used.

## Data

[Amazon Review Data (2018)](https://cseweb.ucsd.edu/~jmcauley/datasets/amazon_v2), Musical Instruments, 5-core file. After removing 12,236 duplicate (user, item) pairs and re-applying the 5-core filter: 206,241 reviews, 24,780 users.

## Limitations

- Reviews are a stand-in for purchases, and not reviewing is not the same as leaving.
- The dataset only contains users with 5 or more reviews overall, so these are already engaged customers.
- All rupee figures are assumptions.
- AUC is modest (about 0.67). Review timing is a weak signal of churn.
- The same users appear in more than one snapshot, but each snapshot only uses information from before its own cutoff.

## Run it

Open `churn_predictor.ipynb` in Google Colab and run all cells. Download the dataset file from the link above and place it as `musical_instruments.json.gz` (the first cell shows how).

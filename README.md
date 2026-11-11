# Customer Review Inactivity Prediction (Churn Proxy)

Predicts which customers are likely to stop reviewing products for the next 180 days and identifies whom to target with retention offers using expected economic value rather than a default 0.5 classification threshold.

Built using the Amazon Musical Instruments review dataset with time-based evaluation.

**Note:** Review inactivity is used as a proxy for customer churn. The dataset does not contain actual purchase or subscription activity.

## Results

| Model | AUC | Average Precision (AP) |
|---|---:|---:|
| Logistic Regression (validation, 8,368 users) | 0.665 | 0.844 |
| XGBoost (validation) | 0.676 | 0.850 |
| **XGBoost (test, 9,203 users)** | **0.674** | **0.852** |

The positive-class prevalence on the test set is approximately 75.6%, providing a random-ranking average precision baseline of approximately 0.756.

Always predicting inactivity achieves 75.6% accuracy on the test set. Therefore, accuracy alone is not a useful measure of model performance. At the selected threshold of 0.61, the model achieves 74.1% accuracy while producing higher simulated net value than blanket targeting.

### Confusion Matrix

Test set at threshold 0.61 (rows: actual, columns: not offered / offered).

| | Not Offered | Offered |
|---|---:|---:|
| Stayed | 738 | 1,503 |
| Inactive (churn proxy) | 879 | 6,083 |

## Cost-Sensitive Retention Decision

All monetary values are hypothetical assumptions, not actual business or transaction data.

**Assumptions:**
- Offer cost: ₹100
- Value of a retained customer: ₹800
- Offer success rate: 20% of customers who would otherwise become inactive

The theoretical break-even threshold is:

\[
t = \frac{\text{Offer Cost}}{\text{Save Rate}\times\text{Customer Value}}
\]

\[
t = \frac{100}{0.20\times800}=0.625
\]

This is the theoretical break-even point under the assumed economics, provided the predicted probabilities are calibrated. Since probability calibration has not been independently established, this threshold is treated as a theoretical reference point. The operating threshold of 0.61 was selected empirically using validation-set net value.

### Test Set Business Simulation

| Targeting Strategy | Simulated Net Value |
|---|---:|
| Target nobody | ₹0 |
| Target everyone | ₹193,620 |
| **Model-based targeting (0.61)** | **₹214,680 (+10.9%)** |

The model achieves ₹21,060 higher simulated net value than blanket targeting under the base-case assumptions.

### Sensitivity Analysis

| Changed Assumption | Best Threshold | Model vs. Target Everyone |
|---|---:|---:|
| Offer cost ₹50 | 0.30 | +₹2,200 (+0.3%) |
| Base case | 0.61 | +₹21,060 (+10.9%) |
| Offer cost ₹200 or ₹400 | None | Offers lose money; target nobody |
| Customer value ₹400 | None | Offers lose money; target nobody |
| Customer value ₹1,600 | 0.30 | +₹4,400 (+0.3%) |
| Save rate 10% | None | Offers lose money; target nobody |
| Save rate 30% | 0.38 | +₹8,040 (+1.1%) |

The model's advantage over blanket targeting is largest under intermediate economic assumptions. When offers are inexpensive or customer value is sufficiently high, targeting everyone becomes competitive. When offers cannot generate positive expected value, the optimal policy is to send none.

**These results represent a cost-sensitive targeting simulation, not measured revenue improvement or causal uplift.** The model does not establish whether an offer actually prevents a customer from becoming inactive.

## Methodology

### 1. Target Definition

Among customers with at least one review in the 365 days before a prediction cutoff, inactivity is defined as having no review during the following 180 days.

Reviews are used as a proxy for purchase activity. Consequently, a customer who continues purchasing without submitting reviews may still be classified as inactive.

### 2. Time-Based Evaluation

Three chronological prediction snapshots are used:

- Training: 2016-01-01
- Validation: 2016-07-01
- Testing: 2017-01-01

Features are calculated exclusively using information available before each snapshot's cutoff date.

The same users can appear across multiple snapshots. This represents temporal evaluation on a recurring customer population rather than evaluation on entirely unseen customers.

### 3. Feature Engineering

Six behavioral features are used:

- Review count
- Average rating
- Days since last review
- Review-history age (time since first observed review)
- Reviews in the last 365 days
- Reviews in the last 90 days

Four additional behavioral features (average review gap, overdue ratio, recent share and last rating) were investigated but excluded because they changed AUC by only approximately 0.004.

### 4. Model Training and Leakage Investigation

Logistic Regression is used as a baseline and compared against XGBoost.

The original 5-core dataset retains users and items with at least five reviews across the complete dataset. Applying this filter before chronological splitting introduces future-informed selection bias.

In particular, users with fewer than five reviews before a prediction cutoff must have additional reviews later to satisfy the dataset's five-review requirement. This artificially reduces their observed inactivity rate.

An investigation found that users with fewer than five prior reviews had inactivity rates of approximately 0.47–0.50, compared with 0.74–0.75 among users with at least five prior reviews. This inflated the original XGBoost validation AUC to approximately 0.72.

To mitigate this user-level leakage, the final modelling population is restricted to users with at least five reviews before the respective prediction cutoff.

This reduces the identified source of leakage but does not eliminate every possible source of temporal selection bias.

## Dataset

**Source:** [Amazon Review Data (2018)](https://cseweb.ucsd.edu/~jmcauley/datasets/amazon_v2)

**Category:** Musical Instruments, 5-core review file.

The dataset is preprocessed to retain user ID, item ID, rating and timestamp. Repeated user-item interactions are deduplicated, retaining the latest review, followed by an iterative 5-core filtering step.

The resulting dataset contains:

- 206,241 reviews
- 24,780 users

## Limitations

- Reviews are a proxy for purchases. Not reviewing does not necessarily mean that a customer has stopped purchasing or left the platform.
- The dataset contains only users and items meeting the 5-core requirement, limiting the representativeness of the population.
- Applying the 5-core filter before chronological splitting introduces future-informed selection bias.
- The model's reported performance is based on temporal snapshots of a recurring user population; the same users may appear across snapshots.
- All rupee figures, including customer value, offer cost and save rate, are hypothetical assumptions.
- The reported 10.9% improvement is a simulated targeting result, not verified revenue improvement or causal uplift.
- Test ROC-AUC is approximately 0.67, indicating moderate predictive discrimination.
- The model has not been tested using actual transaction data or deployed in a real retention campaign.
- Probability calibration has not been independently established, so the theoretical break-even threshold should not be interpreted as a guaranteed optimal operating threshold.

## How to Run

1. Open `churn_predictor.ipynb` in Google Colab.
2. Download the Musical Instruments 5-core dataset from the source linked above.
3. Place the downloaded file in the Colab working directory as `musical_instruments.json.gz`.
4. Run all notebook cells sequentially.

The first notebook cell includes the dataset-loading instructions.

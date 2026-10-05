# Loan Default Risk Scoring and Profitability Analysis

A credit risk project that predicts which loan applicants are likely to default, groups them into risk bands, and uses a simple profit model to decide where a lender should draw the approval line.

I built it as a portfolio project to practise the kind of work a bank analytics team does: predict risk, explain what drives it, and turn the result into a decision with a rupee-style business case.

## What this project does

1. Explores 3.07 lakh loan applications to find what is linked to default.
2. Trains two models (logistic regression and XGBoost) and compares them.
3. Removes gender from the final model to avoid discriminatory lending.
4. Splits customers into four risk bands, each with a suggested action.
5. Chooses the approval cut-off by weighing the cost of a bad loan against the interest lost by turning away a good customer.
6. Tests how sensitive that result is to my assumptions, with an Excel profit model.

## Data

- **Source:** [Home Credit Default Risk](https://www.kaggle.com/competitions/home-credit-default-risk) (Kaggle), file `application_train.csv`.
- **Size:** 307,511 applications, 122 columns, 8.07% defaulted (24,825 defaults vs 282,686 repaid).
- **Not included in this repository.** Please download it from the competition page yourself and place it at `data/application_train.csv`.
- It is a public consumer-lending dataset, not data from a real bank.

## Key findings from the exploration

| Question | What the data showed |
|---|---|
| Does age matter? | Yes. Default falls steadily with age, from 11.46% (ages 20-30) to 4.92% (ages 60-70). |
| Does education matter? | Yes. Default is 5.36% for higher education, 8.94% for secondary, and 10.93% for lower secondary. |
| Do external credit scores matter? | Strongly. Default is 15.21% in the lowest `EXT_SOURCE_2` band and 3.59% in the highest. |
| Does a big loan compared to income mean more risk? | Not simply. The pattern was flat or slightly curved, probably because the lender had already filtered out risky large loans. I kept this finding instead of forcing a story. |
| Does the yearly payment compared to income matter? | A little. Default rises from 7.20% to 8.70% across the bands. |

## Modelling

- **Split:** 80% train and 20% test, keeping the same default rate in both (stratified).
- **Cleaning:** dropped columns that were more than 50% empty (except the external credit scores, which I kept), replaced a placeholder value in `DAYS_EMPLOYED` (365243) with a missing value and a flag, filled gaps with train-set medians for logistic regression, and left gaps as they are for XGBoost.
- **Features I added:** `EXT_SOURCE_MEAN` (average of the three external scores), `CREDIT_TERM` (annuity divided by credit amount), employment years and a not-employed flag.
- **Class imbalance:** only about 1 in 12 customers defaults, so accuracy would be misleading. I used class weighting and looked at AUC, recall and precision instead.

| Model | AUC | Recall | Precision |
|---|---|---|---|
| Logistic regression | 0.750 | 0.678 | 0.163 |
| XGBoost | 0.767 | 0.674 | 0.176 |
| XGBoost without gender (final) | 0.766 | not recorded | not recorded |

Recall and precision above use a 0.5 threshold. The final decision rule is the cost-based cut-off described below, so those two numbers describe the models, not the final policy.

**What drives the score.** `EXT_SOURCE_MEAN`, the feature I built from the external scores, is by far the strongest. Education, employment years, contract type and my `CREDIT_TERM` feature follow.

**Gender.** Gender appeared in the top features, so I retrained without it. AUC fell from 0.767 to 0.766, so the final model leaves it out. A lender should not price credit on gender, and the cost in accuracy was tiny.

**Leakage.** Every column in this file describes the customer at the time of application, so none of them record what happened after the loan.

## From a score to a decision

A probability is not a decision, so I rank the test customers from safest to riskiest and ask how much a lender would earn at each possible approval line.

**Assumptions (mine, not real bank figures):**
- The lender earns **15%** of the loan amount on a loan that is repaid.
- The lender loses **60%** of the loan amount on a loan that defaults.
- With these numbers, a group is worth lending to only if its default rate is below **20%** (0.15 / (0.15 + 0.60)).

**Result on the test set (61,503 customers):**

| | Profit (millions) |
|---|---|
| Approve everyone | 3,433.2 |
| Approve up to the best cut-off | 3,636.8 |
| Gain | 203.6 (about 5.9%) |

The best cut-off approves 91.9% of applicants. Approved customers default at **6.1%**, rejected ones at **30.4%**.

### Risk bands

| Band | Customers | Default rate | Suggested action |
|---|---|---|---|
| Low | 36,901 | 3.28% | Approve at the standard rate |
| Medium | 12,301 | 9.13% | Approve and monitor |
| High | 7,300 | 15.25% | Higher rate, lower limit, or manual review |
| Reject | 5,001 | 30.37% | Decline |

Default rates rise nearly tenfold from the safest to the riskiest band.

### Sensitivity to the loss assumption

| Loss per bad loan | Share approved | Gain from the model |
|---|---|---|
| 40% | 94.7% | 69.2M (1.7%) |
| 60% | 91.9% | 203.6M (5.9%) |
| 80% | 87.6% | 398.1M (13.8%) |
| 100% | 83.3% | 627.8M (27.0%) |

The costlier a bad loan is, the fewer people should be approved, and the more the model is worth. If bad loans cost little, there is almost nothing to gain from screening.

## Excel profit model

`outputs/loan_profit_model.xlsx` rebuilds the band-level profit with live formulas. Change the margin or loss on the Assumptions sheet and everything updates. It uses the average loan in each band, so its totals are close to, but not exactly equal to, the customer-level notebook results (gain of 195.8M against 203.6M).

## Limitations

- **The margin and loss figures are assumptions.** The profit numbers are only as good as those two inputs, which is why I show a range.
- **Not real bank data.** The dataset is from a public competition, so results will not transfer directly to any specific bank.
- **The cut-off was chosen and measured on the same test set,** so the gain is slightly optimistic. A cleaner check would pick the cut-off on one half and measure it on the other.
- **Simplified economics.** There are no operating costs, funding costs, recovery delays or differences between loan types.
- **Model scores are not true probabilities,** because class weighting inflates them. Only the ranking is meaningful, so I use bands and rank-based cut-offs.
- **No tuning or time-based testing.** I used sensible settings for the models without hyperparameter search, and I did not test on a later time period.
- **Small groups.** The academic-degree education group has only 164 customers, so I do not rely on its default rate.

## How to run it

1. Download `application_train.csv` from the Kaggle page above and put it in `data/`.
2. Install the libraries: `pip install pandas numpy matplotlib seaborn scikit-learn xgboost jupyter`
3. Open `notebooks/01_loan_default_risk.ipynb` and run all cells from the top.
4. The notebook writes `outputs/risk_bands.csv` and `outputs/scored_test_customers.csv`. The second file is not stored in this repository, but running the notebook regenerates it.

## Repository structure

```
loan-default-risk/
├── notebooks/
│   └── 01_loan_default_risk.ipynb
├── outputs/
│   ├── risk_bands.csv
│   └── loan_profit_model.xlsx
├── sql/            (coming soon)
└── README.md
```

## Still to add

- SQL queries on the same data (default rate by income band, loan size, age and education)
- A Power BI dashboard of default rate, risk bands and profit by segment

## Tools

Python, pandas, NumPy, scikit-learn, XGBoost, Matplotlib, Seaborn, Jupyter, Excel

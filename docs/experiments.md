# Experiments Log

## Day 1 — Environment + synthetic data
- Environment verified: `pymc-marketing` 1.0.0, `pymc` 6.0.1, `arviz` 1.3.0.
- Generated `scenario_a_train.csv`: shape (130, 8), weekly from 2023-01-02 to 2025-06-23.
- TV spend range: 23.38–79.09.
- Sales (generated target) summary:

| Stat | Value |
|---|---:|
| mean | 4093.54 |
| std | 318.50 |
| min | 3431.69 |
| 25% | 3813.59 |
| 50% | 4153.55 |
| 75% | 4332.51 |
| max | 4822.22 |

## Day 2 — Data validation, distributions, correlations
**Validation (all passed):**
- Nulls: 0 across all columns
- Duplicate weeks: 0
- Date gaps: all exactly 7 days apart
- Negative spend values: none

**Spend/sales summary statistics:**

| | tv_spend | google_search_spend | tiktok_spend | sales |
|---|---:|---:|---:|---:|
| mean | 51.28 | 30.65 | 21.21 | 4093.54 |
| std | 15.08 | 7.88 | 5.65 | 318.50 |
| min | 23.38 | 9.47 | 5.96 | 3431.69 |
| 25% | 39.54 | 25.63 | 17.29 | 3813.59 |
| 50% | 49.98 | 31.33 | 21.60 | 4153.55 |
| 75% | 64.38 | 36.20 | 24.99 | 4332.51 |
| max | 79.09 | 48.62 | 36.48 | 4822.22 |

**Correlation matrix (channels, controls, sales):**

| | tv_spend | google_search_spend | tiktok_spend | promo | trend | yearly_seasonality | sales |
|---|---:|---:|---:|---:|---:|---:|---:|
| tv_spend | 1.00 | 0.16 | -0.03 | 0.09 | -0.02 | 0.50 | 0.66 |
| google_search_spend | 0.16 | 1.00 | 0.08 | -0.05 | 0.01 | 0.10 | 0.25 |
| tiktok_spend | -0.03 | 0.08 | 1.00 | -0.15 | 0.70 | -0.03 | 0.31 |
| promo | 0.09 | -0.05 | -0.15 | 1.00 | -0.06 | 0.06 | 0.28 |
| trend | -0.02 | 0.01 | 0.70 | -0.06 | 1.00 | 0.00 | 0.32 |
| yearly_seasonality | 0.50 | 0.10 | -0.03 | 0.06 | 0.00 | 1.00 | 0.74 |

**Interpretation:** Channels behave largely independently, as designed. TV has the strongest direct correlation with sales among the three channels (0.66), consistent with TV having the largest true β. `yearly_seasonality` correlates most strongly with sales overall (0.74). The TikTok–trend correlation (0.70) is the one figure that needs to carry forward as a flag into model-building — logged in `decisions.md`.

## Day 1 — Synthetic data generation (scenario_a)

Generated 130 weeks of synthetic weekly data (2023-01-02 to 2025-06-23) with three media
channels (TV, Google Search, TikTok), geometric adstock + logistic saturation, matching
the paper's Table 1 approach of using known generating parameters so later parameter-
recovery checks (Day 11) have ground truth to compare against.

True generating parameters (l_max=8 for adstock):

| Channel | alpha (retention) | lam (saturation) | beta (max effect) |
|---|---|---|---|
| TV     | 0.6 | 0.05 | 1200 |
| Search | 0.4 | 0.08 | 900  |
| TikTok | 0.3 | 0.03 | 700  |

Baseline = 2000, trend = linear (0→20 scaled by 10), yearly seasonality = 10*sin(2πt/52)
scaled by 20, promo effect = 300 (binary, ~15% of weeks), noise ~ N(0, 100). Random seed = 42.

130 weeks was chosen as the initial training set size — this sits within the "small
sample size" caution range flagged in Jin et al. (2017) §6-7, so shape-parameter bias
should be expected and checked for later rather than assumed away.
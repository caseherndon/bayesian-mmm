## Day 1

- Dataset: `data/scenario_a/scenario_a_train.csv`, shape (130, 8), weekly, 2023-01-02 to 2025-06-23.
- TV spend range: 23.38–79.09.
- Sales summary: mean 4093.54, std 318.50, min 3431.69, max 4822.22 (25%: 3813.59, 50%: 4153.55, 75%: 4332.51).
- Ground-truth adstock/saturation parameters used to generate sales logged in `docs/decisions.md` for the Day 11 recovery check.
- Spend/sales plot saved to `figures/scenario_a/day1_spend_and_sales.png`.
- Environment: pymc-marketing 1.0.0, pymc 6.0.1, arviz 1.3.0.
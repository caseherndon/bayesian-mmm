# Decisions Log

## Day 1 — Synthetic data generation
Generated a 130-week synthetic dataset (2023-01-02 to 2025-06-23) with **known** adstock/saturation/beta parameters per channel, so later parameter-recovery checks (Day 11) have ground truth to compare against — same approach as the paper's Table 1 simulation setup.

True generating parameters:

| Channel | α (adstock retention) | λ (saturation) | β (max effect) |
|---|---:|---:|---:|
| TV | 0.6 | 0.05 | 1200 |
| Google Search | 0.4 | 0.08 | 900 |
| TikTok | 0.3 | 0.03 | 700 |

Other generating constants: baseline = 2000, `l_max = 8` weeks for adstock, sales noise ~ N(0, 100).

## Day 2 — Correlation findings and a modeling risk to watch
Data validation passed cleanly with no cleaning required (see `experiments.md` for the checks). Channel-to-channel correlations are mostly weak, as intended, with two exceptions worth flagging **before** model-building starts:

- **TikTok spend vs. `trend`: r = 0.697.** Both were constructed with a linear-in-week-index component (TikTok spend includes `0.1 * week_index`; `trend` is `linspace(0, 20, ...)`), so they move together by construction. This is a real collinearity risk for Day 4–8: the model may struggle to cleanly separate "TikTok's true media effect" from "baseline trend growth," which could bias TikTok's estimated ROAS in either direction. Decision: keep both variables rather than dropping one — removing `trend` would misattribute genuine baseline growth to TikTok, and removing TikTok isn't an option — but treat TikTok's contribution/ROAS estimates with extra skepticism once fitted, and revisit this note if TikTok's posterior looks unstable.
- **TV spend vs. `yearly_seasonality`: r = 0.496.** Same root cause (both share the 52-week sine pattern, by design), but the correlation is moderate rather than severe. Noted for awareness; not treated as a blocker.

No orthogonalization step (like the paper's §8 real-data example, where price/distribution/promotion were regressed apart due to r ≈ -0.98) is needed at this correlation strength — that technique stays in reserve if Day 6–8 diagnostics show the sampler struggling.

**Convention noted**: `date_week` requires `parse_dates=["date_week"]` on every CSV reload — pandas does not preserve datetime dtype through a CSV round-trip. Apply this in every later notebook that loads `scenario_a_train.csv`.

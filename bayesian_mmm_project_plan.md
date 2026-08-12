# Bayesian MMM Replication — Project Conventions & Day 1 Steps

Target: `pymc-marketing-retail-mmm` (4 notebooks — EDA, build/fit/diagnostics, posterior predictive/residuals, media deep-dive) built from scratch, 1 hr/day, PyCharm on Windows.

---

## Conventions (carried over from the MIND project, adjusted for MMM)

- **Repo skeleton**: `notebooks/`, `data/scenario_a/`, `docs/` (`decisions.md`, `experiments.md`, `pitfalls.md`), `figures/`.
- **Git**: commit + push at the end of every day, same message pattern (`"Day N: <what happened>"`).
- **No-shared-state rule, adapted**: your MIND rule was "nothing carries over, redefine every function inline" — that worked because sparse-matrix ops rerun in seconds. NUTS fits don't. So here: cheap things (helper functions, plotting code) get redefined inline same as before; the **fit itself gets saved and reloaded**, never rerun — `idata.to_netcdf("../data/processed/mmm_fit.nc")` at the end of the fit notebook, `az.from_netcdf(...)` at the start of every later one.
- **Sanity-check-before-trust**: same pattern as `ndcg_at_k`'s perfect/no-hit checks. Here it means generating synthetic data with *known* adstock/saturation parameters (paper Table 1 does the same thing) so that Day 10–11's "does the model recover the true parameters" check has ground truth to compare against — not just plausible-looking output.
- **Output check** after every cell that produces a result, same as your steps files.
- **docs/decisions.md** logs modeling choices + why (prior choices, adstock lag length, etc.). **docs/experiments.md** logs each day's actual numbers. **docs/pitfalls.md** logs anything that broke and how it got fixed (e.g., divergences, install issues).

---

## Roadmap (mapped to the repo's 4 notebooks)

| Day | Notebook | Focus |
|---|---|---|
| 1 | setup | Env, repo skeleton, synthetic `scenario_a` data with known ground-truth parameters |
| 2–3 | `01_eda_and_data_prep` | Spend/sales time series, correlations, save cleaned data |
| 4–6 | `02_build_fit_and_diagnostics` (build) | `MMM` object, adstock/saturation config, priors, prior predictive check |
| 7–8 | `02_build_fit_and_diagnostics` (fit) | NUTS fit + diagnostics (R-hat, ESS, BFMI, divergences). **Buffer day included** — this is where your MIND project's divergence-debugging equivalent lives, and where the paper's §2.2 identifiability warning is most likely to bite (Google Search-style saturation-parameter correlation) |
| 9–10 | `03_posterior_predictive_and_residuals` | RMSE/MAE/coverage, residual autocorrelation |
| 11–13 | `04_media_deep_dive` | Parameter recovery check vs. Day 1's true values, contributions, historical ROAS, marginal ROAS (full posterior, not point estimates) |
| 14 | wrap-up | README, cross-check against notebooks, resume bullets |

Days 2 onward get their own `DayN_Steps.md`, written at the start of that day once the prior day's actual saved files/column names/shapes are confirmed — same reason your Day 13/14 files reference exact artifacts from Days 1–12 rather than being drafted in advance.

---

## Day 1 — Environment, Repo Skeleton, Synthetic Data

### Note on scope
Installing `pymc-marketing` pulls in PyMC/PyTensor and is the slowest step — kick it off first and do the folder/git setup while it runs. Ground-truth generating parameters get logged to `docs/decisions.md` today because Day 11's parameter-recovery check needs them, exactly like the paper's Table 1.

### Cell 1 — Start the install (PowerShell), then set up the project while it runs
```powershell
cd C:\Users\<you>\Documents
mkdir bayesian-mmm-retail
cd bayesian-mmm-retail
python -m venv .venv
.venv\Scripts\Activate.ps1
pip install pymc-marketing arviz pandas numpy matplotlib jupyter
```

### Cell 2 — Repo skeleton (while install finishes)
```powershell
mkdir notebooks, data, data\scenario_a, docs, figures, figures\scenario_a
git init
New-Item -ItemType File -Path docs\decisions.md, docs\experiments.md, docs\pitfalls.md
```

### Cell 3 — `.gitignore`
```powershell
"@'
.venv/
__pycache__/
*.nc
'@ | Out-File -Encoding utf8 .gitignore"
```
Note: `*.nc` (netCDF fit files) is excluded — these can get large and aren't source, same reasoning as excluding `data/raw/` in the MIND project.

### Cell 4 — Verify the install
```python
import pymc_marketing
import pymc as pm
import arviz as az

print("pymc-marketing:", pymc_marketing.__version__)
print("pymc:", pm.__version__)
print("arviz:", az.__version__)
```
**Output check:** all three print a version with no `ImportError`. If PyTensor complains about a missing C compiler, log it in `docs/pitfalls.md` and move on — it's a known Windows friction point, not worth solving mid-Day-1.

### Cell 5 — Create the notebook and generate synthetic weekly data
```powershell
cd notebooks
New-Item -ItemType File -Name "01_eda_and_data_prep.ipynb"
```
```python
import numpy as np
import pandas as pd

rng = np.random.default_rng(42)
n_weeks = 130
dates = pd.date_range("2023-01-02", periods=n_weeks, freq="W-MON")

trend = np.linspace(0, 20, n_weeks)
yearly_seasonality = 10 * np.sin(2 * np.pi * np.arange(n_weeks) / 52)
promo = rng.binomial(1, 0.15, n_weeks)

tv_spend = np.clip(50 + 20 * np.sin(2 * np.pi * np.arange(n_weeks) / 52 + 1) + rng.normal(0, 5, n_weeks), 0, None)
google_search_spend = np.clip(30 + rng.normal(0, 8, n_weeks), 0, None)
tiktok_spend = np.clip(15 + 0.1 * np.arange(n_weeks) + rng.normal(0, 4, n_weeks), 0, None)

print("Weeks:", n_weeks)
print("TV spend range:", tv_spend.min().round(2), "-", tv_spend.max().round(2))
```
**Output check:** `n_weeks` is 130; all three spend arrays are non-negative.

### Cell 6 — Generate sales from *known* adstock/saturation parameters
```python
def geometric_adstock(x, alpha, l_max=8):
    weights = alpha ** np.arange(l_max)
    padded = np.concatenate([np.zeros(l_max - 1), x])
    return np.array([
        np.dot(padded[i:i + l_max][::-1], weights) / weights.sum()
        for i in range(len(x))
    ])

def logistic_saturation(x, lam):
    return (1 - np.exp(-lam * x)) / (1 + np.exp(-lam * x))

true_params = {
    "tv":     {"alpha": 0.6, "lam": 0.05, "beta": 1200},
    "search": {"alpha": 0.4, "lam": 0.08, "beta": 900},
    "tiktok": {"alpha": 0.3, "lam": 0.03, "beta": 700},
}

tv_effect = true_params["tv"]["beta"] * logistic_saturation(
    geometric_adstock(tv_spend, true_params["tv"]["alpha"]), true_params["tv"]["lam"])
search_effect = true_params["search"]["beta"] * logistic_saturation(
    geometric_adstock(google_search_spend, true_params["search"]["alpha"]), true_params["search"]["lam"])
tiktok_effect = true_params["tiktok"]["beta"] * logistic_saturation(
    geometric_adstock(tiktok_spend, true_params["tiktok"]["alpha"]), true_params["tiktok"]["lam"])

baseline = 2000
sales = (
    baseline + trend * 10 + yearly_seasonality * 20
    + 300 * promo + tv_effect + search_effect + tiktok_effect
    + rng.normal(0, 100, n_weeks)
)

df = pd.DataFrame({
    "date_week": dates,
    "tv_spend": tv_spend,
    "google_search_spend": google_search_spend,
    "tiktok_spend": tiktok_spend,
    "promo": promo,
    "trend": trend,
    "yearly_seasonality": yearly_seasonality,
    "sales": sales,
})
df.to_csv("../data/scenario_a/scenario_a_train.csv", index=False)
print(df.shape)
df["sales"].describe()
```
**Output check:** shape is `(130, 8)`; `sales` is positive throughout with no `NaN`.

### Cell 7 — Log ground-truth parameters to `docs/decisions.md`
Add an entry recording `true_params` exactly as generated above, plus: *"130-week synthetic dataset generated with known adstock/saturation parameters, matching the paper's Table 1 approach, so Day 11's parameter-recovery check has ground truth to compare against."*

### Cell 8 — Quick visual check
```python
import matplotlib.pyplot as plt

fig, axes = plt.subplots(2, 1, figsize=(10, 6), sharex=True)
axes[0].plot(df["date_week"], df["sales"])
axes[0].set_title("Sales")
axes[1].plot(df["date_week"], df["tv_spend"], label="TV")
axes[1].plot(df["date_week"], df["google_search_spend"], label="Google Search")
axes[1].plot(df["date_week"], df["tiktok_spend"], label="TikTok")
axes[1].legend()
axes[1].set_title("Media spend")
plt.tight_layout()
plt.savefig("../figures/scenario_a/day1_spend_and_sales.png")
plt.show()
```
**Output check:** sales should show visible seasonality (yearly sine) and an upward trend; spend series should look like distinct, noisy patterns per channel, not identical.

### Cell 9 — Log the day, commit, push
```powershell
cd C:\Users\<you>\Documents\bayesian-mmm-retail
git add .
git commit -m "Day 1: environment setup, repo skeleton, synthetic scenario_a data with known ground-truth parameters"
```
Add a one-line entry to `docs/experiments.md`: dataset shape, date range, and a note that ground-truth parameters live in `docs/decisions.md` for the Day 11 recovery check.

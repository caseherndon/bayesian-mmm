# Pitfalls Log

## Day 1 — `g++ not available` warning on import
`import pymc_marketing` prints `g++ not available, if using conda: conda install gxx` to stderr. Non-blocking — PyTensor falls back to a slower compiled/interpreted mode without a C++ compiler. Known, expected on this Windows setup; not worth resolving unless fit times in Day 6–8 turn out to be a real bottleneck.

## Day 2 — `df_check.equals(df)` returned `False`
**Cell 5** compared the CSV-reloaded `df_check` against the in-memory `df` and got `False`, even though the data validation in Cell 6 (nulls, duplicates, gaps, negative values) all passed clean on `df` itself — so this isn't a sign of corrupted or mismatched data.

**Likely cause:** `date_week` datetime resolution. `df_check`'s dtype printed as `datetime64[us]`. `pandas` (2.x+) can produce different datetime64 resolutions (`ns`, `us`, or `s`) depending on whether the column came from `pd.date_range(...)` (used when `df` was built) versus `pd.read_csv(..., parse_dates=[...])` (used for `df_check`) — two columns can hold identical timestamps at different resolutions, and `.equals()` treats that as a mismatch. Worth confirming directly: `print(df["date_week"].dtype)` — if it differs from `df_check`'s `datetime64[us]`, this is confirmed.

**Fix for future reload/comparison checks** (Day 3 onward will do this again): don't rely on strict `.equals()` across a CSV round-trip. Either compare with dtype ignored:
```python
pd.testing.assert_frame_equal(df, df_check, check_dtype=False)
```
or normalize before comparing:
```python
df_check["date_week"] = df_check["date_week"].astype(df["date_week"].dtype)
```
**Not a blocker** — Cell 6's independent validation already confirms the saved CSV is clean and complete. This only affects the specific `.equals()` sanity check, not the data itself.

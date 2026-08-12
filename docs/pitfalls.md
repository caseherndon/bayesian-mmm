## Day 1

- **Notebook kernel not connected to project `.venv`.** Initial `import pymc_marketing`
  ran against a global Python 3.13 interpreter instead of the project's `.venv`, causing
  a pydantic `SchemaError` on import ('cls' must be valid as the first argument to
  'isinstance') — root cause was a pydantic version older than 2.8.0, which doesn't
  support Python 3.13. Fixed by pointing PyCharm's interpreter at the existing `.venv`
  (Settings → Python Interpreter → Add → **Select existing interpreter**, since
  "Generate new" fails when a venv already exists at that path), reinstalling packages
  there, and restarting the kernel.
- **`g++ not available`** warning on `import pymc_marketing` (PyTensor C compiler check).
  Known Windows friction point per the setup notes — not solved, moving on; watch for
  whether it affects NUTS sampling speed once fitting starts (Day 7–8).
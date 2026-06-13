# Project Rules

> Short, actionable rules and conventions for this project. Loaded automatically by /aif-implement.

## Rules

- Never copy `config/west.yml` between branches `main` and `zmk-v0.3` without a revision-by-revision review.
- Always keep `config/mix.json` in sync with `config/mix.keymap` when keys are added or removed.
- Firmware builds are authoritative only when produced by GitHub Actions — never trust local-build claims.
- Every change to one half (`mix_left`) must be evaluated against the other half (`mix_right`) before merge.
- Never add traditional source files (`.c`, `.cpp`, `.py`, `.ts`, etc.) — this repository is declarative configuration only.

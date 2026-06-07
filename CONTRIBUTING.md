# Contributing to graphed-core

Part of the `graphed` project, governed by the gated three-role pipeline. The root
[`graphed-project/CLAUDE.md`](https://github.com/graphed-org/graphed-project-mvp) and the project plan
(`graphed-project-plan-gated.md`) are authoritative; the plan always wins.

## Guardrails (M1)

- The graph lives in **Rust**, not Python. This crate **must not import awkward** (enforced by a
  frozen test).
- **No optimization** here — DCE/CSE/canonicalization/stage fusion are M4. (CSE already falls out
  of hash-consing.)
- Any `unsafe` must carry a line-by-line `// SAFETY:` justification (there is none at M1).

## Integrity rules — NON-NEGOTIABLE (plan A.7 / B.6)

Never edit/skip/weaken anything under `tests/frozen/**`; never lower a threshold or relax CI;
never stub the thing under test; never flood `# type: ignore` / `except: pass` / unjustified
`unsafe`. If a frozen test seems wrong, file a Test Dispute under
`.graphed/<Mx>/disputes/<test_id>.md` and stop.

## Local gates

```bash
python -m venv .venv && . .venv/bin/activate
pip install -e ".[dev,docs]"          # builds the extension via maturin
ruff check . && ruff format --check . && mypy
pytest tests/frozen/m1
# Rust (needs libpython on the loader path for the test binary):
export DYLD_FALLBACK_LIBRARY_PATH=$(python -c 'import sysconfig;print(sysconfig.get_config_var("LIBDIR"))')  # macOS
export LD_LIBRARY_PATH=$(python -c 'import sysconfig;print(sysconfig.get_config_var("LIBDIR"))')              # Linux
cargo fmt --check && cargo clippy --all-targets -- -D warnings && cargo test
RUSTFLAGS="--cfg loom" cargo test --lib loom_model     # model-check the locking discipline
sphinx-build -W -b html docs docs/_build/html
```

If both `VIRTUAL_ENV` and `CONDA_PREFIX` are set, run maturin with `env -u CONDA_PREFIX`.

CI runs the Python frozen suite on the full A.5 matrix (building the extension on each OS), a
dedicated Rust job (fmt + clippy + cargo test + loom), free-threaded 3.14/3.14t, a wheel
build, and `sphinx -W`. The matrix is the gate of record.

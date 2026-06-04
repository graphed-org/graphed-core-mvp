# CLAUDE.md — graphed-core

Defers to the root **`graphed-project/CLAUDE.md`**; the **project plan
(`graphed-project-plan-gated.md`) always wins.** This file distills **milestone M1** and the
relevant guardrails.

## What this repo is

`graphed-core`: the **Rust+PyO3 thread-safe interned graph IR** — the spine every other package
builds on. The graph lives in **Rust**, not Python. Later milestones add the optimizer (M4), the
execution protocol (M7 contract), and plan serialization (M8) here; **M1 is the IR only**.

> Hard guardrails (A.4 / M1): **MUST NOT import awkward** · **no optimization** (that is M4) ·
> graph lives in Rust · any `unsafe` needs a line-by-line `// SAFETY:` justification (none at M1).

## M1 — what is implemented

- `Node` = `Source | Op | Reduction | External | Stage` (`Stage` is constructed only by M4).
- **Interning by structural hash**: structurally identical nodes share one `NodeId`, so
  `node_count()` == number of distinct structural keys. CSE falls out of hash-consing.
- `External` carries a **`PayloadDescriptor`** (kind, content hash, framework+version, I/O schema,
  preprocessing ref) that **participates in the structural hash** (plan A.3.1).
- **Total-order float hashing**: every `NaN` interns to itself; `0.0` and `-0.0` are distinct
  (canonicalizing them is M4's job). `ParamMap` keys are sorted → deterministic.
- **`Send + Sync` via a single `Mutex`** (documented locking discipline): interning is an atomic
  read-modify-write; correct under the GIL and free-threaded 3.14t. Model-checked with `loom`
  (`loom_model` test) + stressed from many threads (Rust + Python).
- PyO3 (`Bound` API, 0.28, `#[pymodule(gil_used = false)]`): `GraphStore` with
  `add_source/add_op/add_reduction/add_external/mark_output/node_count/to_dot`, `PayloadDescriptor`,
  `version()`. `to_dot()` is byte-stable.

## Layout

```
src/param.rs   ParamValue + ParamMap (total-order float hashing)
src/node.rs    NodeKey (structural identity) + PayloadDescriptor
src/store.rs   GraphStore (Mutex-guarded intern table) + Rust tests + loom model
src/lib.rs     PyO3 bindings
python/graphed_core/  __init__.py re-export + __init__.pyi stubs + py.typed
tests/frozen/m1/      the frozen acceptance suite (Python)
```

## Gates (run before pushing)

`ruff` + `ruff format` · `mypy --strict` (on the stubs) · `pytest tests/frozen/m1` ·
`cargo fmt --check` · `cargo clippy --all-targets -- -D warnings` · `cargo test` (set
`DYLD_FALLBACK_LIBRARY_PATH`/`LD_LIBRARY_PATH` to the python libdir so the test binary links
libpython) · `RUSTFLAGS="--cfg loom" cargo test --lib loom_model` · `sphinx-build -W`. If both
`VIRTUAL_ENV` and `CONDA_PREFIX` are set, run maturin with `env -u CONDA_PREFIX`.

Coverage note: M1's new code is Rust; coverage.py only sees the thin re-export. Breadth is carried
by the 21 frozen Python tests + 6 `cargo` tests + the loom model, not a coverage %.

Status: see `.graphed/state.json`.

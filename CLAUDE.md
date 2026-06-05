# CLAUDE.md — graphed-core

Defers to the root **`graphed-project/CLAUDE.md`**; the **project plan
(`graphed-project-plan-gated.md`) always wins.** This file distills **milestone M1** and the
relevant guardrails.

## What this repo is

`graphed-core`: the **Rust+PyO3 thread-safe interned graph IR + optimizer** — the spine every other
package builds on. The graph lives in **Rust**, not Python. **M1** is the IR; **M4** is the optimizer.
Later milestones add the execution protocol (M7 contract) and plan serialization (M8) here.

> Hard guardrails (A.4): **MUST NOT import awkward** · graph + optimization live in Rust · any
> `unsafe` needs a line-by-line `// SAFETY:` justification (there is none).

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

## M4 — the optimizer (DCE/CSE + equality-saturation stage fusion)

- **`RewriteEngine` trait** (`optimizer/engine.rs`) is the engine boundary — egg-free types in/out,
  so **no `egg` types leak past it** (Phase-2 egglog swap is one trait impl). `EggEngine` is the MVP.
- **Genuine equality saturation**: the IR is loaded into an egg e-graph; **SOUND rules only**
  (commutativity of `+`/`*`, additive/multiplicative identities — mask-fusion/field-collapse are
  excluded as domain-dependent/unsound). Extraction is an **O(N) cost-quotient** (keep the earliest/
  simplest node per eclass), because egg's recursive `Extractor` is O(depth·N) and blows up to O(N²)
  on the deep chains a systematics graph produces — exactly what the benchmark guards against.
- **DCE** (reachability from outputs) and **CSE** (hash-consing) are plain passes **outside** the
  engine. **Stage fusion** groups maximal op-runs between boundaries into `Stage` nodes (a fan-out op
  or a boundary ends a stage; fusion never crosses a boundary).
- `reduce` / `reduce_incremental` / `reduction_report` exposed via PyO3. Reduction is **byte-stable**.
- A **10,000-node systematics graph reduces to O(stage) nodes in < 1 s**; a **CI benchmark FAILS if
  reduction time grows super-linearly across {1k,2k,4k,8k}** (`tests/frozen/m4/test_benchmark.py`).
- Semantic equivalence (reduced vs un-reduced) is proven by a toy integer interpreter in the Rust
  suite; the full numpy/awkward-backend executor equivalence is M7.

## M8 — plan serialization (the durable, canonical IR form)

- **`GraphStore.serialize()` / `GraphStore.deserialize()`** (Rust, `src/serialize.rs`): a compact
  length-prefixed binary encoding with a version magic (`GIR1`). Nodes are written in interned-id
  order and `ParamMap` is key-sorted, so **identical graphs serialize byte-identically** (the M8
  determinism gate) and a round trip rebuilds a structurally identical store. This is the
  **canonical durable representation** (A.3.1) — never cloudpickle.
- **`DurablePlan`** (pure Python, `python/graphed_core/plan.py`): the glossary "Plan" — wraps the
  serialized IR with executor metadata (partitions, read columns, reduction/stopping/locality/
  resource specs). Versioned, byte-identical `to_bytes`/`from_bytes`; content-addressed `task_id`
  (SHA-256 over IR identity + process spec + partition → cache-poisoning-safe). `OpSpec` resolves
  callables by **import ref** (no source files) or, only for genuinely opaque callables, embeds them
  **by value** (cloudpickle) flagged `opaque=True` (a preservation risk M9 surfaces).
- The content-addressed **Store / checkpoint / resume / error-harvesting** layer is **M8's other
  half in `graphed-checkpoint`** (it consumes `DurablePlan` + `task_id`).

## Layout

```
src/param.rs        ParamValue + ParamMap (total-order float hashing + optimizer tokens)
src/node.rs         NodeKey (structural identity) + PayloadDescriptor + Stage (fused op-DAG)
src/serialize.rs    M8 canonical IR codec (GIR1: versioned, deterministic, byte-identical)
src/store.rs        GraphStore (Mutex-guarded intern table) + reduce() + Rust tests + loom model
src/optimizer/      RewriteEngine + EggEngine (engine.rs), DCE/CSE/stage-fusion pipeline (mod.rs)
src/lib.rs          PyO3 bindings (incl. reduce / reduction_report / serialize / deserialize)
python/graphed_core/  __init__.py re-export + stubs (__init__.pyi + graphed_core.pyi) + plan.py
tests/frozen/m1/    the M1 IR acceptance suite (Python)
tests/frozen/m4/    the M4 optimizer suite: reduce, systematics, super-linear benchmark
tests/frozen/m8/    the M8 suite: IR codec round trip + DurablePlan determinism/task_id/no-source
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

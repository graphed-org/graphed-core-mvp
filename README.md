# graphed-core

Rust+PyO3 **thread-safe interned graph IR** for `graphed` (milestone M1). The graph lives in Rust;
this package **MUST NOT import awkward**. Part of the [`graphed-org`](https://github.com/graphed-org)
project; see [`graphed-project`](https://github.com/graphed-org/graphed-project-mvp) for the root
guidance and the authoritative plan.

## What it does (M1)

A `GraphStore` that interns nodes by structural hash: structurally identical nodes share one
`NodeId`, so `node_count()` equals the number of distinct structural keys. Nodes are
`Source | Op | Reduction | External | Stage`; `External` carries a `PayloadDescriptor` (kind,
content hash, framework+version, I/O schema, preprocessing ref) that participates in the hash.
Float params use a deterministic total order (NaN interns to itself; `0.0` and `-0.0` are distinct).
The store is `Send + Sync` and safe to build from many threads under the GIL and free-threaded
3.14t. `to_dot()` is byte-stable.

```python
import graphed_core as gc
s = gc.GraphStore()
src = s.add_source("events", {"uri": "f.root", "tree": "Events"})
pt = s.add_op("pt", [src])
assert s.add_op("pt", [src]) == pt        # interned
s.mark_output(s.add_reduction("sum", [pt]))
```

## The execution contract + monitor seam (`graphed_core.execution`)

Beyond the IR, this package owns the *pure-Python, data-only* execution contract every executor
implements: `Plan`, `Task`, `Partition`, `StopCondition`, `Executor`, plus the dependency-free
`SequentialRunner` baseline. It imports no awkward/numpy/web — it is a stable, minimal seam.

Part of that contract is the **live-observability seam (M37)**: `TaskEvent` / `TaskPhase` and the
`Monitor` / `WorkerProfiler` protocols. An executor *emits* `TaskEvent`s through a `Monitor` so a
dashboard can watch a run — but the seam is deliberately **render- and transport-agnostic** (it knows
nothing of websockets or Perspective; `graphed-debug` supplies those). It is also **passive**:
emission is best-effort and a `Monitor` that raises is swallowed, so attaching one never changes a
result. The vocabulary lives here, in `graphed-core`, because it is shared by *every* executor —
the principle that a shared primitive belongs at the layer it serves.

## Develop

```bash
pip install -e ".[dev,docs]"
maturin develop                  # build the extension into the venv
ruff check . && ruff format --check . && mypy
pytest tests/frozen/m1
cargo test                       # pure-Rust IR + locking stress test
cargo clippy --all-targets -- -D warnings
```

Guardrails: no optimization (that is M4); no awkward; the graph lives in Rust, not Python.
Status: see `.graphed/state.json` and `CLAUDE.md`.

# graphed-core

Rust+PyO3 **thread-safe interned graph IR** for `graphed` (milestone M1). The graph lives in Rust;
this package **MUST NOT import awkward**. Part of the [`graphed-org`](https://github.com/graphed-org)
project; see [`graphed-project`](https://github.com/graphed-org/graphed-project) for the root
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

# M32 attempts — graphed-core (the reference executor lives with the execution contract)

## Iteration 0 — 2026-06-13 (freeze-M32-0)

- USER finding: SequentialRunner living in graphed.write is a layering artifact — it is a
  general Executor of the Plan contract, not a write concept; it landed there only because the
  M20 write-base was the first dependency-free caller (the frontend may not import
  graphed-exec-local). USER directive: move it to graphed_core.execution, NO re-export from
  graphed.write, point all consumers at graphed_core.execution; sanctioned the resulting frozen
  amendments across 4 consumer repos.
- This repo (the destination): added LocalResources (public reference WorkerResources) +
  SequentialRunner (the dependency-free reference Executor) to graphed_core.execution, beside
  the Plan/Executor/WorkerResources contract they implement; exported from __init__ (+ .pyi
  stubs so consumers' mypy --strict sees them). Behavior is byte-identical to the old
  graphed.write impl (key-ordered in-process fold, LocalResources open_once).
- frozen m32 (6): SequentialRunner satisfies the Executor protocol; runs a plan to the reduced
  result in KEY order; empty plan -> identity; key order independent of task submission order;
  LocalResources opens each uri once; resources reach process via a single per-run
  LocalResources. Non-vacuity for a RELOCATION: reverting the execution.py addition makes the
  import fail (collection error).
- Gates (precommit script): pytest all green · mypy --strict clean (incl. stubs) · ruff/sphinx
  clean. Consumers (graphed, numpy, awkward, histogram, the forks) migrate next and re-freeze.

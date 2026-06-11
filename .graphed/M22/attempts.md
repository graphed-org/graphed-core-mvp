# M22 attempts — graphed-core (output-scoped reduction & serialization)

## Iteration 0 — TEST_AUTHORING/TEST_SANITY/IMPLEMENTING — 2026-06-10 (freeze-M22-0)

- Fixes the compile_ir output-accumulation footgun (user-confirmed plan): outputs are a property
  of the COMPILE REQUEST, not store state.
- frozen suite tests/frozen/m22 (7 tests); NON-VACUOUS (7/7 fail pre-implementation on the
  missing outputs= kwarg).
- Rust: GraphStore.reduce_with_outputs (explicit set, ids validated, stored marks ignored),
  serialize::serialize_with (bytes flag exactly the requested set; same byte FORMAT — serialize()
  is the marks-set case); PyO3: reduce/serialize gain outputs=None, IncrementalReducer.finalize
  gains outputs=None (None = the marks path every earlier frozen suite pins — fully additive).
  Rust unit tests cover the new store/serialize branches (llvm-cov gate scope).
- gates: cargo test 26/26 (local dyld note: needs DYLD_FALLBACK_LIBRARY_PATH to libpython on this
  Mac; CI's Linux LD_LIBRARY_PATH path unaffected) · pytest 125/125 · clippy -D warnings clean ·
  cargo fmt clean · mypy clean · ruff clean.

## Iteration 1 — USER-AUTHORIZED removal of mark_output + llvm-cov gate raise — 2026-06-10 (freeze-M22-1)

- USER DIRECTIVE: "mark_output is clearly no longer needed, remove it, respin tests, and push"
  + "increase the llvm-cov requirement to 90%".
- llvm-cov gate raised 85 -> 90 (measured local: 92.72% lines).
- The PyO3 `mark_output` binding and stub are REMOVED (the Rust-internal store fn remains: the
  reduced-store rebuild and unit tests use it). `reduce_incremental`/`reduction_report` gained
  outputs= for a consistent surface. An unreduced store now simply has no outputs; outputs exist
  only in compile requests and the artifacts they produce.
- RESPUN frozen suites (all amendments under this authorization): m1 (builder + the validity
  test, now reduce(outputs=) accept/reject + the mutator's absence pinned), m4 (reduce /
  benchmark / systematics / topologies builders return their output; report+incremental take
  outputs=), m8 (codec + DurablePlan helpers serialize(outputs=)), m9 (output flag pinned via
  the serialized artifact), m10 (incremental reducer + sound rules + maximal fusion + stage
  members; outputs() pinned as request-order on artifacts), m22 (stale-marks premise moot —
  re-pinned as fresh-request equality + no-mutator).
- gates: cargo test 26/26 · clippy -D warnings clean · fmt clean · llvm-cov 92.72% (>=90) ·
  pytest 118/118 · mypy clean · ruff clean · sphinx -W clean.

## 2026-06-11 — engineering documentation pass (user request)

- docs/design.rst rewritten from a 33-line sketch into the full "How graphed-core works"
  walkthrough: the IR (interning, structural identity, token encoding incl. float bit patterns,
  the Mutex discipline, compile-request-scoped outputs); the OPTIMIZER as the pedagogical
  centerpiece (the four-pass pipeline; an e-graph/equality-saturation primer; the sound rule
  set and why domain rules are excluded; determinism-over-wall-clock budgets; the O(N^2)
  extraction trap and the O(N) earliest-representative quotient escape with the CI benchmark
  tie-in; CSE-after-canonicalization and why; both fusion modes with their determinism
  arguments; a worked example traced through every pass; the incremental reducer with the
  constructor-local one-pass argument and its tripwire); the GIR1 codec; the plan layer; a
  file-by-file reading map. docs/index.rst rewritten to match (the old text still claimed
  "no optimization here"). sphinx -W clean.

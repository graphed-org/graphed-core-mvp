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

# M33 attempts — graphed-core (LocalResources is bounded + closeable)

## Iteration 0 — 2026-06-13 (freeze-M33-0)

- Review finding P0-1: LocalResources.open_once never closed handles — a long-lived worker (a
  persistent process pool over many files) accumulated every uproot handle for its whole
  lifetime (fd/mmap leak). FIX: bound simultaneously-open handles to max_open (default 128),
  LRU-evict + close the least-recently-used over the bound; add close() releasing all handles;
  SequentialRunner closes its resources in a finally. open_count kept as diagnostic.
- frozen m33 (5): reuse within bound (one open); LRU eviction closes the LRU and keeps the MRU;
  reopen after eviction; close() releases all + reusable after; SequentialRunner closes
  resources at end of run. Non-vacuous (eviction/close assertions fail against the old
  unbounded no-close impl).
- m32 unaffected (single-file open_once + runner result unchanged under the bound). Gates green
  via the precommit script. graphed-exec-local will reuse THIS LocalResources (dedup, P3-6).

# Async-exns analysis — target worklist

Selection axes: **importance** (centrality/usage) × **async-exn yield** (presence of
signal handlers, timers/timeouts, Ctrl+C/`Sys.Break`, finalisers, or cleanup that
runs during unwinding — the surface that actually breaks under the new semantics).
Follow `ANALYSIS_PLAYBOOK.md` for each. Done: **alt-ergo** (`findings_alt-ergo.md`),
**dune** (`findings_dune.md` — synchronous signal discipline tames everything except
`Stack_overflow`; no day-one bug for one-shot CLI, but **daemon/`--watch`/RPC mode
loses crash-recovery + accumulates cleanup leaks** → needs a per-build-iteration
`with_async_exns` boundary),
**opam** (`findings_opam.md` — raising SIGINT handler (`Sys.catch_break true`) → blast
radius is `Sys.Break`; GENUINE bug: async Ctrl+C bypasses error-as-value rollback →
partially-installed package left un-rolled-back + terminal left in raw mode).

The two large programs named in the design doc are also **DONE**:
- **Rocq/Coq** (`git+https://github.com/rocq-prover/rocq.git`) — **DONE**
  (`findings_rocq.md`: no genuine bug — the famous "Ctrl+C makes a false proof succeed"
  case is already guarded by `CErrors.is_async`, and state rolls back from immutable
  snapshots, not handlers; needs `Sys.with_async_exns` on the prompt/IDE recovery loops +
  the Unix timeout handler, and the `.vo` writer should write-then-rename).
- **Frama-C** (`git+https://git.frama-c.com/pub/frama-c.git`) — **DONE**
  (`findings_frama-c.md`: only `Sys.Break` matters; no batch-mode bug, but `-server` mode
  has two real bugs — project-switch restore and the "apply once" reset are skipped on an
  interrupting Ctrl+C; cancellation/prover-timeout are cooperative. GUI ships separately,
  not yet reviewed).

## The next 10

| # | Package | Why important | Expected async-exn surface | Clone |
|---|---------|---------------|----------------------------|-------|
| 1 | **dune** | The OCaml build system — ubiquitous | SIGINT/SIGTERM handling, subprocess group management, temp-file/lock/build-dir cleanup, large catch-alls around the build loop | `git+https://github.com/ocaml/dune.git` — **DONE** (`findings_dune.md`: synchronous signal discipline tames everything except `Stack_overflow`; no day-one bug for one-shot CLI, but daemon/`--watch`/RPC loses crash-recovery + accumulates leaks → per-build `with_async_exns`) |
| 2 | **opam** (`opam-client`) | The OCaml package manager — ubiquitous | Ctrl+C during builds, lock files, temp-dir cleanup, `Unix` signal handling, rollback-on-interrupt invariants | `git+https://github.com/ocaml/opam.git` — **DONE** (`findings_opam.md`: raising SIGINT handler (`Sys.catch_break`) → GENUINE BUG: async Ctrl+C bypasses error-as-value rollback, leaving a partly-installed package + terminal in raw mode; throwing side already handled) |
| 3 | **lwt** | Foundational concurrency lib; enormous reverse-dep set | `Lwt_unix` signal handling, `Lwt_timeout`, cancellation, `Lwt.finalize`/`Lwt.catch` cleanup combinators (the bracket pattern at scale) | `git+https://github.com/ocsigen/lwt.git` — **DONE** (`findings_lwt.md`: clean non-target — C-level self-pipe signals, no `Sys.Break`; only `Stack_overflow` reaches it and the default exception filter already skips it; one genuine change only for apps that opt into `handle_all` recovery) |
| 4 | **utop** | The standard REPL; **explicitly cited in the doc** as relying on old Ctrl+C behaviour | `Sys.Break`/SIGINT handling around the read-eval loop — likely a direct, high-signal hit | `git+https://github.com/ocaml-community/utop.git` — **DONE** (`findings_utop.md`: GENUINE BUG — Ctrl+C during evaluation now kills the session instead of returning to the prompt; fix = `Sys.with_async_exns` inside the loop + custom `Term` handler) |
| 5 | **why3** | Verification platform; foundational (Frama-C, SPARK, others depend on it) | Prover-call timeouts, `Unix.alarm`/`setitimer`, external process management, interrupt handling | `git+https://gitlab.inria.fr/why3/why3.git` — **DONE** (`findings_why3.md`: barely affected — timeouts enforced by a C helper process, arrive as values not exceptions; no fixes required) |
| 6 | **eio** | Modern effects-based concurrency/IO | Signal handling, structured cancellation, `Switch`/resource release on unwind, `Fun.protect`-style finalisers; effects × async-exns is directly relevant (cf. the doc's "Async exns x effects" changeset) | `git+https://github.com/ocaml-multicore/eio.git` — **DONE** (`findings_eio.md`: clear — signal-safe SIGCHLD only, cooperative cancellation; fragile clean-up via a few core primitives + one scheduler `with_async_exns` design call) |
| 7 | **merlin** + **ocaml-lsp-server** | Editor tooling used by ~every OCaml dev | Long-running daemon, signal handling, per-request timeouts/cancellation, catch-alls around request handling | `git+https://github.com/ocaml/merlin.git` / `git+https://github.com/ocaml/ocaml-lsp.git` — **DONE** (`findings_merlin-ocaml-lsp.md`: GENUINE BUG — merlin `server` mode self-corrupts after a stack overflow skips its per-query state restore; both lose per-request crash-recovery; fix = per-request `with_async_exns`) |
| 8 | **alcotest** | The widely-used test framework (CI everywhere) | Per-test **timeouts** (alarm-based), catch-all around each test to report+continue, output/temp cleanup | `git+https://github.com/mirage/alcotest.git` — **DONE** (`findings_alcotest.md`: minor — no timer/signal use at all; a stack overflow in one test now aborts the whole run, and skipped stdout-restore hides the crash in a log file; per-test `with_async_exns` + interruption-safe stream restore) |
| 9 | **unison** | Classic, widely-deployed file synchronizer | Signal handling, transactional cleanup on interrupt (partial-transfer invariants), Ctrl+C during sync | `git://github.com/bcpierce00/unison.git` — **DONE** (`findings_unison.md`: largely safe — file consistency rides on an on-disk commit log + atomic archive flip + at_exit lock release, not on catching Ctrl+C; two `with_async_exns` sites (text UI exit, socket-server cleanup)) |
| 10 | **irmin** | Important Git-like store (Tezos/MirageOS ecosystem) | On-disk store consistency invariants, resource/handle cleanup on unwind, `Lwt.finalize` usage | `git+https://github.com/mirage/irmin.git` — **DONE** (`findings_irmin.md`: safe today — no own interrupting exn; on-disk consistency via atomic control-file rename; one real change in `irmin-server`'s per-request loop) |

## Notes on selection

- **High-yield/lower-effort first:** `utop` (4) is small and an almost-certain direct
  hit (`Sys.Break`); good early win. `alcotest` (8) is small with a clear timeout.
- **Lwt/Eio (3, 6)** are *library* analyses — the interesting artefacts are the
  cleanup/cancellation combinators themselves (`Lwt.finalize`, `Switch`), which
  shape how every downstream app behaves. Distinct flavour from app-level analysis.
- **Tarides-relevant cluster:** dune, opam, eio, merlin/ocaml-lsp, irmin.
- **Considered but lower priority:** `async`/`core_unix` (Jane Street authored the
  semantics — likely already handled); `cohttp`/servers (graceful-shutdown pattern,
  but Lwt-derived so overlaps #3); `batteries`/`containers` (resource combinators,
  but declining usage); pure compilers/formatters like `ocamlformat`, `js_of_ocaml`
  (little-to-no signal surface; JS target has none).
- **Blast-radius rule (from the playbook):** for each, the first deliverable is the
  *blast radius* — most will reduce to one or two async exns. Confirm before deep
  reading.

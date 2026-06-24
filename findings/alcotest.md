# Alcotest — first look at the asynchronous-exception change

Repo: `mirage/alcotest` @ `b26e58f62c93895f24a5ee48b550f274db74ee55` (shallow clone in `./alcotest`).
What this pass covers: a first search for code in Alcotest that works correctly
today but would break under OxCaml's new rules for "asynchronous exceptions".

Background in one sentence: an *asynchronous exception* is an event
— pressing Ctrl+C, running out of stack, or an exception thrown from a signal
handler, a GC finaliser, or a memprof callback — that can surface at almost any
point in the program. Today an ordinary `try … with` catches these; under the new
rules it does not, unless you wrap the region in a new helper (`Sys.with_async_exns`,
which turns an asynchronous exception back into an ordinary one so normal handlers
see it again). We call these "asynchronous exceptions" below.

## Bottom line

- Alcotest installs **no signal handlers, no timers, and no finalisers of its own**.
  There is no `Sys.set_signal`, no `Sys.catch_break`, no `Sys.Break`, no
  `Unix.setitimer`/`Unix.alarm`, no `Gc.finalise`, and no memprof callback anywhere
  in the framework (engine or any of the `alcotest` / `alcotest-lwt` /
  `alcotest-async` / `alcotest-mirage` packages, including the C stubs). So there is
  **nothing on the throwing side** — no exception that needs to be re-thrown the new
  way with `Sys.raise_async`, and the "Ctrl+C comes for free via `Sys.catch_break`"
  rule is simply not engaged here.
- The **only** asynchronous exception Alcotest can actually meet is `Stack_overflow`
  (plus `Out_of_memory` in its synchronous, non-GC form, which the design doc keeps
  as an ordinary exception). A test framework runs arbitrary, possibly deeply
  recursive user code, so `Stack_overflow` from inside a test is realistic.
- **Alcotest is a per-test recovery loop, and that is what the change can affect — but
  whether it does depends on the runner.** Each test runs inside
  [`protect_test`](https://github.com/mirage/alcotest/blob/b26e58f62c93895f24a5ee48b550f274db74ee55/src/alcotest-engine/core.ml#L183-L195),
  which catches the test's exception via the concurrency monad's `M.catch`, records it as
  a failed test, and lets the loop continue. That catch is **not** a fixed wildcard — it
  is whatever the engine was instantiated with, and the backends differ on `Stack_overflow`:
  - **default synchronous runner** (identity monad) and **`alcotest-async`** catch
    `Stack_overflow` today, so under OxCaml it stops being caught and **the whole run
    aborts instead of recording one failed test** — a real behaviour change.
  - **`alcotest-lwt`** uses `Lwt.catch`, whose default exception filter already declines
    `Stack_overflow`, so that runner already aborts on a stack-overflowing test today —
    **no change** (see "Which exceptions are affected").
- **The clean-up that an asynchronous `Stack_overflow` would skip is the restoration
  of the program's redirected standard output/error.** For each test, Alcotest
  redirects the operating-system stdout/stderr to a per-test log file and restores
  them afterwards
  ([`with_redirect`](https://github.com/mirage/alcotest/blob/b26e58f62c93895f24a5ee48b550f274db74ee55/src/alcotest/alcotest.ml#L98-L107),
  C stubs
  [`alcotest_before_test`/`alcotest_after_test`](https://github.com/mirage/alcotest/blob/b26e58f62c93895f24a5ee48b550f274db74ee55/src/alcotest/alcotest_stubs.c#L115-L149)).
  The restore (`after_test`) and the log-file close
  ([`log_trap.ml`](https://github.com/mirage/alcotest/blob/b26e58f62c93895f24a5ee48b550f274db74ee55/src/alcotest-engine/log_trap.ml#L102-L113))
  run only on the normal path; an asynchronous `Stack_overflow` skips both, leaving the
  process's stdout/stderr still pointed at that test's log file (and a leaked file
  descriptor). Because the run is aborting anyway this is mostly cosmetic, but it has a
  concrete bad effect: the final report and any `at_exit` flush land in the log file
  instead of on the terminal — verified below.
- **One place catches the exception to decide what to do next**: the top-level
  command-line wrapper asks Cmdliner to catch exceptions and turn them into a non-zero
  exit code
  ([`cli.ml`](https://github.com/mirage/alcotest/blob/b26e58f62c93895f24a5ee48b550f274db74ee55/src/alcotest-engine/cli.ml#L161-L171),
  `Cmd.eval_value ~catch:and_exit`). Under OxCaml an asynchronous `Stack_overflow`
  bypasses that too, so the tidy "internal error" exit code is replaced by the
  runtime's own uncaught-exception termination.
- Everything else checked (`Out_of_memory`, the `at_exit` flush, the
  backtrace-flag and global-formatter restorations, the per-test catch's other
  exception cases, the Lwt/Async/Mirage variants' cooperative timeouts, and the
  JavaScript build) is clear.

## Which exception is affected: `Stack_overflow`

### Why there is nothing on the throwing side

A search of the whole repository for `signal`, `Sys.Break`, `catch_break`,
`setitimer`, `alarm`, `Gc.finalise`, `create_alarm`, and `Memprof` — across every
`.ml`, `.mli`, and `.c` file — finds **none of them**. Alcotest never installs an
OCaml signal handler, never arms a timer, and registers no GC finaliser or memprof
callback. The C stubs
([`alcotest_stubs.c`](https://github.com/mirage/alcotest/blob/b26e58f62c93895f24a5ee48b550f274db74ee55/src/alcotest/alcotest_stubs.c))
do only two things: query terminal dimensions, and `dup`/`dup2` the standard streams
for log capture. None of that raises an exception from an asynchronous context.

Consequently:

- There is **no custom "timeout" exception raised from a signal handler**, so nothing
  needs converting to `Sys.raise_async`.
- There is **no `Sys.Break` / `Sys.catch_break`**, so the "Ctrl+C is handled for free"
  rule does not apply: if a user presses Ctrl+C during a test run, that is the default
  OCaml behaviour (the process is terminated by the signal) — Alcotest does not catch
  it today and does nothing special about it.

So the only asynchronous exception left is the runtime's own `Stack_overflow`, which
can arise inside arbitrary user test code.

### The per-test recovery loop

Alcotest's runner is a loop that runs each test, records its result, and moves on:
[`perform_tests`](https://github.com/mirage/alcotest/blob/b26e58f62c93895f24a5ee48b550f274db74ee55/src/alcotest-engine/core.ml#L256-L279)
folds
[`perform_test`](https://github.com/mirage/alcotest/blob/b26e58f62c93895f24a5ee48b550f274db74ee55/src/alcotest-engine/core.ml#L212-L254)
over every test. Each test was wrapped at registration time by
[`protect_test`](https://github.com/mirage/alcotest/blob/b26e58f62c93895f24a5ee48b550f274db74ee55/src/alcotest-engine/core.ml#L183-L195):

```
let protect_test path (f : 'a run) : 'a -> Run_result.t M.t =
 fun args ->
  M.catch
    (fun () -> f args >|= fun () -> `Ok)
    ((function
       | Check_error err -> `Error (path, err)
       | Skip -> `Skip
       | Failure s -> exn path "failure" …
       | Invalid_argument s -> exn path "invalid" …
       | e -> exn path "exception" …)         (* <- wildcard *)
     >> M.return)
```

For the default synchronous runner (the `alcotest` package, which instantiates the
engine with the
[identity monad](https://github.com/mirage/alcotest/blob/b26e58f62c93895f24a5ee48b550f274db74ee55/src/alcotest/alcotest.ml#L171-L178)),
`M.catch` is
[`match f () with x -> x | exception ex -> on_error ex`](https://github.com/mirage/alcotest/blob/b26e58f62c93895f24a5ee48b550f274db74ee55/src/alcotest-engine/monad.ml#L24)
— an ordinary wildcard handler. The final `e -> exn path "exception" …` arm catches
*everything*, so today a test that overflows the stack is caught, recorded as a failed
test (`` `Exn``), and **the loop continues to the next test**. Verified empirically
(stock OCaml 5.4.1): a wildcard catch around runaway recursion catches the
`Stack_overflow`, reports it, and execution continues normally (exit code 0).

Under OxCaml, `Stack_overflow` is an asynchronous exception that travels the separate
path and does **not** match that wildcard. So a single stack-overflowing test no longer
becomes one failed test — it flies past `protect_test`, past the whole runner loop, and
(absent any `Sys.with_async_exns` wrapper) terminates the process. The framework's core
promise — "one test blowing up still lets the rest of the suite run and report" — is
broken for `Stack_overflow`.

The Lwt and Async variants instantiate the *same* engine, so the same `protect_test`
wraps each test — but `M.catch` is now the concurrency library's own catch, and its
behaviour for `Stack_overflow` is **not** the same as the identity monad's, so the
backends have to be treated separately:

- **`alcotest-async`** instantiates the engine with a `Promise` module whose `catch` is
  [`try_with t >>= function Ok a -> return a | Error exn -> on_error exn`](https://github.com/mirage/alcotest/blob/b26e58f62c93895f24a5ee48b550f274db74ee55/src/alcotest-async/alcotest_async.ml#L11-L19).
  Async's `try_with` runs the test inside a monitor that captures *all* exceptions, with
  no exclusion for the runtime ones, so today a `Stack_overflow` becomes a recorded
  failure just as in the synchronous case — and under OxCaml it bypasses the monitor the
  same way. So the change applies here too. (Async's monitor internals were not read
  directly here; this rests on `try_with` having no `Stack_overflow` exclusion, which is
  worth a quick confirmation.)
- **`alcotest-lwt`** instantiates the engine with the `Lwt` module directly, so `M.catch`
  is `Lwt.catch`. Crucially, `Lwt.catch` guards its call with
  [`try f () with exn when Exception_filter.run exn -> fail exn`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/core/lwt.ml#L2026-L2029),
  and Lwt's **default** filter
  ([`handle_all_except_runtime`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/core/lwt.ml#L715-L722))
  returns `false` for `Stack_overflow` and `Out_of_memory`. So `Lwt.catch` does **not**
  catch a stack-overflowing test *today*: it already flies past `protect_test`, and an
  `alcotest-lwt` run already aborts on such a test. The new model therefore changes
  **nothing** about per-test recovery for `alcotest-lwt` — and note that adding a
  `Sys.with_async_exns` wrapper would not restore recovery either, because the converted
  exception is still `Stack_overflow`, which Lwt's default filter still declines. (The
  shared top-level Cmdliner `~catch`, an ordinary wildcard, is the one place even the Lwt
  run's behaviour shifts — see that row below.)

### What gets left behind when the loop is bypassed

The interesting clean-up is the standard-stream redirection used for per-test log
capture. For each test,
[`with_captured_logs`](https://github.com/mirage/alcotest/blob/b26e58f62c93895f24a5ee48b550f274db74ee55/src/alcotest-engine/core.ml#L200-L210)
calls
[`Log_trap.with_captured_logs`](https://github.com/mirage/alcotest/blob/b26e58f62c93895f24a5ee48b550f274db74ee55/src/alcotest-engine/log_trap.ml#L102-L113),
which opens a log file, calls
[`Platform.with_redirect`](https://github.com/mirage/alcotest/blob/b26e58f62c93895f24a5ee48b550f274db74ee55/src/alcotest/alcotest.ml#L98-L107),
and then closes the file. `with_redirect` (Unix):

1. flushes the formatters and calls `before_test`, which `dup`s the real stdout/stderr
   (saving them) and `dup2`s the log file over file descriptors 1 and 2 — so the test's
   raw output goes to the log file;
2. runs the test body inside `try fn () … with e -> `Error e` — i.e. it catches an
   *ordinary* exception, remembers it, restores the streams via `after_test`, and
   re-raises;
3. on the normal/ordinary-exception path, calls `after_test`, which `dup2`s the saved
   descriptors back over 1 and 2 (restoring the real streams) and closes the saved
   descriptors.

An asynchronous `Stack_overflow` raised inside the test body bypasses the `with e ->`
arm at step 2, so `after_test` (step 3) never runs and the file descriptors 1 and 2 are
left pointing at that test's log file; the `close` of the log file in
`Log_trap.with_captured_logs` is likewise skipped (a leaked descriptor). The process is
about to die from the uncaught asynchronous exception, so the in-memory leak is moot —
but the redirection is externally visible: **everything printed after that point goes to
the log file rather than the terminal.** Verified empirically: after a skipped restore,
a line written to "stdout" appears in the log file and never reaches the terminal. In
practice this means the runtime's uncaught-exception message, and anything flushed by
`at_exit`, would be written into the last test's log file instead of being shown to the
user — making the failure look like a silent disappearance.

## Clean-up that could be skipped on an asynchronous `Stack_overflow`

| Site | Clean-up | Verdict | Why |
|------|---------|---------|-----|
| [`alcotest.ml:98-107`](https://github.com/mirage/alcotest/blob/b26e58f62c93895f24a5ee48b550f274db74ee55/src/alcotest/alcotest.ml#L98-L107) `with_redirect` (+ C [`after_test`](https://github.com/mirage/alcotest/blob/b26e58f62c93895f24a5ee48b550f274db74ee55/src/alcotest/alcotest_stubs.c#L136-L149)) | restore OS stdout/stderr (`dup2` saved descriptors back, close the saved dups) via `try fn () … with e -> … ; after_test; raise` | **the one real change worth fixing** (visible but not fatal) | The restore lives on a `with e ->` arm and a post-body line; an asynchronous `Stack_overflow` skips both. The streams stay redirected to the test's log file, so the final report / runtime error / `at_exit` flush land in the log file, not the terminal. The process is dying anyway, so this is not corruption — but it makes the crash look silent. |
| [`log_trap.ml:102-113`](https://github.com/mirage/alcotest/blob/b26e58f62c93895f24a5ee48b550f274db74ee55/src/alcotest-engine/log_trap.ml#L102-L113) `with_captured_logs` | `Platform.close fd` (close the per-test log file) | benign (process dying) | The `close` runs only after `with_redirect` returns; an asynchronous exception skips it. A single leaked descriptor on a process that is terminating has no observable effect, and `at_exit`/process teardown flush buffered channels. |
| [`core.ml:305-317`](https://github.com/mirage/alcotest/blob/b26e58f62c93895f24a5ee48b550f274db74ee55/src/alcotest-engine/core.ml#L305-L317) `result` | restore `Printexc.record_backtrace` to its initial value after the run | benign (process dying) | The restore is a plain post-loop line (not a `finally`). An asynchronous `Stack_overflow` escaping `perform_tests` skips it, but the value is an in-memory global on a process that is terminating — never observed again. |
| [`core.ml:399-435`](https://github.com/mirage/alcotest/blob/b26e58f62c93895f24a5ee48b550f274db74ee55/src/alcotest-engine/core.ml#L399-L435) `run_with_args'` | restore the global stdout/stderr formatters (`Formatters.set_stdout`/`set_stderr` back to the saved ones) after the run | benign (process dying) | Same shape: a post-loop line, skipped by an asynchronous exception, but only mutates in-memory state on a dying process. The `at_exit` flush of stderr it registers ([`core.ml:427-428`](https://github.com/mirage/alcotest/blob/b26e58f62c93895f24a5ee48b550f274db74ee55/src/alcotest-engine/core.ml#L427-L428)) still runs on uncaught termination (verified). |

There are **no** `Fun.protect`/`~finally` uses anywhere in Alcotest, so the design
doc's discussion of `Fun.protect` not catching asynchronous exceptions does not apply
directly; the clean-up here is all hand-rolled `with e -> … ; raise` or plain post-body
lines.

## Why this is subtle

1. **In the runners where the catch covers `Stack_overflow`, that coverage is the whole
   point.** `protect_test` exists precisely so that *any* misbehaving test — including one
   that overflows the stack — is turned into a recorded failure rather than killing the
   suite. In the default synchronous runner (and `alcotest-async`) that catch does cover
   `Stack_overflow` today, relying on it being an ordinary exception catchable by
   `with e ->`. The new rules remove exactly that property for `Stack_overflow`, so the
   catch silently stops covering the one runtime exception a test framework most needs it
   to cover. (In `alcotest-lwt` the catch never covered `Stack_overflow`, so there is
   nothing to lose there.)

2. **The framework recovers per test, so the "process is dying anyway" reasoning does
   not fully apply.** A one-shot command-line tool can often shrug off an uncaught
   `Stack_overflow` because the process exits and `at_exit` covers the visible
   clean-up. But Alcotest's entire job is to *survive* a failing unit of work and run
   the next one. Under OxCaml the first stack-overflowing test aborts the run, so every
   later test is silently not run — the run goes from "N tests, one failed" to "aborted,
   results unknown".

3. **The skipped clean-up is mostly cosmetic, but in a way that hides the failure.** The
   one externally-visible effect — the standard streams left redirected to the last
   test's log file — means the user does not even see the crash message on their
   terminal; it is buried in a log file. So the failure mode is both "fewer tests ran"
   and "the reason is hard to see".

The change therefore splits in two:

- **Independent of where any wrapper goes:** the per-test wildcard catch
  (`protect_test`) stops catching `Stack_overflow`, so a stack-overflowing test aborts
  the run regardless. This can be asserted without any design decision.
- **Dependent on where a wrapper goes:** if Alcotest wants to *preserve* the current
  "one failed test, keep going" behaviour for `Stack_overflow`, it must turn the
  asynchronous exception back into an ordinary one at a chosen point so that
  `protect_test` can catch it again. Where to put that conversion is the design call
  (see below).

## How to fix it

Two distinct kinds of fix, matching the two kinds of site:

| Site kind | Sites | Fix | Difficulty |
|-----------|-------|-----|------------|
| Code that catches the exception to decide what to do next — *consumes* a `Stack_overflow` to turn it into a recorded per-test failure and continue | `protect_test` per-test catch ([`core.ml:183-195`](https://github.com/mirage/alcotest/blob/b26e58f62c93895f24a5ee48b550f274db74ee55/src/alcotest-engine/core.ml#L183-L195)); the top-level Cmdliner `~catch` ([`cli.ml:161-171`](https://github.com/mirage/alcotest/blob/b26e58f62c93895f24a5ee48b550f274db74ee55/src/alcotest-engine/cli.ml#L161-L171)) | wrap the per-test run in a `Sys.with_async_exns` boundary so an asynchronous `Stack_overflow` is converted back to an ordinary one *before* it reaches `protect_test`'s wildcard, preserving "one failed test, keep going" | design call (where to put the boundary, and whether to bother for `Stack_overflow`) |
| Resource / stream-state clean-up that only needs "the restore must run" | stream restore in `with_redirect` ([`alcotest.ml:98-107`](https://github.com/mirage/alcotest/blob/b26e58f62c93895f24a5ee48b550f274db74ee55/src/alcotest/alcotest.ml#L98-L107)) and log-file close in `with_captured_logs` ([`log_trap.ml:102-113`](https://github.com/mirage/alcotest/blob/b26e58f62c93895f24a5ee48b550f274db74ee55/src/alcotest-engine/log_trap.ml#L102-L113)) | an interruption-safe clean-up helper (`new_protect`) that restores the streams / closes the file and then re-throws the interruption unchanged | mechanical, local |

A second, interruption-aware bracket is expected to be available (the design doc's
"second version" of `Fun.protect`; e.g.
`new_protect : init:(unit -> 'a) -> body:('a -> 'b) -> finaliser:('a -> unit) -> 'b`)
— it runs the clean-up even when an asynchronous exception unwinds, then re-throws the
interruption. The natural shape here is to move the `before_test`/`after_test` pair into
the `init`/`finaliser` of such a bracket in `with_redirect`, and the open/close of the
log file into another, so the streams are always restored on the way out.

If Alcotest wants to keep the current "a stack-overflowing test is just a failed test"
behaviour, the per-test boundary is a single `Sys.with_async_exns` placed so it wraps
the test body but sits *below* `protect_test`'s wildcard — i.e. convert the asynchronous
`Stack_overflow` back to an ordinary one right around the test, and let the existing
wildcard catch it as today. Placing it *too high* (e.g. once around the whole run) would
not help: the run would still abort on the first overflow.

**Caveat — how the helper re-throws.** A `new_protect` that restores the streams and
then re-throws the interruption *unchanged* frees the resource but keeps the exception
on the asynchronous path (you still need the explicit `Sys.with_async_exns` per test if
you want `protect_test` to catch it). A `new_protect` that instead converts the
interruption to an *ordinary* exception would re-enable every downstream wildcard
`with e ->` to swallow a future asynchronous exception, against the new model. So:
restore the streams with a masking bracket that re-throws the interruption unchanged,
and convert to ordinary only at the one explicit per-test `Sys.with_async_exns` boundary
(if the "keep going" behaviour is wanted).

## Resolved during this pass

- **There is genuinely nothing on the throwing side** — verified by a full-repository
  search for signals, timers, finalisers, and memprof: none exist in the engine or any
  of the four front-end packages or the C stubs. So no `Sys.raise_async` work, and the
  `Sys.catch_break` "free Ctrl+C" rule is not engaged.
- **The per-test wildcard catches `Stack_overflow` today and the loop continues** —
  verified empirically on stock OCaml 5.4.1: a wildcard `with e ->` around runaway
  recursion catches the `Stack_overflow`, lets the program report it, and continue
  (exit 0). This is exactly `protect_test`'s shape.
- **`at_exit` runs on an uncaught `Stack_overflow`** — verified empirically on stock
  OCaml 5.4.1 (the `at_exit` callback ran; exit code 2). So the `at_exit` stderr flush
  in
  [`core.ml:427-428`](https://github.com/mirage/alcotest/blob/b26e58f62c93895f24a5ee48b550f274db74ee55/src/alcotest-engine/core.ml#L427-L428)
  still fires on an uncaught asynchronous termination — *provided OxCaml's
  asynchronous-termination path matches stock uncaught behaviour* (worth one
  confirmation on OxCaml). Note that its output goes to the *redirected* stderr if the
  crash happened mid-test (see the stream-restore finding).
- **The skipped stream restore really does send later output to the log file** —
  verified empirically: redirecting stdout via `dup2`, skipping the restore, then
  printing shows the text landing in the file, not the terminal.
- **The Lwt/Async/Mirage timeouts are cooperative, not signal-based** — `alcotest-async`
  uses
  [`Clock.with_timeout`](https://github.com/mirage/alcotest/blob/b26e58f62c93895f24a5ee48b550f274db74ee55/src/alcotest-async/alcotest_async.ml#L4-L10)
  (an Async scheduler timeout that yields a `` `Timeout`` result, no signal, no
  exception from a handler). `alcotest-lwt`
  ([`alcotest_lwt.ml:17-24`](https://github.com/mirage/alcotest/blob/b26e58f62c93895f24a5ee48b550f274db74ee55/src/alcotest-lwt/alcotest_lwt.ml#L17-L24))
  has no timeout at all and uses `Lwt.pick`/`Lwt_switch` plus an
  `Lwt.async_exception_hook` that wakes a promise — all on the normal cooperative path.
  Neither raises from a signal handler or a timer, so neither adds anything to the
  throwing side.

## Open questions (design calls / runtime confirmations)

- **Whether to preserve the "a stack-overflowing test is just a failed test" behaviour,
  and where to put the wrapper.** If yes, add a per-test `Sys.with_async_exns` boundary
  around the test body, below `protect_test`'s wildcard, so an asynchronous
  `Stack_overflow` is turned back into an ordinary one that the existing handler records
  as a failure and the loop continues. This is the one real design decision; it is only
  worth doing if recovering from a per-test stack overflow is considered in scope (it
  arguably is, for a test framework).
- **OxCaml confirmations** (need the runtime, not the source): (1) that `at_exit` runs
  on asynchronous-uncaught termination (so the stderr flush survives); and (2) the
  framework would also benefit from restoring the standard streams *before* that final
  message is printed, so the user sees the crash — covered by the stream-restore fix
  above.

## Checked and fine

- **`Stack_overflow`** — the one realistic asynchronous exception. Covered above: today
  it is caught per test and the run continues; under OxCaml it aborts the run unless a
  per-test wrapper is added.
- **`Out_of_memory`** — there are **zero** explicit catches of `Out_of_memory`. The
  design doc makes GC out-of-memory fatal; only the synchronous kind (e.g.
  `Array.make` with a bad size) remains an ordinary exception, and that would be caught
  by `protect_test`'s wildcard today and reported as a failed test exactly as now. No
  recovery path is broken.
- **Signals / Ctrl+C** — none installed anywhere. No `Sys.set_signal`,
  `Sys.catch_break`, or `Sys.Break`. Ctrl+C terminates the process via the default
  signal disposition today; Alcotest does nothing special, so nothing changes.
- **Timers** — no `Unix.setitimer` / `Unix.alarm`. The per-test timeouts in the Async
  variant are cooperative scheduler timeouts, not signal-based (see "Resolved").
- **`Gc.finalise` / `Gc.create_alarm` / memprof** — none. No "exception from a
  finaliser/callback" risk.
- **`Fun.protect` / `~finally`** — none used anywhere in Alcotest; all clean-up is
  hand-rolled.
- **Other wildcard / `exception _` handlers** — the remaining ones are not state
  restorations:
  [`config.ml:58`](https://github.com/mirage/alcotest/blob/b26e58f62c93895f24a5ee48b550f274db74ee55/src/alcotest-engine/config.ml#L58)
  and
  [`config.ml:186`](https://github.com/mirage/alcotest/blob/b26e58f62c93895f24a5ee48b550f274db74ee55/src/alcotest-engine/config.ml#L186)
  parse a config value and fall back on failure (no resource held);
  [`alcotest.ml:130`](https://github.com/mirage/alcotest/blob/b26e58f62c93895f24a5ee48b550f274db74ee55/src/alcotest/alcotest.ml#L130)
  and
  [`callsite_loc.412.ml:66`](https://github.com/mirage/alcotest/blob/b26e58f62c93895f24a5ee48b550f274db74ee55/src/alcotest-engine/callsite_loc.412.ml#L66)
  are TTY/`is_a_tty` probes. The `collect_exception` in
  [`test.ml:239-243`](https://github.com/mirage/alcotest/blob/b26e58f62c93895f24a5ee48b550f274db74ee55/src/alcotest-engine/test.ml#L239-L243)
  underlies the user-facing `check_raises`/`match_raises`: it catches whatever the body
  raised to test that an expected exception occurred. If a user's code-under-test
  overflows the stack while inside `check_raises`, under OxCaml that asynchronous
  `Stack_overflow` would bypass `collect_exception` rather than being reported as
  "wrong exception" — same per-test recovery story as `protect_test`, fixed by the same
  per-test boundary.
- **JavaScript build** — `alcotest`'s
  [`runtime.js`](https://github.com/mirage/alcotest/blob/b26e58f62c93895f24a5ee48b550f274db74ee55/src/alcotest/runtime.js)
  stubs the redirection (no real OS file descriptors), and there are no Unix signals in
  a JavaScript runtime, so the asynchronous-exception concerns are a native-runtime
  matter; the JS build is essentially out of scope.

## Recommendations

In priority order. The whole picture is small: there is exactly one asynchronous
exception to reason about (`Stack_overflow`), and nothing to change on the throwing
side.

1. **Decide whether a per-test stack overflow should remain a single failed test rather
   than aborting the run — the one real design call.** If yes, add a per-test
   `Sys.with_async_exns` boundary around the test body, *below* `protect_test`'s
   wildcard catch
   ([`core.ml:183-195`](https://github.com/mirage/alcotest/blob/b26e58f62c93895f24a5ee48b550f274db74ee55/src/alcotest-engine/core.ml#L183-L195)),
   so an asynchronous `Stack_overflow` is converted back to an ordinary one and the
   existing handler records it as a failed test and the loop continues. A boundary
   placed once around the whole run would not preserve this — it must be per test. This
   is the only change that preserves a currently-relied-on behaviour.

2. **Make the stream restoration interruption-safe.** Rewrite
   [`with_redirect`](https://github.com/mirage/alcotest/blob/b26e58f62c93895f24a5ee48b550f274db74ee55/src/alcotest/alcotest.ml#L98-L107)
   so `after_test` (the `dup2` that restores stdout/stderr) and the log-file
   [`close`](https://github.com/mirage/alcotest/blob/b26e58f62c93895f24a5ee48b550f274db74ee55/src/alcotest-engine/log_trap.ml#L102-L113)
   run even when an asynchronous exception unwinds (an interruption-safe bracket whose
   `finaliser` restores the streams). This guarantees that, however a run ends, the
   terminal's stdout/stderr are restored before the final report or crash message is
   printed — so the failure is visible rather than buried in a log file. Re-throw the
   interruption *unchanged* (do not convert it to ordinary here); pair it with the
   per-test boundary from (1) if recovery is wanted.

3. **Nothing to throw the new way.** Alcotest raises no exception from any signal
   handler, finaliser, or memprof callback, so there is no `Sys.raise_async` work, and
   no `Sys.catch_break` change.

4. **Confirm on the OxCaml runtime** (not derivable from source): that `at_exit` runs on
   asynchronous-uncaught termination (so the final stderr flush survives), and — together
   with (2) — that the streams are restored before that flush so the user sees it.

**Sequencing note:** (1) and (2) are independent and can land separately; (2) is pure
robustness (always restore the terminal streams) and is worth doing regardless of the
decision in (1). (3) is a no-op. **Do *not*** rewrite the stream-restore bracket so it
turns the asynchronous exception into an ordinary one — that would re-enable
`protect_test`'s wildcard (and every other `with e ->`) to swallow a future asynchronous
exception against the new model; restore-and-re-throw-unchanged, with one explicit
per-test `Sys.with_async_exns` if "keep going" is wanted.

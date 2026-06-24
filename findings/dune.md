# Dune — async-exceptions initial triage

Repo: `ocaml/dune` @ `4d89cd6d5f6246996db8c4c12ae5d0ee77836a15` (shallow clone in `./dune`).
Scope of this pass: a first look for code that works correctly today but would
break under OxCaml's new handling of "asynchronous exceptions".

An asynchronous exception is an event — pressing Ctrl+C, a timeout
going off, or running out of stack — that can surface at almost any point in the
program. Under the new model these asynchronous exceptions no longer travel through
ordinary `try … with` handlers; an ordinary handler will not catch them. There is a
new wrapper, `Sys.with_async_exns`, that turns one of these asynchronous exceptions
back into an ordinary one so the normal handlers can see it again.

## Bottom line

- **Dune's signal handling already side-steps the usual problem.** Ctrl+C
  (SIGINT), SIGTERM, SIGQUIT and SIGCHLD are **globally blocked** with
  `Unix.sigprocmask SIG_BLOCK` and consumed on a dedicated signal-watcher thread
  via `Thread.wait_signal` (`sigwait`), then turned into ordinary `Event.Queue`
  messages / `sys_exit`. They are **never raised as exceptions through OCaml
  code**, so the new model does not change their behaviour at all. There is **no
  `Sys.Break`** anywhere in dune's own code; dune's own cancellation
  (`Build_cancelled`) is an ordinary exception, raised deliberately at safe fiber
  check-points — not an asynchronous one.
- **The only asynchronous exception that actually reaches dune's code is
  `Stack_overflow`.** It has **no recovery handlers**; the only places it passes
  through are the universal Fiber task wrapper and the top-level reporter, both of
  which catch everything.
- **Nothing to do on the throwing side.** Dune raises no exception from a signal
  handler, finaliser, or memprof callback (signal handlers either write a byte to a
  pipe, send an event, or `exit`; the Memprof tracker callbacks only record
  samples). So nothing needs to be changed to throw it the new way
  (`Sys.raise_async`).
- **No genuine day-one bug *for one-shot CLI invocations*.** In a normal
  `dune build`, the one live asynchronous exception (`Stack_overflow`) stops the
  process, so nothing observable is corrupted — skipped clean-up doesn't matter
  because the process is exiting.
- **⚠ Daemon / watch / RPC mode breaks this (see dedicated section).** In
  `dune build --watch` / RPC server mode the process is **long-lived and recovers
  from build failures**: a `Stack_overflow` raised in a build is *today* caught by
  the universal Fiber task wrapper
  ([`core.ml:84-96`](https://github.com/ocaml/dune/blob/4d89cd6d5f6246996db8c4c12ae5d0ee77836a15/src/fiber/src/core.ml#L84-L96)),
  routed to `Fiber.map_reduce_errors`'s `on_error` reporter, and the poll loop
  ([`build_loop.ml:390`](https://github.com/ocaml/dune/blob/4d89cd6d5f6246996db8c4c12ae5d0ee77836a15/src/dune_engine/build_loop.ml#L390),
  [`:75-94`](https://github.com/ocaml/dune/blob/4d89cd6d5f6246996db8c4c12ae5d0ee77836a15/src/dune_engine/build_loop.ml#L75-L94))
  **waits for the next filesystem change and keeps serving**. Under the new model
  the asynchronous `Stack_overflow` slips past those wrappers, so (1) the daemon
  **dies instead of recovering** — a real behaviour change — and (2) the clean-up
  it skips now **leaks/corrupts over the daemon's lifetime** instead of being moot.
  So in daemon mode those wrappers are code that catches the exception to decide
  what to do next (report it and recover), not merely clean-up that happens to be
  skipped harmlessly.
- **The fragile spots are the clean-up helpers** — `Exn.protect`/`protectx`
  (stdune), `Mutex.protect` (stdune), `Fiber.finalize`/`Fiber.protect`, and the
  threaded-console hand-rolled lock/clean-up loop. Each catches the exception only
  to clean up and then re-throw, using a `match … with exception e ->` or
  `with exn ->` arm that an asynchronous `Stack_overflow` bypasses. They are safe
  today, but only by luck (**only for one-shot runs**, where the process exits
  anyway); in daemon mode they are genuine accumulating leaks (see daemon section).
  They would also break the instant a *second* asynchronous exception (an
  async-catchable `Sys.Break`, or asynchronous finaliser/memprof exceptions) can
  travel through the same scope.
- **One spot catches the exception to decide what to do next:** the top-level
  [`bin/main.ml:107-118`](https://github.com/ocaml/dune/blob/4d89cd6d5f6246996db8c4c12ae5d0ee77836a15/bin/main.ml#L107-L118)
  `| exn -> Report_error.report exn; exit` turns an uncaught exception into a
  reported error + flush + exit. Today it reports an uncaught `Stack_overflow`;
  under the new model that exception stops the program directly unless this scope is
  wrapped in `Sys.with_async_exns`. Whether to wrap it (to keep the diagnostic +
  `Console.finish` flush) is the one design call.
- Everything else checked (`Out_of_memory`, `Gc.finalise`, memprof callbacks, the
  TUI/console signal handlers, the Windows SIGINT handler, dune's own cooperative
  `Build_cancelled` cancellation) is clear.

## Which exceptions are affected

### Signals never become exceptions (why dune is mostly immune)

Dune deliberately blocks these signals and handles them on a dedicated thread, so
they never appear as exceptions:

- [`thread0.ml:13-31`](https://github.com/ocaml/dune/blob/4d89cd6d5f6246996db8c4c12ae5d0ee77836a15/src/dune_scheduler/thread0.ml#L13-L31):
  `block_signals` installs *no-op* handlers (`Signal_handle (fun _ -> ())`) for
  USR1/USR2/CHLD and then `Unix.sigprocmask SIG_BLOCK`s the full set
  (`Terminal_signals.signals @ [Usr1; Usr2; Chld; Int; Quit; Term]`). Every thread
  is spawned *after* this, so no thread takes these signals as an interruption.
- [`signal_watcher.ml:39-75`](https://github.com/ocaml/dune/blob/4d89cd6d5f6246996db8c4c12ae5d0ee77836a15/src/dune_scheduler/signal_watcher.ml#L39-L75):
  one watcher thread loops on `Thread.wait_signal` (`sigwait`) and converts each
  signal into a queue message — `Event.Queue.send_shutdown q (Signal signal)` for
  Int/Quit/Term, `send_job_completed_ready` for Chld — or, on the third Ctrl+C
  within a second, calls `sys_exit 1` directly. Nothing is raised as an exception
  into build code.
- The only `Sys.set_signal … Signal_handle` that *could* run as an interruption is
  the **Windows** SIGINT path,
  [`signal_watcher.ml:32`](https://github.com/ocaml/dune/blob/4d89cd6d5f6246996db8c4c12ae5d0ee77836a15/src/dune_scheduler/signal_watcher.ml#L32),
  whose body merely `Unix.write`s one byte to a self-pipe — it does **not** raise.

So under the new model, Ctrl+C in dune behaves *exactly as today*: it is delivered
to `sigwait`, becomes a shutdown event, and the build winds down cooperatively.
This is the single most important finding — dune already keeps signals from being
raised as exceptions through user code, which is exactly what the new model assumes.

### `Stack_overflow` — the one live asynchronous exception

The design doc lists `Stack_overflow` as asynchronous (raised by the runtime's
SIGSEGV handler via `caml_raise_async`). Dune is deeply recursive (rule evaluation,
Memo, S-expr parsing) so it *can* occur. But there are **no** `Stack_overflow`
handlers in `src/`/`bin/`, so there is no recovery to break — today it is uncaught
and fatal, and under the new model it stays fatal. The only handlers it passes
through are the two catch-everything wrappers below (the universal Fiber wrapper and
the top-level reporter), and the change to their behaviour is the subject of the
verdict table.

### The universal Fiber task wrapper (how an asynchronous exception escapes the fiber world)

Every task run by the fiber scheduler is wrapped by
[`core.ml:84-96`](https://github.com/ocaml/dune/blob/4d89cd6d5f6246996db8c4c12ae5d0ee77836a15/src/fiber/src/core.ml#L84-L96)
`apply`/`apply2`: `try f x with exn -> Reraise (capture exn)`. This is what turns
an ordinary exception into a `Reraise` effect that the fiber error-handler chain
(`with_error_handler` → `map_reduce_errors` → `collect_errors` →
[`finalize`, `core.ml:497-509`](https://github.com/ocaml/dune/blob/4d89cd6d5f6246996db8c4c12ae5d0ee77836a15/src/fiber/src/core.ml#L497-L509))
collects and re-raises in an orderly way, running every `~finally`. Under the new
model an asynchronous `Stack_overflow` at a safe point inside `f x` **bypasses this
`with exn ->` arm**: it is *not* turned into a `Reraise`, so it is *not* collected,
**no `Fiber.finalize ~finally` runs**, and it unwinds straight out of `Fiber.run`
([`scheduler.ml:398-418`](https://github.com/ocaml/dune/blob/4d89cd6d5f6246996db8c4c12ae5d0ee77836a15/src/dune_scheduler/scheduler.ml#L398-L418))
to the top-level handler. That is the mechanism behind every clean-up verdict below.

## Daemon / watch / RPC mode changes the verdict

The "safe today because the process is dying anyway" reasoning below holds **only
for one-shot `dune build` invocations**. In `dune build --watch` and the RPC/daemon
server, dune is **long-lived and recovers from build failures**:

- The universal Fiber task wrapper
  [`core.ml:84-96`](https://github.com/ocaml/dune/blob/4d89cd6d5f6246996db8c4c12ae5d0ee77836a15/src/fiber/src/core.ml#L84-L96)
  (`try f x with exn -> Reraise`) catches *any* exception raised in a fiber —
  **including a `Stack_overflow` today**, because today it propagates like an
  ordinary exception — and turns it into a fiber error.
- `Run_once.run`
  ([`scheduler.ml:407-418`](https://github.com/ocaml/dune/blob/4d89cd6d5f6246996db8c4c12ae5d0ee77836a15/src/dune_scheduler/scheduler.ml#L407-L418))
  wraps the build in `Fiber.map_reduce_errors ~on_error:Report_error.report` and a
  catch-all `exception exn -> Error (Exn …)`, so build errors are *reported*, not
  fatal.
- The poll loop
  ([`build_loop.ml:390`](https://github.com/ocaml/dune/blob/4d89cd6d5f6246996db8c4c12ae5d0ee77836a15/src/dune_engine/build_loop.ml#L390))
  then sets the status to *"Had N errors, waiting for filesystem changes…"*
  ([`build_loop.ml:75-94`](https://github.com/ocaml/dune/blob/4d89cd6d5f6246996db8c4c12ae5d0ee77836a15/src/dune_engine/build_loop.ml#L75-L94))
  and serves the next build — **the daemon survives.**

Under the new model an asynchronous `Stack_overflow` bypasses both `with exn ->`
arms, so:

1. **Recovery is lost (real behaviour change).** A build that overflows the stack
   today is reported and the watch/RPC daemon keeps running; under the new model the
   asynchronous exception propagates past the fiber wrapper and `Run_once`, hits no
   `Sys.with_async_exns`, and **terminates the daemon**. In daemon mode the fiber
   wrapper + `Run_once` catch-all are therefore code that catches the exception to
   decide what to do next (report and recover), not harmless clean-up skips.
2. **Skipped clean-up now accumulates.** Because the daemon would otherwise keep
   running, the `~finally`/`unlock`/fd-close clean-ups skipped by the asynchronous
   exception are no longer "moot at process exit" — leaked sandboxes, unreaped
   children, held mutexes and fds **build up over the daemon's lifetime**.

This is triggered rarely (a build deep enough to overflow the stack), but it is a
genuine regression for the long-lived modes, and it is the one place where dune is
**not** a clean non-target. Fix: put a `Sys.with_async_exns` wrapper **per build
iteration** inside the poll loop (turning the asynchronous exception back into an
ordinary one so the existing fiber/`Run_once` machinery reports it and the daemon
continues), plus making the clean-up helpers interruption-safe so per-build
resources are released on the way out.

## Clean-up that could be skipped

All sites catch the exception only to clean up and then re-throw, via an
`exception e`/`with exn ->` arm (or `Fiber.finalize`) that an asynchronous
`Stack_overflow` bypasses. No active bug *for one-shot CLI runs* (the process exits
regardless), so all are **safe today, but only by luck (and therefore fragile)** —
but see "Daemon / watch / RPC mode" above, where the skipped clean-up accumulates
and recovery is lost. Worth fixing for hygiene regardless, and because the luck runs
out the moment a second asynchronous exception (an asynchronous `Sys.Break`,
finaliser/memprof exceptions) can travel through the same scope.

| Site | Cleanup | Verdict | Why |
|------|---------|---------|-----|
| [`exn.ml:9-19`](https://github.com/ocaml/dune/blob/4d89cd6d5f6246996db8c4c12ae5d0ee77836a15/otherlibs/stdune/src/exn.ml#L9-L19) `Exn.protectx`/`protect` | run `finally x` then `raise e` | safe today, only by luck | The stdune analogue of `Fun.protect`. Used at ~12 sites (`csexp_rpc.ml`, `digest.ml`, `md5.ml`, `ocaml_config.ml`, `async_io.ml`, `scheduler.ml:408`) for fd/socket/cwd clean-up. An asynchronous `Stack_overflow` in the body skips `finally`, leaking the fd/leaving cwd changed — but the process is dying anyway, so it goes unnoticed today. This is the doc's "second version of `Fun.protect`" candidate. |
| [`mutex0.ml:3-12`](https://github.com/ocaml/dune/blob/4d89cd6d5f6246996db8c4c12ae5d0ee77836a15/otherlibs/stdune/src/mutex0.ml#L3-L12) `Mutex.protect` | `unlock mutex` then `raise exn` | safe today, only by luck | ~25 callers (scheduler `async_io`, `thread_safe_channel`, `inotify`, `fsevents`, `dune_trace/out`, `alloc`). A skipped unlock would deadlock the owning worker thread, not the build — and only on a `Stack_overflow` inside the (tiny) critical section, after which the process is unwinding to death. Unnoticed today; a real deadlock risk the moment a recoverable asynchronous exception enters scope. |
| [`core.ml:497-509`](https://github.com/ocaml/dune/blob/4d89cd6d5f6246996db8c4c12ae5d0ee77836a15/src/fiber/src/core.ml#L497-L509) `Fiber.finalize` / [`fiber.ml:35`](https://github.com/ocaml/dune/blob/4d89cd6d5f6246996db8c4c12ae5d0ee77836a15/src/fiber/src/fiber.ml#L35) `Fiber.protect` | run `finally` fiber, reraise collected errors | safe today, only by luck | The fiber-level clean-up behind process cleanup ([`process.ml:247-253`](https://github.com/ocaml/dune/blob/4d89cd6d5f6246996db8c4c12ae5d0ee77836a15/src/dune_engine/process.ml#L247-L253)), sandbox destroy ([`sandbox.ml:510`](https://github.com/ocaml/dune/blob/4d89cd6d5f6246996db8c4c12ae5d0ee77836a15/src/dune_engine/sandbox.ml#L510)), RPC session close, build-loop cleanup, timeout-task cancel ([`scheduler.ml:502`](https://github.com/ocaml/dune/blob/4d89cd6d5f6246996db8c4c12ae5d0ee77836a15/src/dune_scheduler/scheduler.ml#L502)). An asynchronous `Stack_overflow` bypasses `core.ml:84` `apply`'s `with exn ->`, so it never enters the fiber error machinery and **no `~finally` runs** — leaked sandbox dir / unreaped child / undeleted temp. Unnoticed today because it co-occurs with process death; the scheduler's own `kill_and_wait_for_all_processes` + `at_exit` temp cleanup are the actual backstops (see below). |
| [`dune_threaded_console.ml:89-154`](https://github.com/ocaml/dune/blob/4d89cd6d5f6246996db8c4c12ae5d0ee77836a15/src/dune_threaded_console/dune_threaded_console.ml#L89-L154) console event loop | `| exn -> Mutex.lock; cleanup (Some exn)` (re-acquire mutex, unlock, broadcast `finish_cv`) | safe today, only by luck (low severity) | Runs in a dedicated console thread; `render`/`handle_user_events` are shallow so `Stack_overflow` is improbable, and the clean-up it would skip (unlock + `Condition.broadcast finish_cv`) only matters for orderly shutdown of a thread that is itself dying. Diagnostic/UI bookkeeping, no soundness or build-result impact. |

(There are **no** `Fun.protect` uses and **no** hand-rolled `match … with exception
e -> cleanup; raise e` outside the helpers above — dune funnels all clean-up
through these, which is convenient: fixing the helpers fixes the callers.)

## Why this is subtle

1. **Today's catching is unpredictable** *in principle* — an asynchronous exception
   fires at an arbitrary safe point and is caught by whichever catch-everything
   handler is innermost. But in dune the only live asynchronous exception is
   `Stack_overflow`, which nobody recovers from, so in practice the only observable
   effect today is: the universal Fiber wrapper or the top-level reporter catches
   it, clean-up runs on the way out, dune prints an "Internal error / Uncaught
   exception" and exits.
2. **The chain is short** and there is no link where the exception is consumed to
   make a decision:
   ```
   bin/main.ml:107  | exn -> Report_error.report exn; exit        <- report + flush + abort  (wrapper candidate)
     scheduler.ml:411  Fiber.run ... | exception exn -> Error (Exn ...) <- orderly capture (bypassed by asynchronous exn)
       core.ml:84  apply: try f x with exn -> Reraise              <- enters fiber error machinery (bypassed by asynchronous exn)
         Fiber.finalize / Exn.protect / Mutex.protect              <- cleanup, then RE-RAISE
   ```
   Under the new model an asynchronous `Stack_overflow` skips the three inner
   `with exn ->` frames entirely (no clean-up, no orderly capture) and reaches
   `bin/main.ml`'s catch-everything handler — which *also* will not catch it unless
   wrapped in `Sys.with_async_exns`.

The change splits in two:

- **Independent of where the wrapper goes (guaranteed):** every
  `Fiber.finalize`/`Exn.protect`/`Mutex.protect` between the overflow's origin and
  any wrapper is skipped no matter where the wrapper goes — this can be asserted
  now. Whether it *matters* is per-site (the table: all safe today only by luck,
  because `Stack_overflow` is process-fatal and the scheduler/`at_exit` backstops
  cover the externally-visible resources).
- **Dependent on where the wrapper goes:** whether the top-level reporter still
  fires. With no `Sys.with_async_exns`, an uncaught asynchronous `Stack_overflow`
  stops the program *without* the `Report_error.report` diagnostic and *without* the
  `Console.finish ()` flush on the direct path (the `at_exit` flush still runs — see
  below).

## How to fix it

| Site kind | Sites | Fix | Difficulty |
|-----------|-------|-----|------------|
| Resource / lock / invariant clean-up — only needs "finaliser must run" | `Exn.protectx`/`protect` ([`exn.ml:9`](https://github.com/ocaml/dune/blob/4d89cd6d5f6246996db8c4c12ae5d0ee77836a15/otherlibs/stdune/src/exn.ml#L9-L19)), `Mutex.protect` ([`mutex0.ml:3`](https://github.com/ocaml/dune/blob/4d89cd6d5f6246996db8c4c12ae5d0ee77836a15/otherlibs/stdune/src/mutex0.ml#L3-L12)), `Fiber.finalize`/`protect` ([`core.ml:497`](https://github.com/ocaml/dune/blob/4d89cd6d5f6246996db8c4c12ae5d0ee77836a15/src/fiber/src/core.ml#L497-L509)), threaded-console loop | an interruption-safe version of the clean-up helper (`new_protect`) that runs the clean-up, then re-throws the interruption unchanged — fix the *helper*, callers inherit it | mechanical, ~4 definitions |
| Catches the exception to decide what to do next — *consumes* the exn, makes a control-flow decision | top-level [`bin/main.ml:107-118`](https://github.com/ocaml/dune/blob/4d89cd6d5f6246996db8c4c12ae5d0ee77836a15/bin/main.ml#L107-L118) | `Sys.with_async_exns` wrapper if dune wants to keep reporting/flushing on asynchronous termination | design call |

**Caveat — two re-throw modes.** A `new_protect` that runs the clean-up and then
re-throws the *asynchronous* exception unchanged frees the resource but keeps the
exception flying past downstream ordinary handlers (you still need the explicit
`with_async_exns` at the top if you want the diagnostic). A `new_protect` that
instead turns it back into an *ordinary* exception would silently re-enable
downstream `with exn ->`/`with _ ->` swallowing of any future asynchronous
`Sys.Break` — against the model. Best reading: re-throw the interruption unchanged
in the helper + (optionally) one explicit `with_async_exns` at `bin/main.ml`. The
four clean-up helpers are resource-safe under either mode.

Fixing the *helpers* (not the call sites) is the high-leverage move here: dune
already funnels essentially all clean-up through `Exn.protect`/`Mutex.protect`/
`Fiber.finalize`, so a single interruption-safe reimplementation of each covers
every caller.

## Resolved during this pass

- **Signals never become exceptions through OCaml code** — verified by reading the
  block-then-`sigwait` discipline (`thread0.ml`, `signal_watcher.ml`). Confirms
  nothing needs to throw it the new way (`Sys.raise_async`) and no `Sys.Break`
  catching work is needed.
- **`Build_cancelled` is an ordinary, deliberate exception** — dune's own
  cancellation, raised at safe fiber check-points
  ([`scheduler.ml:27-34`](https://github.com/ocaml/dune/blob/4d89cd6d5f6246996db8c4c12ae5d0ee77836a15/src/dune_scheduler/scheduler.ml#L27-L34)),
  caught at `build_system.ml:1090`/`action_exec.ml:99`. Not asynchronous;
  unaffected.
- **Memprof callbacks don't raise** — the tracker
  ([`alloc.ml:169-186`](https://github.com/ocaml/dune/blob/4d89cd6d5f6246996db8c4c12ae5d0ee77836a15/src/dune_trace/alloc.ml#L169-L186))
  only does `Table.set` into private hashtables; no `raise`. So the doc's "exception
  from a memprof callback terminates the program" change is inert here. (Tracing is
  also opt-in.)
- **`at_exit` runs on uncaught `Stack_overflow`** — verified empirically on stock
  OCaml 5.x (a program dying from a self-recursion `Stack_overflow` still ran its
  `at_exit` callback; exit code 2). Dune leans on `at_exit` for temp-file cleanup
  and trace flushing ([`assets.ml:35`](https://github.com/ocaml/dune/blob/4d89cd6d5f6246996db8c4c12ae5d0ee77836a15/src/dune_rules/assets.ml#L35),
  [`single_run_file_cache.ml:13`](https://github.com/ocaml/dune/blob/4d89cd6d5f6246996db8c4c12ae5d0ee77836a15/src/dune_pkg/single_run_file_cache.ml#L13),
  [`rev_store.ml:198`](https://github.com/ocaml/dune/blob/4d89cd6d5f6246996db8c4c12ae5d0ee77836a15/src/dune_pkg/rev_store.ml#L198),
  RPC socket unlink, `Dune_trace.at_exit`). So the externally-visible clean-up that
  the skipped `Fiber.finalize` brackets would have done is *also* covered by
  `at_exit` — **provided OxCaml's asynchronous-uncaught path runs `at_exit`** (one
  confirmation needed on the OxCaml runtime).

## Open questions (design calls / runtime confirmations)

- **Where to put the wrapper — two candidates.**
  - *One-shot CLI (low stakes):* the top-level
    [`bin/main.ml:107-118`](https://github.com/ocaml/dune/blob/4d89cd6d5f6246996db8c4c12ae5d0ee77836a15/bin/main.ml#L107-L118).
    Wrap `Cmd.eval_value` in `Sys.with_async_exns` so an uncaught asynchronous
    `Stack_overflow` still becomes the standard "Internal error" report +
    `Console.finish ()` flush rather than a bare termination. The `at_exit` flush
    survives either way; the only loss is the pretty diagnostic.
  - *Daemon / watch / RPC (real stakes):* a `Sys.with_async_exns` wrapper **per
    build iteration** in the poll loop
    ([`build_loop.ml:390`](https://github.com/ocaml/dune/blob/4d89cd6d5f6246996db8c4c12ae5d0ee77836a15/src/dune_engine/build_loop.ml#L390),
    around the `Run_once.run` build), so an asynchronous `Stack_overflow` is turned
    back into an ordinary exception, caught by the existing fiber/`Run_once`
    machinery, reported, and the daemon **continues** — preserving today's
    crash-recovery. This is the genuine design decision (where exactly to wrap).
- **OxCaml confirmations** (need the runtime, not the source): (1) `at_exit` runs on
  asynchronous-uncaught termination (so temp-file/trace cleanup survives); (2) the
  behaviour of a `Stack_overflow` that occurs *inside a worker/console/scheduler
  thread* under the new model (does it terminate just the domain/thread or the
  process?).

## Checked and fine

- **`Out_of_memory`** — **no** catch sites in `src/`/`bin/`. The doc's "OOM from the
  GC becomes fatal" breaks no recovery path here.
- **`Stack_overflow`** — no *dedicated* catch sites, but it **is** caught by the
  universal fiber wrapper + `Run_once` catch-all, which form a recovery loop in
  **daemon/watch mode** (see "Daemon / watch / RPC mode" — *not* fine there). In
  one-shot CLI mode it is effectively uncaught/fatal.
- **`Gc.finalise` / `Gc.create_alarm`** — none in dune's own code. No "exception
  from a finaliser" risk.
- **Memprof callbacks** — present (`dune_trace/alloc.ml`) but never raise; opt-in.
- **TUI signal handlers** — [`dune_tui.ml:9-37`](https://github.com/ocaml/dune/blob/4d89cd6d5f6246996db8c4c12ae5d0ee77836a15/src/dune_tui/dune_tui.ml#L9-L37)
  handle SIGCONT/SIGTSTP by releasing/recreating the terminal and writing a
  self-pipe / re-`kill`ing with SIGSTOP — they do not raise through OCaml code.
- **Windows SIGINT handler** — `signal_watcher.ml:32` writes one pipe byte; no
  raise.
- **`Sys.Break`** — not used anywhere in dune's own code (only in the vendored
  `opam`/`notty` copies, which are out of scope: 3rd-party code under `vendor/`).
- **No `Fun.protect`** anywhere; clean-up is funnelled through the four helpers
  above.

## Recommendations

In priority order. dune's signal handling already matches the new model, so for
one-shot CLI use this is just hardening the clean-up helpers; the substantive work
is preserving daemon-mode crash-recovery.

0. **Daemon/watch/RPC: add a per-build-iteration `Sys.with_async_exns` wrapper**
   ([`build_loop.ml:390`](https://github.com/ocaml/dune/blob/4d89cd6d5f6246996db8c4c12ae5d0ee77836a15/src/dune_engine/build_loop.ml#L390),
   around `Run_once.run`) so an asynchronous `Stack_overflow` in a build is turned
   back into an ordinary exception, caught by the existing fiber/`Run_once`
   reporter, and the daemon keeps serving instead of dying. This is the one place
   dune has a real behaviour change; pair it with (1) so per-build resources are
   actually released on the way out.

1. **Make the four clean-up helpers interruption-safe (the bulk of the structural
   work).** Reimplement `Exn.protectx`/`protect` ([`exn.ml:9`](https://github.com/ocaml/dune/blob/4d89cd6d5f6246996db8c4c12ae5d0ee77836a15/otherlibs/stdune/src/exn.ml#L9-L19)),
   `Mutex.protect` ([`mutex0.ml:3`](https://github.com/ocaml/dune/blob/4d89cd6d5f6246996db8c4c12ae5d0ee77836a15/otherlibs/stdune/src/mutex0.ml#L3-L12)),
   and `Fiber.finalize`/`Fiber.protect` ([`core.ml:497`](https://github.com/ocaml/dune/blob/4d89cd6d5f6246996db8c4c12ae5d0ee77836a15/src/fiber/src/core.ml#L497-L509),
   [`fiber.ml:35`](https://github.com/ocaml/dune/blob/4d89cd6d5f6246996db8c4c12ae5d0ee77836a15/src/fiber/src/fiber.ml#L35))
   as interruption-safe helpers (`new_protect`) so the finaliser/unlock runs even
   when an asynchronous exception unwinds. Have them re-throw the interruption
   unchanged, **not** turn it back into an ordinary exception — turning it back
   would re-enable downstream catch-everything swallowing of a future asynchronous
   `Sys.Break`. This single change covers all ~40 call sites without touching them.
   Also harden the threaded-console loop
   ([`dune_threaded_console.ml:89`](https://github.com/ocaml/dune/blob/4d89cd6d5f6246996db8c4c12ae5d0ee77836a15/src/dune_threaded_console/dune_threaded_console.ml#L89-L154)).

2. **Decide where to put the top-level wrapper (the one design call).** Optionally
   wrap `Cmd.eval_value` at [`bin/main.ml:107-118`](https://github.com/ocaml/dune/blob/4d89cd6d5f6246996db8c4c12ae5d0ee77836a15/bin/main.ml#L107-L118)
   in `Sys.with_async_exns` so an uncaught asynchronous `Stack_overflow` still
   reaches the `| exn -> Report_error.report exn; exit` reporter (pretty diagnostic
   + `Console.finish` flush) instead of terminating bare. Low stakes.

3. **Nothing to throw the new way** — dune raises no exception from any signal
   handler / finaliser / memprof callback, so there is nothing to convert on the
   throwing side (`Sys.raise_async`).

4. **Confirm on the OxCaml runtime** (not derivable from source): `at_exit` runs on
   asynchronous-uncaught termination (preserving temp-file/trace cleanup); and
   whether a worker/console-thread `Stack_overflow` kills just that thread or the
   whole process.

**Sequencing note:** (1) is independent and the bulk of the value; (2) is a small
optional add-on; (3) is a no-op confirmation; (4) gates how much (1)/(2) actually
buy you. **Do *not*** reimplement the helpers to turn the interruption back into an
ordinary exception — that would re-enable downstream catch-everything swallowing of
a future asynchronous `Sys.Break`, defeating the model.

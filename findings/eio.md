# Eio — async-exceptions initial triage

Repo: `ocaml-multicore/eio` @ `b114ab09d28809afa92f4e23f81ef6f1aa623b62` (shallow clone in `./eio`).

This is a first pass looking for code in Eio that works correctly today but would
break under OxCaml's new rules for "asynchronous exceptions".

## What an asynchronous exception is

A few kinds of event can interrupt an OCaml program at almost any moment: pressing
Ctrl+C, running out of stack space, or an exception thrown from a signal handler, a
GC finaliser, or a memprof callback. Today these surface as ordinary exceptions, so a
normal `try … with` catches them. OxCaml changes this: these asynchronous exceptions
now travel a separate route, an ordinary `try … with` no longer catches them, and
unless the program wraps the right region in a new special handler
(`Sys.with_async_exns`, which turns an asynchronous exception back into an ordinary
one so normal handlers see it again) the exception flies straight past every normal
handler and either reaches that wrapper or stops the program.

## Bottom line

- **Eio already does the thing the new model wants for signals.** The one OCaml
  signal handler Eio installs in its own library code is the SIGCHLD handler in
  [`process.ml:168-169`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio/unix/process.ml#L168-L169),
  and its whole body is `Eio.Condition.broadcast sigchld`. `broadcast` calls
  `Broadcast.resume_all`, which is documented as "lock-free and can be used safely even
  from a signal handler or GC finalizer"
  ([`broadcast.mli:24-28`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio/core/broadcast.mli#L24-L28);
  the callbacks it runs "must not raise", [`broadcast.mli:13-22`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio/core/broadcast.mli#L13-L22)),
  and the warning comment in
  [`examples/signals/main.ml:21-26`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/examples/signals/main.ml#L21-L26)):
  it only appends already-waiting fibers to the run queue and writes one wake-up byte
  to a pipe. It raises nothing through OCaml frames. So this handler produces **no
  exception** — there is no `Sys.Break` and nothing to convert to the new throwing
  primitive (`Sys.raise_async`).
- **The only other signal use is to *ignore* SIGPIPE**
  ([`eio_linux.ml:600`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio_linux/eio_linux.ml#L600),
  [`eio_posix.ml:23`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio_posix/eio_posix.ml#L23)),
  which installs no handler at all. There is **no `Sys.Break`** and **no
  `Sys.catch_break`** anywhere in Eio's own code, so the "Ctrl+C comes for free" rule
  is simply not engaged here.
- **Eio's own cancellation is an ordinary, cooperative exception, not an asynchronous
  one.** `Eio.Cancel.Cancelled` is raised only at cancellation check-points —
  `Cancel.check` / `Fiber.check` / `get_error`
  ([`cancel.ml:69-79`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio/core/cancel.ml#L69-L79))
  — on the normal exception path. It is unaffected by the change.
- **The only asynchronous exception that can actually reach Eio's code is
  `Stack_overflow`** (plus `Out_of_memory` only in its synchronous, non-GC form — the
  design doc makes GC out-of-memory fatal). Eio is deeply recursive (the scheduler,
  the lock-free cell structures, deep fiber trees), so `Stack_overflow` is realistic.
- **`Stack_overflow` has no recovery handlers anywhere in Eio.** There is not a single
  `with Stack_overflow ->` or `with Out_of_memory ->` in the library. So there is no
  *recovery* path that stops working — today an uncaught `Stack_overflow` already
  unwinds out of the scheduler and stops the program, and under OxCaml it stays fatal.
  The change is therefore confined to clean-up that gets skipped on the way out, not
  to lost recovery.
- **The clean-up that an asynchronous `Stack_overflow` skips funnels through a small
  number of core primitives.** Almost all of Eio's resource ownership and teardown
  goes through `Switch` — `Switch.run` / `await_idle` / `on_release` / `with_op`
  ([`switch.ml`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio/core/switch.ml#L77-L152))
  — plus a handful of hand-rolled `match … with … | exception ex -> cleanup; raise ex`
  brackets (`Cancel.with_cc`, `Fiber` fork wrappers, `Pool.run_with`, the backend
  schedulers' `with_sched`). Every one of these uses an ordinary exception arm that an
  asynchronous `Stack_overflow` bypasses, so the finaliser / `on_release` handler /
  fd-close does not run.
- **No genuine day-one bug for the common case.** In a normal program, the one live
  asynchronous exception (`Stack_overflow`) has no handler, so it unwinds out of
  `Eio_main.run` and the process stops. Skipped in-memory clean-up does not matter
  because the process is exiting, and the externally-visible parts (open fds, child
  processes) are released by the OS at process exit; Eio adds no `at_exit` hooks of its
  own. So these sites are **safe today, but only by luck** — they break the moment a
  recoverable asynchronous exception (an asynchronous `Sys.Break`, or
  finaliser/memprof exceptions) can travel through the same scope, or the moment a
  program tries to keep running after catching one.
- **Two important things to confirm with the *new* model rather than the source**
  (see "Still open"): (1) what happens when an asynchronous `Stack_overflow` arrives
  *while a fiber is running under the scheduler's effect handler* — the effect
  handler's exception arm (`exnc`) is an ordinary handler that re-raises, and under
  the new model an asynchronous exception bypasses it, escaping the whole scheduler;
  and (2) whether discontinuing a fiber's continuation (which is how Eio cancels and
  tears fibers down) still runs that fiber's clean-up under the new model.
- Everything else checked (`Out_of_memory`, the absence of `Gc.finalise` / memprof,
  the io_uring and Windows backends having no signal handler at all, the
  `Eio.Cancel.Cancelled` cooperative path, `Path.with_open_*` / `Pool` / `Buf_write` /
  `Eio_mutex` / process-spawn clean-up all flowing through `Switch`) is clear.

## Which exceptions are affected

### Signals never become OCaml exceptions in Eio (why Eio is mostly immune)

There is exactly one OCaml signal handler in Eio's own library code:

- [`process.ml:166-169`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio/unix/process.ml#L166-L169):
  ```
  let sigchld = Eio.Condition.create ()
  let install_sigchld_handler () =
    Sys.(set_signal sigchld) (Signal_handle (fun (_:int) -> Eio.Condition.broadcast sigchld))
  ```
  This is installed once, by the POSIX backend
  ([`eio_posix.ml:24`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio_posix/eio_posix.ml#L24)),
  so that a finished child wakes the fiber that is waiting to reap it
  ([`low_level.ml:590-600`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio_posix/low_level.ml#L590-L600)).
  The handler body is *only* `Eio.Condition.broadcast`, which is `Broadcast.resume_all`
  ([`condition.ml:41`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio/condition.ml#L41),
  [`broadcast.ml:103-104`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio/core/broadcast.ml#L103-L104)).
  `resume_all` walks the already-suspended cells and runs each cell's stored callback
  ([`broadcast.ml:57-69`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio/core/broadcast.ml#L57-L69)).
  For a waiter created by `Condition.loop_no_mutex` / `await_generic`, that callback is
  just `enqueue (Ok ())` / a `wake` that calls `enqueue`
  ([`condition.ml:74-118`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio/condition.ml#L74-L118)),
  and `enqueue` is `enqueue_thread` / `enqueue_failed_thread`, which only push to the
  lock-free run queue and optionally write one byte to the wake-up pipe
  ([`sched.ml:88-95`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio_posix/sched.ml#L88-L95)).
  Those functions are explicitly documented as safe to call "from anywhere (other
  systhreads, domains, signal handlers, GC finalizers)" and to take no locks
  ([`sched.ml:87-95`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio_posix/sched.ml#L87-L95));
  the whole `resume_all` path is likewise documented signal-safe with a "must not raise"
  obligation on its callbacks
  ([`broadcast.mli:13-28`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio/core/broadcast.mli#L13-L28),
  [`cells.mli:68`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio/core/cells.mli#L68)).
  **Nothing raises an exception through OCaml frames from this handler.**

- The io_uring (Linux) backend installs **no** SIGCHLD handler at all: it reaps
  children with a pidfd and a plain `waitpid`
  ([`low_level.ml:636-665`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio_linux/low_level.ml#L636-L665)).
  The Windows backend installs none either. So the SIGCHLD handler is POSIX-only and,
  even there, raises nothing.

- The only other `Sys.set_signal` calls set SIGPIPE to `Signal_ignore`
  ([`eio_linux.ml:600`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio_linux/eio_linux.ml#L600),
  [`eio_posix.ml:23`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio_posix/eio_posix.ml#L23)) —
  no handler, no exception.

There is **no `Sys.Break`**, **no `Sys.catch_break`**, **no `setitimer` / `Unix.alarm`
timer**, **no `Gc.finalise` / `Gc.create_alarm`**, and **no memprof callback** anywhere
in Eio's library code. So all of the "raise side" concerns (a custom exception thrown
from a signal handler / finaliser / memprof callback that would now need
`Sys.raise_async`) are absent. This is the most important finding: Eio already keeps
signals from being raised as exceptions through OCaml code.

A *user* who, like
[`examples/signals/main.ml`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/examples/signals/main.ml#L21-L26),
installs their own signal handler whose body is only `Eio.Condition.broadcast` is
likewise safe and needs no change. A user who instead *raises* a custom exception from
their own signal handler would need `Sys.raise_async` for that exception — but that is
the user's code, not Eio's, and the recommended Eio idiom (broadcast a condition) does
not raise.

### `Stack_overflow` — the one live asynchronous exception, and how it leaves the scheduler

The design doc lists `Stack_overflow` as asynchronous (raised by the runtime's SIGSEGV
handler). Eio is deeply recursive, so it can occur. There are **no** `Stack_overflow`
handlers in the library, so there is no recovery to break — it is uncaught and fatal
today, and stays fatal under OxCaml.

The mechanism by which it leaves the fiber world is the scheduler's effect handler. Each
fiber is run with `Effect.Deep.match_with`
([POSIX `sched.ml:334-376`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio_posix/sched.ml#L334-L376),
[Linux `sched.ml:317-385`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio_linux/sched.ml#L317-L385)),
whose exception arm is

```
exnc = (fun ex ->
    Fiber_context.destroy fiber;
    Printexc.raise_with_backtrace ex (Printexc.get_raw_backtrace ()))
```

This `exnc` is an *ordinary* exception handler. Today an uncaught `Stack_overflow`
raised inside a fiber is delivered here, the fiber context is destroyed, and the
exception is re-raised out of the scheduler. Under OxCaml an asynchronous
`Stack_overflow` travels the separate path and **does not enter `exnc` at all**: it
unwinds straight out of `match_with`, past `next` / `schedule`, past `Sched.run`'s
result-capture in the Linux backend
([`sched.ml:393-407`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio_linux/sched.ml#L393-L407)),
and out of `Eio_main.run`. So `Fiber_context.destroy fiber` and every clean-up bracket
between the overflow and the top are skipped. The same is true for the
`| exception ex -> cleanup (); …` arms in `with_sched`
([POSIX `sched.ml:258-263`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio_posix/sched.ml#L258-L263),
[Linux `sched.ml:435-491`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio_linux/sched.ml#L435-L491)),
which close the event pipe / uring ring on the way out: an asynchronous `Stack_overflow`
bypasses them. (For a one-shot program these resources are reclaimed by the OS at
process exit anyway.)

### Switch is the central clean-up funnel

Almost all resource ownership flows through `Switch`. `Switch.run_internal`
([`switch.ml:132-152`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio/core/switch.ml#L132-L152))
runs the body and, on both the success and the `| exception ex ->` arms, calls
`await_idle`, which collects and runs every `on_release` handler
([`switch.ml:95-114`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio/core/switch.ml#L95-L114)).
`Switch.with_op` and `with_daemon` use `Fun.protect ~finally:(fun () -> dec_fibers t)`
([`switch.ml:77-89`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio/core/switch.ml#L77-L89)).
An asynchronous `Stack_overflow` raised inside the switch body never reaches the
`| exception ex ->` arm and never runs `Fun.protect`'s `~finally`, so `await_idle`
never runs and the `on_release` handlers (which close fds, kill and reap child
processes, etc.) are skipped. Because `Path.with_open_*`
([`path.ml:143-152`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio/path.ml#L143-L152)),
`Eio.Process` spawn cleanup
([`low_level.ml:616-625`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio_posix/low_level.ml#L616-L625)),
and most other resource brackets are built on `Switch`, fixing `Switch` covers them all.

## Clean-up that could be skipped

All sites below clean up and re-throw (or run clean-up as a `~finally` / `on_release`)
and are bypassed by an asynchronous `Stack_overflow`. The recurring verdict is "safe
today, only by luck": the one live asynchronous exception has no recovery handler, so
it unwinds out of `Eio_main.run` and the process exits — the skipped clean-up is moot
because the OS reclaims fds/children at exit and Eio installs no `at_exit` hooks. They
become genuine skips the moment a program tries to keep running after an asynchronous
exception, or a second (recoverable) asynchronous exception enters scope.

| Site | Clean-up | Verdict | Why |
|------|----------|---------|-----|
| [`switch.ml:132-152`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio/core/switch.ml#L132-L152) `Switch.run_internal` + [`:95-114`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio/core/switch.ml#L95-L114) `await_idle` + [`:77-89`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio/core/switch.ml#L77-L89) `with_op`/`with_daemon` | run all `on_release` handlers (close fds, kill+reap children, drop resources) and decrement the live-fiber count | safe today, only by luck | The central teardown path for the whole library. An asynchronous `Stack_overflow` inside the switch body skips the `| exception ex ->` arm and the `Fun.protect ~finally`, so `await_idle` and every `on_release` handler are skipped. No active bug today (process exits, OS reclaims fds/children, no `at_exit` hooks to lose); fixing this one primitive fixes `Path.with_open_*`, `Pool`, process spawn, and every other switch-scoped resource at once. |
| [`fiber.ml:18-25`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio/core/fiber.ml#L18-L25) `Fiber.fork` / [`:27-44`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio/core/fiber.ml#L27-L44) `fork_daemon` / [`:46-72`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio/core/fiber.ml#L46-L72) `fork_promise` | the `with ex -> Switch.fail sw ex` / `exception ex -> Promise.resolve_error r ex` arms turn a fiber failure into a switch failure or a rejected promise | safe today, only by luck | These are the per-fiber wrappers that report a fiber's failure to its switch. An asynchronous `Stack_overflow` bypasses the `with ex ->` arm, so the switch is never failed (and sibling fibers are never cancelled) and the promise is never rejected — instead the exception escapes the scheduler's `exnc` and stops the program. The code comment at [`fiber.ml:9`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio/core/fiber.ml#L9) ("`f` must not raise … as that would terminate the whole scheduler") already documents that an escaping exception here is terminal; under OxCaml an asynchronous `Stack_overflow` is exactly such an escape. |
| [`cancel.ml:112-120`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio/core/cancel.ml#L112-L120) `with_cc` / [`:228-235`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio/core/cancel.ml#L228-L235) `with_vars` | restore the fiber's cancellation context / variable bindings (`cleanup (); … raise ex`) | safe today, only by luck | Both restore the fiber to its parent context / old `vars` on exit. An asynchronous `Stack_overflow` skips `cleanup ()`, leaving the fiber linked into the wrong cancellation context. In-memory only, and the fiber is being torn down by a fatal exception anyway — no observable effect today. Fragile: a recoverable asynchronous exception would leave the cancellation tree corrupt. |
| [`pool.ml:100-134`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio/pool.ml#L100-L134) `Pool.run_with` / `run_new_and_dispose` | return the slot to the pool / dispose the resource on the `| exception ex ->` arm | safe today, only by luck | An asynchronous `Stack_overflow` skips returning the slot, so a pooled resource is lost. The pool is in-memory and the process is exiting, so no active bug; becomes a leak only if the program keeps running. |
| [`eio_mutex.ml:104-118`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio/eio_mutex.ml#L104-L118), [`stream.ml:18-22`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio/stream.ml#L18-L22), [`condition.ml:31-36`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio/condition.ml#L31-L36)/[`:84-88`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio/condition.ml#L84-L88), [`hook.ml:13`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio/hook.ml#L13), [`debug.ml:22`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio/core/debug.ml#L22) | unlock a mutex (`unlock t; raise ex` or `Fun.protect ~finally:unlock`) | safe today, only by luck (low severity) | These unlock an internal `Mutex` on the failure path. An asynchronous `Stack_overflow` inside the critical section skips the unlock, leaving the mutex held — but the critical sections are tiny and the process is unwinding to death, so no deadlock is observed today. A real deadlock risk only if a recoverable asynchronous exception enters one of these critical sections. |
| [`buf_write.ml:519-527`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio/buf_write.ml#L519-L527), [`net.ml:357`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio/net.ml#L357)/[`:471`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio/net.ml#L471), [`thread_pool.ml:68-72`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio/unix/thread_pool.ml#L68-L72), [`rcfd.ml:165-172`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio/unix/rcfd.ml#L165-L172), [`process.ml:46-56`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio/unix/process.ml#L46-L56) `with_close_list` | close a flow / fd / fd-list, mark a write done, run a `~finally` | safe today, only by luck | Assorted leaf clean-up brackets (`Fun.protect` or `| exception ex -> close; raise`). Same reasoning: an asynchronous `Stack_overflow` skips them, but the fds are reclaimed at process exit. Worth hardening for the same reason as the rest. |

(There is no top-level `with _ ->` / `with exn ->` recovery wrapper in Eio's library
code at all — Eio is a library, so the "decide what to do next" handler belongs to the
application built on top of it, not to Eio itself.)

## Why this is subtle

1. **Today's catching is, in principle, unpredictable** — an asynchronous exception
   fires at an arbitrary safe point and is caught by whichever ordinary handler is
   innermost. But in Eio the only live asynchronous exception is `Stack_overflow`, and
   nobody recovers from it, so in practice the only observable effect today is: it
   unwinds through the clean-up brackets (running them on the way out), reaches the
   scheduler's `exnc`, and stops `Eio_main.run`.
2. **The chain is a clean-up chain with no recovery link.** From the point an overflow
   happens up to the top there is a stack of brackets, each of which only cleans up and
   re-raises — there is no frame that *consumes* the exception to make a decision:
   ```
   Eio_main.run / Sched.run               -> result := Error … ; re-raise   (Linux backend)
     scheduler match_with exnc            -> destroy fiber; re-raise         (ordinary handler — bypassed)
       Fiber.fork wrapper  with ex -> Switch.fail                            (report to switch — bypassed)
         Switch.run_internal | exception ex -> await_idle (run on_release)   (close fds/kill children — bypassed)
           Cancel.with_cc / Pool / Mutex / close brackets                    (restore context / unlock / close — bypassed)
   ```
   Under OxCaml an asynchronous `Stack_overflow` skips *every* frame in this chain and
   leaves the scheduler directly. Because none of them is a recovery frame, nothing is
   "no longer caught that used to be caught" in the sense of behaviour — what changes is
   only that the clean-up steps no longer run on the way out.
3. **The effects interaction is the genuinely new part.** Eio drives fibers by
   resuming captured continuations (`Suspended.continue` / `discontinue`,
   [`suspended.ml:13-19`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio/utils/suspended.ml#L13-L19))
   under one `Effect.Deep.match_with` per fiber. The scheduler's only exception arm is
   the ordinary `exnc`; there is no `Sys.with_async_exns` anywhere. So an asynchronous
   exception arriving while a fiber's continuation is running bypasses `exnc` and tears
   through the scheduler loop itself, not just the user's fiber. This is the place a
   wrapper would have to go if Eio wanted to turn an asynchronous exception back into an
   ordinary one and route it through the existing fiber-failure machinery.
4. **Signals are entirely out of the picture.** Eio converts the one signal it handles
   (SIGCHLD) into a `broadcast` that only enqueues, so none of the "Ctrl+C / `Sys.Break`
   / custom exception from a signal handler" concerns apply.

## How to fix it

| Site kind | Sites | Fix | Difficulty |
|-----------|-------|-----|------------|
| Resource / lock / context clean-up that only needs "the finaliser must run" | `Switch.run_internal`/`await_idle`/`with_op` ([`switch.ml`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio/core/switch.ml#L77-L152)), the `Fiber` fork wrappers ([`fiber.ml:18-72`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio/core/fiber.ml#L18-L72)), `Cancel.with_cc`/`with_vars` ([`cancel.ml`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio/core/cancel.ml#L112-L120)), `Pool`, the mutex/close leaf brackets | an interruption-safe clean-up helper (`new_protect`) that runs the clean-up and then re-throws the interruption *unchanged* — fix the core primitives and the leaf brackets inherit it | mechanical; a handful of primitives cover the rest |
| Code that catches the exception to decide what to do next | the scheduler's per-fiber `match_with` exnc / `Sched.run` ([POSIX `sched.ml:334-376`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio_posix/sched.ml#L334-L376), [Linux `sched.ml:317-407`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio_linux/sched.ml#L317-L407)) | a `Sys.with_async_exns` wrapper *only if* Eio wants an asynchronous `Stack_overflow` to be reported through the normal fiber-failure machinery / leave the scheduler cleanly rather than terminate the process directly — where exactly to put it (per fiber vs around the whole `Sched.run`) is the design call | design call |

**Caveat — how the helper re-throws.** An interruption-safe helper (`new_protect`) that
runs the clean-up and then re-throws the *asynchronous* exception unchanged frees the
resource but keeps the exception travelling the separate path (you still need an
explicit `Sys.with_async_exns` if you want the existing fiber/`Switch.fail` machinery to
see it). A helper that instead turned it back into an *ordinary* exception would
silently re-enable every downstream ordinary handler — including the catch-everything
arms in `Fiber.fork` and the application's own handlers — to swallow a future
asynchronous `Sys.Break`, against the new model. So: re-throw the interruption unchanged
in the primitives, and convert to ordinary at most at one explicit `Sys.with_async_exns`
in the scheduler if recovery is wanted.

Fixing the **primitives** (`Switch`, the `Fiber` fork wrappers, `Cancel.with_cc`), not
the call sites, is the high-leverage move: essentially all of Eio's clean-up funnels
through these, so a single interruption-safe reimplementation of each covers every
downstream resource bracket (`Path.with_open_*`, `Pool`, process spawn, `Buf_write`,
`Eio_mutex`, …) without touching them.

## Resolved during this pass

- **Signals never become exceptions through OCaml code in Eio** — verified by reading
  the SIGCHLD handler ([`process.ml:168-169`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio/unix/process.ml#L168-L169)),
  `Condition.broadcast` → `Broadcast.resume_all` → the enqueue callbacks
  ([`broadcast.ml`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio/core/broadcast.ml#L57-L104),
  [`condition.ml`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio/condition.ml#L74-L118),
  [`sched.ml:88-95`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio_posix/sched.ml#L88-L95)),
  all of which only enqueue and write a wake byte. Confirms nothing to throw the new way
  (`Sys.raise_async`) and no `Sys.Break` catch-side work.
- **`Eio.Cancel.Cancelled` is an ordinary, cooperative exception** — raised only at
  cancellation check-points via `Cancel.check` / `get_error`
  ([`cancel.ml:69-79`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio/core/cancel.ml#L69-L79)),
  propagated on the normal path. Not raised from a signal handler or the runtime; not
  asynchronous; unaffected.
- **No `Gc.finalise`, no memprof, no timers** in Eio's library code (`grep` clean). The
  doc's "exception from a finaliser / memprof callback / timer" changes are inert here.
- **`at_exit` runs on an uncaught `Stack_overflow`** — verified empirically on stock
  OCaml 5.4.1 (a program dying from runaway recursion still ran its `at_exit` callback;
  exit code 2). Eio itself installs **no** `at_exit` hooks, so it relies on the OS to
  reclaim fds/children at process exit; this is why the one-shot case is benign.

## Still open (design calls / runtime confirmations)

- **The scheduler effect-handler interaction (the real design call).** Decide whether
  an asynchronous `Stack_overflow` should be allowed to terminate the program directly
  (current de-facto behaviour) or be turned back into an ordinary exception and routed
  through the normal fiber-failure path so that `on_release` clean-up runs. If the
  latter, add a `Sys.with_async_exns` boundary in the scheduler — either around each
  fiber's `match_with` (so each fiber's `exnc` and the switch clean-up still fire) or
  around the whole `Sched.run` (simpler, but the per-fiber clean-up between the overflow
  and the boundary is still skipped). This is coupled with making the clean-up
  primitives interruption-safe.
- **Continuation discontinuation under the new model (runtime confirmation).** Eio
  cancels and tears fibers down by `discontinue`-ing their captured continuation
  ([`suspended.ml:17-19`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio/utils/suspended.ml#L17-L19)),
  which runs that fiber's clean-up brackets on the way out. Confirm that this still
  behaves as today under OxCaml — i.e. that discontinuing a continuation with an
  *ordinary* exception is unchanged (it should be; only *asynchronous* exceptions take
  the new path), and that an asynchronous `Stack_overflow` occurring *during* a
  `continue`/`discontinue` step escapes the scheduler as analysed above.
- **`Stack_overflow` inside a worker domain / systhread.** Eio spawns domains
  (`Domain.spawn`, [`eio_linux.ml:282`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio_linux/eio_linux.ml#L282))
  and a systhread pool ([`thread_pool.ml`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio/unix/thread_pool.ml#L60-L110)).
  Confirm on the OxCaml runtime whether an asynchronous `Stack_overflow` in a worker
  domain/thread terminates just that domain/thread or the whole process, since that
  determines whether the worker-loop clean-up (`Fun.protect ~finally`,
  [`domain_mgr.ml:102`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio_posix/domain_mgr.ml#L102)) matters.

## Obviously fine (gaps checked and cleared)

- **`Stack_overflow`** — the one realistic asynchronous exception. No recovery handler
  anywhere in Eio, so no recovery regression; it is fatal today and stays fatal. The
  only effect is the skipped clean-up covered above, which is benign for one-shot
  programs (process exits, OS reclaims fds/children, no `at_exit` hooks to lose).
- **`Out_of_memory`** — the design doc makes GC out-of-memory fatal, so only the
  synchronous kind remains; Eio has no `Out_of_memory` catch sites, so no regression.
- **Signals** — only SIGCHLD (POSIX backend) gets a handler, and its body is a
  signal-safe `broadcast` that only enqueues; SIGPIPE is merely ignored; io_uring and
  Windows backends install no handler. No `Sys.Break`, no `Sys.catch_break`.
- **`Gc.finalise` / `Gc.create_alarm` / memprof / `setitimer` / `Unix.alarm`** — none
  in Eio's library code. (The only `setitimer` / SIGALRM hit is in a Linux backend
  *test*, [`lib_eio_linux/tests/test.ml:201-212`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio_linux/tests/test.ml#L201-L212),
  which checks an OCaml signal handler still runs while sleeping in liburing — test
  scaffolding, not shipped library code, so out of scope.)
- **`Eio.Cancel.Cancelled` / cooperative cancellation** — ordinary exception on the
  normal path; unaffected.
- **Wildcard handlers** — Eio's library code has no bare `with _ ->` recovery handler;
  the `| exception ex ->` arms are all clean-up-and-re-raise (covered in the table).
- **Windows backend** — installs no signal handler; its clean-up brackets
  ([`sched.ml`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio_windows/sched.ml#L258-L320),
  [`domain_mgr.ml`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio_windows/domain_mgr.ml#L102-L112))
  are the same clean-up-and-re-raise shape as POSIX/Linux and get the same verdict.

## Recommendations

In priority order. Eio's signal handling already matches the new model and the one live
asynchronous exception has no recovery path, so for ordinary programs this is mostly
hardening; the substantive work is the scheduler design call.

1. **Make the core clean-up primitives interruption-safe (the bulk of the structural
   work).** Reimplement `Switch.run_internal`/`await_idle`/`with_op`
   ([`switch.ml`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio/core/switch.ml#L77-L152)),
   the `Fiber.fork`/`fork_daemon`/`fork_promise` wrappers
   ([`fiber.ml:18-72`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio/core/fiber.ml#L18-L72)),
   and `Cancel.with_cc`/`with_vars`
   ([`cancel.ml:112-120`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio/core/cancel.ml#L112-L120),
   [`:228-235`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/lib_eio/core/cancel.ml#L228-L235))
   as interruption-safe helpers (`new_protect`) so the `on_release` handlers / context
   restoration run even when an asynchronous exception unwinds. Re-throw the
   interruption *unchanged* — do **not** turn it back into an ordinary exception, or the
   catch-everything arms in `Fiber.fork` (and the application's own handlers) would
   start swallowing a future asynchronous `Sys.Break`. This single change covers all the
   downstream resource brackets (`Path.with_open_*`, `Pool`, process spawn, `Buf_write`,
   `Eio_mutex`, the close brackets) without touching them.

2. **Decide the scheduler boundary (the one real design call), coupled with (1).**
   Decide whether an asynchronous `Stack_overflow` should terminate the process directly
   (status quo) or be converted with `Sys.with_async_exns` in the scheduler and routed
   through the existing fiber-failure machinery so `Switch` clean-up runs. Per-fiber
   placement (around each `match_with`) preserves the most clean-up; whole-`Sched.run`
   placement is simpler but skips per-fiber clean-up below it. Only worth doing if Eio
   wants graceful teardown rather than immediate termination on stack overflow.

3. **Nothing to throw the new way.** Eio raises no exception from any signal handler,
   finaliser, or memprof callback (the SIGCHLD handler only `broadcast`s), so there is no
   `Sys.raise_async` work. Worth a documentation note that a *user* signal handler
   should follow the `Eio.Condition.broadcast`-only idiom
   ([`examples/signals/main.ml`](https://github.com/ocaml-multicore/eio/blob/b114ab09d28809afa92f4e23f81ef6f1aa623b62/examples/signals/main.ml#L21-L26))
   and must use `Sys.raise_async` if it raises a custom exception instead.

4. **Confirm on the OxCaml runtime** (not derivable from source): (a) an asynchronous
   `Stack_overflow` arriving during a fiber `continue`/`discontinue` escapes the
   scheduler as analysed; (b) discontinuing a continuation with an ordinary exception is
   unchanged; (c) whether a worker-domain / systhread `Stack_overflow` kills just that
   domain/thread or the whole process.

**Sequencing note:** (1) is independent and the bulk of the value; (2) is the one design
decision and lands together with (1) if graceful teardown is wanted; (3) is a no-op plus
a doc note; (4) gates how much (1)/(2) actually buy you. **Do *not*** reimplement the
primitives to turn the interruption back into an ordinary exception — that would
re-enable the catch-everything fiber-fork arms (and application handlers) to swallow a
future asynchronous `Sys.Break`, defeating the model.

# Frama-C — first look at the asynchronous-exception change

Repo: `pub/frama-c` @ `44585036cb05125b3128a173c7d98881c4bbdb42`
(shallow clone in `./frama-c`, cloned from the canonical GitLab
`https://git.frama-c.com/pub/frama-c.git`).
What this pass covers: a first search for code that works correctly today but
would break under OxCaml's new rules for "asynchronous exceptions".

Background in one sentence: an *asynchronous exception* is an event
— pressing Ctrl+C, running out of stack, or an exception thrown from inside a
signal handler — that can surface at almost any point in the program. Today an
ordinary `try … with` catches these; under the new rules it does not, unless you
use a new wrapper (`Sys.with_async_exns`) that turns the asynchronous exception
back into an ordinary one. We call these "asynchronous exceptions" below.

## Bottom line

- **Frama-C turns Ctrl+C into an exception in two different ways, and only one of
  them is asynchronous.**
  - At start-up the batch driver runs
    [`Sys.catch_break true`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/kernel_internals/runtime/boot.ml#L42-L44),
    which asks the standard library to install a `SIGINT` handler that **raises
    `Sys.Break` from inside the signal handler**. So `Sys.Break` is a genuine
    asynchronous exception, caught all over Frama-C with ordinary `try … with`.
  - The Eva plug-in additionally installs **its own** `SIGINT` handler that also
    raises `Sys.Break` directly from the handler
    ([`signal.ml:40-50`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/plugins/eva/src/engine/signal.ml#L36-L51)),
    and a `SIGUSR1` handler that does **not** raise — it only sets a flag that is
    read later at cooperative check-points
    ([`signal.ml:16-27`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/plugins/eva/src/engine/signal.ml#L16-L27)).
- **Only `Sys.Break` (Ctrl+C) is actually affected.** "Out of stack"
  (`Stack_overflow`) can always happen too, but Frama-C has **zero** places that
  catch it. "Out of memory" (`Out_of_memory`) is caught in exactly one spot, and
  that one is a synchronous out-of-memory from a C math library, not the GC. There
  are no finalisers (`Gc.finalise`), no GC alarms, and no memprof callbacks. The
  one custom interruption exception, `Async.Cancel`, and the WP/Task `Canceled`
  and `Timeout` outcomes, are **cooperative** — they are produced at explicit
  check-points, never from a signal handler — so they are unaffected (see below).
- **Throwing side — almost nothing to do, with one exception.** Where Ctrl+C goes
  through the standard `Sys.catch_break true`, OxCaml already makes the escaping
  `Sys.Break` an asynchronous exception, so no code change is needed to throw it.
  **The one place that needs a throwing-side check is Eva's own `SIGINT` handler**
  ([`signal.ml:41`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/plugins/eva/src/engine/signal.ml#L41)),
  which does a literal `raise Sys.Break` inside the handler — that `Sys.Break` is
  already the kind OxCaml turns into an asynchronous one, so it is fine, but it is
  worth confirming that a hand-installed handler raising `Sys.Break` gets the same
  free treatment as `Sys.catch_break` (see "Open questions").
- **No present-day soundness bug from skipped clean-up.** Frama-C's two most
  important clean-up sites are protected against being skipped *for the reason that
  actually matters*: partial Eva results are saved through a dedicated wrapper, and
  the per-run state save on Ctrl+C is driven by the top-level catch, not by a deep
  finaliser.
- **Two spots that are safe today only by luck (and therefore fragile):** the
  project-switch restore in
  [`project.ml:426-428`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/libraries/project/project.ml#L426-L428)
  and the "apply once" status reset in
  [`state_builder.ml:1032-1041`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/libraries/project/state_builder.ml#L1029-L1042).
  Both restore a long-lived global through a handler / `Fun.protect ~finally` that an
  asynchronous `Sys.Break` would skip. The skipped restore is **real** — verified by
  reading both — but **a Ctrl+C does not, in fact, leave the server running**: the only
  `Sys.Break` handling in `-server` mode is the main-loop guard
  [`while … with Sys.Break -> ()`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/plugins/server/main.ml#L524-L531),
  whose comment is "Ctr+C, just leave the loop normally". Today a Ctrl+C **shuts the
  server down** (exit the loop → clean shutdown at
  [`:532-535`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/plugins/server/main.ml#L532-L535)).
  Under the new model the asynchronous `Sys.Break` skips that loop guard and is caught
  instead by the outer
  [`with exn -> … ; raise exn`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/plugins/server/main.ml#L536-L541),
  which prints "Server interrupted (fatal error)", goes offline, and **re-raises** — the
  server still terminates. So in **both** models the server stops after a Ctrl+C and no
  later request is served. The corrupted globals (`Project.on`'s wrong current project,
  `apply_once`'s stuck flag) are therefore **never observed by a later request** in the
  current code, because there is no later request. These remain real hygiene problems
  (an in-memory invariant is broken on the way out, and the change *visibly* turns
  today's clean "leave the loop normally" shutdown into a "fatal error" shutdown), but
  the claimed "wrong project for every later request" consequence does **not** occur as
  the server is written. It *would* occur if a future `Sys.with_async_exns`-per-request
  boundary were added that let the loop survive a Ctrl+C — which is exactly the
  remediation below, so the fix must restore both the loop guard *and* the skipped
  state restores together.
- **Places that catch the exception to decide what to do next** (turn it into a
  result, save partial work, choose an exit code): the batch top-level
  [`cmdline.ml:191-205`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/kernel_services/cmdline_parameters/cmdline.ml#L191-L205),
  Eva's
  [`signal.ml:58-76`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/plugins/eva/src/engine/signal.ml#L58-L76)
  and
  [`analysis.ml:264-269`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/plugins/eva/src/engine/analysis.ml#L264-L269),
  and the server request/loop handlers in
  [`server/main.ml`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/plugins/server/main.ml#L151-L166).
  These need the `Sys.with_async_exns` wrapper; *where* to put it is the design call,
  and it differs between batch and server mode.
- Everything else checked (`Stack_overflow`, the one `Out_of_memory`, finalisers,
  the WP/Why3 prover timeout, the cooperative `Async.Cancel`, the `SIGUSR1` and
  `SIGPIPE` handlers, the `Killed` request exception) is clear or cooperative.

## Which exceptions are affected

### `Sys.Break` is asynchronous in Frama-C (`Sys.catch_break true`)

The batch driver
[`Boot.boot`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/kernel_internals/runtime/boot.ml#L42-L55)
runs `Sys.catch_break true` once at start-up. In the standard library that is
`Sys.set_signal Sys.sigint (Signal_handle (fun _ -> raise Break))` — i.e. Frama-C
installs a `SIGINT` handler that **raises `Sys.Break` from inside the handler**.
That is exactly the asynchronous-exception pattern: today the runtime lets the
raised `Sys.Break` surface at the next safe point, where one of Frama-C's many
`try … with Sys.Break` / catch-all handlers catches it. Under OxCaml it takes the
asynchronous path and **none** of those handlers catch it — it goes to the nearest
`Sys.with_async_exns`, or it stops Frama-C.

The server's
[`run`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/plugins/server/main.ml#L516-L531)
also calls `Sys.catch_break true` (there is a comment that this is a workaround
"to be removed once Why3 signal handler is fixed"), so the same `Sys.Break` is
asynchronous in `-server` mode too.

Two consequences:
1. **Throwing side — already handled** for the standard `catch_break` route.
   OxCaml already makes the `Sys.Break` escaping that handler an asynchronous
   exception. The one hand-written handler (Eva, below) raises *the very same*
   `Sys.Break`, which the new model also recognises — so the throwing side needs
   essentially no change.
2. **Catching side — the work.** Every `with Sys.Break` / catch-all handler stops
   firing; a `Sys.with_async_exns` wrapper is needed to turn the asynchronous
   exception back into an ordinary one. This is where Frama-C needs changes.

### Eva installs a second SIGINT handler that also raises `Sys.Break`

Eva's signal module installs its own handlers during an analysis and restores the
previous ones afterwards
([`signal.ml:36-51`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/plugins/eva/src/engine/signal.ml#L36-L51)):

- **`SIGINT` → `interrupt _ = warn (); raise Sys.Break`** — raises `Sys.Break`
  *directly from inside the signal handler*. This is the asynchronous case. Because
  it is `Sys.Break` specifically, OxCaml treats it the same way it treats the one
  installed by `Sys.catch_break` — it becomes an asynchronous exception for free,
  so no `Sys.raise_async` is needed here (worth one confirmation that a
  hand-installed `Signal_handle` raising `Sys.Break` is covered, not only
  `Sys.catch_break`).
- **`SIGUSR1` → `stop _ = warn (); kill ()`** where `kill ()` just sets
  `signal_emitted := Some Sys.Break`
  ([`signal.ml:16-17`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/plugins/eva/src/engine/signal.ml#L16-L17)).
  The flag is read by `Signal.check ()`
  ([`signal.ml:25-27`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/plugins/eva/src/engine/signal.ml#L25-L27)),
  which Eva calls at chosen safe points
  ([`iterator.ml:362`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/plugins/eva/src/engine/iterator.ml#L362),
  [`initialization.ml:381`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/plugins/eva/src/engine/initialization.ml#L381)).
  This `SIGUSR1` path is **cooperative** — the handler does not raise; the
  `Sys.Break` it eventually produces comes out of an ordinary `check ()` call on
  the main stack — so it is *not* an asynchronous exception and is unaffected by
  the change. (Note `check` raises a plain `Sys.Break`; that ordinary raise from
  non-handler code stays ordinary.)

### `Async.Cancel`, WP `Canceled`/`Timeout` — cooperative, not affected

Frama-C's general interruption mechanism is `Async.cancel`/`Async.yield`
([`async.ml`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/libraries/utils/async.ml#L57-L108)):
`cancel ()` only sets a `canceled` boolean
([`async.ml:63-64`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/libraries/utils/async.ml#L63-L64)),
and the `Async.Cancel` exception is raised **only** at explicit yield points by
`raise_if_canceled`
([`async.ml:95-97`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/libraries/utils/async.ml#L95-L97),
called from `yield`/`flush`/`sleep`). It is never raised from a signal handler, so
it travels the normal path and ordinary `try … with Async.Cancel` keeps working.

The WP prover-call timeout is likewise cooperative: `Command.spawn`
([`command.ml:136-158`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/libraries/utils/command.ml#L136-L158))
polls the child with `Unix.waitpid`, calls `Async.yield ()`, compares wall-clock
elapsed time against the timeout, and `raise Async.Cancel` from ordinary code when
it is exceeded — there is **no `Unix.alarm`/`setitimer` and no signal-handler raise**.
The WP/Task layer's `Timeout` and `Canceled` are *result variants* of the task
monad ([`task.ml:34-50`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/libraries/utils/task.ml#L34-L50)),
matched as values, not caught as exceptions. None of this is affected.

## Clean-up that could be skipped on a Ctrl+C

All the sites below restore state or release a resource through a handler or
`Fun.protect ~finally` that an asynchronous `Sys.Break` would bypass. After tracing
each one, **none is a day-one soundness bug in batch mode**; the two project/state
restorations are safe today only by luck and become genuine bugs in the long-lived
server mode. Verdicts below state the mode explicitly.

| Site | Clean-up | Verdict | Why |
|------|---------|---------|-----|
| [`iterator.ml:643-648`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/plugins/eva/src/engine/iterator.ml#L631-L648) `Signal.protect compute ~cleanup` | save partial dataflow results (`mark_degeneration`, `merge_results`) when interrupted | safe today *because of* this wrapper — but the wrapper is the very thing that breaks | This is Eva's deliberate "save partial results on user interrupt" feature: `Signal.protect` (below) catches `Sys.Break`/`Self.Abort`, runs `cleanup`, then re-raises. Under the new model the `Sys.Break` case of `Signal.protect` stops firing, so partial results are **not** saved. Not a soundness bug (the lost data is best-effort partial output), but a **lost feature** — converting this wrapper to the interruption-aware bracket restores it. |
| [`project.ml:426-428`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/libraries/project/project.ml#L400-L428) `Project.on` | restore the previously-current project (`Fun.protect ~finally:set_to_old`) | safe today only by luck; fragile, **not** an observed server-mode bug as written | `Project.on p f x` temporarily makes `p` the current project, runs `f`, and restores the old current project in `finally` ([`project.ml:427-428`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/libraries/project/project.ml#L426-L428)). An asynchronous `Sys.Break` skips the restore, leaving the **wrong project current** — verified. But this corruption is only *observed* if the program keeps using projects afterwards. In one-shot batch the process exits. In `-server` mode the server also stops on a Ctrl+C (the loop guard / outer handler both end the loop; see Bottom line), so no later request reads the wrong project as the code stands. Real if a per-request `Sys.with_async_exns` boundary is later added so the loop survives Ctrl+C — then the skipped restore must be fixed together with that boundary. |
| [`state_builder.ml:1032-1041`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/libraries/project/state_builder.ml#L1029-L1042) `apply_once` | reset the "already run" flag on failure so the computation can be retried (`with exn -> First.set true; raise exn`) | safe today only by luck; fragile, **not** an observed server-mode bug as written | `apply_once` runs a computation at most once: it flips the flag to `false` before running, and on failure the `with exn -> First.set true; raise exn` arm flips it back so a later call retries ([`state_builder.ml:1029-1042`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/libraries/project/state_builder.ml#L1029-L1042)). Eva's whole `compute` is wrapped in this ([`analysis.ml:281-283`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/plugins/eva/src/engine/analysis.ml#L281-L283)). An asynchronous `Sys.Break` skips the reset, leaving the flag at `false` → a later call sees `First.get () = false`, skips the body, and silently returns nothing — verified. But, as above, the server stops on a Ctrl+C, so there is no later Eva request to hit the stuck flag as the code stands. Real once a per-request boundary lets the server survive Ctrl+C; fix together with that boundary. |
| [`analysis.ml:243-269`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/plugins/eva/src/engine/analysis.ml#L243-L269) `compute_from` | (a) `Fun.protect ~finally:restore_signals` restores the previous signal handlers; (b) `with exn -> ComputationState.set Aborted; …` marks Eva aborted | (a) fragile; (b) catch-to-decide | (a) If the `Fun.protect` is skipped by an asynchronous `Sys.Break`, Eva's `SIGINT`/`SIGUSR1` handlers stay installed after the analysis. In batch this is moot; in server mode it leaves Eva's handlers active outside an analysis, so a later Ctrl+C raises `Sys.Break` where the server doesn't expect it. (b) The `with exn ->` arm sets `ComputationState` to `Aborted` and re-raises — it *decides* the analysis state; under the new model it is skipped, so the state stays `Computing`. See remediation. |
| [`signal.ml:58-76`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/plugins/eva/src/engine/signal.ml#L58-L76) `Signal.protect` | runs `cleanup` on `Sys.Break`/`Self.Abort`/`Log.Abort*`, then re-raises | catch-to-decide (the wrapper itself) | This is the bracket that the `iterator.ml:648` clean-up flows through. Its `Sys.Break` arm stops firing under the new model, so it must be rebuilt as an interruption-aware bracket (and/or paired with a `Sys.with_async_exns` boundary). The `protect_only_once` guard ([`signal.ml:56-64`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/plugins/eva/src/engine/signal.ml#L56-L64)) ensures `cleanup` runs once even on nested `protect`; rebuilding must preserve that. |
| [`current_loc.ml:19-23`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/kernel_services/ast_queries/current_loc.ml#L19-L23) `with_loc` | restore the previous "current source location" (`Fun.protect ~finally:set oldLoc`) | low severity (diagnostic only) | The current location only feeds error/warning messages. A skipped restore leaves a stale location for later messages; no soundness, resource, or result impact. Moot in batch; cosmetic in server mode. |

A handful of other `Fun.protect`/restore sites
([`cil_printer.ml:560`/`:570`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/kernel_services/ast_printing/cil_printer.ml#L555-L571)
restore printer flags;
[`mergecil.ml:1006`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/kernel_internals/typing/mergecil.ml#L1001-L1006);
[`floating_point.ml:85`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/libraries/floating_point/floating_point.ml#L83-L85)
restores the FPU rounding mode) are all transient, reset-before-next-read state of
the same "diagnostic / locally re-initialised" kind — fragile but not bugs.

## Why this is subtle

1. **Which handler catches Ctrl+C today is unpredictable.** `Sys.Break` fires at
   some arbitrary safe point, so whichever matching handler happens to be innermost
   at that instant catches it. The sites that catch it (the batch top-level, Eva's
   `Signal.protect`, the server request handler) are written defensively precisely
   because Ctrl+C can land anywhere.
2. **The handlers form a clean-up-then-decide chain.** A Ctrl+C deep inside an Eva
   analysis runs, innermost-out:
   ```
   Sys.with_async_exns wrapper                                  <- to be added
     cmdline.ml:199-202 catch_toplevel_run (Sys.Break -> print, on_error, exit 2)   <- decide exit code; run save-on-".break" hook
       analysis.ml:264 compute_from (with exn -> set Aborted; re-raise)             <- mark state, re-raise
         analysis.ml:246 Fun.protect ~finally:restore_signals                       <- uninstall Eva's signal handlers
           iterator.ml:648 Signal.protect ~cleanup (Sys.Break -> save partial; re-raise)  <- save partial results, re-raise
             project.ml:428 / current_loc.ml:23 Fun.protect ~finally               <- restore current project / location, re-raise
   ```
   i.e. inner frames clean up / save partial work and re-raise; outer frames mark
   the analysis state, choose the exit code, and trigger the save-on-`.break` hook.

The change splits in two:

- **Independent of where the wrapper goes (always happens):** every
  `Fun.protect`/`Signal.protect`/`with exn ->` frame between where the Break starts
  and the wrapper is skipped *no matter where the wrapper is*. So the partial-result
  save, the signal-handler restore, the project/location restore and the
  `ComputationState := Aborted` are all lost unconditionally. Whether each one
  *matters* is per-site and per-mode (see the table): all are moot for in-memory
  state in one-shot batch (the process exits). They are *also* moot in `-server` mode
  **as the server is currently written**, because a Ctrl+C ends the main loop and the
  server shuts down rather than serving more requests (see Bottom line). They would
  become real bugs only if a per-request `Sys.with_async_exns` boundary were added that
  lets the loop survive a Ctrl+C — i.e. the fix has to restore both the loop guard and
  the skipped restores at once.
- **Depends on where the wrapper goes:** which *outer* handler wins. A high wrapper
  at `catch_toplevel_run` restores the clean exit code 2 and the save-on-`.break`
  hook, but bypasses Eva's partial-result save and `Aborted` marking below it.
  Preserving today's "save partial Eva results on Ctrl+C" needs the wrapper placed
  **below** `Signal.protect`.

## What is and isn't backstopped

- **`at_exit` runs when `Sys.Break` (and `Stack_overflow`) is uncaught** — verified
  by experiment on stock OCaml 5.4.1: a program dying from an uncaught `Sys.Break`
  *and* from an uncaught `Stack_overflow` both ran the `at_exit` callback (exit
  code 2). So anything Frama-C registers via `Stdlib.at_exit` / `Extlib.safe_at_exit`
  still runs on an asynchronous-uncaught termination *if OxCaml's uncaught path
  matches stock behaviour* (one thing to confirm on the runtime).
- **External prover processes and their temp files are backstopped by `at_exit`.**
  `Command.command_generic`
  ([`command.ml:69-131`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/libraries/utils/command.ml#L69-L131))
  registers both the temp-file deletion and the child-process kill via
  `Extlib.safe_at_exit`
  ([`extlib.ml:117-122`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/libraries/stdlib/extlib.ml#L116-L122),
  a parent-process-guarded `at_exit`). So even if a Ctrl+C skips the in-line
  `delete ()`/`terminate ()` calls, the temp files are removed and the child prover
  is killed at process exit. Robust in batch; in server mode these `at_exit` hooks
  only fire when the *server* exits, so a long-lived server accumulates them — but
  the cooperative `Async.Cancel` path
  ([`command.ml:151-153`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/libraries/utils/command.ml#L151-L153))
  is the normal cancellation route there and is unaffected.
- **The save-on-Ctrl+C of the whole session is driven by the top-level catch, not a
  finaliser.** `special_hooks.ml`
  ([`special_hooks.ml:156-167`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/kernel_internals/runtime/special_hooks.ml#L156-L167))
  registers an `at_error_exit` hook that, on `Sys.Break`, saves the project with a
  `.break` suffix. That hook is run by `catch_toplevel_run`'s `run_on_error`
  ([`cmdline.ml:165-205`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/kernel_services/cmdline_parameters/cmdline.ml#L153-L205)),
  which only fires if `Sys.Break` is **caught** at the top level. So if the
  top-level `with exn ->` stops catching `Sys.Break`, the save-on-`.break` is
  skipped — making the top-level wrapper the high-value place to add the boundary.

## How to fix it

Use an interruption-safe version of the clean-up helper (the doc's "second version"
of `Fun.protect`; e.g.
`new_protect : init:(unit -> 'a) -> body:('a -> 'b) -> finaliser:('a -> unit) -> 'b`),
which runs the clean-up and then re-throws the interruption unchanged, paired with a
`Sys.with_async_exns` wrapper that converts the interruption back to an ordinary
exception *at the boundary*. The fixes split by *what each site needs from the
exception*:

| Site kind | Sites | Fix | Difficulty |
|-----------|-------|-----|------------|
| Resource / state restore — only needs "the finaliser must run" | `Project.on` ([`project.ml:428`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/libraries/project/project.ml#L426-L428)), `apply_once` reset ([`state_builder.ml:1039-1041`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/libraries/project/state_builder.ml#L1029-L1042)), `restore_signals` ([`analysis.ml:246`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/plugins/eva/src/engine/analysis.ml#L246)), `with_loc` ([`current_loc.ml:23`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/kernel_services/ast_queries/current_loc.ml#L19-L23)), printer/FPU restores | `new_protect` | mechanical, local |
| Catches the exception to decide / save partial work | `Signal.protect` (Eva partial-result save, [`signal.ml:58-76`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/plugins/eva/src/engine/signal.ml#L58-L76)), `compute_from`'s `with exn -> set Aborted` ([`analysis.ml:264-269`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/plugins/eva/src/engine/analysis.ml#L264-L269)), `catch_toplevel_run` ([`cmdline.ml:191-205`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/kernel_services/cmdline_parameters/cmdline.ml#L191-L205)), server `run` / loop ([`server/main.ml:151-166`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/plugins/server/main.ml#L151-L166), [`:524-531`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/plugins/server/main.ml#L516-L531)) | `Sys.with_async_exns` wrapper (placement = design call) | design call (where to put the wrapper, per mode) |

**Caveat — how the helper re-throws.** A `new_protect` that re-throws the
*asynchronous* exception frees the resource but keeps the Break flying past
downstream normal handlers (you still need an explicit `with_async_exns` for the
graceful result). A version that re-throws an *ordinary* exception instead would
silently re-enable Frama-C's many downstream `with Sys.Break ->` / catch-all arms,
letting them swallow a future Ctrl+C — against the model. Best approach:
interruption-safe helper + one explicit `with_async_exns` per mode. The state-restore
sites are correct either way.

`new_protect`'s `init`/`finaliser` shape also closes an acquire→enter race in
`Project.on` and in Eva's `compute_from`: `compute_from` installs the signal
handlers (`Signal.setup`) *before* entering the `Fun.protect`
([`analysis.ml:245-246`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/plugins/eva/src/engine/analysis.ml#L245-L246)),
so a Ctrl+C landing between installing the handlers and entering the body already
leaks the handlers today; moving the install into `init` closes that gap.

## Resolved during this pass

- **`Sys.Break` really is asynchronous in Frama-C** — confirmed via
  `Sys.catch_break true` at start-up
  ([`boot.ml:44`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/kernel_internals/runtime/boot.ml#L42-L44))
  and again in the server
  ([`server/main.ml:519`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/plugins/server/main.ml#L516-L520)).
  This is the single most important fact for this analysis.
- **Eva's `SIGINT` handler raises `Sys.Break` directly; its `SIGUSR1` handler does
  not raise** — confirmed
  ([`signal.ml:40-50`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/plugins/eva/src/engine/signal.ml#L36-L51)).
  `SIGUSR1` is cooperative (sets a flag read by `Signal.check`); only `SIGINT`
  raises from the handler, and it raises `Sys.Break`, which OxCaml already treats as
  asynchronous — so no `Sys.raise_async` is needed.
- **Frama-C's general cancellation (`Async.Cancel`) and the WP/Task timeout are
  cooperative** — `cancel` sets a flag, `Async.Cancel` is raised only at `yield`
  points ([`async.ml:63-108`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/libraries/utils/async.ml#L57-L108)),
  and `Command.spawn`'s timeout is a wall-clock poll with `raise Async.Cancel` from
  ordinary code ([`command.ml:136-158`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/libraries/utils/command.ml#L136-L158)),
  not a `setitimer`/signal-handler raise. No throwing-side change, and the existing
  `with Async.Cancel` handlers keep working.
- **`at_exit` runs on uncaught `Sys.Break`/`Stack_overflow`** — verified by
  experiment on stock OCaml 5.4.1 (both ran `at_exit`, exit code 2). So the
  prover-process/temp-file `Extlib.safe_at_exit` clean-up and the json/log flush
  survive an asynchronous-uncaught termination, *if OxCaml's uncaught path matches*.
- **Prover processes and temp files are doubly protected** — deleted/killed both
  in-line on success and via `Extlib.safe_at_exit`
  ([`command.ml:69-131`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/libraries/utils/command.ml#L69-L131)).
- **The save-on-Ctrl+C session save is driven by the top-level catch** — the
  `at_error_exit` `Sys.Break -> save_binary ".break"` hook
  ([`special_hooks.ml:156-167`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/kernel_internals/runtime/special_hooks.ml#L156-L167))
  only runs if the top-level `with exn ->` catches the Break, so the boundary must
  sit at/above `catch_toplevel_run` to keep it.

## Open questions (design calls / runtime confirmations)

- **Where to put the wrapper — the real design call, and it differs by mode.**
  - *Batch:* a `Sys.with_async_exns` around `f ()` inside `catch_toplevel_run`
    ([`cmdline.ml:191-205`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/kernel_services/cmdline_parameters/cmdline.ml#L153-L205))
    restores the clean exit code 2 and the save-on-`.break` hook. To *also* keep
    Eva's "save partial results on Ctrl+C", a second, lower boundary is needed below
    `Signal.protect` ([`iterator.ml:648`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/plugins/eva/src/engine/iterator.ml#L631-L648)).
  - *Server:* note first that the per-request handler `run`
    ([`server/main.ml:151-166`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/plugins/server/main.ml#L151-L166))
    **re-raises** `Sys.Break` ([`:158`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/plugins/server/main.ml#L158)),
    as does `process` ([`:363-365`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/plugins/server/main.ml#L363-L365)),
    so today a Ctrl+C is only ever *handled* by the main-loop guard
    [`while … with Sys.Break -> ()`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/plugins/server/main.ml#L524-L531),
    which **ends the loop and shuts the server down** — Ctrl+C is the shutdown
    mechanism, not a per-request abort. Under the new model the asynchronous `Sys.Break`
    bypasses that guard and is caught by the outer
    [`with exn -> … ; raise exn`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/plugins/server/main.ml#L536-L541),
    so the server still terminates (now via the "fatal error" path). The **observable**
    change is therefore the *shutdown style* (clean → fatal-error), not "wrong project
    for later requests" (there are no later requests). If the desired behaviour is
    instead a per-request abort that lets the server keep serving, that is a *new*
    design: add a `Sys.with_async_exns` per request (around `proc.handler`), change the
    per-request handler so it does **not** re-raise `Sys.Break`, and fix `Project.on` /
    `apply_once` so the now-surviving server isn't left with corrupted globals. Those
    pieces are coupled — adding the per-request boundary *without* fixing the restores
    is what would introduce the "wrong project current" bug.
- **OxCaml confirmations** (need the runtime, not the source):
  - that a *hand-installed* `Signal_handle` raising `Sys.Break` (Eva's `interrupt`
    handler) is made asynchronous for free, the same way `Sys.catch_break` is — if
    not, Eva's `SIGINT` handler needs `Sys.raise_async`;
  - that `at_exit` runs on asynchronous-uncaught termination (so the
    prover/temp-file clean-up and json/log flush survive).

## Checked and fine

- **`Stack_overflow`** — listed as asynchronous by the doc; Frama-C is deeply
  recursive (CIL traversals, the Eva dataflow), but there are **zero** catch sites.
  It already crashes uncaught; asynchronous or not, no recovery-path change. On its
  way out it skips the same `Fun.protect`/`Signal.protect` clean-up frames — the same
  "moot in batch, matters in server" analysis as `Sys.Break`, minus any recovery.
- **`Out_of_memory`** — exactly one catch site,
  [`apron_domain.ml:161`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/plugins/eva/src/domains/apron/apron_domain.ml#L150-L161)
  `with Out_of_memory -> top ()`. This catches a *synchronous* out-of-memory from
  Apron's GMP/MPFR C arithmetic (the optimistic-coercion path), turning it into a
  "give up precision" result — not an out-of-memory from the OCaml GC. The doc's
  "OOM from the GC becomes fatal" change therefore does not affect this recovery,
  but it is worth a note since it is the only `Out_of_memory` handler in the tree.
- **`Gc.finalise` / GC alarms / memprof** — none in Frama-C's own code. No risk of
  an exception escaping a finaliser or callback.
- **`SIGPIPE`** ([`server_socket.ml:309`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/plugins/server/server_socket.ml#L307-L309))
  — set to `Signal_ignore`; no handler, no raise.
- **The server `Killed` exception** ([`server/main.ml:105`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/plugins/server/main.ml#L105),
  raised by `kill` at [`:373`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/plugins/server/main.ml#L373))
  — raised from ordinary code at a check (`if killed then raise Killed`,
  [`:219`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/plugins/server/main.ml#L219)),
  not a signal handler. Cooperative; unaffected.
- **WP/Task `Canceled` and `Timeout`** — *result variants* of the task monad
  ([`task.ml:34-50`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/libraries/utils/task.ml#L34-L50)),
  matched as values (e.g. [`ProverWhy3.ml:118-119`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/plugins/wp/ProverWhy3.ml#L118-L119)),
  not caught as exceptions. The Task scheduler's only exception catch is
  `with Async.Cancel -> cancel_all` ([`task.ml:471-472`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/libraries/utils/task.ml#L470-L472)),
  on the cooperative `Async.Cancel`. Unaffected.
- **The `wp_parameters.ml` / `cmdline.ml` exit-code/`protect` predicates** — these
  *classify* an exception (e.g.
  [`wp_parameters.ml:1219-1224`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/plugins/wp/wp_parameters.ml#L1219-L1224)
  decides whether to swallow it;
  [`cmdline.ml:141-146`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/kernel_services/cmdline_parameters/cmdline.ml#L136-L146)
  maps it to an exit code). They are only reached once an exception has been caught,
  so they matter only insofar as the surrounding `try … with` still catches Break
  (handled by the boundary placement above).
- **The GUI** — **not in this repository.** The clone contains no GUI plug-in
  sources (only documentation tutorials under `doc/`); the interactive GUI ships as
  a separate package and is out of scope for this pass. No `Sys.set_signal` /
  `catch_break` / `Sys.Break` handling exists in any in-tree GUI code. Worth a
  separate look at the `frama-c-gui` package.

## Recommendations

In priority order. There is essentially one asynchronous exception (`Sys.Break`)
to reason about, and the analysis is dominated by *mode*: one-shot batch vs the
long-lived `-server` mode.

1. **Choose where to put the `Sys.with_async_exns` wrapper(s) — the real design
   call.**
   - *Batch:* wrap `f ()` in `catch_toplevel_run`
     ([`cmdline.ml:191-205`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/kernel_services/cmdline_parameters/cmdline.ml#L153-L205))
     to restore the clean exit code 2 and the save-on-`.break` hook; add a second,
     lower wrapper below Eva's `Signal.protect`
     ([`iterator.ml:648`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/plugins/eva/src/engine/iterator.ml#L631-L648))
     to keep "save partial Eva results on Ctrl+C".
   - *Server:* first decide the *intended* behaviour. Today a Ctrl+C is the server's
     **shutdown** signal — the main-loop guard
     [`while … with Sys.Break -> ()`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/plugins/server/main.ml#L524-L531)
     ends the loop and the server stops (the per-request `run` re-raises `Sys.Break` at
     [`:158`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/plugins/server/main.ml#L158)).
     To merely **preserve** today's behaviour, wrap the loop body (or the whole loop)
     in `Sys.with_async_exns` so the asynchronous `Sys.Break` becomes ordinary and the
     `with Sys.Break -> ()` guard fires → clean shutdown instead of the "fatal error"
     re-raise. If instead the goal is a per-request abort that keeps the server
     serving, that is a behaviour *change*: add `Sys.with_async_exns` around
     `proc.handler` ([`:151-166`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/plugins/server/main.ml#L151-L166)),
     stop re-raising `Sys.Break` there, *and* fix the project/state restores below it
     (recommendation 3) so the now-surviving server isn't left with corrupted globals.

2. **Make Eva's `Signal.protect` interruption-aware.**
   ([`signal.ml:58-76`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/plugins/eva/src/engine/signal.ml#L58-L76))
   Rebuild it so its clean-up (save partial results) runs on an asynchronous
   `Sys.Break`/`Self.Abort`, preserving the `protect_only_once` once-only guarantee.
   This restores Eva's deliberate partial-result feature, which otherwise silently
   stops working.

3. **Make the two state restorations interruption-safe via `new_protect`
   (hygiene; required *if* server-keeps-serving is adopted):** `Project.on`
   ([`project.ml:426-428`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/libraries/project/project.ml#L426-L428))
   and `apply_once`
   ([`state_builder.ml:1032-1041`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/libraries/project/state_builder.ml#L1029-L1042)).
   An asynchronous `Sys.Break` skips both restores (verified), leaving the wrong project
   current and Eva's "already run" flag stuck. As the server is written this is **not**
   observed (a Ctrl+C stops the server, so nothing reads the corrupted globals) — but
   these are kernel-wide primitives with many callers, and the moment a per-request
   boundary lets the server survive a Ctrl+C (recommendation 1, server variant) the
   corruption becomes a live "wrong project current" / "Eva won't re-run" bug. Rebuilding
   them as interruption-safe brackets fixes every caller at once and must land together
   with that boundary.

4. **Convert the remaining fragile restores to `new_protect` for robustness:**
   Eva's `restore_signals` `Fun.protect`
   ([`analysis.ml:246`](https://git.frama-c.com/pub/frama-c/-/blob/44585036cb05125b3128a173c7d98881c4bbdb42/src/plugins/eva/src/engine/analysis.ml#L246)),
   `Current_loc.with_loc`, the printer-flag and FPU-rounding restores. Safe today in
   batch, but they break the instant a Ctrl+C (or `Stack_overflow`) unwinds through
   them in a context where the state is read later (server mode).

5. **Confirm on the OxCaml runtime** (not derivable from source): that a
   hand-installed `Signal_handle` raising `Sys.Break` (Eva's `interrupt`) is treated
   as asynchronous like `Sys.catch_break` — otherwise add `Sys.raise_async` there;
   and that `at_exit` runs on asynchronous-uncaught termination (so the
   prover/temp-file clean-up and json/log flush survive).

**Sequencing note:** the throwing side needs essentially no work (`Sys.catch_break`
and the hand-installed `Sys.Break` are already asynchronous on OxCaml), so (1) the
wrapper placement can land on its own and restores today's behaviour. There is **no
day-one correctness bug in either batch or server mode as the code stands**: a Ctrl+C
ends a one-shot run or shuts the server down, and nothing reads the in-memory state
left broken by a skipped restore. The (3) restores are a *latent* correctness hazard
that only becomes a real bug if (1)'s server variant is taken in the
"keep serving across Ctrl+C" direction — in which case (3) must land with it. Without
the wrapper, the one *visible* server-mode change is the shutdown style (today's clean
"leave the loop normally" becomes a "Server interrupted (fatal error)" re-raise).
**Do *not*** rebuild the helpers so they turn the asynchronous exception into an
ordinary one: Frama-C has many downstream `with Sys.Break` / catch-all arms that
would then silently re-enable Ctrl+C swallowing — defeating the model.

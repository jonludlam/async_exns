# opam — async-exceptions initial triage

Repo: `ocaml/opam` @ `be8d399c36cc24ad3adfb90a8189af9a9bbad747` (shallow clone in `./opam`).

This is a first pass looking for code that works correctly today but breaks under
OxCaml's new rules for "asynchronous exceptions".

## What an asynchronous exception is

A few kinds of event can interrupt an OCaml program at almost any moment: pressing
Ctrl+C, a time-limit going off, or the program running out of stack space. Today they
surface as ordinary exceptions, so a normal `try … with` catches them. OxCaml changes
this: these interrupting exceptions now travel a separate route, an ordinary `try … with`
no longer catches them, and unless the program wraps the right region in a new special
handler the exception flies straight past every normal handler and either reaches that
wrapper or stops the program. Below we call them "interrupting exceptions".

## Bottom line

- **opam deliberately turns Ctrl+C into an exception.** At startup `OpamSystem.init` calls
  [`Sys.catch_break true`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/core/opamSystem.ml#L1285-L1289),
  which asks the standard library to install a `SIGINT` handler that **raises `Sys.Break`
  from inside the signal handler**. So `Sys.Break` is a genuine interrupting exception
  here, and opam catches it all over the place with ordinary `try … with Sys.Break` and
  catch-all handlers.
- **Only one exception is actually affected: `Sys.Break` (Ctrl+C).** "Out of stack"
  (`Stack_overflow`) can always happen too, but opam has **zero** places that catch it;
  "out of memory" (`Out_of_memory`) likewise has zero catch sites. There are no
  finalisers (`Gc.finalise`), no memprof callbacks, and no `setitimer` timers. The
  `sigalrm` use in `OpamProcess.safe_wait` and the `SIGWINCH` handler in `OpamStd`
  **do not raise** — they print verbose output / update a ref.
- **Throwing side: nothing to do.** opam installs its SIGINT handler with the standard
  `Sys.catch_break true`, and OxCaml already arranges for the `Sys.Break` escaping that
  handler to be an interrupting exception — so Ctrl+C propagates the new way with no code
  change. (The two *ordinary* `raise Sys.Break` sites
  ([`opamSystem.ml:33`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/core/opamSystem.ml#L32-L34),
  [`opamLocal.ml:45`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/repository/opamLocal.ml#L40-L46))
  come from a *child* process's exit status; those are normal synchronous raises, caught
  normally, and are unaffected.) So all the work is on the catching side below.
- **Genuine bug class — Ctrl+C during a package install leaves real, observable mess on
  disk.** opam's whole install engine works by *catching the exception and turning it
  into an ordinary return value*: a raised exception becomes a `` `Exception``/`Right e`
  value at `Job.catch` / `try … with e -> Right e` points, and *that value* is what
  drives the undo. An interrupting `Sys.Break` skips all of those catch arms, so it never
  becomes a value — it escapes the Job entirely, skipping (a) the per-package
  **roll-back** (`remove_package` of a half-installed package), (b) the parallel engine's
  killing and reaping of child processes, and (c) the apply-layer clean-up. Concretely:
  - **Half-installed package not rolled back** —
    [`opamAction.ml:1062-1066`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/client/opamAction.ml#L1062-L1066)
    / [`:1128-1132`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/client/opamAction.ml#L1128-L1132):
    files already copied into the switch prefix by `process_dot_install` are **left
    orphaned**, with the package *not* recorded as installed → the switch prefix and the
    `switch-state` record disagree, and every later opam command sees the mismatch.
    **GENUINE BUG.**
  - **Child processes orphaned** —
    [`opamProcess.ml:763-768`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/core/opamProcess.ml#L763-L768)
    `run` and the engine's
    [`fail`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/core/opamParallel.ml#L202-L239)
    skip `interrupt p`, so the running compiler/build child is not killed. This is
    temporary (the child is reparented to init when opam exits), so it is lower severity
    for a one-shot command.
  - **Terminal left in raw mode** —
    [`opamConsole.ml:756-767`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/core/opamConsole.ml#L756-L767):
    a Ctrl+C at an interactive menu or password prompt bypasses both the `Exn.finally
    reset` (which restores the terminal via `tcsetattr`) and the `with Sys.Break` arm, so
    the user's terminal is left with echo and canonical mode switched off. Visible to the
    user. **GENUINE BUG.**
- **The clean-up helpers are the high-leverage fix.** Almost all of opam's
  clean-up/roll-back goes through a handful of its *own* helper functions, each with many
  callers: `OpamStd.Exn.finally` / `Exn.finalise`
  ([`opamStd.ml:1625-1633`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/core/opamStd.ml#L1625-L1634)),
  `OpamProcess.Job.finally` / `Job.catch`
  ([`opamProcess.ml:920-939`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/core/opamProcess.ml#L920-L939)),
  plus 5 raw `Fun.protect` uses. The lock/temp/state wrappers
  (`OpamSystem.with_tmp_dir`, `OpamFilename.with_flock`/`with_tmp_dir`,
  `OpamGlobalState.with_`, `OpamSwitchState.with_`) are all built on `Exn.finalise`.
  Rebuilding those ~4 helpers as interruption-safe versions fixes all their callers.
- **Code that needs the new wrapper (`Sys.with_async_exns`):** the apply-layer catch-all
  [`opamSolution.ml:810-819`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/client/opamSolution.ml#L810-L819)
  (`with … | e -> `Exception e`, which today turns Ctrl+C into a recorded outcome and
  still runs the state-save / clean-up), and the top-level
  [`main_catch_all`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/client/opamCliMain.ml#L337-L394)
  `| Sys.Break -> exit User_interrupt`. *Where* to put the wrapper is the design call.
- **opam is a one-shot command — no daemon, server, watch, or REPL mode** (checked). So
  the "the process is exiting anyway" reasoning is valid for *in-memory* state and for
  locks *held only via a file descriptor*, but **not** for *on-disk* effects
  (half-installed packages, leftover temp dirs, terminal state), which the next opam
  command or the user's shell will see.
- Everything else checked (`Stack_overflow`, `Out_of_memory`, `Gc.finalise`, memprof,
  the `sigalrm`/`SIGWINCH`/`SIGPIPE` handlers, the lock-wait `Sys.Break` re-raise, the
  switch-state backup) is clear or low-severity.

## Which exceptions are affected

### `Sys.Break` is interrupting in opam (`Sys.catch_break true`)

[`OpamSystem.init`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/core/opamSystem.ml#L1285-L1289)
(called once at startup, documented in
[`opamSystem.mli:364-367`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/core/opamSystem.mli#L364-L367))
runs `Sys.catch_break true`. In the standard library that is
`Sys.set_signal Sys.sigint (Signal_handle (fun _ -> raise Break))` — i.e. **opam
installs a SIGINT handler that raises `Sys.Break` from inside the handler.** That is
exactly the interrupting-exception pattern: today the runtime lets the raised `Sys.Break`
surface at the next safe point, where one of opam's many `try … with Sys.Break` / `with
_` handlers catches it. Under OxCaml it takes the interrupting path and **none** of those
handlers catch it — it goes to the nearest `Sys.with_async_exns`, or it stops opam.

Two consequences:
1. **Throwing side — already handled.** Because opam uses the standard
   `Sys.catch_break true`, OxCaml already makes the `Sys.Break` escaping that handler an
   interrupting exception. opam needs no code change to throw it the new way; Ctrl+C
   propagates correctly as an interrupting `Sys.Break`.
2. **Catching side — the work.** Every `with Sys.Break` / catch-all handler stops firing;
   a `Sys.with_async_exns` wrapper is needed to turn the interrupting exception back into
   an ordinary one. This is where opam needs changes.

(`Sys.Break` is *also* raised the ordinary way from a child's exit status —
[`process_error`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/core/opamSystem.ml#L32-L34),
[`opamLocal.ml:45`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/repository/opamLocal.ml#L40-L46)
— when an opam-spawned process was itself killed by SIGINT. Those are normal raises;
catching them is unaffected, but they share the catch sites below.)

### The install engine turns the exception into a value; the catch arms are where that happens

`OpamSolution.parallel_apply` runs each package action as a `Job` and **passes results
around as values** (`` `Successful``/`` `Exception e``/`` `Error (`Aborted …)``). A
*raised* exception only becomes one of those values at explicit catch arms:
- [`opamAction.ml:1062-1066`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/client/opamAction.ml#L1062-L1066)
  `try … process_dot_install () … with e -> Right e` (per-package install),
- [`Job.catch`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/core/opamProcess.ml#L920-L925)
  (20 callers, e.g.
  [`opamAction.ml:289`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/client/opamAction.ml#L289-L294),
  [`:452`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/client/opamAction.ml#L452)),
- the parallel engine's loop catch
  [`opamParallel.ml:269`/`:290-293`/`:299-301`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/core/opamParallel.ml#L267-L303)
  `try … with e -> fail n e` (which kills/reaps the sibling jobs and builds the `Errors`
  value), and
- the apply-layer
  [`opamSolution.ml:810-819`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/client/opamSolution.ml#L810-L819)
  `with Parallel.Errors … -> … | e -> `Exception e`.

An interrupting `Sys.Break` skips *every one of these arms*, so it is never turned into a
value: the per-package roll-back
([`opamAction.ml:1128-1132`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/client/opamAction.ml#L1128-L1132)
`remove_package` of the half-install), the engine's child reaping, and the
state-save / clean-up
([`opamSolution.ml:852-864`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/client/opamSolution.ml#L852-L864))
are all skipped, and the exception propagates to `main_catch_all`, which also can't catch
it.

### What is and isn't backstopped (one-shot command)

- `at_exit` **does** run when `Sys.Break` is uncaught (verified on stock OCaml 5.x:
  `at_exit` fired, exit 2). opam relies on it: `logs_cleaner`
  ([`opamSystem.ml:199-218`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/core/opamSystem.ml#L199-L232)
  cleans temp files in the log dir), repo-state temp cleanup
  ([`opamRepositoryState.ml:334`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/state/opamRepositoryState.ml#L334)),
  and json/flush
  ([`opamCliMain.ml:491-494`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/client/opamCliMain.ml#L491-L494)).
  These survive interrupting-uncaught termination *as long as OxCaml's uncaught path runs
  `at_exit`* (one thing to confirm on the runtime).
- **Not** backstopped: the temp dirs from `OpamSystem.with_tmp_dir` /
  `OpamFilename.with_tmp_dir` (not in `at_exit`), half-installed switch prefixes, and
  terminal raw mode.
- **flock locks are OS file-descriptor locks**
  ([`opamSystem.ml:1179-1192`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/core/opamSystem.ml#L1179-L1194)):
  the OS releases them automatically when the process exits, so a skipped `funlock` does
  **not** leave a stale lock on disk for the next command. Safe for a one-shot command.

## Clean-up that could be skipped

All the sites below do clean-up or roll-back via an
`Exn.finalise`/`Exn.finally`/`Job.*`/`Fun.protect`/`with e ->` arm that an interrupting
`Sys.Break` bypasses. Verdicts assume a one-shot command (opam has no long-running mode).

| Site | Cleanup | Verdict | Why |
|------|---------|---------|-----|
| [`opamAction.ml:1062-1066`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/client/opamAction.ml#L1062-L1066) + [`:1128-1132`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/client/opamAction.ml#L1128-L1132) install + rollback | `try process_dot_install () with e -> Right e`, then `Right e → remove_package` rollback | **GENUINE BUG** | An interrupting `Sys.Break` after files are copied into the switch prefix but before the install finishes skips the `with e -> Right e` step, so the `remove_package` roll-back never runs. Orphan files stay in the switch prefix while the package is *not* in `switch-state`. Every later opam command sees the mismatch. On disk; not "exiting anyway". |
| [`opamConsole.ml:756-767`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/core/opamConsole.ml#L756-L767) menu/password prompt | `Exn.finally reset` (tcsetattr restore) + `with Sys.Break -> …` | **GENUINE BUG** | Ctrl+C at an interactive prompt bypasses the `Exn.finally` *and* the `with Sys.Break` arm, leaving the terminal in raw mode (echo and canonical mode off) after opam dies. Visible to the user (broken shell); no `at_exit` restore. |
| [`opamProcess.ml:763-768`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/core/opamProcess.ml#L763-L768) `run` + [`opamParallel.ml:202-239`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/core/opamParallel.ml#L202-L239) `fail` | `with e -> interrupt p; raise e` / reap siblings | low severity | An interrupting `Sys.Break` skips `interrupt p`, so the running build/compiler child is not killed and is reparented to init when opam exits. A temporary orphan process, not on-disk mess. (The "be careful to handle `Sys.Break`" warning at [`opamProcess.mli:109-111`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/core/opamProcess.mli#L109-L111) is exactly this.) |
| [`opamSwitchState.ml:1383-1394`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/state/opamSwitchState.ml#L1383-L1394) `with_` (+ [`opamGlobalState.ml:200-203`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/state/opamGlobalState.ml#L200-L203)) | `with e -> finalise e (drop st)` — `funlock` + `cleanup_backup false` | safe today, but only by luck (low) | `drop` releases the OS file-descriptor lock (auto-released at process exit anyway) and runs `cleanup_backup false`. The backup snapshot is *written at the start* ([`do_backup`, opamSwitchState.ml:1353-1357](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/state/opamSwitchState.ml#L1353-L1381)); the skipped `cleanup_backup true` would have *deleted* it, so on an interrupting Break the recovery snapshot is **kept**, not lost. Only the "restore with `opam switch import …`" *message* is skipped (diagnostic). |
| [`opamSystem.ml:406-415`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/core/opamSystem.ml#L406-L415) `with_tmp_dir` / [`opamFilename.ml:70`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/core/opamFilename.ml#L70) | `try … remove_dir dir … with e -> finalise e (remove_dir dir)` | safe today, but only by luck → fragile | The temp dir is not in `at_exit`, so an interrupting Break leaves it on disk. But build dirs are cleared at the start of the next build ([`opamSolution.ml:651-652`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/client/opamSolution.ml#L646-L654) `rmdir dir`), and other temp dirs land under `~/.opam/.../log`, which `logs_cleaner` cleans at `at_exit`. Wasted disk space at worst; no broken invariant. |
| [`opamFilename.ml:478-527`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/core/opamFilename.ml#L478-L527) `with_flock` / `with_flock_upgrade` / `with_flock_write_then_read` | `with e -> finalise e (funlock …)` | safe today, but only by luck | A skipped `funlock` leaks an OS file-descriptor lock that the kernel releases at process exit. No stale on-disk lock for the next command. Would become a real in-process deadlock only in a (non-existent) long-running mode. |
| `OpamStd.Exn.finally`/`finalise` ([`opamStd.ml:1625-1633`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/core/opamStd.ml#L1625-L1634)); `Job.finally`/`Job.catch` ([`opamProcess.ml:920-939`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/core/opamProcess.ml#L920-L939)); 5× `Fun.protect` ([`opamSystem.ml:388`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/core/opamSystem.ml#L388), [`opamConfigCommand.ml:176`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/client/opamConfigCommand.ml#L176), [`opamRepositoryState.ml:142`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/state/opamRepositoryState.ml#L142)/[`:220`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/state/opamRepositoryState.ml#L220), [`opamPatch.ml:406`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/core/opamPatch.ml#L406)) | the **helpers** underlying all the above | fragile (fix here) | These are the leverage points: ~11 `Exn.finally` + 21 `Exn.finalise` + 5 `Job.finally` callers + 20 `Job.catch` callers. Rebuilding the helpers as interruption-safe versions fixes every caller, including external library users (`opam-core`/`opam-state`/`opam-client` are public). |

## Why this is subtle

1. **Which handler catches Ctrl+C today is unpredictable.** `Sys.Break` fires at some
   arbitrary safe point, so whichever matching handler happens to be innermost at that
   instant catches it. The engine that catches the exception and turns it into a value is
   written carefully precisely because Ctrl+C can land anywhere mid-action.
2. **The handlers form a clean-up-then-convert chain.** A Ctrl+C deep inside a package
   install runs, innermost-out:
   ```
   Sys.with_async_exns wrapper                               <- to be added
     main_catch_all (opamCliMain.ml:392  Sys.Break -> exit User_interrupt)   <- abort + exit code + exec_at_exit
       opamSolution.ml:819  (| e -> `Exception e)            <- turn into a value; then state-save + clean-up
         opamParallel.ml:293 (with e -> fail n e)            <- reap sibling children, build Errors value
           opamAction.ml:1066 (with e -> Right e)            <- turn into a value; Right e -> remove_package ROLLBACK
             with_tmp_dir / with_flock / Exn.finalise        <- release temp dir / lock, RE-RAISE
   ```
   i.e. inner frames clean up and re-throw / turn the exception into a value; outer frames
   do the roll-back, child reaping, state-save, and the final exit-code decision.

The change splits in two:

- **Independent of where the wrapper goes (always happens):** every
  `Exn.finalise`/`Job.*`/`with e ->` frame between where the Break starts and the wrapper
  is skipped *no matter where the wrapper is*. So the per-package roll-back, child
  reaping, temp/lock release, and terminal restore are all lost unconditionally. Whether
  each one *matters* is per-site (see the table): the half-install roll-back and terminal
  restore are genuine bugs; the lock/temp release and the backup-cleanup are safe today
  but only by luck for a one-shot command.
- **Depends on where the wrapper goes:** which *outermost* handler wins. A
  `Sys.with_async_exns` placed high, at `main_catch_all`, restores the clean
  `User_interrupt` exit code and `exec_at_exit`, but still bypasses the apply-layer
  `` `Exception e`` step (so the per-package roll-back and state-save are still skipped).
  Preserving today's *graceful Ctrl+C* (roll-back + state-save + "Aborting.") needs the
  wrapper placed **below** `opamSolution.ml:819`, ideally per-action.

## How to fix it

Use an interruption-safe version of the clean-up helper (`new_protect`, which runs the
clean-up and then re-throws the interruption unchanged):
`new_protect : init:(unit -> 'a) -> body:('a -> 'b) -> finaliser:('a -> unit) -> 'b`.
The fixes split by *what each site needs from the exception*:

| Site kind | Sites | Fix | Difficulty |
|-----------|-------|-----|------------|
| Resource / state / lock clean-up — only needs "the finaliser must run" | `Exn.finally`/`finalise` ([`opamStd.ml:1625`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/core/opamStd.ml#L1625-L1634)), `Job.finally` ([`opamProcess.ml:934`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/core/opamProcess.ml#L934-L939)), `with_tmp_dir`, `with_flock`, `with_` state wrappers, terminal `reset` ([`opamConsole.ml:757`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/core/opamConsole.ml#L756-L765)), 5× `Fun.protect` | `new_protect` — fix the *helpers*; callers inherit it | mechanical, ~4 definitions |
| Catches the exception to decide what to do next — *consumes* the Break to make a control-flow / roll-back decision | `Job.catch` ([`opamProcess.ml:920`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/core/opamProcess.ml#L920-L925)), `opamAction.ml:1066` install rollback, `opamParallel.ml fail`, `opamSolution.ml:819`, `main_catch_all` | `Sys.with_async_exns` wrapper (placement = design call) | design call (where to put the wrapper) |

**Caveat — how the helper re-throws.** A `new_protect` that re-throws the *interrupting*
exception frees the resource but keeps the Break flying past downstream normal handlers
(you still need an explicit `with_async_exns` for the graceful-Ctrl+C result). A version
that re-throws an *ordinary* exception instead would silently re-enable opam's many
downstream `with _ ->` / `with Sys.Break ->` arms, letting them swallow a future Ctrl+C —
against the model. Best approach: interruption-safe helper + one explicit
`with_async_exns`. The clean-up helpers are resource-safe either way.

`new_protect`'s `init`/`finaliser` shape also closes acquire→enter races that the current
helpers leave open — e.g. `with_flock`
([`opamFilename.ml:478`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/core/opamFilename.ml#L478-L501))
acquires the lock *outside* the `try`, so a Break between `flock` and entering the body
already leaks the lock today.

## Resolved during this pass

- **`Sys.Break` really is interrupting in opam** — confirmed via `Sys.catch_break true`
  ([`opamSystem.ml:1287`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/core/opamSystem.ml#L1285-L1289)),
  which installs a SIGINT handler that raises. This is the single most important fact
  about opam for this analysis.
- **Throwing side needs no change** — OxCaml already makes the `Sys.Break` raised by the
  standard `Sys.catch_break` an interrupting exception. So opam does not need to install
  its own SIGINT handler or call any "throw it the new way" primitive; Ctrl+C already
  propagates correctly. All remaining work is on the catching side. (This makes the
  roll-back bug below a *present-day* bug on OxCaml, not merely a future risk: the
  interrupting `Sys.Break` is delivered today and already skips the catch-and-undo arms.)
- **`at_exit` runs when `Sys.Break` is uncaught** — verified by experiment on stock OCaml
  5.x (the `at_exit` callback fired; exit code 2). So `logs_cleaner`, repo-state cleanup
  and json/flush survive interrupting-uncaught termination *if OxCaml's path matches* (one
  thing to confirm). Half-installs / temp dirs / terminal state are **not** in `at_exit`.
- **flock locks are auto-released at process exit** — they are `Unix.lockf` advisory
  locks on file descriptors
  ([`opamSystem.ml:1186-1192`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/core/opamSystem.ml#L1186-L1192)),
  so a skipped `funlock` leaves no stale on-disk lock for the next command.
- **The install engine catches the exception and turns it into a value** — verified: the
  roll-back
  ([`opamAction.ml:1128-1132`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/client/opamAction.ml#L1128-L1132))
  is driven by the `` `Exception``/`Right e` value produced at the catch arms, not by a
  live `try…with` at the apply layer. State is flushed *between* actions
  ([`opamSolution.ml:353-354` comment + `add_to_install`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/client/opamSolution.ml#L353-L377)),
  so a Break *between* actions is safe; the bug window is a Break *inside* an action.
- **The switch-state backup is kept, not lost, on an interrupting Break** — `do_backup`
  writes it up front; only the success-path *delete* is skipped.
- **No daemon/server/watch/REPL mode** — opam is a one-shot command; the long-running-mode
  exemption does not apply (so on-disk effects matter, in-memory ones don't).

## Open questions (design calls / runtime confirmations)

- **Where to put the wrapper (the real design call).**
  - *Cheap, partial:* a high `Sys.with_async_exns` wrapping `run ()` in `main_catch_all`
    ([`opamCliMain.ml:337-394`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/client/opamCliMain.ml#L337-L394))
    restores the clean `User_interrupt` exit code and `exec_at_exit`, but still bypasses
    the per-package roll-back / state-save below it.
  - *Correct, more work:* a `Sys.with_async_exns` wrapper **per package action** (around
    `command`/`cont` in
    [`opamParallel.ml:267-303`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/core/opamParallel.ml#L267-L303),
    or around `job` in
    [`opamSolution.ml:544`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/client/opamSolution.ml#L544))
    so a Ctrl+C is turned back into a normal `Sys.Break` that the existing engine catches
    → roll-back + child reaping + state-save run as today. This preserves opam's current
    graceful-interrupt behaviour and is the recommended placement.
- **OxCaml confirmation** (needs the runtime, not the source): does `at_exit` run on
  interrupting-uncaught termination (so `logs_cleaner`/repo-cleanup/flush survive)?
  (The throwing-side question is *settled*: `Sys.catch_break` already yields an
  interrupting `Sys.Break` on OxCaml, so opam's SIGINT path needs no change — see
  "Resolved during this pass".)

## Checked and fine

- **`Stack_overflow`** — listed as interrupting by the doc; opam is recursive (graphs,
  solver), but has **zero** catch sites. Uncaught and fatal today; stays fatal. On its way
  out it does skip the same clean-up helpers (the ones that only clean up and re-throw) —
  same "safe today, only by luck" analysis as `Sys.Break`, minus any recovery path.
- **`Out_of_memory`** — **zero** catch sites; the doc's "out-of-memory from the GC becomes
  fatal" breaks no recovery path here.
- **`Gc.finalise` / `Gc.create_alarm` / memprof** — none in opam's own code. No risk of
  an exception escaping from a finaliser or callback.
- **`sigalrm`** ([`opamProcess.ml:674-689`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/core/opamProcess.ml#L674-L703)
  `safe_wait`) — the handler only calls `print_verbose_f` (prints buffered child output);
  it **does not raise**. The `Unix.alarm`/`setitimer`-style arming is for semi-synchronous
  verbose printing, not a timeout exception. Fine.
- **`SIGWINCH`** ([`opamStd.ml:860-863`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/core/opamStd.ml#L860-L863))
  — the handler only updates a `lazy` ref for terminal width; no raise. Fine.
- **`SIGPIPE`** ([`opamSystem.ml:1288`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/core/opamSystem.ml#L1288),
  reset to default in
  [`opamCliMain.ml:369`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/client/opamCliMain.ml#L369-L370))
  — a do-nothing handler; no raise.
- **Lock-wait `Sys.Break` re-raise**
  ([`opamSystem.ml:1170-1175`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/core/opamSystem.ml#L1170-L1175))
  — `with Sys.Break as e -> errmsg "\n"; raise e` while waiting on a contended lock.
  Diagnostic-only (prints a newline) then re-throws; an interrupting Break that skips it
  just omits the newline. Low severity — this is code that catches the exception only to
  print before re-throwing.
- **`display_error` / `parallel_apply` `` `Exception `` matches**
  ([`opamSolution.ml:230`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/client/opamSolution.ml#L230),
  [`:913`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/client/opamSolution.ml#L908-L915))
  — these are **pattern matches on an already-stored exception value**
  (`match error with Sys.Break ->`), not a live `try…with`. Unaffected by the change.
- **`OpamStd.Exn.fatal`** ([`opamStd.ml:787-790`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/core/opamStd.ml#L785-L790))
  — `fun e -> match e with Sys.Break -> raise e | … -> ()`, the helper opam uses to avoid
  swallowing Ctrl+C in catch-all handlers. Under the new model `Sys.Break` won't reach
  those catch-alls at all, so `fatal` becomes redundant for Break (still useful for
  `Assert_failure`/`Match_failure`). Not a bug.

## Recommendations

In priority order. The throwing side already works (`Sys.catch_break` gives an
interrupting `Sys.Break` on OxCaml), so all the work is on the catching side.

1. **Choose where to put the `Sys.with_async_exns` wrapper — the one real design call.**
   Recommended: **per package action**, around `command`/`cont` in the
   parallel engine
   ([`opamParallel.ml:267-303`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/core/opamParallel.ml#L267-L303))
   or `job` in
   ([`opamSolution.ml:544`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/client/opamSolution.ml#L544)),
   so a Ctrl+C turns back into a normal `Sys.Break` and the existing roll-back /
   child-reaping / state-save machinery runs exactly as today. Add a second high wrapper
   in `main_catch_all`
   ([`opamCliMain.ml:392`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/client/opamCliMain.ml#L392))
   for the clean exit code on any Break that escapes.

2. **Make the clean-up helpers interruption-safe (the bulk of the structural work).**
   Rebuild `OpamStd.Exn.finally`/`finalise`
   ([`opamStd.ml:1625-1633`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/core/opamStd.ml#L1625-L1634))
   and `OpamProcess.Job.finally`/`Job.catch`
   ([`opamProcess.ml:920-939`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/core/opamProcess.ml#L920-L939))
   as interruption-safe `new_protect` helpers (re-throw the *interrupting* exception, do
   **not** turn it into an ordinary one). This fixes `with_tmp_dir`,
   `with_flock`/`with_flock_*`, the `with_` state wrappers, and all `Fun.protect` callers
   (including external library users) at once.

3. **Fix the two genuine on-disk bugs explicitly** (independent; can land first):
   - Half-install roll-back
     ([`opamAction.ml:1062-1132`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/client/opamAction.ml#L1062-L1132)):
     make sure `remove_package` runs even on an interrupting Break (covered by the
     per-action wrapper from (1), or wrap the install body in `new_protect` that rolls
     back).
   - Terminal restore
     ([`opamConsole.ml:756-767`](https://github.com/ocaml/opam/blob/be8d399c36cc24ad3adfb90a8189af9a9bbad747/src/core/opamConsole.ml#L756-L767)):
     move `reset` into an interruption-safe `new_protect` so raw mode is restored on a
     Ctrl+C at a prompt (and/or register a tcsetattr restore in `at_exit`).

4. **Confirm on the OxCaml runtime** (not derivable from source): does `at_exit` run on
   interrupting-uncaught termination (so `logs_cleaner`/repo-cleanup/flush survive)?

**Sequencing note:** the throwing side needs no work (`Sys.catch_break` already yields an
interrupting `Sys.Break`), so (1) the wrapper placement can land on its own and restores
today's graceful-interrupt behaviour. (2) is independent and the bulk of the leverage.
(3) is the concrete user-visible payoff and is largely covered by (1)+(2). **Do *not***
rebuild the helpers so they turn the interrupting exception into an ordinary one: opam has
dozens of downstream `with Sys.Break`/`with _` arms that would then silently re-enable
Ctrl+C swallowing — defeating the model.

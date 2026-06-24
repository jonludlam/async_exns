# Rocq — first look at the asynchronous-exception change

Repo: `rocq-prover/rocq` @ `1add672f8d81b00eb5e107dc3ce4fc200322ee5e`
(shallow clone in `/home/jon/async_exns/rocq`; origin
`https://github.com/rocq-prover/rocq.git`).
What this pass covers: a first search for code that works correctly today but
would break under OxCaml's new rules for "asynchronous exceptions".

Background in one sentence: an *asynchronous exception* is an interrupting event
— pressing Ctrl+C, a time limit going off, or running out of stack — that can
surface at almost any point in the program. Today an ordinary `try … with`
catches these; under the new rules it does not, unless you use a new wrapper
(`Sys.with_async_exns`) that turns the interrupting exception back into an
ordinary one. We call these "interrupting exceptions" below.

This program is the original motivating example for the whole change: the design
note recalls that *"pressing Ctrl+C at an inopportune moment in the middle of a
Coq proof would cause it to succeed, even if false"*. So the first thing to pin
down is exactly how Rocq turns Ctrl+C (and time limits) into exceptions, and
where it catches them.

## Bottom line

- Two interrupting exceptions can travel the new route, and **only in some
  modes**:
  - **`Control.Timeout`**, raised **from inside a `SIGALRM` signal handler** by
    the per-command `Timeout` / `-time` / `Set Default Timeout` machinery on
    Unix. This is a genuine interrupting raise.
  - **`Sys.Break`** (Ctrl+C). How it is produced depends on the mode, and this
    is the crux of the whole analysis:
    - In the normal compiler and interactive prompt (`rocq compile` / `coqc`,
      and `rocq repl` / `coqtop`), Ctrl+C is handled by OCaml's standard
      `Sys.catch_break true`, which raises `Sys.Break` from the runtime's
      `SIGINT` handler — so it is interrupting.
    - In the editor/IDE backend (`rocqide` document manager, the
      language-server-style process) Rocq installs its **own** `SIGINT` handler
      that *sometimes* raises `Sys.Break` directly (interrupting) and *otherwise*
      only sets a flag that ordinary code later checks (cooperative).
    - The general `Control.check_for_interrupt` checkpoint used throughout the
      kernel and tactics raises `Sys.Break` **cooperatively** from ordinary
      code — that is *not* an interrupting exception and is unaffected.
- **`Stack_overflow`** is always live (the deeply recursive kernel can produce
  it), but it is **never caught for recovery** — only formatted into an error
  message at the very top — so there is no recovery path that regresses.
- **Throwing side:** the one place that needs a throwing-side change is the
  Unix timeout handler in `Control.unix_timeout`, which must raise
  `Control.Timeout` the new way (`Sys.raise_async`). Ctrl+C via
  `Sys.catch_break` needs **no throwing-side change** — OxCaml makes that
  `Sys.Break` interrupting automatically.
- **Good news on the catching side:** Rocq is unusually well-prepared. It does
  *not* clean up proof/environment state in `try … with` handlers around the
  failing command; instead the recovery loop restores a saved snapshot
  afterwards. And its general-purpose cleanup helper (`Util.try_finally`) is
  **already** built on a masking bracket (`Memprof_coq.Masking.with_resource`),
  not on `Fun.protect`. So most of the "skipped cleanup" risk is already
  handled.
- **No genuine state-corruption bug found.** The places that catch interrupting
  exceptions to *decide what to do* (the recovery loops and the `Fail` / `Succeed`
  / `Timeout` vernacular controls) are the ones that need a `Sys.with_async_exns`
  wrapper so they keep working; placing that wrapper is the real design call.
- A few cleanup spots are **safe today only by luck**: the `.vo` writer's
  "delete the half-written file" handler (`library.ml`), and the diagnostic
  timing/`-time` wrappers. None corrupt logical state; the `.vo` one is the only
  externally-visible one.
- Everything else checked (`Stack_overflow`/`Out_of_memory` recovery,
  finalisers, memprof callbacks, other signals, the `noncritical` discipline,
  the checker `coqchk`) is clear.

## The mechanisms that matter

### Ctrl+C in the compiler and the prompt — `Sys.catch_break`

`init_ocaml` calls `Sys.catch_break true` once at startup
([`sysinit/coqinit.ml#L47-L50`](https://github.com/rocq-prover/rocq/blob/1add672f8d81b00eb5e107dc3ce4fc200322ee5e/sysinit/coqinit.ml#L47-L50)).
This is OCaml's stock mechanism: the runtime's `SIGINT` handler raises
`Sys.Break`. Under OxCaml this `Sys.Break` is automatically made interrupting,
so there is **no throwing-side change to make** for Ctrl+C in `coqc` /
`rocq compile` and in the `coqtop` / `rocq repl` prompt.

### The general interrupt checkpoint — cooperative, unaffected

The pervasive interrupt check is
[`lib/control.ml#L21-L29`](https://github.com/rocq-prover/rocq/blob/1add672f8d81b00eb5e107dc3ce4fc200322ee5e/lib/control.ml#L21-L29):

```ocaml
let check_for_interrupt () =
  if !interrupt then begin interrupt := false; raise Sys.Break end;
  ...
```

`check_for_interrupt` raises `Sys.Break` from **ordinary code**, only when the
`Control.interrupt` flag has been set. A signal handler that merely sets that
flag (as the IDE backend does in its flag mode, and as the Windows timeout
thread does) produces a `Sys.Break` that is an ordinary synchronous exception —
**not interrupting**, and therefore unaffected by the change. This is the
cooperative-cancellation pattern, and it is the dominant interruption mechanism
inside the kernel, tactics and the proof engine.

### Time limits — `Control.Timeout` from a `SIGALRM` handler (the real throwing-side case)

The `Timeout` vernacular, the `-time` machinery's timeout and `Set Default
Timeout` all go through `Control.timeout`. On Unix this is `unix_timeout`
([`lib/control.ml#L31-L56`](https://github.com/rocq-prover/rocq/blob/1add672f8d81b00eb5e107dc3ce4fc200322ee5e/lib/control.ml#L31-L56)):

```ocaml
let timeout_handler _ = raise Timeout in
...
let psh = Sys.signal Sys.sigalrm (Sys.Signal_handle timeout_handler) in
...
let _ = setitimer ITIMER_REAL {it_interval = 0.; it_value = n} in
try Ok (f x)
with Timeout as exn -> ... Error info
```

The timer is armed with `setitimer ITIMER_REAL` and `Control.Timeout` is raised
**from inside the `SIGALRM` handler** — so it is genuinely interrupting. It is
immediately caught by the `try … with Timeout` right around the body and turned
into a `result` (`Ok`/`Error`); it does *not* propagate as an exception beyond
`unix_timeout`. The restoration of the previous timer/handler is done with the
masking bracket (`Masking.with_resource ~release:restore_timeout`,
[`lib/control.ml#L50-L51`](https://github.com/rocq-prover/rocq/blob/1add672f8d81b00eb5e107dc3ce4fc200322ee5e/lib/control.ml#L50-L51)).

Two consequences under the new rules:

1. **Throwing side:** `timeout_handler` must raise `Control.Timeout` via
   `Sys.raise_async`, or the timeout will stop the program instead of being
   caught.
2. **Catching side:** the `try … with Timeout` *immediately around the body* in
   `unix_timeout` will no longer fire for an interrupting `Control.Timeout`. So
   this same function needs the `Sys.with_async_exns` wrapper placed around
   `f x` (turning the interrupting `Timeout` back into an ordinary one that the
   adjacent `with Timeout` still catches). Because the catch and the body sit in
   the same small function, the fix is local and the placement is obvious — this
   is the easiest wrapper to place in the whole program.

The Windows fallback `windows_timeout`
([`lib/control.ml#L58-L94`](https://github.com/rocq-prover/rocq/blob/1add672f8d81b00eb5e107dc3ce4fc200322ee5e/lib/control.ml#L58-L94))
uses a watcher thread that sets `Control.interrupt := true`; the actual
`Sys.Break` is then raised cooperatively from `check_for_interrupt`. So the
Windows timeout is cooperative and unaffected.

There is also `Control.protect_sigalrm`
([`lib/control.ml#L107-L123`](https://github.com/rocq-prover/rocq/blob/1add672f8d81b00eb5e107dc3ce4fc200322ee5e/lib/control.ml#L107-L123)),
which deliberately *defers* a `SIGALRM` so that the timeout fires only after a
block of I/O finishes (preventing a timeout from corrupting half-written
output). It catches no interrupting exception itself and is unaffected.

### The recovery loops — where interrupting exceptions are caught today

Each long-lived mode catches everything (including today's interrupting
`Sys.Break`) and recovers. These are the catch-to-decide sites that need a
`Sys.with_async_exns` wrapper to keep recovering:

- **Interactive prompt (`coqtop` / `rocq repl`).** `read_and_execute` has a
  catch-all `| any ->` that prints the error and **keeps the previous state**,
  then loops
  ([`toplevel/coqloop.ml#L512-L543`](https://github.com/rocq-prover/rocq/blob/1add672f8d81b00eb5e107dc3ce4fc200322ee5e/toplevel/coqloop.ml#L512-L543)).
  The comment at
  [`toplevel/coqloop.ml#L346-L350`](https://github.com/rocq-prover/rocq/blob/1add672f8d81b00eb5e107dc3ce4fc200322ee5e/toplevel/coqloop.ml#L346-L350)
  says "Ctrl-C is handled internally as `Sys.Break` instead of aborting Rocq".
  Today that catch-all is exactly what lets the prompt survive a Ctrl+C and
  print "User interrupt." Under the new rules an interrupting `Sys.Break` walks
  straight past `| any ->` and (with no wrapper) terminates the prompt.
- **Editor/IDE backend (`rocqide` document manager).** Its own `SIGINT`
  handler raises `Sys.Break` directly while a request is being evaluated
  ([`ide/rocqide/idetop.ml#L39-L42`](https://github.com/rocq-prover/rocq/blob/1add672f8d81b00eb5e107dc3ce4fc200322ee5e/ide/rocqide/idetop.ml#L39-L42)):

  ```ocaml
  let f _ = if valid_interrupt () then
              if !catch_break then raise Sys.Break else Control.interrupt := true in
  Sys.set_signal Sys.sigint (Sys.Signal_handle f);
  ```

  `catch_break` is turned on only around request evaluation by the
  `interruptible` wrapper
  ([`ide/rocqide/idetop.ml#L512-L519`](https://github.com/rocq-prover/rocq/blob/1add672f8d81b00eb5e107dc3ce4fc200322ee5e/ide/rocqide/idetop.ml#L512-L519)).
  When it is on, that `raise Sys.Break` *is* an interrupting raise; when it is
  off, the handler only sets the cooperative flag. The interrupting `Sys.Break`
  is caught by the catch-all `with any -> … Fail (handler.handle_exn any)` in
  `abstract_eval_call`
  ([`ide/rocqide/protocol/xmlprotocol.ml#L775-L809`](https://github.com/rocq-prover/rocq/blob/1add672f8d81b00eb5e107dc3ce4fc200322ee5e/ide/rocqide/protocol/xmlprotocol.ml#L775-L809)),
  which converts it to a `Fail` answer so the backend keeps serving the editor.
  Under the new rules the interrupting `Sys.Break` bypasses that `with any`, and
  with no wrapper the whole backend process dies instead of answering "the
  command was interrupted". *Because this handler raises `Sys.Break` directly
  from its own signal handler, it is the one Ctrl+C site that — like the timeout
  — also depends on the throwing side: it gets the interrupting `Sys.Break` for
  free from OxCaml, but only because OxCaml special-cases `Sys.Break` escaping a
  handler.*
- **One-shot compile (`coqc` / `rocq compile`).** `coqc_run` wraps the whole
  compilation in `with exn -> … exit exit_code`
  ([`toplevel/coqc.ml#L51-L61`](https://github.com/rocq-prover/rocq/blob/1add672f8d81b00eb5e107dc3ce4fc200322ee5e/toplevel/coqc.ml#L51-L61)).
  Because the action is to print and `exit`, the only observable change under
  the new rules is the formatted message and the exit code: today a Ctrl+C is
  caught here and `CErrors.exit_code` returns 129; under the new rules it would
  terminate as an uncaught interrupting exception. `at_exit flush_all` still
  runs (verified — see below), so output is still flushed.

### State protection does *not* live in command handlers

This is why the doc's "false proof succeeds" bug is structurally hard to
reproduce here today, and why the change is comparatively safe. When a command
fails, the proof state machine is **not** repaired by a `try … finally` around
the command. Instead:

- `process_expr` catches the failure, prints it, then calls
  `Stm.edit_at ~doc state.sid` to roll the document back to the pre-command
  snapshot, and re-raises
  ([`toplevel/vernac.ml#L164-L176`](https://github.com/rocq-prover/rocq/blob/1add672f8d81b00eb5e107dc3ce4fc200322ee5e/toplevel/vernac.ml#L164-L176)).
- State itself is captured as immutable snapshots
  (`Vernacstate.freeze_full_state`, `Summary.freeze`/`unfreeze`,
  [`library/summary.ml#L111-L176`](https://github.com/rocq-prover/rocq/blob/1add672f8d81b00eb5e107dc3ce4fc200322ee5e/library/summary.ml#L111-L176))
  and restored by the document manager, not by exception handlers.

So an interrupting exception that skips an intermediate handler does not leave a
dangling "half-applied tactic" — the next thing the recovery loop does is reset
to a saved snapshot.

### The `noncritical` discipline — already excludes interrupting exceptions

Rocq has a long-standing convention that inner `try … with` handlers must not
swallow critical exceptions. `CErrors.noncritical`
([`lib/cErrors.ml#L164-L171`](https://github.com/rocq-prover/rocq/blob/1add672f8d81b00eb5e107dc3ce4fc200322ee5e/lib/cErrors.ml#L164-L171))
returns `false` for `Sys.Break`, `Control.Timeout`, `Stack_overflow`,
`Out_of_memory` and the memprof interrupt, and `CErrors.is_async`
([`lib/cErrors.ml#L85-L91`](https://github.com/rocq-prover/rocq/blob/1add672f8d81b00eb5e107dc3ce4fc200322ee5e/lib/cErrors.ml#L85-L91))
returns `true` for exactly those. There are ~170 `with e when noncritical e ->`
sites; today they deliberately re-raise an interrupting exception. Under the new
rules they would not even be entered by an interrupting exception — the outcome
is the same (the interrupting exception keeps propagating), so these are not a
regression. The directly relevant use is the `Fail` vernacular (next section).

### The `Fail` / `Succeed` / `Timeout` vernacular controls — the motivating bug, already guarded

`with_fail` is the implementation of `Fail …` (which expects its body to fail)
and is the closest thing to the doc's "a Ctrl+C turns a failed proof into a
success" scenario
([`vernac/vernacControl.ml#L185-L211`](https://github.com/rocq-prover/rocq/blob/1add672f8d81b00eb5e107dc3ce4fc200322ee5e/vernac/vernacControl.ml#L185-L211)):

```ocaml
let with_fail f =
  try let x = f () in Error x
  with e ->
    let _, info as exn = Exninfo.capture e in
    (* Don't catch async exceptions, don't turn anomalies into successes *)
    if CErrors.is_async e || CErrors.is_sync_anomaly e then Exninfo.iraise exn;
    Ok (Loc.get_loc info, CErrors.iprint exn)
```

Today this *already* refuses to treat an interrupting exception (Ctrl+C, a
timeout, stack overflow, out-of-memory) as "the command failed as expected": the
`is_async` guard re-raises it. Under the new rules an interrupting `Sys.Break`
would bypass the whole `with e` and propagate anyway — so the soundness property
("Ctrl+C inside `Fail` must not be reported as a success") is preserved either
way. The change makes this *more* robust, not less: it removes the program's
reliance on getting the `is_async` test right. (The timeout flavour of this is
covered by the `unix_timeout` analysis above: a `Timeout` is already converted
to a result inside `Control.timeout`, then surfaced as the ordinary
`CmdTimeout`, [`vernac/vernacControl.ml#L130-L148`](https://github.com/rocq-prover/rocq/blob/1add672f8d81b00eb5e107dc3ce4fc200322ee5e/vernac/vernacControl.ml#L130-L148).)

## Cleanup that could be skipped on an interrupting exception

Each site below restores state or releases a resource in a handler, and that
cleanup would no longer run when an interrupting exception passes through. After
tracing each, **none corrupts logical proof/environment state**; the only
externally-visible one is the `.vo` writer.

| Site | Cleanup | Verdict | Why |
|------|---------|---------|-----|
| [`vernac/library.ml#L489-L500`](https://github.com/rocq-prover/rocq/blob/1add672f8d81b00eb5e107dc3ce4fc200322ee5e/vernac/library.ml#L489-L500) `save_library_base` | `with exn -> Sys.remove f; raise` — delete the half-written `.vo` | safe today, only by luck | The channel *close* is already in a masking bracket (`Masking.with_resource ~release:ObjFile.close_out`), but the "delete the partial file" step is a plain `with exn` *outside* that bracket. The output is written directly to the final path `f` (no temp-file-then-rename), so an interrupting exception mid-marshal would leave a truncated `.vo` on disk that the delete no longer removes. In one-shot `coqc` the process exits non-zero, so a `make`-style build re-runs; but the stale file is externally visible. The source already flags this as imperfect ("would be nice … extra cleanup in the case the computation raises"). |
| [`vernac/library.ml#L57-L70`](https://github.com/rocq-prover/rocq/blob/1add672f8d81b00eb5e107dc3ce4fc200322ee5e/vernac/library.ml#L57-L70) `fetch_delayed` | `with e -> close_in ch; raise e` | safe today, only by luck | A read-only input channel; skipping the close leaks one file descriptor on the way to process exit. No logical state, no soundness impact. |
| [`lib/system.ml#L358-L367`](https://github.com/rocq-prover/rocq/blob/1add672f8d81b00eb5e107dc3ce4fc200322ee5e/lib/system.ml#L358-L367) `measure_duration` and [`lib/system.ml#L395-L404`](https://github.com/rocq-prover/rocq/blob/1add672f8d81b00eb5e107dc3ce4fc200322ee5e/lib/system.ml#L395-L404) `count_instructions` | `with e -> … Error (exn, duration)` — record elapsed time/instructions, then re-raise | low severity (diagnostic) | Used by the `Time` / `-time` / instruction-count controls. Skipping the measurement on an interrupting exception only loses the timing line for that interrupted command; no result, soundness or resource impact. |
| [`lib/control.ml#L107-L123`](https://github.com/rocq-prover/rocq/blob/1add672f8d81b00eb5e107dc3ce4fc200322ee5e/lib/control.ml#L107-L123) `protect_sigalrm` | `with e -> Sys.set_signal sigalrm old_handler; raise` — restore the previous `SIGALRM` handler | safe today, only by luck | Restores the saved handler on the way out. The only interrupting exception that could pass through is a Ctrl+C `Sys.Break` (the `SIGALRM` it guards is *deferred*, so no `Timeout` fires here). Skipping the restore would leave the deferring handler installed; in practice this runs inside a single output operation and the next `protect_sigalrm` re-installs its own handler. Worth converting to the masking bracket for tidiness. |

`Util.try_finally` / `Util.with_finally`
([`lib/util.ml#L131-L161`](https://github.com/rocq-prover/rocq/blob/1add672f8d81b00eb5e107dc3ce4fc200322ee5e/lib/util.ml#L131-L161)),
the generic cleanup helper used in the file loader, profiling and the `-time`
output, is **already** built on `Memprof_coq.Masking.with_resource` rather than
`Fun.protect`. So the loader's `close_out`/`emit_time` finalisers in
`load_vernac` / `load_vernac_core`
([`toplevel/vernac.ml#L100-L219`](https://github.com/rocq-prover/rocq/blob/1add672f8d81b00eb5e107dc3ce4fc200322ee5e/toplevel/vernac.ml#L100-L219))
are run through the masking path already, not a plain handler.

## Why this is subtle

1. **Which "Ctrl+C exception" you have depends on the mode and the moment.** The
   same exception value, `Sys.Break`, is produced three different ways: by the
   stock `Sys.catch_break` runtime handler (interrupting), by the IDE backend's
   own handler when a request is being evaluated (interrupting), and by
   `check_for_interrupt` reading a flag (cooperative, *not* interrupting). Only
   the first two change behaviour. You cannot tell from a `with Sys.Break ->`
   line which kind it will catch — it depends on which mode is running and
   whether the break arrived during evaluation or at a checkpoint.

2. **Today's catching is nondeterministic.** When `Sys.Break` or `Control.Timeout`
   is interrupting, it fires at an arbitrary safe point, so whichever matching
   handler is innermost at that instant catches it. The recovery loops are
   written defensively (catch-all `| any ->`) precisely because of this.

3. **The handlers form a print-then-recover chain, with state restored
   separately.** Innermost-out for a command failure in the interactive prompt:

   ```
   with_async_exns wrapper                  ← to be added (per command)
     coqloop read_and_execute (| any -> print; keep old state; continue)   ← recover
       vernac process_expr  (with reraise -> print; Stm.edit_at rollback; re-raise)  ← roll back state, re-raise
         vernac interp_vernac (with reraise -> annotate loc; re-raise)             ← annotate, re-raise
   ```

   The crucial point is that the *state rollback* (`Stm.edit_at`) and the
   *recovery decision* (keep going) are in different frames, and neither relies
   on a `finally` inside the failing command. So an interrupting exception that
   skips the inner annotate frame still reaches the rollback frame and the
   recovery frame — unless the new rules let it skip *those* too, which is
   exactly why the wrapper has to sit at or above the recovery frame.

4. **The change splits in two.** Independently of where the wrapper goes, the
   inner annotate/print frames are skipped by an interrupting exception — but
   they only add a location and a message, so nothing breaks. What *depends* on
   the wrapper is whether the recovery frame (the prompt's `| any ->`, the IDE's
   `with any -> Fail`, the compiler's `with exn -> exit`) still runs. Put the
   wrapper at or just above each recovery frame and behaviour is preserved; omit
   it and the long-lived modes die on Ctrl+C instead of recovering.

## How to fix it

A second, interruption-aware cleanup bracket is expected to be available (the
doc's "second version" of `Fun.protect`; here we call it `new_protect`): it runs
the cleanup, then re-throws the interruption still-interrupting. Rocq already has
the right primitive in `Memprof_coq.Masking.with_resource`, so the cleanup-side
work is mostly "route the remaining hand-written handlers through it".

The fixes split by what each site needs from the exception:

| Site kind | Sites | Fix | Difficulty |
|-----------|-------|-----|------------|
| Catches the interrupting exception to *decide what to do next* (recover, convert to a `Fail` answer, exit) | the timeout catch in `Control.unix_timeout`; the prompt's `read_and_execute` recovery; the IDE backend's `abstract_eval_call` recovery; `coqc_run` / the `coqchk` top-level | `Sys.with_async_exns` wrapper | design call: where to place each wrapper |
| Resource / state cleanup that only needs "the finaliser must run" | `library.ml` `.vo` writer's "delete partial file"; `fetch_delayed` `close_in`; `protect_sigalrm` handler restore; the `system.ml` timing wrappers | route through the existing `Memprof_coq.Masking.with_resource` (i.e. `new_protect`) | mechanical, local |

The most important wrapper is the cheapest: inside `Control.unix_timeout`, wrap
the `f x` body in `Sys.with_async_exns` so the adjacent `with Timeout` still
catches the (now interrupting) `Control.Timeout`. After that, one wrapper per
long-lived recovery loop (the prompt loop and the IDE request loop) keeps Ctrl+C
recovery working; the one-shot `coqc` / `coqchk` paths only need a wrapper if you
want to preserve the exact formatted message and exit code 129 on Ctrl+C.

**Caveat — how the bracket re-throws.** For the four cleanup sites it does not
matter whether the bracket re-throws the interruption as still-interrupting or
as ordinary — the resource is released either way. But do **not** make the
cleanup bracket convert interrupting→ordinary as a side effect: that would let a
downstream `with _ ->` swallow a Ctrl+C again, which is the very thing the new
model is meant to stop. Use a masking bracket that re-throws still-interrupting,
plus the explicit `Sys.with_async_exns` wrappers above.

## Resolved during this pass

- **No proof/environment state is repaired inside command-level handlers.** The
  STM restores immutable snapshots (`Stm.edit_at`, `Summary.unfreeze`) from the
  recovery loop after the failure, not via a `finally` around the failing
  command. So an interrupting exception skipping the inner frames does not leave
  a half-applied tactic or a corrupt environment — verified by reading
  `process_expr` ([`toplevel/vernac.ml#L164-L176`](https://github.com/rocq-prover/rocq/blob/1add672f8d81b00eb5e107dc3ce4fc200322ee5e/toplevel/vernac.ml#L164-L176))
  and the freeze/unfreeze machinery.
- **The `Fail` vernacular already refuses to absorb interrupting exceptions**
  via `CErrors.is_async` ([`vernac/vernacControl.ml#L185-L196`](https://github.com/rocq-prover/rocq/blob/1add672f8d81b00eb5e107dc3ce4fc200322ee5e/vernac/vernacControl.ml#L185-L196)),
  so the historical "Ctrl+C makes a false proof succeed" bug is already guarded;
  the new model preserves the guarantee and removes the reliance on the test.
- **`at_exit` runs on an uncaught interrupting exception** — verified empirically
  on OCaml 5.4.1: a program dying from an uncaught `Sys.Break` and from
  `Stack_overflow` both ran their `at_exit` callback (exit code 2). So the
  `at_exit flush_all` in `coqc`/`coqtop`/`coqchk`
  ([`toplevel/coqtop.ml#L17`](https://github.com/rocq-prover/rocq/blob/1add672f8d81b00eb5e107dc3ce4fc200322ee5e/toplevel/coqtop.ml#L17),
  [`checker/coqchk_main.ml#L18`](https://github.com/rocq-prover/rocq/blob/1add672f8d81b00eb5e107dc3ce4fc200322ee5e/checker/coqchk_main.ml#L18))
  still flushes output even if a Ctrl+C terminates the one-shot run — *provided
  OxCaml's interrupting-termination path matches stock uncaught behaviour* (worth
  one confirmation on the OxCaml runtime).
- **The memprof allocation limit is already result-based.**
  `Control.alloc_limit` uses `Memprof_coq.limit_allocations`, which catches its
  own memprof-callback interrupt internally and returns `Ok`/`Error`
  ([`lib/control.ml#L125-L131`](https://github.com/rocq-prover/rocq/blob/1add672f8d81b00eb5e107dc3ce4fc200322ee5e/lib/control.ml#L125-L131)),
  so no interrupt from a memprof callback propagates as an exception into user
  code.

## Still open (design calls, not investigations)

- **Where to place each `Sys.with_async_exns` wrapper.** Inside
  `Control.unix_timeout` is clear-cut. For the long-lived modes, the wrapper
  should sit per command / per request (so one interrupted command yields a
  recoverable error, not a dead process): around `read_and_execute` in the
  prompt loop and around `abstract_eval_call` (or the per-request body in
  `process_xml_msg`) in the IDE backend. For one-shot `coqc` / `coqchk`, decide
  whether preserving the exact Ctrl+C message and exit code 129 is worth a
  wrapper at all.
- **OxCaml confirmations** (cheap, need the OxCaml runtime, not the source):
  that `at_exit` runs on interrupting-uncaught termination (so the final
  `flush_all` happens); and that the throwing-side change in `unix_timeout` plus
  the free `Sys.Break` from `Sys.catch_break` behave as the doc describes.

## Obviously fine (gaps checked and cleared)

- **`Stack_overflow`** — the kernel is deeply recursive and can raise it, but
  there are **no recovery catches**: the only matches are the error-message
  formatters `explain_exn` in the checker
  ([`checker/coqchk_main.ml#L256-L259`](https://github.com/rocq-prover/rocq/blob/1add672f8d81b00eb5e107dc3ce4fc200322ee5e/checker/coqchk_main.ml#L256-L259))
  and `himsg`, which match on the exception as data after it is already caught at
  the top level. No recovery path regresses. It does, like any interrupting
  exception, skip the cleanup spots in the table above on its way out — that is
  the same "safe today only by luck" surface, already assessed there.
- **`Out_of_memory`** — same: matched only by the checker's `explain_exn`
  formatter; no recovery. The doc's "out-of-memory from the GC becomes fatal"
  change breaks no recovery path here.
- **Finalisers / `Gc.create_alarm` / memprof callbacks raising** — none in the
  tree except the memprof allocation limit, which is result-based (above).
- **Other signals** — `SIGUSR1` is used by the IDE's Break button to enter the
  Ltac debugger ([`plugins/ltac/tactic_debug.ml#L477`](https://github.com/rocq-prover/rocq/blob/1add672f8d81b00eb5e107dc3ce4fc200322ee5e/plugins/ltac/tactic_debug.ml#L477))
  and is set to `Signal_ignore` by default; the worker manager's `Unix.kill …
  Sys.sigint` is process control, not an in-process raise. The `coqworkmgr`
  helper uses `Sys.catch_break true` and a top-level `with Sys.Break`
  ([`tools/coqworkmgr/rocqworkmgr.ml#L207`](https://github.com/rocq-prover/rocq/blob/1add672f8d81b00eb5e107dc3ce4fc200322ee5e/tools/coqworkmgr/rocqworkmgr.ml#L207),
  [`#L223`](https://github.com/rocq-prover/rocq/blob/1add672f8d81b00eb5e107dc3ce4fc200322ee5e/tools/coqworkmgr/rocqworkmgr.ml#L223))
  to shut down its socket server — a small standalone helper; the same
  per-loop-wrapper advice applies if it is kept long-lived.
- **The async task queue worker manager** catches `Sys.Break` in a `kill_if`
  thread that loops on `Unix.sleep`
  ([`stm/asyncTaskQueue.ml#L180-L186`](https://github.com/rocq-prover/rocq/blob/1add672f8d81b00eb5e107dc3ce4fc200322ee5e/stm/asyncTaskQueue.ml#L180-L186))
  — a cooperative loop; treat it like the other recovery sites if it must keep
  working under Ctrl+C.
- **The ~170 `with e when noncritical e ->` handlers** — `noncritical` already
  excludes every interrupting exception, so they deliberately do not catch them
  today; under the new rules they are not entered by an interrupting exception
  either. No behaviour change.
- **No existing `Sys.with_async_exns` / `Sys.raise_async`** — the new API is not
  yet used anywhere (expected).

## Recommendations

In priority order. The set of interrupting exceptions to reason about is small:
`Control.Timeout` (Unix `SIGALRM`) and `Sys.Break` (Ctrl+C, in the compiler /
prompt / IDE modes); `Stack_overflow` is live but never recovered.

1. **Make the Unix timeout throw the new way, and wrap its body (required,
   coupled).** In `Control.unix_timeout`
   ([`lib/control.ml#L33-L56`](https://github.com/rocq-prover/rocq/blob/1add672f8d81b00eb5e107dc3ce4fc200322ee5e/lib/control.ml#L33-L56)):
   raise `Control.Timeout` from `timeout_handler` via `Sys.raise_async`, and wrap
   `f x` in `Sys.with_async_exns` so the adjacent `with Timeout` still converts
   it to a result. These two land together: the first alone turns every timeout
   into a program exit; the second alone does nothing.

2. **Add a per-command / per-request `Sys.with_async_exns` wrapper to each
   long-lived recovery loop.** The interactive prompt's `read_and_execute`
   ([`toplevel/coqloop.ml#L512-L543`](https://github.com/rocq-prover/rocq/blob/1add672f8d81b00eb5e107dc3ce4fc200322ee5e/toplevel/coqloop.ml#L512-L543))
   and the IDE backend's request handling around `abstract_eval_call`
   ([`ide/rocqide/idetop.ml#L512-L550`](https://github.com/rocq-prover/rocq/blob/1add672f8d81b00eb5e107dc3ce4fc200322ee5e/ide/rocqide/idetop.ml#L512-L550),
   [`ide/rocqide/protocol/xmlprotocol.ml#L775-L809`](https://github.com/rocq-prover/rocq/blob/1add672f8d81b00eb5e107dc3ce4fc200322ee5e/ide/rocqide/protocol/xmlprotocol.ml#L775-L809))
   each need a wrapper so a Ctrl+C is converted back to an ordinary `Sys.Break`
   that the existing catch-all still turns into "User interrupt" / a `Fail`
   answer — otherwise these processes die on Ctrl+C instead of recovering. The
   IDE handler additionally relies on its own `raise Sys.Break` becoming
   interrupting, which OxCaml provides automatically.

3. **Decide whether the one-shot paths need a wrapper.** `coqc_run`
   ([`toplevel/coqc.ml#L51-L61`](https://github.com/rocq-prover/rocq/blob/1add672f8d81b00eb5e107dc3ce4fc200322ee5e/toplevel/coqc.ml#L51-L61))
   and the `coqchk` top-level
   ([`checker/coqchk_main.ml#L469-L475`](https://github.com/rocq-prover/rocq/blob/1add672f8d81b00eb5e107dc3ce4fc200322ee5e/checker/coqchk_main.ml#L469-L475))
   only print and `exit`; a Ctrl+C that bypasses them terminates anyway and
   `at_exit` still flushes. Add a wrapper only if the exact formatted message and
   exit code 129 on Ctrl+C are worth preserving.

4. **Route the remaining hand-written cleanup handlers through the masking
   bracket for robustness.** The `.vo` writer's "delete partial file"
   ([`vernac/library.ml#L489-L500`](https://github.com/rocq-prover/rocq/blob/1add672f8d81b00eb5e107dc3ce4fc200322ee5e/vernac/library.ml#L489-L500)),
   `fetch_delayed`'s `close_in`
   ([`vernac/library.ml#L57-L70`](https://github.com/rocq-prover/rocq/blob/1add672f8d81b00eb5e107dc3ce4fc200322ee5e/vernac/library.ml#L57-L70)),
   `protect_sigalrm`'s handler-restore
   ([`lib/control.ml#L107-L123`](https://github.com/rocq-prover/rocq/blob/1add672f8d81b00eb5e107dc3ce4fc200322ee5e/lib/control.ml#L107-L123)),
   and the `system.ml` timing wrappers
   ([`lib/system.ml#L358-L404`](https://github.com/rocq-prover/rocq/blob/1add672f8d81b00eb5e107dc3ce4fc200322ee5e/lib/system.ml#L358-L404))
   are safe today only by accident; routing them through
   `Memprof_coq.Masking.with_resource` (already the basis of `Util.try_finally`)
   makes them robust. The `.vo` writer is the only one with an externally-visible
   effect; consider also writing to a temp path and renaming on success.

5. **Confirm on the OxCaml runtime** (not derivable from source): that `at_exit`
   runs on interrupting-uncaught termination, and that the `Sys.catch_break`
   `Sys.Break` and the `unix_timeout` `Sys.raise_async` behave as the doc
   describes.

**Sequencing note:** (1) is self-contained and is the highest-value, lowest-risk
change (the catch and the body are in the same small function). (2) is the main
user-visible work for the long-lived modes and must land before relying on Ctrl+C
recovery. (3) and (4) are independent polish. **Do not** convert the cleanup
brackets in (4) into interrupting→ordinary converters — that would re-enable
`with _ ->` swallowing of Ctrl+C, defeating the point of the new model.

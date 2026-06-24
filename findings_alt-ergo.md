# Alt-Ergo — first look at the asynchronous-exception change

Repo: `OCamlPro/alt-ergo` @ `4471dadc` (shallow clone in `./alt-ergo`).
What this pass covers: a first search for code that works correctly today but
would break under OxCaml's new rules for "asynchronous exceptions".

Background in one sentence: an *asynchronous exception* is an interrupting event
— pressing Ctrl+C, a time-limit going off, or running out of stack — that can
surface at almost any point in the program. Today an ordinary `try … with`
catches these; under the new rules it does not, unless you use a new wrapper
(`Sys.with_async_exns`) that turns the interrupting exception back into an
ordinary one. We call these "interrupting exceptions" below.

## Bottom line

- In Alt-Ergo, only **one** exception travels this new interrupting route:
  `Util.Timeout`, raised from the `SIGVTALRM`/`ITIMER_VIRTUAL` signal handler.
  Nothing else is affected — Ctrl+C calls `exit` directly, and there is no caught
  `Stack_overflow` or `Out_of_memory`, and no finalisers.
- **Throwing side:** the `SIGVTALRM` handler must throw `Util.Timeout` the new way
  (`Sys.raise_async`), or the timeout will stop the program instead of propagating.
- **One genuine, active bug:** `steps.ml:206` `apply_without_step_limit` — a timeout
  during `SAT.push` permanently turns off `--steps-bound` for the rest of the session.
- **Three spots that are safe today only by luck (and therefore fragile):**
  `matching.ml:664`, `options.ml:939`, `timers.ml:353` — they work today only because
  of a defensive re-initialisation or a one-shot timer, not because the clean-up is
  unneeded. Fix all four clean-up spots with the interruption-safe `new_protect`.
- **Two places that catch the exception to decide what to do next** (turn it into a
  result or abort): `satml_frontend`/`fun_sat`'s `i_dont_know` and `solving_loop`'s
  `exit_as_timeout`. These need the `Sys.with_async_exns` wrapper; *where* to put it
  is the one real open design decision.
- Everything else checked (`Stack_overflow`, `Out_of_memory`, finalisers, other
  signals, wildcard handlers, plugins, the JS build) is clear.

## Which exception is affected: `Util.Timeout`

Alt-Ergo's timeout is a CPU-time timer that raises an exception **from inside a
signal handler** — which is exactly an interrupting exception under the new model.

- Installed: `signals_profiling.ml:69-73` sets the `SIGVTALRM` handler to
  `fun _ -> Options.exec_timeout ()`.
- Fired:   `options.ml:899` sets `timeout := fun () -> raise Util.Timeout`
  (`exec_timeout` calls it). The timer is armed with `Unix.setitimer ITIMER_VIRTUAL`
  in `options.ml:921` `Time.set_timeout`.
- Caught: there are 13 ordinary `with Util.Timeout ->` sites, mostly in
  `satml_frontend.ml` (956, 1096, 1139, 1149, 1261), plus `frontend.ml:478`,
  and the outermost `solving_loop.ml:269` / `solving_loop.ml:854`.

Because the timer can fire at **any** safe point, `Util.Timeout` is the only
interrupting exception this program can produce this way (see "Checked and fine"
for why nothing else applies).

This has two separate consequences under the new rules:

1. **Throwing side (investigation point 2):** the `SIGVTALRM` handler must throw it
   the new way with `Sys.raise_async` (otherwise the runtime treats an ordinary
   exception escaping the handler as a fatal one). Without this, raising
   `Util.Timeout` from the handler *stops the program*.
2. **Catching side (investigation point 1):** none of the `with Util.Timeout ->`
   handlers will fire anymore — an interrupting `Util.Timeout` jumps straight to the
   nearest `Sys.with_async_exns` wrapper, or stops the program. So we need such a
   wrapper, and we need to decide *where* to put it.

(Aside: the SIGINT / Ctrl+C handler at `signals_profiling.ml:30-41` calls `exit 1`
**directly** — it never raises an exception through OCaml code — so the change does
not affect it. There is no use of `Sys.Break` anywhere.)

## Clean-up that could be skipped on a timeout

All four sites below restore state or release a resource through a handler or
`Fun.protect ~finally`, and that clean-up will **no longer run** when `Util.Timeout`
arrives as an interrupting exception. But after tracing each one, only **one is an
active bug**. The others are safe today only by luck — for an incidental reason (state
is re-initialised at the next use, or the timer is one-shot), not because the clean-up
is unnecessary. That makes them **fragile**, not broken — they should still move to
`new_protect` (see "How to fix it"), both for tidiness and because the lucky accident
protecting them disappears the moment a *second* interrupting exception (e.g. a
catchable `Sys.Break`) could pass through the same code.

| Site | Clean-up | Verdict | Why |
|------|---------|---------|-----|
| `steps.ml:206-215` `apply_without_step_limit` | restore `steps_bound` (`exception e -> steps_bound := bound; raise e`) | **GENUINE BUG** | `steps_bound` is a long-lived global (set only by `--steps-bound` / `set_steps_bound`; *not* reset per goal — `reset_steps` only zeroes counters). A timeout during the wrapped `SAT.push` (`frontend.ml:297`) leaves it stuck at `-1`, which **silently disables the step limit for the rest of the session** (`steps.ml:148` `!steps_bound <> -1 && …`). A lasting broken invariant. |
| `matching.ml:664-671` `query` | `reset_cache_refs ()` (`with e -> reset_cache_refs (); raise e`) | safe today, only by luck | `query` also calls `reset_cache_refs ()` at its **start** (`matching.ml:662`), and the caches (`cache_are_equal_light/full`) are module-private, query-local memoization. The next `query` resets them before any read, so a skipped trailing reset only **keeps the maps in memory** until then — no broken invariant. Fragile if those caches ever gain a reader outside `query`. |
| `options.ml:939` `Time.with_timeout` | `Fun.protect ~finally:unset_timeout` | safe today, only by luck | `ITIMER_VIRTUAL` is **one-shot** (`it_interval = 0.`, `options.ml:926`). The *only* exception that skips `finally` is `Util.Timeout` — i.e. the timer having **already fired and spent itself**. So no live timer leaks into the next goal. It becomes a real leak only if a second interrupting exception (Ctrl+C) can unwind through `with_timeout` while the timer is still armed. |
| `timers.ml:353-359` `with_timer` | `Fun.protect ~finally:timer_pause` | low severity (diagnostic) | profiling-only; see below. |

`Fun.protect` matters twice over: the design doc says it will **not** catch
interrupting exceptions, and a second, interruption-aware variant is wanted. All three
`Fun.protect`/handler sites rely on `finally` running on a timeout.

## `timers.ml` `with_timer` — low-severity detail (diagnostics only)

- **`timers.ml:353-359` `with_timer`** (`Fun.protect ~finally:(fun _ -> !timer_pause …)`).
  `start` (`timers.ml:295-309`) *pushes* the current timer onto `env.stack`; `pause`
  (`312-321`) *pops* it — so the `finally` keeps a nested timer **stack** balanced. An
  interrupting `Util.Timeout` skipping it leaves a dangling stack entry / wrong
  durations. But:
  - it is gated on `Options.get_timers ()` (off by default);
  - it is pure **profiling/timing bookkeeping** (`TimerTable` elapsed-time
    accumulation) — no solver result, no soundness, no OS resource;
  - it does **not** feed back into the timeout (the timer is driven by
    `Options.Time` / `ITIMER_VIRTUAL`, independent of this stack).

  **Library exposure:** `with_timer` is in the public interface `timers.mli:109` and
  the module ships as `alt-ergo-lib` (`src/lib/dune` `public_name`), so an external
  `AltErgoLib` consumer *could* call it — but it is an odd low-level primitive to call
  directly. All ~35 in-tree callers are uniform solver internals
  (`assume`/`query`/`make`/`solve`/`union`/`add`).

  **Do the callers need investigating?** No. `with_timer`'s only clean-up is timing
  bookkeeping; callers consume `f`'s *return value*, and `f`'s own invariants/resources
  are protected (or not) independently at their own sites. So the wrapper adds nothing
  safety-relevant regardless of what `f` is.

## Why this is subtle

Once you fix where the new wrapper goes, the behaviour change is easy to compute, so
the real work is pinning down *today's* behaviour — and it is not as simple as "site X
handles timeouts":

1. **Today's catching is nondeterministic.** The timer fires at an *arbitrary* safe
   point, so the handler that catches a given timeout is *whichever matching handler is
   innermost on the call stack at the instant the signal lands*. Any of the 13 might
   catch it, depending on timing. The sites are written defensively precisely because
   of this.
2. **The handlers form a clean-up-then-decide chain.** A timeout deep in matching today
   runs, innermost-out:
   ```
   with_async_exns wrapper             ← to be added
     solving_loop.ml:854 / handle_exn        (Util.Timeout -> exit_as_timeout)         ← abort
       satml_frontend unsat_rec              (Util.Timeout -> i_dont_know …)           ← turn into a graceful "unknown(timeout)"
         matching.ml:669 query               (with e -> reset_cache_refs (); raise e)  ← clean up, then RE-RAISE
   ```
   That is: inner frames clean up and **re-raise**; an outer frame makes the final
   decision — turning it into a graceful per-goal "unknown(timeout)" — and the run
   continues (it never reaches the top-level `exit`).

So the change splits into two parts:

- **Independent of where the wrapper goes (guaranteed):** every intermediate handler
  between the timeout's origin and the wrapper is skipped *regardless of where* the
  wrapper goes. So the `matching.ml` / `steps.ml` restorations are lost
  unconditionally — you can assert this without any design decision. Whether that
  *matters* is per-site (see the clean-up table): `steps.ml` is an active bug,
  `matching.ml` is safe today by luck.
- **Dependent on where the wrapper goes:** which *final* handler "wins". The wrapper
  turns an interrupting exception back into an ordinary one *at its location*, so any
  handler nested *below* the wrapper (e.g. `satml_frontend`'s `i_dont_know`) is
  bypassed. Putting the wrapper high up turns "graceful per-goal unknown" into
  "whole-run exit"; preserving the graceful answer requires placing the wrapper *below*
  `i_dont_know`.

## How to fix it

A second, interruption-aware bracket is expected to be available (the doc's "second
version" of `Fun.protect`; e.g.
`new_protect : init:(unit -> 'a) -> body:('a -> 'b) -> finaliser:('a -> unit) -> 'b`).
This is an interruption-safe version of the clean-up helper: it runs the clean-up, then
re-throws the interruption unchanged. It cleanly **splits** the fixes by *what each site
needs from the exception*:

| Site kind | Sites | Fix | Difficulty |
|-----------|-------|-----|------------|
| Resource / invariant clean-up — only needs "the finaliser must run" | `options.ml:939`, `timers.ml:353`, `matching.ml:664`, `steps.ml:206` | `new_protect` | mechanical, local |
| Catches the exception to decide what to do next — *consumes* the timeout and makes a control-flow decision | `satml_frontend.ml` `i_dont_know`, `solving_loop.ml` `exit_as_timeout` | `Sys.with_async_exns` wrapper (or rewrite the handler to catch the interrupting exception) | design call (where to put the wrapper) |

The clean-up sites are not just rewritable but **race-improved** by the
`init`/`finaliser` shape: e.g. `Time.with_timeout` currently arms the timer
(`set_timeout`) *inside* the protected body; moving the arming into `init` lets the
bracket hold off interrupting exceptions across the arm→enter transition, closing the
gap where a timeout could land between arming the timer and entering `f`.

**Caveat — how it re-throws.** Whether `new_protect` *on its own* restores the old
end-to-end behaviour depends on how it re-throws after the finaliser:
- re-throw as **still-interrupting**: the resource is freed, but the exception keeps
  flying past downstream ordinary handlers → you *still* need an outer
  `with_async_exns` for the graceful-unknown result.
- re-throw as **ordinary**: `new_protect` doubles as a conversion point — which would
  silently re-enable downstream `with _ ->` swallowing of a Ctrl+C/timeout, arguably
  against the point of the new model.

For pure **resource safety** at the four clean-up sites it doesn't matter which. The
best reading of the doc's intent is the version that holds off interruptions and
re-throws still-interrupting, so `new_protect` (4 sites) and one `with_async_exns`
wrapper (2 sites) are **complementary, not substitutes**.

## Resolved during this pass

- **`i_dont_know` only turns the timeout into a result** — verified.
  `satml_frontend.ml:115` = `env.unknown_reason <- Some ur; raise I_dont_know`;
  `fun_sat.ml:166` = `raise (I_dont_know { env with unknown_reason = Some ur })`. The
  `model_gen_on_timeout` / `update_model_and_return_unknown` variants additionally
  compute a *best-effort* model (re-arming the one-shot timer for a model-generation
  phase). All of this is **producing a result, not restoring an invariant** — if an
  interrupting timeout bypasses it you get a *degraded result* (a raw timeout at the
  wrapper, no partial model), not corruption. So these stay in the
  catch-to-decide bucket; none are hidden `new_protect` sites.
- **`at_exit` runs on uncaught exceptions** — verified empirically on OCaml 5.4.1: a
  program dying from an uncaught `Not_found` *and* from `Stack_overflow` both run the
  `at_exit` callback (exit code 2). So `Output.close_all` (`parse_command.ml:1562`)
  still flushes even on an uncaught interrupting termination — *provided OxCaml's
  interrupting-termination path matches stock uncaught behaviour* (worth one
  confirmation on OxCaml). Moot anyway once the wrapper is placed (timeout → normal
  exit). **Low concern.**
- **The `satml_frontend.ml:1085` / model-generation `set_timeout` re-arm dances are
  fine** — same one-shot-timer reasoning as `options.ml:939`: every re-arm is a fresh
  one-shot, and the only exception that skips a finally/handler is the timer firing
  itself, which spends it. No `ITIMER_VIRTUAL` can leak into the next goal while
  `Util.Timeout` is the only interrupting exception in play.

## Open questions (design calls, not investigations)

- **Where to put the wrapper.** The right `Sys.with_async_exns` spot is probably
  per-goal/per-statement, so one goal's timeout still yields a `Timeout`/`unknown`
  answer instead of killing the whole run. Candidates: the outer `solving_loop.ml:854`
  / `handle_exn`, versus *below* `satml_frontend`'s `i_dont_know` to preserve the
  graceful per-goal "unknown(timeout)" (see "Why this is subtle"). This is the one
  genuinely unresolved design decision.
- **OxCaml confirmations** (cheap, need the OxCaml runtime, not the source): that
  `at_exit` runs on interrupting-uncaught termination; and that `alt-ergo-js` truly
  never arms `Unix.setitimer`.

## Checked and fine

- **`Stack_overflow`** — listed as an interrupting exception by the doc, and Alt-Ergo
  is deeply recursive, but there are **zero catch sites**. It already crashes uncaught;
  interrupting or not, no behaviour change.
- **`Out_of_memory`** — **zero catch sites**, so the doc's "OOM from the GC becomes
  fatal" change breaks no recovery path here.
- **`Gc.finalise`** — none. The only GC callback is `Gc.create_alarm` in `gc_debug.ml`
  (debug-only). No "exception from a finaliser" risk.
- **Other signals** — SIGINT/SIGTERM/SIGQUIT handlers `exit` or print; SIGPROF prints
  (`signals_profiling.ml`); profiling-mode Ctrl+C just toggles display. Only SIGVTALRM
  raises. (`profiling.ml:121` arms a *separate* `ITIMER_PROF`/SIGPROF timer purely for
  periodic profiling prints — harmless.) No `Sys.Break` anywhere.
- **Wildcard handlers** — 7 total; the only one that restores state is `matching.ml:669`
  (covered in the clean-up table above). The rest: `with _ -> assert false`
  impossible-case guards (`theory.ml:157`, `satml.ml:643` — an interrupting exception
  skipping them is, if anything, *more* correct); a `with _ -> false` char check
  (`frontend.ml:312`); a pure memoization fallback (`models.ml:145`); startup
  plugin-load error reporting (`config.ml:50`, outside any timeout scope).
- **Plugins (`fm-simplex`)** — no `Timeout` catches, no signals, no
  `Fun.protect`/clean-up. Clean.
- **JS / worker build (`worker_js.ml`)** — no signals; its `Timeout _` is a result-type
  *pattern match*, not an exception catch. Interrupting exceptions are a native-runtime
  concern, so `alt-ergo-js` is essentially out of scope (worth a one-line confirmation
  that `Unix.setitimer` is unused/stubbed there).

## Recommendations

In priority order. The whole change is small and localized — there is exactly one
interrupting exception (`Util.Timeout`) to reason about.

1. **Make the timeout throw the new way (required, or nothing else matters).** Change
   the `SIGVTALRM` handler so `Util.Timeout` leaves the handler via `Sys.raise_async`
   (`options.ml:899` `exec_timeout` / `signals_profiling.ml:69-73`). Without this a
   timeout stops the program instead of propagating.

2. **Decide where the `Sys.with_async_exns` wrapper goes — the one real design call.**
   Place it *below* `satml_frontend`/`fun_sat`'s `i_dont_know` (i.e. wrapping the raw
   solve, per goal) so a timeout is converted back to an ordinary `Util.Timeout` that
   the existing inner handlers still catch and turn into a graceful "unknown(timeout)".
   A single high wrapper (e.g. at `solving_loop.ml:854`) is simpler but downgrades
   per-goal timeouts to a whole-run abort — only acceptable if that behaviour change is
   intended. Confirm against the `i_dont_know` / `print_status` path before committing.

3. **Fix the one active bug: `steps.ml:206` `apply_without_step_limit`.** Convert it to
   the interruption-aware bracket (`new_protect` with a `finaliser` restoring
   `steps_bound`) so a timeout during `SAT.push` can't leave `--steps-bound`
   permanently disabled. This is the only site with a present-day soundness impact.

4. **Convert the three fragile clean-up sites to `new_protect` for robustness:**
   `options.ml:939` `with_timeout`, `timers.ml:353` `with_timer`, `matching.ml:664`
   `query`. Safe today (one-shot timer / defensive re-init / profiling-only), but they
   break the instant a second interrupting exception (e.g. a catchable `Sys.Break`) can
   pass through them. Prefer the `init`/`finaliser` shape — it also closes the
   arm→enter race in `with_timeout`.

5. **Confirm on the OxCaml runtime** (not derivable from source): that `at_exit` runs
   on interrupting-uncaught termination (so `Output.close_all` still flushes), and that
   `alt-ergo-js` never arms `Unix.setitimer`.

**Sequencing note:** (1) and (2) are coupled and must land together — (1) alone turns
every timeout into a program exit; (2) alone does nothing. (3) is independent and can
land first. (4) is tidy-up/future-proofing and can land anytime.

**Do *not*** reflexively wrap clean-up in a converting (interrupting→ordinary) bracket:
that would re-enable downstream `with _ ->` swallowing of `Sys.Break`/timeouts,
defeating the point of the new model. Use a bracket that holds off interruptions and
re-throws them still-interrupting, paired with the single explicit `with_async_exns`
from (2).

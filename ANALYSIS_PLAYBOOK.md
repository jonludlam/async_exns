# Async-exceptions impact analysis — generic playbook

How to analyse any large OCaml package for breakage under OxCaml's asynchronous
exception semantics. Written for an LLM to follow. Terse; assumes you've read
`OxCaml asynchronous exceptions.md`.

## The semantic change (what you're hunting)

Today an async exception (`Sys.Break`/Ctrl+C, `Stack_overflow`, and exns from C
finalisers / OCaml signal handlers / memprof callbacks) can surface at *almost any
safe point*, so ordinary `try … with` (especially wildcards) reliably catches it.
Under OxCaml it travels a **separate path**: a normal `try … with` no longer catches
it — it propagates to the nearest `Sys.with_async_exns` boundary or terminates the
program. Two consequences:

- **Raise side:** a *custom* exn raised from a signal handler must be raised via
  `Sys.raise_async`, else it terminates the program. **BUT `Sys.Break` is already
  handled:** OxCaml automatically makes the `Sys.Break` escaping a signal handler —
  including the handler installed by the standard `Sys.catch_break true` — an async
  exception. So a program that catches Ctrl+C via `Sys.catch_break` (or by raising
  `Sys.Break` from its own SIGINT handler) needs **no raise-side change at all**; the
  async `Sys.Break` comes for free. Raise-side work is only for *other* exceptions a
  program throws from a handler/finaliser (e.g. alt-ergo's custom `Util.Timeout` from a
  SIGVTALRM handler). A plain `raise Sys.Break` from ordinary (non-handler) code stays a
  normal synchronous exception.
- **Catch side:** existing handlers stop firing; you need a `Sys.with_async_exns`
  boundary to convert async→normal *at that point*.

Target class: code **sound today but silently broken** — cleanup / state-restoration
/ resource release living in a handler or `Fun.protect ~finally` that will no longer
run, leaving leaked resources or broken invariants.

## What the survey found (priors — anchor on these, but still verify per package)

Across 13 packages (build systems, package managers, provers, concurrency libraries,
editor daemons, a test framework, a file syncer, a store) a few regularities held hard
enough to use as priors:

- **Most packages reduce to ONE affected exception, and it's usually `Stack_overflow`.**
  Once signals are handled safely (see "Good patterns"), `Stack_overflow` is typically
  the only thing left on the new path — and it is almost never caught for recovery. So
  for a *one-shot* command there is frequently **no day-one bug**.
- **Genuine bugs cluster in exactly two places — look here first:**
  1. **Long-lived recovery loops** (daemon / server / `--watch` / REPL / IDE / per-test).
     This was the #1 source of real bugs (utop, merlin, dune `--watch`, irmin-server,
     Frama-C `-server`, alcotest). The per-iteration catch-all stops catching → the
     process dies instead of recovering, and skipped cleanup accumulates across its life.
  2. **Externally-visible / on-disk cleanup that gets skipped:** files left on disk, a
     switch left inconsistent, the terminal left in raw mode, a truncated output file
     (opam, utop, Rocq's `.vo`, alcotest's stdout). In-memory-only invariants on a dying
     one-shot process do **not** matter.
- **The decisive question for any signal-raised exception: asynchronous or cooperative?**
  A `raise` *inside the signal-handler body* is asynchronous (affected). A handler that
  only *sets a flag / writes a pipe*, with the exception raised later at a check-point in
  ordinary code, is **cooperative → an ordinary exception → unaffected**. This one
  distinction settled the verdict for Rocq, Unison, Frama-C and Eio. Read the handler body.
- **Throwing-side work is narrow.** `Sys.Break`/`Sys.catch_break` is free. The only
  recurring throwing-side change is a **timeout raised from an alarm handler** (alt-ergo,
  Rocq) or a **custom termination exception from a signal handler** (utop's `Term`).
  Everything else is catch-side.
- **The fix is small and stereotyped:** make the few cleanup primitives masking-`new_protect`;
  add `Sys.with_async_exns` **per unit of work** in each recovery loop (a single high
  wrapper does *not* preserve per-iteration recovery); and `Sys.raise_async` the
  alarm/timeout handler if there is one.

## Good patterns that make a program safe (recognizers — and what to recommend upstream)

Several disciplines side-step the change almost entirely. Spotting one quickly clears a
package; their *absence* is where the bugs are. They are also the fixes to recommend.

- **Signals as data, not exceptions.** Block signals and consume them on a dedicated
  thread via `sigwait`/`Thread.wait_signal` (dune), or a C-level handler that only writes
  a self-pipe/eventfd with the OCaml callback run later on the event loop (Lwt, Eio). No
  exception is ever raised from a handler.
- **Cooperative interruption.** Handler sets a flag; ordinary code raises at explicit
  check-points. The exception is ordinary and unaffected. (Rocq, Unison, Frama-C `Async`.)
- **Timeouts out-of-process.** A separate helper/child process applies the limit and
  reports a *value*; the OCaml side pattern-matches a result, never catches a signal-raised
  timeout. (Why3; Frama-C's prover calls.)
- **Roll back from an immutable snapshot, not in a handler.** The recovery loop catches the
  failure and restores saved state, rather than repairing state in a `finally` around the
  failing step — so skipped inner handlers can't corrupt anything. (Rocq STM.)
- **On-disk consistency via atomic rename / commit log.** Write-to-temp-then-rename, or an
  on-disk commit record, makes an interruption mid-write detected/ignored rather than
  corrupting — safe regardless of handlers. (Unison commit log, irmin control-file rename.)
  *Counter-example:* writing in place with a "delete the partial file" handler is fragile
  (Rocq `.vo`).
- **Decline to catch the runtime exceptions.** A catch-all that already excludes
  `Stack_overflow`/`Out_of_memory` (Lwt's default exception filter) means those already
  bypass the cleanup/recovery machinery, so the new model changes nothing for the default.

## Method (in order)

1. **Clone shallow; record the commit SHA for links.**
   `git clone --depth 1 <url> && (cd <repo> && git rev-parse HEAD)`.
   All source references in your output MUST be GitHub permalinks (see convention
   below) pinned to that SHA.

2. **Establish the async-exn *blast radius* first — this is the highest-value step.**
   Find every source of an async exception and, for each, decide whether it actually
   *raises through OCaml frames* (vs `exit`/print/return). Greps:
   - signals: `Sys.set_signal`, `Sys.signal`, `Sys.Signal_handle`, `Sys.Break`, `sigint|sigalrm|sigvtalrm|sigterm` (-i)
   - timers: `setitimer`, `Unix.alarm`
   - finalisers / GC / memprof: `Gc.finalise`, `Gc.create_alarm`, `Memprof`
   - the package's own "timeout"/"interrupt"/"limit" exceptions: `exception `, `timeout|interrupt|alarm` (-i)
   A handler that calls `exit`/just prints is **not** an async-raise concern — note
   and dismiss it. The output of this step is usually short: most packages reduce to
   **one or two** async exceptions (alt-ergo reduced to a single `Util.Timeout` from
   `SIGVTALRM`). State the blast radius explicitly; everything else keys off it.

   Two recognizers that *eliminate* surface (check for these — they can shrink the
   blast radius to almost nothing):
   - **Synchronous signal discipline (a non-target pattern).** If the package
     **blocks** signals (`Unix.sigprocmask SIG_BLOCK`) and consumes them on a
     dedicated thread via `Thread.wait_signal` / `sigwait`, turning them into queue
     events, then **no exception is raised from a signal handler** — there is no
     `Sys.Break` and no `Sys.raise_async` work to do. This is the "already does what
     the new model wants" pattern. *(dune: `sigprocmask` + `wait_signal` watcher
     thread → signals never raise → blast radius collapses to just `Stack_overflow`.)*
     Flag it as a **good pattern** worth recommending elsewhere.
   - **Cooperative cancellation ≠ async exn.** Concurrency libs raise their *own*
     cancellation exception (dune `Build_cancelled`, Lwt `Canceled`) along the
     **normal** path at cooperative points — these are unaffected by the change.
     Only genuinely-async sources (signal handler raise, runtime `Stack_overflow`)
     are in scope. Don't conflate the two. **The concrete tell:** look at the signal
     handler body — does it `raise` (asynchronous, affected), or only set a ref / write
     a pipe / send an event, with the exception raised later at a check-point in ordinary
     code (cooperative, unaffected)? This is the decisive question; answer it for every
     handler. (The same custom exception, e.g. `Sys.Break` or a `Timeout`, can be either,
     depending on *where* it's raised.)

   **`Stack_overflow` is always live.** Even after signals are tamed, the runtime can
   still raise `Stack_overflow` asynchronously. It is frequently the *residual* blast
   radius (see step 6) — and it matters for cleanup brackets (A-sites), not only for
   catch sites.

3. **Find the catch sites** for each blast-radius exception: `with <Exn>`, plus
   wildcard handlers `with _ ->`, `with e ->`, `with exn ->`, and re-raise patterns
   `raise e` / `Printexc.raise_with_backtrace`. Also `Fun.protect`, `~finally`,
   `Mutex.lock`/`unlock`, hand-rolled `match … with exception e -> cleanup; raise e`.

4. **Classify each catch/cleanup site** by *what it needs from the exception*:
   - **(A) cleanup / state-restoration / resource release** → wants "finaliser must
     run" → fix with the async-aware bracket (`new_protect`).
   - **(B) result-conversion / control-flow decision** (turns the exn into a
     result, exits, retries) → wants a real catch → fix with `Sys.with_async_exns`.
   - **(C) pure abort / impossible-case guard** (`with _ -> assert false`) → fine.

   **Fix at the primitives, not the call sites.** Most packages funnel cleanup
   through a handful of *own* bracket combinators — `protect`/`protectx`,
   `Mutex.protect`, `finalize`, a universal task/fiber wrapper — each with many
   callers. Identify these first: re-implementing the ~handful of primitives as
   masking `new_protect` brackets fixes all their callers at once, and is the
   highest-leverage change. *(dune: ~4 primitives cover ~40 call sites.)* Report the
   primitive, with its caller count, rather than enumerating every caller.

5. **For every (A) site, check "benign-by-accident" before calling it a bug.** This
   is the most important trick — naive grepping over-reports. A skipped cleanup is
   only an *active* bug if the broken state is actually observed. Ask:
   - Does the function **re-initialise the same state at its start** (next call)? If
     the state is local and reset-before-read, the trailing cleanup is redundant →
     benign (memory retention at most). *(alt-ergo `matching.ml` `reset_cache_refs`.)*
   - Is the resource a **one-shot** (e.g. `setitimer` with `it_interval = 0.`)? Then
     the only exn that skips the `finally` is the timer *firing* — which already
     spent it → no leak → benign. *(alt-ergo `Time.with_timeout`.)*
   - Is it **diagnostic only** (profiling/timing counters), gated behind a debug
     flag, with no soundness/resource/result impact? → low severity. *(alt-ergo
     `timers.ml` `with_timer`.)*
   - Is the restored state a **sticky global not reset elsewhere**, read by later
     work? → **genuine active bug.** *(alt-ergo `steps.ml` `apply_without_step_limit`
     leaves `--steps-bound` disabled.)*
   - **Is the process dying anyway?** If the only async exn is terminal (e.g.
     `Stack_overflow` with no recovery catch, per step 6), then in-memory invariants
     don't matter — only *externally-visible* cleanup does (fds, temp files, child
     processes, terminal/tty state). And that is often already backstopped by
     `at_exit` (verify — it runs on uncaught exns, incl. `Stack_overflow`) or an
     outer process-teardown. → benign-by-accident. *(dune one-shot `dune build`:
     cleanup brackets skipped by `Stack_overflow`, but the process exits and
     `at_exit`/scheduler-kill cover the visible parts.)*
     **⚠ This check is invalid for long-lived modes.** If the tool has a
     daemon / server / `--watch` / REPL / RPC mode, the process does **not** die —
     it recovers from a failed unit of work and serves the next one. Then: (i) any
     async exn the recovery loop *catches today* (often a top-level `with exn ->`
     wrapper) becomes a **(B) recovery site** that stops catching → the daemon dies
     instead of recovering = a real behaviour change; and (ii) the cleanup it skips
     now **accumulates across the process lifetime** (leaked fds/locks/children,
     corrupted caches) instead of being moot. *Always check for a long-lived mode and
     analyse it separately* — the same async exn can be benign one-shot and a genuine
     bug in daemon mode. *(dune `--watch`/RPC: a build `Stack_overflow` is caught by
     the fiber wrapper today and the daemon keeps serving; under the new model it
     bypasses the wrapper and kills the daemon — fix with a per-iteration
     `with_async_exns` boundary in the poll loop.)*
   Label survivors **benign-by-accident → fragile**: still convert to `new_protect`
   (hygiene + they break the moment a second async exn enters scope, e.g. an
   async-catchable `Sys.Break`), but they're not day-one bugs.

6. **Check the "usually clear" gaps** (record them as cleared, don't skip):
   - `Stack_overflow` catch sites — often **zero**, so no *recovery*-path regression.
     But note it's still a **live async exn** (step 2): with zero catches there's no
     day-one bug, yet it still **skips every (A) cleanup bracket** on its way out —
     that's the benign-by-accident surface to assess via step 5, not something to
     wave away. When signals are tamed, this is usually the entire blast radius.
   - `Out_of_memory` catch sites — often zero. The doc makes OOM-from-GC fatal, so
     any *recovery* on `Out_of_memory` is the thing to flag.
   - `Gc.finalise` raising; memprof callbacks raising.
   - Distinguish an exception **catch** (`with Timeout ->`) from a **pattern match**
     on a result/variant type (`| Timeout _ -> …`) — the latter is not affected.
     (JS/`js_of_ocaml` builds have no Unix signals — usually out of scope; confirm.)
   - **Library exposure:** if a suspect function is in an `.mli` and the `dune` has a
     `public_name`, external callers exist — note it, but in-tree callers usually
     settle severity.

7. **Determine boundary placement (the real design call).** `Sys.with_async_exns`
   converts async→normal *at the boundary*, bypassing every handler nested below it.
   So a **high** boundary turns a graceful per-item result (e.g. per-goal "timeout →
   unknown") into a whole-run abort; preserving current behaviour needs the boundary
   placed **below** the result-conversion (B) handler. Trace the chain to recommend.
   **For a long-lived recovery loop the answer is almost always "per unit of work"**
   (per command / request / test / build), placed *just inside* the loop and *below* the
   existing per-iteration catch-all — so the asynchronous exn becomes ordinary, the
   existing report-and-continue handler fires, and the loop keeps going. A **single high
   wrapper around the whole loop does NOT preserve per-iteration recovery** — it converts
   the exn only after the loop has already been torn down. (Recurred in utop, merlin,
   alcotest, irmin-server, Frama-C `-server`.)

8. **Verify runtime claims empirically when cheap.** E.g. "does `at_exit` run on an
   uncaught exception?" — write a 2-line `.ml` and run it (`at_exit` *does* run, incl.
   on `Stack_overflow`, on stock OCaml 5.x). Flag anything needing the *OxCaml*
   runtime specifically as a separate confirmation.

## Key conceptual framings to include

- **Current catching is nondeterministic:** the async exn fires at an arbitrary safe
  point, so *whichever matching handler is innermost at that instant* catches it.
  Sites are defensive precisely because of this.
- **Handlers form a cleanup-then-convert chain:** inner frames clean up and re-raise;
  an outer frame does the final result-conversion. Draw the chain.
- **Delta splits in two:** *boundary-independent* (intermediate cleanup always
  skipped — assert without any design decision) and *boundary-dependent* (which
  terminal handler wins — depends on where you put `with_async_exns`).
- **`new_protect` re-raise mode caveat:** a masking bracket re-raises *async* (resource
  freed, exn still async → still needs the explicit boundary); a converting bracket
  re-raises *normal* (silently re-enables downstream `with _` swallowing Ctrl+C —
  against the model). Recommend masking + one explicit `with_async_exns`. Don't
  reflexively wrap cleanup in a converting bracket.
- `new_protect`'s `init`/`finaliser` shape also closes acquire→enter races that
  `Fun.protect` leaves open.

## GitHub link convention (mandatory for all source refs)

Format: `https://github.com/<org>/<repo>/blob/<FULL_SHA>/<path>#L<start>-L<end>`
- Use the **full 40-char SHA** from step 1 (permalink stability), not a branch name.
- Single line: `#L420`. Range: `#L420-L437`.
- Use these inline everywhere you'd write `file.ml:line` (tables, prose, summaries).
- For non-GitHub forges use the equivalent permalink (GitLab `/-/blob/<sha>/…#L`,
  Bitbucket, etc.).

## Deliverables (produce all three)

**Write-up style rules (apply to both deliverables):**
- **Each per-package write-up must be self-contained — no comparisons to other
  packages.** Do not write "unlike dune…", "the opposite of opam", "same shape as
  alt-ergo's…", etc. A reader of one report must not need any other report. (Real
  references to a package's *own* code are fine even when the path happens to contain
  another name — e.g. alt-ergo's `src/lib/dune` build file, or dune's vendored
  `vendor/opam` copy. Those are not comparisons.) All cross-package observation and
  contrast belongs **only** in the shared summary doc.
- **Plain English; avoid jargon in the deliverables.** Do not use "blast radius",
  "benign-by-accident", "(A)/(B) site", "boundary", etc. *in the reports* (those terms
  are fine as working shorthand inside this playbook). Say plainly: "which exceptions
  are affected", "safe today only by luck (fragile)", "code that catches the exception
  to decide what to do next", "where to put the wrapper". Keep real API names
  (`Sys.with_async_exns`, `Sys.raise_async`, `new_protect`) but gloss each once.
- **Use the standard term "asynchronous exception" — do NOT invent a synonym** (no
  "interrupting exception" etc.). It is the term the design doc and reviewers use.
  Define it once in plain terms near the top of each report (e.g. *"an asynchronous
  exception is an event — pressing Ctrl+C, a timeout going off, or running out of stack —
  that can surface at almost any point in the program"*), then use it consistently.
  Ordinary words like "interrupt"/"interruption" are fine in their everyday sense.

### 1. One-paragraph summary (max one paragraph)

Pattern: *which exceptions are affected (the 1–2 + their source) → the two-sided fix
(raise via `Sys.raise_async`, except `Sys.Break`/`catch_break` which is already handled;
existing handlers stop firing) → the genuine bug(s) vs safe-today-only-by-luck sites and
that all use `new_protect` → the catch-to-decide site(s) needing a `Sys.with_async_exns`
wrapper and that its placement is the open design call → "everything else checked is
clear".* Embed GitHub links on the named sites.

### 2. Detailed per-package doc `findings_<pkg>.md`

Mirror `findings_alt-ergo.md`. Sections:
- **Header**: repo + pinned SHA + clone path.
- **Bottom line**: bulleted blast radius, raise-side need, genuine bug(s), fragile
  sites, result-conversion sites, "everything else clear".
- **The mechanism(s) that matter**: how each async exn is installed/fired/caught,
  with links.
- **Cleanup-site verdict table**: columns *Site | Cleanup | Verdict (GENUINE BUG /
  benign-by-accident / low severity) | Why*. Links in the Site column.
- **Current semantics (why subtle)**: nondeterminism + the chain diagram + the
  two-part delta.
- **Remediation shape**: the (A)→`new_protect` vs (B)→`with_async_exns` partition
  table; the re-raise-mode caveat.
- **Resolved during this pass** / **Still open (design calls)**.
- **Obviously fine (gaps checked and cleared)**: Stack_overflow, Out_of_memory,
  finalisers, other signals, wildcard handlers, plugins, JS build.
- **Recommendations**: prioritized & sequenced — (1) `Sys.raise_async` on the
  handler [required], (2) choose the `with_async_exns` boundary [the design call,
  coupled with 1], (3) fix the genuine bug(s) via `new_protect` [independent], (4)
  convert fragile sites for robustness, (5) OxCaml runtime confirmations. Note
  coupling (1+2 land together) and the "don't use a converting bracket" warning.

### 3. Append a plain-English entry to the shared `findings_summary.md` (REQUIRED — don't skip)

A package is **not done** until its entry is added to `findings_summary.md`. This is the
easy step to forget, so treat it as part of "finishing" every package.
- Add a new `## <Package> (<one-line what-it-is>) — <verdict>` section, in the same
  plain-English style as the existing entries (a few short paragraphs, not the terse
  one-paragraph summary, and not the detailed report).
- Link to the detailed `findings_<pkg>.md`.
- Insert it *before* the "Still to do" section, and remove the package from "Still to do"
  / the worklist line.
- This shared file is the **one place cross-package comparison is allowed** — but keep
  each entry readable on its own.

## One-line reusable grep

```
grep -rni -e "Sys.Break" -e "set_signal" -e "Sys.signal" -e "setitimer" -e "Unix.alarm" \
  -e "Gc.finalise" -e "create_alarm" -e "Memprof" -e "at_exit" -e "Fun.protect" -e "finally" \
  -e "Stack_overflow" -e "Out_of_memory" -e "with _ ->" -e "with e ->" -e "with exn ->" \
  --include=*.ml --include=*.mli <srcdir>
```
Then read each hit in context — the verdict is never in the grep line, it's in
whether the skipped cleanup is observed (step 5).

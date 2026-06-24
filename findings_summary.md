# Asynchronous exceptions — summary of findings

A plain-language summary of what we found when checking large OCaml programs for
problems caused by OxCaml's change to "asynchronous exceptions". Each program also has
a detailed, code-linked write-up in `findings_<name>.md`.

## The problem in plain terms

A few kinds of event can interrupt an OCaml program at almost any moment: pressing
Ctrl+C, a time-limit going off, or the program running out of stack space. Today these
surface as ordinary exceptions, so a normal `try … with` block catches them — and
programmers rely on that to do clean-up work: release a lock, delete a temporary file,
undo a half-finished change, put a setting back, and so on.

OxCaml changes this. These "interrupting" exceptions now travel a *separate* route. An
ordinary `try … with` no longer catches them. Unless the program wraps the right region
in a new special handler, the exception flies straight past all the normal handlers and
either reaches that wrapper or stops the program outright.

The risk we are hunting for: clean-up code that runs reliably today will be **silently
skipped** tomorrow, leaving leaked resources or broken state behind.

Fixing each program has two sides:
- **Throwing side:** code that deliberately raises one of these exceptions may need to be
  updated to throw it the new way, or it will simply stop the program. (Ctrl+C is a
  special case: if a program uses OCaml's standard way of catching it, OxCaml already
  delivers it the new way, so no change is needed. A program's *own* exception thrown from
  a handler — such as a timeout — does need updating.)
- **Catching side:** places that need to catch one of these exceptions must be wrapped
  in the new handler, or they stop catching it.

And a key judgement call throughout: a skipped clean-up is only an *actual* bug if
something later notices the mess. Many suspicious-looking spots turn out to be harmless
today — but often only by luck, so they are still worth tidying.

---

## Alt-Ergo (an automated theorem prover) — one real bug; tricky to place the wrapper

Detailed write-up: `findings_alt-ergo.md`.

Alt-Ergo enforces a time limit with a CPU timer. When time runs out, the timer's handler
throws a `Timeout` exception — exactly the kind of exception that changes behaviour. This
single `Timeout` is the only thing affected here: Ctrl+C just exits immediately, and
nothing else (out-of-stack, out-of-memory, finalisers) is relevant.

So two things are needed: the timer must throw `Timeout` the new way, and the many places
deep inside the solver that catch `Timeout` (to report "timed out / unknown" and carry
on) must be wrapped so they keep working.

**One genuine bug.** A helper temporarily switches off the "step limit", does some work,
then switches it back on. If a timeout strikes during that work, the "switch back on" is
skipped — and because the step limit is a long-lived global setting that nothing else
resets, it stays off for the rest of the session. (This affects a niche resource-limit
flag, not the correctness of the prover's answers.)

A few other spots look dangerous but are safe today by luck: one resets its working state
at the *start* of its next use anyway; another relies on the timer being one-shot (the
only way to skip its clean-up is the timer firing, which has already used it up). Not bugs
now, but fragile — they would break if a second kind of interrupting exception were ever
allowed here — so they should still be moved to the safe pattern.

**Why the wrapper placement is the hard part.** It is tempting to put one wrapper high up
and be done, but Alt-Ergo handles timeouts at several nested levels with *different*
outcomes (different "unknown" reasons, a two-phase "try to produce a model after timing
out" feature, and some catch-all spots), the tidy per-goal answer actually travels back
as an ordinary result *value* rather than as an exception, and Alt-Ergo already treats a
per-goal time limit differently from a whole-run one. A single wrapper can't reproduce all
that, so the placement has to be worked out for each existing timeout mode before any
change lands.

---

## Dune (the OCaml build system) — almost entirely safe, with one caveat

Detailed write-up: `findings_dune.md`.

Dune is already built the way the new model wants. It deliberately blocks Ctrl+C and the
other signals and deals with them on a dedicated thread, turning them into ordinary
internal messages. Because of that they are never thrown as exceptions at all, so the
change doesn't affect them — and there is nothing to fix on the throwing side.

The only interrupting exception left is "out of stack space". For a normal one-off
`dune build` this is not a problem: nothing tries to recover from it, so the program just
stops, exactly as today.

**The caveat is dune's long-running modes** (`dune build --watch` and the background
server). There, dune is meant to survive a failed build and carry on serving the next one:
today it catches the failure, reports it, and waits for the next change. Under the new
rules an "out of stack" during a build would slip past that safety net and kill the
long-running process instead of letting it recover — and because the process keeps
running, any clean-up skipped along the way would slowly pile up. The fix is to wrap each
build in the new handler so the recover-and-continue behaviour is preserved.

Dune is also a useful good example: handling signals on a dedicated thread side-steps
almost the whole problem, and is worth recommending to other programs.

---

## opam (the OCaml package manager) — real, visible bugs

Detailed write-up: `findings_opam.md`.

opam is the opposite of dune: on start-up it deliberately asks OCaml to turn Ctrl+C into
an exception. So Ctrl+C is the event that changes behaviour here. (Out-of-stack and
out-of-memory are possible too, but nothing tries to recover from them.)

The bugs here are visible on disk, which is what makes them serious — the mess outlives
the program and affects your next opam command:

- When opam installs a package and you press Ctrl+C, it normally **rolls back** the
  half-installed package. opam drives that roll-back by catching the interruption and
  turning it into a value that says "undo". Under the new rules the Ctrl+C skips all of
  those catch points, the roll-back never happens, and **stray files are left behind** in
  your switch — now out of step with opam's record of what is installed.
- Pressing Ctrl+C at a prompt can leave your **terminal stuck in "raw" input mode**.

Other clean-up (file locks, temporary directories) turns out to be safe by luck: the
operating system releases the locks when opam exits, temporary build directories are
recreated fresh next time, and log files are tidied by a separate exit handler.

The fix is on the catching side only: opam uses OCaml's standard way of catching Ctrl+C,
and OxCaml already delivers that interruption the new way, so nothing needs changing to
*produce* it. What's left is two parts: wrap each package action so its roll-back still
runs when interrupted; and route clean-up through the handful of shared helper functions
opam already has (fixing those few helpers covers everything at once).

---

## Lwt (a cooperative-concurrency library) — almost entirely safe

Detailed write-up: `findings_lwt.md`.

Lwt is the library a large part of the OCaml world builds its asynchronous code on. It is
already arranged the way the new rules want. Its signal handling happens at the C level: a
signal merely writes a byte into an internal pipe, and the OCaml code that responds to it
runs later as ordinary work — so signals never arrive as exceptions, and there is nothing
to fix for Ctrl+C.

The only interrupting exception that can reach Lwt is running out of stack space. And here
an existing design choice helps: the place where Lwt runs your callbacks already declines,
by default, to catch "out of stack" (and "out of memory"). So those events already skip all
of Lwt's catch-and-clean-up machinery today — which means the new rules change nothing for
normal use.

There is one case that does change. Lwt lets a program opt in to catching everything,
including "out of stack", so it can try to recover and carry on. For those programs the new
rules let the "out of stack" slip past that recovery and stop the program instead. Keeping
that behaviour needs the new wrapper around the step where Lwt runs callbacks. (We also
noticed a stale comment in Lwt's interface file that should be corrected.)

---

## utop (the OCaml REPL) — a real bug: Ctrl+C during evaluation kills the session

Detailed write-up: `findings_utop.md`.

utop deliberately turns Ctrl+C into an exception (it calls `Sys.catch_break true`) so that
an interruption while a user expression is running can be caught. Today the catch-all
wrapped around evaluation catches it, reports it, and returns you to the prompt — which is
the whole point of an interactive loop. Under the new rules that interruption is no longer
caught by an ordinary handler: it escapes the loop, reaches the top level, and utop prints
"Fatal error: Sys.Break" and exits. So a Ctrl+C that should abort a runaway computation
instead terminates the session. Because OxCaml already delivers the interruption from
`Sys.catch_break` the new way, this is a present-day bug, and running out of stack regresses
the same way.

The fix is small. Wrap the evaluation step in the new wrapper (which turns the interruption
back into an ordinary exception), placed inside the loop below the existing catch-all, so
the interruption is reported and the loop continues exactly as today; where to put it is the
only real design call. Separately, utop's own SIGHUP/SIGTERM handler raises a custom `Term`
exception and must throw it the new way (or simply be dropped). A few save/restore sites
(formatter state, a temp file) are safe today only by luck and should move to the
interruption-safe helper once recovery is restored. Everything else checked is clear.

---

## Why3 (the verification platform) — barely affected; no fixes required

Detailed write-up: `findings_why3.md`.

Why3 is the least affected of the programs looked at, for an architectural reason: it does
not enforce prover time limits with an OCaml signal or timer. It runs provers through a
separate C helper process that applies the limits itself and reports back over a socket, so
a timeout arrives as an ordinary return value, not an exception — every "timeout" or
"interrupted" case is a pattern match on a value, not an exception handler. A
whole-repository search finds no Ctrl+C handling, no OCaml timers, and no garbage-collector
finalisers, so there is nothing to change on the throwing side.

The only interrupting exception Why3 can produce is running out of stack, and it is caught
nowhere — so it is fatal today and stays fatal, and no recovery path changes. A few clean-up
spots (flushing and closing output files, removing a temp file) would be skipped, but the
process is terminating and the external prover process is reaped by an exit handler, so none
is a real bug. The only recommendations are optional tidying and two cheap runtime
confirmations.

---

## Eio (the effects-based concurrency library) — clear; fragile clean-up and one design call

Detailed write-up: `findings_eio.md`.

Eio already does what the new model wants for signals: the only signal handler it installs
(for child-process exit) does nothing but a signal-safe wake-up, and there is no Ctrl+C
handling, no timer, and no finaliser — so nothing needs changing on the throwing side. Its
own cancellation is an ordinary cooperative exception and is unaffected. The only
interrupting exception that can reach it is running out of stack, which it catches nowhere,
so it stays fatal as today with no lost recovery.

What does change: an interruption skips Eio's resource-teardown brackets, which nearly all
funnel through a few core primitives (`Switch`, the fiber-fork wrappers, and the cancellation
context). The release actions they run — closing file descriptors, killing child processes —
would be skipped, which is harmless for a one-shot program (the process is exiting and the
operating system reclaims everything), so they are safe today only by luck. Making those few
primitives interruption-safe fixes every resource bracket downstream at once. The one genuine
design question is whether to add the new wrapper in the scheduler's per-fiber step so an
interruption is routed through Eio's normal fiber-failure path rather than stopping the
program directly; a few effects-interaction details need confirming on the OxCaml runtime.

---

## Merlin and ocaml-lsp-server (editor tooling) — one real long-lived-mode bug, plus lost crash-recovery

Detailed write-up: `findings_merlin-ocaml-lsp.md`.

These are the OCaml editor back-ends (ocaml-lsp wraps merlin), both long-lived processes that
answer many requests and recover from a failed one. Neither turns Ctrl+C into an exception
(merlin ignores signals; ocaml-lsp handles them on a dedicated thread as events), so there is
nothing to change on the throwing side. The only interrupting exception that can reach them is
running out of stack.

There is one genuine bug, in merlin's normal `server` mode: each query swaps the compiler's
global state and restores it on the way out — clearing a process-wide "in use" flag — through
a clean-up handler. If running out of stack skips that restore, the flag stays set and every
later query in that process fails an "already in use" guard; the server keeps running but
returns errors for everything until it is replaced. Separately, both tools lose
crash-recovery: their per-request catch-alls stop catching a stack overflow, so the back-end
dies instead of returning an error and serving the next request. The fix is per-request: wrap
each request body in the new wrapper so the existing report-and-continue machinery still
fires, and make the per-query state restore (and a few smaller clean-up helpers)
interruption-safe. Cooperative request cancellation is unaffected; everything else is clear.

---

## Alcotest (the test framework) — minor: one visible clean-up gap and a per-test recovery choice

Detailed write-up: `findings_alcotest.md`.

Alcotest installs no signal handlers, no timers, and no finalisers, and has no Ctrl+C
handling — so there is nothing to change on the throwing side. The only interrupting exception
it can meet is running out of stack. Alcotest is a per-test recovery loop: each test runs
inside a catch-all that today turns any failure, including a stack overflow in deeply
recursive test code, into one recorded "failed test" and continues. Under the new rules that
catch-all no longer catches a stack overflow, so a single such test aborts the whole run and
the remaining tests silently never run.

There is no data corruption (the process is terminating), but one visible clean-up is worth
fixing: for each test Alcotest redirects the operating-system stdout/stderr to a per-test log
file and restores them afterwards; if the restore is skipped, the runtime's crash message and
any final flush land in the log file instead of on the terminal, so the failure looks like a
silent disappearance. Fix the stream restore with the interruption-safe helper (pure
robustness), and, if a stack-overflowing test should remain just a failed test, add a per-test
wrapper placed below the existing catch-all (a single wrapper around the whole run would not
preserve this). Everything else is clear.

---

## Unison (the file synchronizer) — largely safe; file consistency does not rely on catching the interruption

Detailed write-up: `findings_unison.md`.

Unison handles Ctrl+C in two stages and only the second is an interrupting exception: the
first one or two presses set a flag that the engine notices cooperatively (an ordinary
exception on the normal path), and only the third press raises the interruption to force
termination. The throwing side needs no change. Its main clean-up helpers only ever catch
Unison's own recoverable exceptions, so they already ignore the interruption today — the new
model changes nothing there. Crucially, the things that protect your files do not depend on
catching the interruption at all: file replacement is guarded by an on-disk commit log, the
archive is committed by a careful write-then-flip, and the lock is released by an exit
handler. So there is no on-disk-corruption bug.

What remains is small: one clean-up helper that closes transfer descriptors and drops a
partial temp file is safe today only by luck (the descriptors are closed independently and
the temp file is harmlessly ignored), and is worth moving to the interruption-safe helper. And
the two places that catch the interruption to decide what to do next — the text UI's clean
exit and the persistent socket server's removal of its socket file — each need the new
wrapper. Repeat/watch mode is meant to stop on Ctrl+C, and the graphical UI uses cooperative
cancellation; both are fine.

---

## Irmin (the Git-like store) — safe today; one real change in the network server

Detailed write-up: `findings_irmin.md`.

Irmin produces no interrupting exception of its own: no Ctrl+C handler, no raising signal
handler, no finaliser, no timer. The only interrupting exception that can reach its code is
running out of stack — realistic, because it walks deep object graphs — and its core library
and on-disk backend catch it nowhere, so it stays fatal with no recovery lost. Any Ctrl+C
handling is the embedding application's responsibility.

The clean-up an interruption would skip runs through a couple of helpers and the write path,
but none is a live bug: the skipped work is in-memory flags and file descriptors the operating
system reclaims, and the on-disk store cannot be corrupted because it commits by writing data
and then atomically renaming a small control file — a skipped flush mid-write is no worse than
a power loss, which the format already tolerates. These are worth moving to the
interruption-safe helper for hygiene. The one place behaviour actually changes is the
long-lived network server (`irmin-server`): its per-request loop catches any exception and
keeps serving, but under the new rules a stack overflow flies past that catch and stops the
whole server. Keeping recover-and-continue needs the new wrapper around each request (placement
is the open design call). Everything else is clear or low-severity.

---

## Rocq (the theorem prover, formerly Coq) — no genuine bug; the famous example is already guarded

Detailed write-up: `findings_rocq.md`.

Rocq is the original motivating example: the design note recalls that pressing Ctrl+C at the
wrong moment in a Coq proof "would cause it to succeed, even if false." That specific bug is
already guarded in the current code — the `Fail` command, which expects its body to fail,
explicitly refuses to treat an interruption as an expected failure and re-raises Ctrl+C,
timeouts, and stack overflow instead of reporting success. Under the new rules the interruption
would bypass that handler entirely and propagate anyway, so the soundness guarantee is preserved
either way; the change actually removes the reliance on getting that test exactly right.

Only two interrupting exceptions are in play, and which one you get depends on the mode. The
time-limit machinery (the `Timeout` tactic, `-time`) raises a timeout from inside a SIGALRM
handler and catches it immediately in the same small function. Ctrl+C arrives as `Sys.Break`:
interrupting when it comes from OCaml's standard Ctrl+C catching (used by `rocq compile`/`coqc`
and the prompt) or from the editor back-end's own handler while evaluating a request, but
cooperative — and so unaffected — when it comes from the pervasive interrupt check-point the
kernel and tactics use. Running out of stack can happen in the deeply recursive kernel but is
never caught for recovery.

The work is small. The one throwing-side change is making the timeout handler throw the new way;
Ctrl+C needs nothing. On the catching side, three places catch an interruption to decide what to
do next and need the new wrapper to keep working: the timeout's own local catch, the interactive
prompt's recovery loop, and the editor back-end's per-request recovery (which turns a Ctrl+C into
a "command interrupted" answer rather than killing the process); placement is the design call.
Rocq is unusually well-prepared: proof and environment state is rolled back from immutable
snapshots by the recovery loop rather than repaired in handlers around the failing command, and
its general clean-up helper is already built on an interruption-safe bracket. No genuine
state-corruption bug was found; a few clean-up spots are safe today only by luck — most visibly
the compiled-file (`.vo`) writer, which currently leaves a truncated file at the final path if
interrupted (no write-to-temp-then-rename) — and should move to the existing safe bracket.

---

## Frama-C (the C analysis platform) — minor and mode-dependent: no batch bug, two real server-mode bugs

Detailed write-up: `findings_frama-c.md`.

Frama-C analyses C code either as a one-shot batch command or as a long-lived `-server` process.
Only one interrupting exception matters: Ctrl+C (`Sys.Break`). Frama-C asks the standard library
to turn Ctrl+C into `Sys.Break` at start-up and again in the server, and the Eva plug-in installs
its own handler that also raises `Sys.Break` — all of which OxCaml makes interrupting for free, so
the throwing side needs essentially no change (one cheap runtime confirmation for the hand-written
handler). Its general cancellation and the prover timeout are not signal-driven at all — they set
a flag and raise at explicit check-points — so they keep working unchanged. There are no
finalisers, and nothing catches running out of stack.

In one-shot batch mode there is no real bug: the process is exiting, and the exit handler still
runs (verified), so prover temp files are deleted, child provers killed, and output flushed. The
only losses are cosmetic (Eva's "save partial results on Ctrl+C" feature, the clean exit code) and
are restored by placing the wrapper correctly. The genuine bugs appear in the long-lived server
mode, which recovers from an interrupt and keeps serving: two kernel-wide restore helpers — one
that restores which "project" is current, one that resets a "has this run?" flag — restore
long-lived global state through a clean-up handler that an interrupting Ctrl+C skips. Skipping them
leaves the wrong project current for every later request, and makes the Eva analysis believe it has
already run, so it silently returns nothing.

The fix is to rebuild those two restore helpers on the interruption-safe bracket (which fixes all
their callers at once) and to add the new wrapper where interruptions are caught to decide what to
do next — one high wrapper in batch mode, and one per request in the server so a Ctrl+C becomes an
ordinary exception the existing handlers catch and the restores below it still run. (The
interactive GUI ships as a separate package and is worth its own pass.)

---

## Status

All targets on the worklist (`TARGETS.md`) plus the two large named programs (Rocq, Frama-C) have
now been reviewed. The only deferred item is Frama-C's interactive GUI, which ships as a separate
package (`frama-c-gui`) not in the analysed repository.

---

## Patterns we keep seeing

- **Start by asking which interrupting exceptions a program can actually produce.** It is
  usually just one or two, and everything else follows. (Alt-Ergo: a timeout. Dune: only
  "out of stack". opam: Ctrl+C.)
- **A skipped clean-up is only a real bug if something later notices.** State that is
  reset before its next use, locks the OS releases for you, or anything that only matters
  while the program is already shutting down — these look risky but are fine. The genuine
  bugs leave a lasting mess: a setting stuck off, files left on disk.
- **Long-running programs are riskier than one-shot commands.** A server, watcher, or
  interactive session normally recovers from failures and keeps going; the new rules can
  break that recovery and let problems accumulate. Always check for such a mode and look
  at it separately.
- **Most clean-up flows through a few shared helper functions.** Fixing those handful of
  helpers fixes all their users at once, so that is where the effort should go.
- **Handling signals on a dedicated thread (dune's approach) avoids the problem almost
  entirely.**

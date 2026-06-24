# Why3 — first look at the asynchronous-exception change

Repo: `why3/why3` @ `2d6fbb02b4101d0fb9062d2b2a2a62f0838c0734` (shallow clone in
`./why3`). Why3 lives on GitLab; every source link below is a GitLab permalink pinned
to that commit.

What this pass covers: a search for code that works correctly today but would break
under OxCaml's new rules for "asynchronous exceptions".

Background in one sentence: an *asynchronous exception* is an interrupting event —
pressing Ctrl+C, running out of stack, or an exception thrown from a signal handler or
a garbage-collector callback — that can surface at almost any point in the program.
Today an ordinary `try … with` catches these; under the new rules it does not, unless
you wrap the region in a new helper (`Sys.with_async_exns`) that turns the interrupting
exception back into an ordinary one. We call these "interrupting exceptions" below.

## Bottom line

- **Why3 does almost nothing that the change touches.** It never turns Ctrl+C into an
  OCaml exception (no `Sys.Break`, no `Sys.catch_break` anywhere), it arms no OCaml
  timer (`Unix.setitimer`/`Unix.alarm` appear nowhere in the OCaml code), it registers
  no garbage-collector finalisers (`Gc.finalise`), and it catches `Stack_overflow`
  nowhere. The one residual interrupting exception is **`Stack_overflow`**, which is
  never caught and is therefore fatal today and stays fatal.
- **Time limits are enforced outside OCaml.** Prover time/memory limits are imposed by
  a separate helper process, `why3server`, written in C
  ([`src/server/cpulimit-unix.c`](https://gitlab.inria.fr/why3/why3/-/blob/2d6fbb02b4101d0fb9062d2b2a2a62f0838c0734/src/server/cpulimit-unix.c#L97-L120)
  uses `alarm`/`SIGALRM`, `setrlimit` and `kill`). The OCaml side only *reads a result
  back over a socket*. A timeout therefore arrives as the **value** `Call_provers.Timeout`
  ([`call_provers.ml:28`](https://gitlab.inria.fr/why3/why3/-/blob/2d6fbb02b4101d0fb9062d2b2a2a62f0838c0734/src/driver/call_provers.ml#L25-L34)),
  not as an exception. No OCaml signal handler ever raises a timeout.
- **Interruption ("stop this proof") is cooperative, not signal-driven.** The IDE's
  Interrupt button and the `Interrupt_req` request call
  [`Controller_itp.interrupt`](https://gitlab.inria.fr/why3/why3/-/blob/2d6fbb02b4101d0fb9062d2b2a2a62f0838c0734/src/session/controller_itp.ml#L540-L556),
  which sends an `interrupt` message to `why3server` and dispatches the result as the
  value `Interrupted`. Again, no exception travels through OCaml frames.
- **Throwing side: nothing to do.** There is no OCaml signal handler that raises an
  application exception (see "Which exceptions are affected"), so there is nothing to
  re-route through `Sys.raise_async`.
- **No genuine bugs found, and nothing that is "safe today only by luck" in a worrying
  way.** Because the only interrupting exception is an uncaught `Stack_overflow`, the
  clean-up that it skips on its way out is limited (three `Fun.protect` sites and a
  couple of hand-rolled temp-file removals), and the externally-visible parts are
  backstopped by an `at_exit` handler that disconnects from and reaps `why3server`.
- **No code needs a `Sys.with_async_exns` wrapper for correctness**, because no
  interrupting exception is caught anywhere today. The top-level catch-all handlers in
  the command-line tools and the per-request/per-connection handlers in the web server
  catch *ordinary* exceptions; an uncaught `Stack_overflow` slipping past them changes
  nothing meaningful (it was going to terminate that unit of work either way).
- Everything else checked (`Stack_overflow`, `Out_of_memory`, finalisers, memprof, the
  Isabelle and SIGPIPE handlers, the disabled SIGINT handler, the GTK IDE, the web
  server's long-lived loop, the JavaScript build) is clear or already harmless.

## Which exceptions are affected

The decisive question — *which interrupting exceptions can this program actually
produce?* — has an unusually short answer for Why3: only **`Stack_overflow`**.

### No Ctrl+C exception, no OCaml timer

A whole-repository search finds **no** `Sys.Break`, **no** `Sys.catch_break`, and **no**
`Unix.setitimer`/`Unix.alarm`. Why3 does not install a SIGINT handler that raises, and
does not arm an OCaml-level timer that raises. Pressing Ctrl+C at a Why3 command-line
tool uses the operating system's default behaviour (the process is terminated); it does
not become an OCaml exception that any `try … with` would catch.

The only OCaml signal handlers in the whole tree are:

- [`isabelle_client_main.ml:263`](https://gitlab.inria.fr/why3/why3/-/blob/2d6fbb02b4101d0fb9062d2b2a2a62f0838c0734/src/isabelle-client/isabelle_client_main.ml#L263)
  installs a `SIGINT` handler, but
  [`handle_interrupt`](https://gitlab.inria.fr/why3/why3/-/blob/2d6fbb02b4101d0fb9062d2b2a2a62f0838c0734/src/isabelle-client/isabelle_client_main.ml#L118-L124)
  only *sends a "cancel" message down a socket* — it raises nothing. Not affected.
- [`wserver.ml:383`](https://gitlab.inria.fr/why3/why3/-/blob/2d6fbb02b4101d0fb9062d2b2a2a62f0838c0734/src/ide/wserver.ml#L383)
  sets `SIGPIPE` to `Signal_ignore` — it does nothing, and certainly raises nothing.
- [`debug.ml:184-190`](https://gitlab.inria.fr/why3/why3/-/blob/2d6fbb02b4101d0fb9062d2b2a2a62f0838c0734/src/util/debug.ml#L184-L190)
  contains a `SIGINT` handler that calls `exit 2`, but it is **commented out** (disabled
  for issue #383). Even if it were live, `exit` does not raise an exception through
  OCaml frames, so it would not be affected either.

So there is no place where Why3's own code raises an application exception from a signal
handler, and therefore no `Sys.raise_async` work and no throwing-side change of any kind.

### Time limits and interruption are not exceptions

This is the heart of why Why3 is barely touched. Why3 calls provers through a long-lived
helper process, `why3server`, and communicates with it over a Unix-domain socket
([`prove_client.ml`](https://gitlab.inria.fr/why3/why3/-/blob/2d6fbb02b4101d0fb9062d2b2a2a62f0838c0734/src/driver/prove_client.ml#L156-L188)).

- The **time and memory limits are imposed by `why3server` in C**, using `alarm` /
  `SIGALRM` and `setrlimit`, and the over-limit prover child is killed with `kill`
  ([`cpulimit-unix.c:97-120`](https://gitlab.inria.fr/why3/why3/-/blob/2d6fbb02b4101d0fb9062d2b2a2a62f0838c0734/src/server/cpulimit-unix.c#L97-L120),
  [`:128-142`](https://gitlab.inria.fr/why3/why3/-/blob/2d6fbb02b4101d0fb9062d2b2a2a62f0838c0734/src/server/cpulimit-unix.c#L128-L142)).
  This is all outside the OCaml runtime, so it is entirely outside the scope of the
  asynchronous-exception change.
- The OCaml client **reads a result back as data**. `why3server` reports a finished run
  with a `timeout` flag and an exit code
  ([`prove_client.ml:225-242`](https://gitlab.inria.fr/why3/why3/-/blob/2d6fbb02b4101d0fb9062d2b2a2a62f0838c0734/src/driver/prove_client.ml#L213-L242));
  the client turns that into the variant `Call_provers.Timeout`
  ([`call_provers.ml:342-390`](https://gitlab.inria.fr/why3/why3/-/blob/2d6fbb02b4101d0fb9062d2b2a2a62f0838c0734/src/driver/call_provers.ml#L342-L390)),
  delivered to a callback by the scheduler
  ([`controller_itp.ml:452-516`](https://gitlab.inria.fr/why3/why3/-/blob/2d6fbb02b4101d0fb9062d2b2a2a62f0838c0734/src/session/controller_itp.ml#L452-L516)).
- "Interrupt" is the same shape: `Controller_itp.interrupt` sends an `interrupt` message
  to the server ([`controller_itp.ml:540-556`](https://gitlab.inria.fr/why3/why3/-/blob/2d6fbb02b4101d0fb9062d2b2a2a62f0838c0734/src/session/controller_itp.ml#L540-L556),
  [`call_provers.ml:583-588`](https://gitlab.inria.fr/why3/why3/-/blob/2d6fbb02b4101d0fb9062d2b2a2a62f0838c0734/src/driver/call_provers.ml#L583-L588))
  and the result comes back as the variant `Interrupted`.

Every `| Timeout ->` / `| Interrupted ->` you will find in Why3 is therefore a **pattern
match on a result value**, not an exception catch, and the change does not touch it.

The scheduler that drives all of this is a **cooperative polling loop**, not a
signal-driven one: it computes a delay and waits in `Unix.select`, running registered
"timeout" callbacks when their time arrives
([`unix_scheduler.ml:52-118`](https://gitlab.inria.fr/why3/why3/-/blob/2d6fbb02b4101d0fb9062d2b2a2a62f0838c0734/src/session/unix_scheduler.ml#L52-L119)).
The word "timeout" here means "a function to run after a delay", not an interrupting
exception.

### The residual interrupting exception: `Stack_overflow`

Why3 manipulates large terms and is deeply recursive, so the runtime can still raise
`Stack_overflow` asynchronously. But Why3 **catches it nowhere** (a whole-repository
search for `Stack_overflow` finds a single hit, and that is a *printer* — see "Checked
and fine"). It is uncaught and fatal today, and it stays uncaught and fatal under the
new rules. There is therefore **no recovery path that the change can break**, and the
only thing to assess is the clean-up that a `Stack_overflow` skips on its way out
(next section).

## Clean-up that could be skipped

The only interrupting exception in play is an uncaught, fatal `Stack_overflow`. So the
question for each clean-up site is narrower than usual: not "does a recovery handler
stop firing" (none exists), but "does a `Stack_overflow` flying out of the process skip
some clean-up that is *externally* visible (a file, a child process) and not otherwise
backstopped?" In every case the answer is no.

| Site | Clean-up | Verdict | Why |
|------|---------|---------|-----|
| [`sysutil.ml:69-76`](https://gitlab.inria.fr/why3/why3/-/blob/2d6fbb02b4101d0fb9062d2b2a2a62f0838c0734/src/util/sysutil.ml#L69-L76) `write_file_fmt` and [`sysutil.ml:79-95`](https://gitlab.inria.fr/why3/why3/-/blob/2d6fbb02b4101d0fb9062d2b2a2a62f0838c0734/src/util/sysutil.ml#L79-L95) `write_unique_file_fmt` | `Fun.protect ~finally:(flush + close_out)` | safe today only by luck → mildly fragile | Today the `finally` flushes and closes the output channel on any exception. A `Stack_overflow` skipping it leaves a channel unflushed, but the process is terminating, so the only loss is the buffered tail of one output file — and the process is dying anyway. No leaked resource the next run sees. Becomes a real concern only if a *second*, recoverable interrupting exception is ever introduced through this code. |
| [`debug.ml:162-167`](https://gitlab.inria.fr/why3/why3/-/blob/2d6fbb02b4101d0fb9062d2b2a2a62f0838c0734/src/util/debug.ml#L162-L167) `record_timing` | `Fun.protect ~finally:(add_timing …)` | low severity (diagnostic) | Pure profiling: records elapsed time into a table, and only when the `stats` debug flag is on (off by default). No resource, no result, no soundness impact. |
| [`sysutil.ml:98-109`](https://gitlab.inria.fr/why3/why3/-/blob/2d6fbb02b4101d0fb9062d2b2a2a62f0838c0734/src/util/sysutil.ml#L98-L109) `open_temp_file` | hand-rolled `try … with e -> Sys.remove file; close_out cout; raise e` | safe today only by luck → mildly fragile | Removes a temp file and closes its channel on any exception. A `Stack_overflow` skipping it leaves one temp file on disk as the process dies. Wasted disk space at worst; no broken invariant, and not observed by a later run. |
| [`call_provers.ml:602-616`](https://gitlab.inria.fr/why3/why3/-/blob/2d6fbb02b4101d0fb9062d2b2a2a62f0838c0734/src/driver/call_provers.ml#L602-L616) VC temp file → [`call_provers.ml:479-494`](https://gitlab.inria.fr/why3/why3/-/blob/2d6fbb02b4101d0fb9062d2b2a2a62f0838c0734/src/driver/call_provers.ml#L479-L494) deletion in `handle_answer` | temp VC file deleted when the `Finished` value arrives | safe today only by luck | The VC file is *not* deleted in a `finally`; it is deleted when the server's `Finished` result is received and dispatched. A `Stack_overflow` between sending the file and receiving the result leaves that temp file behind as the process dies — same "wasted disk space, process exiting anyway" reasoning. The running prover child is owned and time-limited by `why3server`, which the `at_exit` below reaps. |

There is no place where skipped clean-up corrupts a persistent invariant: the only
interrupting exception is terminal, so in-memory state does not matter, and the
externally-visible pieces (the `why3server` connection and its child processes) are
covered by an `at_exit` handler:

- [`prove_client.ml:147-149`](https://gitlab.inria.fr/why3/why3/-/blob/2d6fbb02b4101d0fb9062d2b2a2a62f0838c0734/src/driver/prove_client.ml#L147-L150)
  registers `at_exit` to disconnect the socket and `Unix.waitpid` the `why3server`
  child. I verified empirically (stock OCaml, see "Resolved") that `at_exit` runs on an
  uncaught `Stack_overflow`, so this teardown happens even on that fatal path — *provided
  OxCaml's uncaught-termination path also runs `at_exit`* (one runtime confirmation).

## Why this is subtle

For most programs the hard part is reasoning about *which* of many handlers catches an
interrupting exception, and *where* to put the new wrapper. For Why3 the subtlety is the
opposite: it is easy to *assume* there is interrupting-exception machinery to fix —
there is a prover scheduler, "timeout" everywhere, an interrupt feature, an IDE — and
then to discover that **none of it raises an exception through OCaml frames**:

- "Timeout" is a result value parsed from a C helper process, never an OCaml signal
  raising an exception.
- "Interrupt" is a cooperative socket message, never a `Sys.Break`.
- The scheduler's "timeout" is a poll-loop callback, not a timer signal.

So the analysis collapses early. The only genuinely interrupting exception the program
can produce is `Stack_overflow`, and because nothing catches it, the new rules cannot
change any control flow: a `Stack_overflow` was fatal before and is fatal after. The
only delta is which clean-up runs on the way out — and that delta is the same whether or
not a wrapper is added, because there is no handler to convert anything *to*.

## How to fix it

There is no correctness fix required, because there is no interrupting exception that is
caught today and would stop being caught. The (optional) hygiene work is small:

| Site kind | Sites | Fix | Difficulty |
|-----------|-------|-----|------------|
| Resource / file clean-up that only needs "the finaliser must run" | `sysutil.ml` `write_file_fmt`, `write_unique_file_fmt`, `open_temp_file`; `debug.ml` `record_timing` | the interruption-safe bracket `new_protect` (an interruption-aware version of the clean-up helper that runs the clean-up and then re-throws the interruption unchanged) | mechanical, local, optional |
| Code that catches the exception to decide what to do next | none for interrupting exceptions | — | — |

`Sys.with_async_exns` (the helper that turns an interrupting exception back into an
ordinary one at the point it is placed) is **not needed anywhere** for correctness,
because no interrupting exception is caught today. If Why3 ever decides it wants
`Stack_overflow` (or a future Ctrl+C handler) to be *recoverable* rather than fatal —
for example, to keep the IDE or the web server alive across a stack overflow in one
proof task — *then* it would add a `Sys.with_async_exns` boundary around each unit of
work; the natural places would be the per-request handler in the web server
([`wserver.ml:283-286`](https://gitlab.inria.fr/why3/why3/-/blob/2d6fbb02b4101d0fb9062d2b2a2a62f0838c0734/src/ide/wserver.ml#L283-L286))
and the scheduler's per-task callback dispatch
([`controller_itp.ml:519-538`](https://gitlab.inria.fr/why3/why3/-/blob/2d6fbb02b4101d0fb9062d2b2a2a62f0838c0734/src/session/controller_itp.ml#L519-L539)).
That is a *new feature*, not a fix for a regression, and is out of scope for this change.

**Caveat on how a clean-up bracket re-throws.** If the four clean-up sites are ever
converted, use a bracket that holds off the interruption, runs the clean-up, and
re-throws the exception *still interrupting* (`new_protect`). Do not use a converting
bracket that re-throws an *ordinary* exception: Why3 has many downstream `with _ ->` /
`with e ->` catch-alls (the top-level tool handlers, the web-server handlers) that would
then silently swallow a future interrupting exception — against the point of the new
model. For the four sites here, which only release a file/channel and re-raise, plain
resource safety is the same either way.

## Resolved during this pass

- **No `Sys.Break` / `Sys.catch_break` anywhere** — confirmed by whole-repository
  search. Why3 does not turn Ctrl+C into an OCaml exception. This is the single most
  important fact for this analysis: there is no Ctrl+C catching side to fix and no
  throwing side to re-route.
- **Time limits are enforced in C, not by an OCaml signal** — confirmed in
  [`cpulimit-unix.c`](https://gitlab.inria.fr/why3/why3/-/blob/2d6fbb02b4101d0fb9062d2b2a2a62f0838c0734/src/server/cpulimit-unix.c#L97-L142);
  the OCaml side receives a `Timeout`/`Interrupted` *value*
  ([`call_provers.ml`](https://gitlab.inria.fr/why3/why3/-/blob/2d6fbb02b4101d0fb9062d2b2a2a62f0838c0734/src/driver/call_provers.ml#L342-L390)).
  Every `| Timeout ->` / `| Interrupted ->` in the OCaml code is a result-value match,
  not an exception catch.
- **The scheduler is cooperative, not signal-driven** — confirmed in
  [`unix_scheduler.ml`](https://gitlab.inria.fr/why3/why3/-/blob/2d6fbb02b4101d0fb9062d2b2a2a62f0838c0734/src/session/unix_scheduler.ml#L52-L119):
  it waits in `Unix.select` and runs callbacks when their delay elapses.
- **`at_exit` runs on an uncaught `Stack_overflow`** — verified empirically on stock
  OCaml: a program dying from an uncaught `Stack_overflow` still ran its `at_exit`
  callback (exit code 2). So the `why3server` disconnect-and-reap
  ([`prove_client.ml:147-149`](https://gitlab.inria.fr/why3/why3/-/blob/2d6fbb02b4101d0fb9062d2b2a2a62f0838c0734/src/driver/prove_client.ml#L147-L150))
  survives the only fatal interrupting path here — provided OxCaml's uncaught path also
  runs `at_exit` (one runtime confirmation).
- **The disabled SIGINT handler is genuinely disabled** —
  [`debug.ml:184-190`](https://gitlab.inria.fr/why3/why3/-/blob/2d6fbb02b4101d0fb9062d2b2a2a62f0838c0734/src/util/debug.ml#L184-L190)
  is commented out; and even live it would only `exit`, which is not an exception.

## Still open (runtime confirmations, not investigations)

- **OxCaml `at_exit` behaviour.** Confirm on the OxCaml runtime that `at_exit` runs on
  interrupting-uncaught termination (specifically `Stack_overflow`), so the `why3server`
  disconnect/reap still happens. Verified on stock OCaml; needs one check on OxCaml.
- **OxCaml `Stack_overflow` stays fatal-and-uncaught.** The whole analysis rests on
  `Stack_overflow` being uncaught in Why3 (it is) and remaining a terminal interrupting
  exception under OxCaml (the design doc says it is raised via the asynchronous path).
  No source change follows from this; it is just the assumption that makes the "no
  recovery path to break" conclusion hold.

## Checked and fine

- **`Stack_overflow`** — listed as an interrupting exception by the design doc; Why3 is
  deeply recursive, but there are **zero** catch sites. It crashes uncaught today and
  stays that way; no behaviour change beyond the (backstopped / benign) skipped clean-up
  in the table above.
- **`Out_of_memory`** — the single hit
  ([`wserver.ml:133`](https://gitlab.inria.fr/why3/why3/-/blob/2d6fbb02b4101d0fb9062d2b2a2a62f0838c0734/src/ide/wserver.ml#L123-L152))
  is inside `print_exc`, a function that *formats an exception for an error message*;
  it is not a recovery handler. The design doc's "out-of-memory from the GC becomes
  fatal" breaks no recovery path here.
- **`Gc.finalise` / `Gc.create_alarm`** — none in the whole tree. No risk of an
  exception escaping from a finaliser.
- **memprof callback** —
  [`statmemprof.real.ml`](https://gitlab.inria.fr/why3/why3/-/blob/2d6fbb02b4101d0fb9062d2b2a2a62f0838c0734/src/ide/statmemprof.real.ml#L11)
  starts a sampling memory profiler, but it is selected only when the optional
  `statmemprof-emacs` package is installed
  ([`src/ide/dune:8-14`](https://gitlab.inria.fr/why3/why3/-/blob/2d6fbb02b4101d0fb9062d2b2a2a62f0838c0734/src/ide/dune#L8-L14);
  otherwise the empty `statmemprof.dummy.ml` is used). It is a debug profiling tool whose
  callback records allocation samples; it does not raise application exceptions.
- **Signal handlers** — the only OCaml ones are the Isabelle `SIGINT` handler (sends a
  socket "cancel", raises nothing), the web server's `SIGPIPE` ignore (raises nothing),
  and the commented-out `debug.ml` `SIGINT` (would `exit`). None raises an application
  exception.
- **Long-lived modes analysed separately:**
  - *GTK IDE* (`why3 ide`,
    [`why3ide.ml`](https://gitlab.inria.fr/why3/why3/-/blob/2d6fbb02b4101d0fb9062d2b2a2a62f0838c0734/src/ide/why3ide.ml#L328)):
    "signals" here are GTK UI callbacks, not Unix signals; the only Unix signal influence
    is via the shared scheduler/`prove_client`, which is cooperative. Interruption is the
    cooperative Interrupt button. No interrupting exception is caught, so the daemon-mode
    caveat (a recovery loop that stops catching) does not apply — there is nothing it
    catches today.
  - *Web/HTTP server* (`why3web`,
    [`wserver.ml`](https://gitlab.inria.fr/why3/why3/-/blob/2d6fbb02b4101d0fb9062d2b2a2a62f0838c0734/src/ide/wserver.ml#L283-L427)):
    a genuinely long-lived loop with a per-request catch-all
    ([`:283-286`](https://gitlab.inria.fr/why3/why3/-/blob/2d6fbb02b4101d0fb9062d2b2a2a62f0838c0734/src/ide/wserver.ml#L283-L286))
    and a per-iteration catch-all
    ([`:409-426`](https://gitlab.inria.fr/why3/why3/-/blob/2d6fbb02b4101d0fb9062d2b2a2a62f0838c0734/src/ide/wserver.ml#L409-L426)).
    These catch *ordinary* exceptions and keep the server alive. The only interrupting
    exception that could reach them is `Stack_overflow`; today an uncaught
    `Stack_overflow` in a request would already escape both (it propagates past
    `with exc -> …` only at safe points, and the server is not written to recover from a
    stack overflow), so under the new rules the server's behaviour is unchanged. If Why3
    *wanted* the web server to survive a per-request `Stack_overflow`, a per-request
    `Sys.with_async_exns` at `:283` would be the place — but that is a new robustness
    feature, not a regression to fix.
  - *Server / session mode* (`controller_itp` + `itp_server`): prover results, timeouts
    and interruptions all arrive as values over the socket and are dispatched to
    callbacks ([`controller_itp.ml:452-538`](https://gitlab.inria.fr/why3/why3/-/blob/2d6fbb02b4101d0fb9062d2b2a2a62f0838c0734/src/session/controller_itp.ml#L452-L539)).
    The `with e when not stack_trace -> spa.spa_callback (InternalFailure e)`
    ([`controller_itp.ml:534-535`](https://gitlab.inria.fr/why3/why3/-/blob/2d6fbb02b4101d0fb9062d2b2a2a62f0838c0734/src/session/controller_itp.ml#L531-L536))
    catches a *synchronous* exception from building one prover call and turns it into an
    `InternalFailure` value; no interrupting exception is in play.
- **One-shot tools** (`why3prove`, `why3replay`, `main`): each has a top-level
  `try … with e when not (Debug.test_flag Debug.stack_trace) -> print; exit 1`
  ([`main.ml:109-118`](https://gitlab.inria.fr/why3/why3/-/blob/2d6fbb02b4101d0fb9062d2b2a2a62f0838c0734/src/tools/main.ml#L109-L119),
  [`why3prove.ml:211-258`](https://gitlab.inria.fr/why3/why3/-/blob/2d6fbb02b4101d0fb9062d2b2a2a62f0838c0734/src/tools/why3prove.ml#L211-L258)).
  These are ordinary-exception reporters. The only interrupting exception that could
  bypass them is `Stack_overflow`; bypassing them just changes the printed message, and
  the process exits either way.
- **Wildcard handlers** — the handful found
  ([`controller_itp.ml:275`](https://gitlab.inria.fr/why3/why3/-/blob/2d6fbb02b4101d0fb9062d2b2a2a62f0838c0734/src/session/controller_itp.ml#L275)
  turns a parse failure into a value; [`itp_server.ml:647`](https://gitlab.inria.fr/why3/why3/-/blob/2d6fbb02b4101d0fb9062d2b2a2a62f0838c0734/src/session/itp_server.ml#L647)
  is a pretty-printing fallback) catch ordinary exceptions only; none restores a
  persistent invariant that an interrupting `Stack_overflow` would damage.
- **JavaScript build** (`why3_js.ml`, the web worker): no Unix signals, no timers; the
  native asynchronous-exception machinery does not apply. Out of scope.

## Recommendations

In priority order. The change touches Why3 very little.

1. **No required code change.** There is no interrupting exception that is caught today
   and would stop being caught, so there is no regression to fix and no
   `Sys.with_async_exns` boundary that correctness demands. The throwing side is empty
   too (no application exception is raised from any signal handler), so no
   `Sys.raise_async` is needed.

2. **Confirm two runtime facts on OxCaml** (cheap, need the runtime, not the source):
   that `at_exit` runs on interrupting-uncaught termination (so the `why3server`
   disconnect/reap at [`prove_client.ml:147-149`](https://gitlab.inria.fr/why3/why3/-/blob/2d6fbb02b4101d0fb9062d2b2a2a62f0838c0734/src/driver/prove_client.ml#L147-L150)
   still happens), and that `Stack_overflow` remains a terminal interrupting exception.

3. **Optional hygiene — convert the four clean-up sites to `new_protect`.**
   `sysutil.ml`'s `write_file_fmt`, `write_unique_file_fmt`, `open_temp_file`, and
   `debug.ml`'s `record_timing` rely on their `finally`/handler running. They are safe
   today only because the one interrupting exception (`Stack_overflow`) is terminal and
   their skipped clean-up is either invisible or backstopped. Moving them to the
   interruption-safe bracket is tidy and future-proofs them against a *second*,
   recoverable interrupting exception ever being introduced. Use a masking bracket that
   re-throws still-interrupting, not a converting one.

4. **Future feature, not a fix.** If Why3 later wants the GTK IDE or the web server to
   survive a `Stack_overflow` in one proof task instead of dying, add a
   `Sys.with_async_exns` boundary around each unit of work — per request in
   [`wserver.ml:283`](https://gitlab.inria.fr/why3/why3/-/blob/2d6fbb02b4101d0fb9062d2b2a2a62f0838c0734/src/ide/wserver.ml#L283-L286)
   and per task callback in
   [`controller_itp.ml:519-538`](https://gitlab.inria.fr/why3/why3/-/blob/2d6fbb02b4101d0fb9062d2b2a2a62f0838c0734/src/session/controller_itp.ml#L519-L539).
   This is new robustness, decoupled from the migration.

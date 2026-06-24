# Lwt — async-exceptions initial triage

Repo: `ocsigen/lwt` @ `59daedd52c76eeca65e6198e4196660845177458` (shallow clone in `./lwt`).

This is a first pass looking for code in Lwt that works correctly today but breaks
under OxCaml's new rules for "asynchronous exceptions".

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

- **Lwt already does the thing the new model wants for signals.** Lwt never installs
  an OCaml signal handler that raises. Its signal handling is done in C: the C
  handler installed by
  [`lwt_unix_set_signal`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/unix/lwt_unix_stubs.c#L833-L869)
  only writes a byte to a self-pipe / eventfd
  ([`handle_signal`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/unix/lwt_unix_stubs.c#L802-L820)),
  and the user's OCaml callback registered with
  [`Lwt_unix.on_signal`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/unix/lwt_unix.cppo.ml#L2228-L2254)
  runs **later, in the ordinary event loop** (via the notification system), on the
  normal path. So Lwt produces **no `Sys.Break`** and raises **no exception from a
  signal handler**. There is no `Sys.catch_break` and no `Sys.set_signal` with an
  OCaml handler anywhere in Lwt's own code.
- **Lwt's GC finalisers don't raise either.** Every `Gc.finalise` Lwt installs uses
  the same deferral trick: the finaliser only fills a cell and sends a self-pipe
  notification, or it `ignore`s a promise-returning close — the real work happens
  later in the event loop. So Lwt raises nothing from a finaliser. (One exception in
  `Lwt_react` runs a *user-supplied* function directly in the finaliser; see "Checked
  and fine".)
- **The only asynchronous exception that actually reaches Lwt's code is
  `Stack_overflow`** (plus `Out_of_memory` only in its synchronous, non-GC form — the
  design doc makes GC out-of-memory fatal). Lwt is a promise library whose whole job
  is deep callback chains, so `Stack_overflow` is realistic.
- **Lwt *already* treats `Stack_overflow`/`Out_of_memory` as un-catchable by default,
  which is the single most important fact here.** Lwt's central callback runner wraps
  every user callback in `try … with exn when Exception_filter.run exn -> …`, and the
  default filter is
  [`handle_all_except_runtime`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/core/lwt.ml#L712-L722),
  which **returns `false` for `Out_of_memory` and `Stack_overflow`**. So today, by
  default, those two already slip past every `Lwt.catch`/`finalize`/`try_bind`/`async`
  guard and bubble straight out of `Lwt_main.run`. Under OxCaml `Stack_overflow`
  becomes an asynchronous exception that *also* slips past those guards — so for the
  default configuration **the catch-side behaviour is essentially unchanged**.
- **The one genuine behaviour change: programs that opt in to catching the runtime
  exceptions.** A program that calls
  [`Lwt.Exception_filter.set Lwt.Exception_filter.handle_all`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/core/lwt.ml#L713-L723)
  is explicitly asking Lwt to catch `Stack_overflow`/`Out_of_memory` and turn them
  into rejected promises. Under OxCaml that opt-in **silently stops working for
  `Stack_overflow`** (it is now asynchronous and bypasses the `with exn when …`
  guard regardless of what the filter says) — the program that expected a rejected
  promise now sees the event loop die. That is the place where Lwt's documented,
  supported behaviour changes.
- **Nothing to do on the throwing side.** Lwt raises no exception from a signal
  handler, finaliser, or memprof callback (it has no memprof callbacks at all), so
  there is nothing to convert to the new throwing primitive (`Sys.raise_async`). And
  because there is no `Sys.Break` in Lwt, the `Sys.catch_break` "comes for free" rule
  is simply not engaged here.
- **Lwt is fundamentally a long-lived event loop**, so the "process is dying anyway"
  reasoning does *not* automatically apply — but with the default filter the one live
  asynchronous exception (`Stack_overflow`) already escapes `Lwt_main.run` today and
  is documented as leaving Lwt's internal state inconsistent and `Lwt_main.run`
  un-reusable. So the relevant clean-up that gets skipped is the same set that is
  *already* skipped today, not a new regression — except for the `handle_all` opt-in
  above.
- **The clean-up that an asynchronous `Stack_overflow` skips** lives behind a handful
  of core combinators — the callback runner's
  [`Exception_filter` guards](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/core/lwt.ml#L1143-L1150),
  `Lwt.finalize`/`Lwt.catch`/`Lwt.try_bind`
  ([`lwt.ml:2307-2315`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/core/lwt.ml#L2307-L2315)),
  and the engine's
  [`ev_unloop` re-raise](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/unix/lwt_engine.ml#L192-L197).
  All of these are safe today *only because the default filter already lets
  `Stack_overflow` through* — i.e. safe today only by luck; they become real skips
  the moment a program opts in to `handle_all` or a second asynchronous exception
  enters scope.
- Everything else checked (`Out_of_memory`, all `Gc.finalise` sites, memprof, the C
  signal path, `Lwt.Canceled`, `Lwt_timeout`, `Lwt_main.run`, `Lwt_mutex`/`Lwt_pool`/
  `Lwt_io`/`Lwt_stream` clean-up, the `fork`-child `unix_exit`) is clear.

## Which exceptions are affected

### Signals never become OCaml exceptions in Lwt (why Lwt is mostly immune)

Lwt does not use OCaml signal handlers. `Lwt_unix.on_signal` /
`on_signal_full`
([`lwt_unix.cppo.ml:2228-2254`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/unix/lwt_unix.cppo.ml#L2228-L2254))
register the user's callback into a `Lwt_sequence` and ask the C stub
`lwt_unix_set_signal` to install a **C** signal handler. That handler,
[`handle_signal`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/unix/lwt_unix_stubs.c#L802-L820),
does nothing but `lwt_unix_send_notification(id)` — a write to the notification
self-pipe / eventfd
([`pipe_notification_send`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/unix/lwt_unix_stubs.c#L734-L737),
[`eventfd_notification_send`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/unix/lwt_unix_stubs.c#L720-L723)).
The event loop later notices the pipe is readable and runs
[`handle_notifications`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/unix/lwt_unix.cppo.ml#L2189-L2193),
which calls the registered OCaml callbacks **on the ordinary event-loop path**. So:

- A signal in Lwt never raises an exception through OCaml frames — it becomes an
  ordinary event. This is exactly the discipline the new model assumes, and it means
  Ctrl+C, SIGCHLD
  ([`sigchld_handler`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/unix/lwt_unix.cppo.ml#L2336-L2355)),
  etc. behave under OxCaml exactly as today.
- There is **no `Sys.Break`** in Lwt and **no `Sys.catch_break`**, so the "Ctrl+C via
  a signal handler" path that needs a `Sys.with_async_exns` wrapper in other programs
  simply does not exist here. If a *user's* `on_signal` callback raises, that happens
  on the normal event-loop path and is handled by the ordinary callback machinery
  below — it is not an asynchronous exception.

This is the most important finding: Lwt already keeps signals from being raised as
exceptions through OCaml code.

### The central callback runner and its `Exception_filter` guard

Lwt turns "an exception raised in a callback" into "a rejected promise" in one place:
the callback is run inside `try … with exn when Exception_filter.run exn -> fail exn`
(or `→ !async_exception_hook exn` for fire-and-forget callbacks,
[`handle_with_async_exception_hook`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/core/lwt.ml#L1143-L1150)).
The same `with exn when Exception_filter.run exn ->` guard appears in
[`bind`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/core/lwt.ml#L1853-L1854),
[`catch`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/core/lwt.ml#L2026-L2029),
[`try_bind`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/core/lwt.ml#L2152-L2172),
[`map`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/core/lwt.ml#L1979-L1980),
[`async`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/core/lwt.ml#L2542-L2545),
and `with_value`
([`lwt.ml:807-813`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/core/lwt.ml#L807-L813)).

The filter
([`Exception_filter`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/core/lwt.ml#L712-L722))
decides what the guard catches. Its default value is `handle_all_except_runtime`:

```
let handle_all_except_runtime = function
  | Out_of_memory -> false
  | Stack_overflow -> false
  | _ -> true
let v = ref handle_all_except_runtime
```

So by default the guard returns `false` for `Out_of_memory`/`Stack_overflow`, meaning
the `try … with exn when …` does **not** match them and they propagate as real
exceptions out of the callback, out of the resolution loop, and out of
`Lwt_main.run`. (Note: the `.mli` prose at
[`lwt.mli:2020-2023`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/core/lwt.mli#L2020-L2023)
says "`handle_all` is the default", but the code default is
`handle_all_except_runtime` — the doc comment is stale.)

Consequence for OxCaml: the *one* asynchronous exception Lwt can actually meet,
`Stack_overflow`, is exactly the one the default filter already lets bypass these
guards. So for the default configuration nothing about the catch side changes. The
change bites only when a program has opted in to catching the runtime exceptions (see
the verdict table).

### How an asynchronous `Stack_overflow` leaves the event loop

[`Lwt_main.run`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/unix/lwt_main.ml#L62-L118)
finishes with `| exception exn when Lwt.Exception_filter.run exn -> finished (); raise
exn`. With the default filter, `Stack_overflow` makes that guard `false`, so the
`finished ()` book-keeping (resetting `run_already_called`) is **skipped** and the
exception escapes — which is exactly the documented behaviour ("you will not be able
to call `Lwt_main.run` again",
[`lwt.mli:2029-2032`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/core/lwt.mli#L2029-L2032)).
Under OxCaml the asynchronous `Stack_overflow` bypasses the same guard with the same
result. No change.

## Clean-up that could be skipped

All sites below clean up and re-throw (or run clean-up as a follow-on callback) and
are bypassed by an asynchronous `Stack_overflow`. The recurring verdict is "safe
today, only by luck": they are safe today **only because the default filter already
lets `Stack_overflow` through them**, so they already don't run on a `Stack_overflow`
today — OxCaml doesn't change that. They become genuine skips for a program that opts
in to `Lwt.Exception_filter.handle_all`, or if a second asynchronous exception enters
scope.

| Site | Cleanup | Verdict | Why |
|------|---------|---------|-----|
| [`lwt.ml:1143-1150`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/core/lwt.ml#L1143-L1150) `handle_with_async_exception_hook` + the `Exception_filter` guards in `bind`/`catch`/`try_bind`/`map`/`async`/`with_value` | turn a callback exception into a rejected promise / route it to `async_exception_hook`; `with_value` also restores `current_storage` | safe today, only by luck | The central place where "exception in a callback" becomes "rejected promise". For `Stack_overflow` the default filter already returns `false`, so these guards already don't fire today — the exception already escapes the loop. Under OxCaml the asynchronous `Stack_overflow` likewise bypasses them: identical behaviour for the default filter. With `handle_all` set, today the guard catches `Stack_overflow` and produces a rejected promise; under OxCaml it cannot — that conversion is lost (see "genuine change" below). |
| [`lwt.ml:2307-2315`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/core/lwt.ml#L2307-L2315) `Lwt.finalize` / `backtrace_finalize` | run the finaliser `f' ()` on both the success and failure continuations | safe today, only by luck | `finalize` is built on `try_bind`/`bind`, so the finaliser runs as a *callback* keyed off the body's result. An asynchronous `Stack_overflow` raised *inside the body* never becomes a result (the `try_bind` guard doesn't catch it under the default filter), so the `f' ()` finaliser never runs. This is the cleanup-skipping shape, but it already behaves this way today for `Stack_overflow`. It underlies `Lwt_mutex.with_lock` ([`lwt_mutex.ml:39`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/core/lwt_mutex.ml#L39)), `Lwt_io` flush/close ([`lwt_io.ml:361`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/unix/lwt_io.ml#L361)), `Lwt_pool` ([`lwt_pool.ml:94`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/core/lwt_pool.ml#L94)), and the `Lwt_unix` `closedir`/`with_*` wrappers — fixing `finalize`/`catch`/`try_bind` covers them all. |
| [`lwt_engine.ml:192-197`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/unix/lwt_engine.ml#L192-L197) `libev#iter` | `try ev_loop loop block with exn -> ev_unloop loop; raise exn` | safe today, only by luck (low severity) | This is the one bare `with exn ->` (no filter) on a hot path. An asynchronous `Stack_overflow` raised inside a callback that libev invokes during `ev_loop` bypasses this arm, so `ev_unloop` (which marks the libev loop as no longer iterating) is skipped. But the only live asynchronous exception is fatal-by-default here (it escapes `Lwt_main.run` and the loop is being torn down anyway), so the skipped `ev_unloop` is moot. Becomes relevant only if a caller wraps `Lwt_main.run` to recover and re-enter — Lwt already documents that as unsupported after a runtime exception. |
| [`lwt_main.ml:62-118`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/unix/lwt_main.ml#L62-L118) `Lwt_main.run` outer guard | `| exception exn when Exception_filter.run exn -> finished (); raise exn` (reset `run_already_called`) | safe today, only by luck | For `Stack_overflow` the guard is already `false` under the default filter, so `finished ()` is already skipped today and `Lwt_main.run` is left non-reusable — the documented behaviour. OxCaml's asynchronous `Stack_overflow` does the same. No change for the default; the `finished ()` reset is the book-keeping that a `Sys.with_async_exns` wrapper here could restore if Lwt wanted `run` to be re-callable after an asynchronous exception. |

(There are **no** raw `Fun.protect` clean-up sites in Lwt's library code except the
tracing emit in
[`lwt_main.ml:50-52`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/unix/lwt_main.ml#L50-L52),
whose finaliser only emits a runtime-events trace marker — diagnostic only, no
resource impact. All real clean-up funnels through `finalize`/`catch`/`try_bind`.)

## The one genuine behaviour change: the `handle_all` opt-in

A program that has called `Lwt.Exception_filter.set Lwt.Exception_filter.handle_all`
([`lwt.ml:713-723`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/core/lwt.ml#L713-L723),
public API in
[`lwt.mli:2013-2043`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/core/lwt.mli#L2013-L2043))
has explicitly asked Lwt to catch `Out_of_memory`/`Stack_overflow` in its callback
guards and turn them into rejected promises — so that, e.g., a `Lwt.catch` around a
deeply recursive computation recovers from a stack overflow instead of dying. Today
that works: the `with exn when Exception_filter.run exn -> fail exn` arm fires for
`Stack_overflow` (the filter now returns `true`) and the promise is rejected.

Under OxCaml, `Stack_overflow` is an asynchronous exception that travels the separate
path and **does not match an ordinary `with exn when …` arm at all**, no matter what
the filter returns. So:

1. The `handle_all` opt-in **silently stops catching `Stack_overflow`.** A program
   relying on it to keep a long-lived event loop alive across stack overflows now
   sees the asynchronous exception bypass every guard and stop the loop. This is a
   real, silent behaviour change for those programs.
2. To preserve it, the place that wants to *recover* from a `Stack_overflow` needs a
   `Sys.with_async_exns` wrapper that turns the asynchronous `Stack_overflow` back
   into an ordinary one *before* it reaches the Lwt callback guard — i.e. a wrapper
   around the callback-running step in the resolution loop / `Lwt_main.run`, gated on
   the filter being `handle_all`. Where exactly to put it is the design call.

(`Out_of_memory` is less of a concern: the design doc makes GC out-of-memory a fatal
error rather than an exception, so the only `Out_of_memory` left is the synchronous
kind — e.g. `Array.make` with a bad size — which stays an ordinary exception and is
still caught by `handle_all`. Only `Stack_overflow` regresses.)

## Why this is subtle

1. **Lwt has already, deliberately, decided that the runtime exceptions are
   un-catchable by default.** The `Exception_filter` machinery exists precisely
   because `Stack_overflow`/`Out_of_memory` "are thrown at different points depending
   on the machine" and "recovering may be impossible"
   ([`lwt.mli:2001-2011`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/core/lwt.mli#L2001-L2011)).
   So for the default configuration, OxCaml's change is almost a no-op: it makes the
   runtime treat `Stack_overflow` the way Lwt already chose to treat it.
2. **The chain is short and mostly already bypassed:**
   ```
   Lwt_main.run  | exception exn when Exception_filter.run exn -> finished (); raise   <- reset book-keeping (already skipped for Stack_overflow by default)
     libev#iter  with exn -> ev_unloop; raise exn                                      <- mark loop idle (skipped)
       callback runner  with exn when Exception_filter.run exn -> fail/hook            <- turn into rejected promise (returns false for Stack_overflow by default)
         Lwt.finalize / Lwt.catch / Lwt.try_bind                                       <- run finaliser as a follow-on callback (never scheduled if body overflows)
   ```
   Under the default filter, an asynchronous `Stack_overflow` skips every one of these
   frames *today* — OxCaml does not change that. The change is confined to the
   `handle_all` opt-in, where today the callback-runner frame *does* catch
   `Stack_overflow` and under OxCaml will not.
3. **Signals are entirely out of the picture.** Because Lwt converts every signal to a
   self-pipe event, none of the "Ctrl+C / `Sys.Break` / custom exception from a signal
   handler" concerns apply.

## How to fix it

| Site kind | Sites | Fix | Difficulty |
|-----------|-------|-----|------------|
| Code that catches the exception to decide what to do next — *consumes* a `Stack_overflow` to recover (only when the user opted in to `handle_all`) | the callback runner / `Lwt_main.run` callback step, gated on `Exception_filter = handle_all` | `Sys.with_async_exns` wrapper around the callback-running step so an asynchronous `Stack_overflow` is turned back into an ordinary one and the existing `Exception_filter.run` guard can catch it as today | design call (placement + whether to bother) |
| Resource / lock / state clean-up that only needs "the finaliser must run" | `Lwt.finalize`/`catch`/`try_bind` ([`lwt.ml:2307`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/core/lwt.ml#L2307-L2315)), `libev#iter` ([`lwt_engine.ml:192`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/unix/lwt_engine.ml#L192-L197)) | an interruption-safe clean-up helper (`new_protect`) that runs the finaliser and then re-throws the interruption unchanged — fix the *combinators*, all of `Lwt_mutex`/`Lwt_pool`/`Lwt_io`/`Lwt_unix` clean-up inherit it | mechanical, but only buys something for the `handle_all` opt-in / future second asynchronous exception |

**Caveat — how the helper re-throws.** A `new_protect` that runs the clean-up and
then re-throws the *asynchronous* exception unchanged frees the resource but keeps the
exception flying past downstream ordinary handlers (you still need the explicit
`Sys.with_async_exns` if you want the existing `Exception_filter` guard to convert it
to a rejected promise). A `new_protect` that instead turns it back into an *ordinary*
exception would silently re-enable every downstream `with exn when …` guard to catch a
future asynchronous exception against the model — and, worse for Lwt, would make
`Stack_overflow` catchable again even when the filter is `handle_all_except_runtime`,
contradicting Lwt's own default policy. So: re-throw the interruption unchanged in the
combinators, and convert to ordinary only at one explicit `Sys.with_async_exns` placed
where the `handle_all` user wants recovery.

## Resolved during this pass

- **Signals never raise in Lwt** — verified by reading the C stub: the signal handler
  is `handle_signal`
  ([`lwt_unix_stubs.c:802-820`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/unix/lwt_unix_stubs.c#L802-L820)),
  which only sends a self-pipe / eventfd notification; the OCaml `on_signal` callback
  runs later on the event-loop path
  ([`lwt_unix.cppo.ml:2189-2193`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/unix/lwt_unix.cppo.ml#L2189-L2193)).
  No `Sys.Break`, no `Sys.catch_break`, no OCaml `Signal_handle`. So nothing to throw
  the new way (`Sys.raise_async`) and no Ctrl+C catch-side work.
- **The default `Exception_filter` already excludes the runtime exceptions** — read at
  [`lwt.ml:712-722`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/core/lwt.ml#L712-L722):
  the default is `handle_all_except_runtime`, so `Stack_overflow`/`Out_of_memory`
  already bypass every Lwt callback guard today. This is what makes the default
  configuration essentially unaffected. (The `.mli` comment claiming `handle_all` is
  the default is stale.)
- **`Lwt.Canceled` is an ordinary, cooperative exception** — `Lwt.cancel`
  ([`lwt.ml:1418-1449`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/core/lwt.ml#L1418-L1449))
  sets the target promise's state to `Rejected Canceled` and propagates through the
  normal callback machinery. It is not raised from a signal handler or the runtime; it
  is not an asynchronous exception and is unaffected.
- **GC finalisers don't raise** — `Lwt_gc.finalise`
  ([`lwt_gc.ml:23-45`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/unix/lwt_gc.ml#L23-L45))
  sends a self-pipe notification; `cleanup_dir_handle`
  ([`lwt_unix.cppo.ml:1360-1365`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/unix/lwt_unix.cppo.ml#L1360-L1365))
  and the process/io `ignore_close` finalisers
  ([`lwt_process.cppo.ml:206`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/unix/lwt_process.cppo.ml#L206),
  [`lwt_io.ml:1858`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/unix/lwt_io.ml#L1858))
  `ignore` a promise-returning close; `lwt_fmt`
  ([`lwt_fmt.ml:35`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/unix/lwt_fmt.ml#L35))
  pushes to a stream. None raise from the finaliser context.
- **`at_exit` runs on uncaught `Stack_overflow`** — verified empirically on stock
  OCaml 5.4.1 (a program dying from runaway recursion still ran its `at_exit`
  callback; exit code 2). `Lwt_main`'s `at_exit` hook
  ([`lwt_main.ml:132-138`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/unix/lwt_main.ml#L132-L138))
  that drains the Lwt exit hooks therefore still fires on an uncaught asynchronous
  exception *if OxCaml's uncaught path runs `at_exit`* (one runtime confirmation).

## Open questions (design calls / runtime confirmations)

- **Whether to support `Stack_overflow` recovery under the `handle_all` opt-in, and
  where to put the wrapper.** If Lwt wants `Lwt.Exception_filter.set handle_all` to
  keep working under OxCaml, it needs a `Sys.with_async_exns` wrapper around the
  callback-running step (in the resolution loop / `Lwt_main.run`) that converts an
  asynchronous `Stack_overflow` back into an ordinary one so the existing
  `Exception_filter.run` guard catches it. This is the one real design decision; it is
  only worth doing if the recovery use case is to be preserved.
- **Fix the stale `.mli` comment** at
  [`lwt.mli:2020-2023`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/core/lwt.mli#L2020-L2023)
  ("`handle_all` is the default") — the code default is `handle_all_except_runtime`.
  Independent of OxCaml, but relevant to anyone reasoning about this behaviour.
- **OxCaml confirmations** (need the runtime, not the source): (1) `at_exit` runs on
  asynchronous-uncaught termination (so the `Lwt_main` exit-hook drain survives);
  (2) whether a `Stack_overflow` occurring inside a `Lwt_preemptive` worker thread
  terminates just that thread or the whole process under the new model.

## Checked and fine

- **`Stack_overflow`** — the one realistic asynchronous exception. Lwt's default
  filter already lets it bypass every callback guard and escape `Lwt_main.run`, so the
  default configuration is unaffected. The only regression is the `handle_all` opt-in
  (covered above).
- **`Out_of_memory`** — the design doc makes GC out-of-memory fatal, so only the
  synchronous kind remains; the default filter already excludes it, and `handle_all`
  still catches the synchronous kind (it is not asynchronous). No regression.
- **Signals** — converted to self-pipe events in C; never raise through OCaml. No
  `Sys.Break`, no `Sys.catch_break`, no OCaml `Signal_handle`.
  ([`lwt_unix_stubs.c:802-869`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/unix/lwt_unix_stubs.c#L802-L869))
- **`Gc.finalise`** — every Lwt-installed finaliser defers via notification or
  `ignore`s a promise; none raise from the finaliser context (see "Resolved").
- **`Lwt_react.with_finaliser`**
  ([`lwt_react.ml:18-23`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/react/lwt_react.ml#L18-L23),
  [`:271-276`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/react/lwt_react.ml#L271-L276))
  — this one runs a *user-supplied* `f ()` directly inside the `Gc.finalise` callback.
  If that user function raises, then under the design doc's strict reading an ordinary
  exception from a finaliser terminates the program. But this is the *user's* function,
  not Lwt's code; Lwt itself raises nothing here. Worth a doc note that finaliser
  functions passed to `with_finaliser` must not raise under the new model. Not a Lwt
  bug.
- **Memprof** — Lwt registers no memprof callbacks. The doc's "exception from a
  memprof callback" change is inert here.
- **`Lwt.Canceled` / `Lwt.cancel`** — ordinary cooperative exception, normal path;
  unaffected.
- **`Lwt_timeout`** — the timeout `action` runs as an ordinary event-loop callback
  driven by `Lwt_unix.sleep`
  ([`lwt_timeout.ml:65-79`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/unix/lwt_timeout.ml#L65-L79)),
  guarded by `Exception_filter.run`. Not a signal/timer that raises; not asynchronous.
- **`Lwt_mutex` / `Lwt_pool` / `Lwt_io` / `Lwt_stream` clean-up** — all built on
  `Lwt.finalize`/`Lwt.catch`/`Lwt.try_bind`
  ([`lwt_mutex.ml:39`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/core/lwt_mutex.ml#L39),
  [`lwt_pool.ml:49`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/core/lwt_pool.ml#L49)/[`:94`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/core/lwt_pool.ml#L94),
  [`lwt_io.ml:361`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/unix/lwt_io.ml#L361)),
  so they inherit whatever the core combinators do — no separate fix needed.
- **`fork`-child `unix_exit`** — the `with _ -> unix_exit 127` arms
  ([`lwt_process.cppo.ml:167-169`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/unix/lwt_process.cppo.ml#L167-L169),
  [`lwt_unix.cppo.ml:2419-2424`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/unix/lwt_unix.cppo.ml#L2419-L2424))
  run in the forked child to catch an `execv`/`execvp` failure and exit. They catch to
  decide-and-exit, not to clean up; an asynchronous exception here would just take a
  different exit path in a child that is about to be replaced anyway. Fine.
- **`discover.ml`** — build-time configuration script, not shipped library code. Out
  of scope.

## Recommendations

In priority order. Lwt's signal discipline already matches the new model and its
default treatment of the runtime exceptions already matches what OxCaml enforces, so
the substantive work is small.

1. **Decide whether to preserve `Stack_overflow` recovery for the `handle_all`
   opt-in** ([`lwt.ml:713-723`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/core/lwt.ml#L713-L723)).
   This is the only place Lwt's documented behaviour changes under OxCaml. If yes, add
   a `Sys.with_async_exns` wrapper around the callback-running step in the resolution
   loop / `Lwt_main.run`
   ([`lwt_main.ml:62-118`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/unix/lwt_main.ml#L62-L118))
   so an asynchronous `Stack_overflow` is turned back into an ordinary one and the
   existing `Exception_filter.run` guard catches it as today. This is the design call.

2. **Optionally make the core clean-up combinators interruption-safe.** Reimplement
   `Lwt.finalize`/`catch`/`try_bind`
   ([`lwt.ml:2307-2315`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/core/lwt.ml#L2307-L2315))
   and the `libev#iter` re-raise
   ([`lwt_engine.ml:192-197`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/unix/lwt_engine.ml#L192-L197))
   so the finaliser/`ev_unloop` runs even when an asynchronous exception unwinds, and
   re-throw the interruption *unchanged* (do **not** turn it into an ordinary
   exception — that would re-enable catching `Stack_overflow` even under the
   `handle_all_except_runtime` default, against Lwt's own policy). This buys nothing
   for the default configuration today (the default filter already skips these for
   `Stack_overflow`), but it is hygiene and protects against a future second
   asynchronous exception. Lower priority precisely because the default is already
   safe.

3. **Fix the stale `.mli` comment** at
   [`lwt.mli:2020-2023`](https://github.com/ocsigen/lwt/blob/59daedd52c76eeca65e6198e4196660845177458/src/core/lwt.mli#L2020-L2023)
   to say the default is `handle_all_except_runtime`, and add a note to
   `Lwt_react.with_finaliser` that the supplied function must not raise under the new
   model.

4. **Nothing to throw the new way.** Lwt raises no exception from any signal handler,
   finaliser, or memprof callback, so there is no `Sys.raise_async` work.

5. **Confirm on the OxCaml runtime** (not derivable from source): `at_exit` runs on
   asynchronous-uncaught termination (so the `Lwt_main` exit-hook drain survives); and
   whether a worker-thread `Stack_overflow` kills just the thread or the process.

**Sequencing note:** (1) is the only change that preserves a currently-documented
behaviour and is the single real decision; (2) is optional hardening with no day-one
payoff for the default configuration; (3) is independent doc hygiene; (4) is a no-op.
**Do *not*** reimplement the combinators so they turn the asynchronous exception into
an ordinary one — Lwt deliberately makes `Stack_overflow` un-catchable by default, and
that would defeat both its own policy and the new model.

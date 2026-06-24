# utop — async-exceptions initial triage

Repo: `ocaml-community/utop` @ `8cd6716d0f90efd67da90afa5bc31c7abe9a7a57` (shallow clone in `./utop`).

This is a first pass looking for code in utop that works correctly today but breaks
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

utop is an interactive read-eval-print loop: it reads a phrase, evaluates it, prints
the result, and loops, recovering after each phrase. The whole point of catching
Ctrl+C is to abandon the *current* phrase and return to the prompt without killing
the session. That recovery is exactly what the new model breaks.

## Bottom line

- **utop deliberately turns Ctrl+C into an exception while evaluating user code.**
  At startup [`common_init`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop_main.ml#L1417-L1418)
  calls `Sys.catch_break true`, which asks the standard library to install a `SIGINT`
  handler that **raises `Sys.Break` from inside the signal handler**. So while a user
  expression is running (inside `execute_phrase`), pressing Ctrl+C raises a genuine
  asynchronous `Sys.Break`.
- **The genuine bug: Ctrl+C during evaluation kills the whole utop session instead of
  returning to the prompt.** While a phrase is being evaluated, the only thing that
  catches a `Sys.Break` is the catch-all `with exn ->` around
  [`execute_phrase`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop_main.ml#L770-L802)
  in the console loop (and the analogous one in the Emacs loop,
  [`process_checked_phrase`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop_main.ml#L986-L1002)).
  Today that arm catches the `Sys.Break`, prints it as an error message and the loop
  continues — you get your prompt back. Under OxCaml the asynchronous `Sys.Break`
  bypasses that arm entirely, escapes
  [`loop term`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop_main.ml#L724-L805),
  is not matched by the only handler around it
  ([`with LTerm_read_line.Interrupt -> ()`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop_main.ml#L1514-L1517)),
  and falls through to
  [`main_internal`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop_main.ml#L1528-L1544)'s
  `with exn ->` which prints `Fatal error: exception Stdlib.Sys.Break` and `exit 2`.
  So a Ctrl+C that today aborts a runaway computation and drops you back at the prompt
  will, under OxCaml, terminate utop. **GENUINE BUG**, and the one the design doc
  refers to when it says utop "relied on the old behaviour".
- **Throwing side for Ctrl+C: nothing to do.** utop installs its SIGINT handler with
  the standard `Sys.catch_break true`, and OxCaml already arranges for the `Sys.Break`
  escaping that handler to be an asynchronous exception. So Ctrl+C propagates the new
  way with no code change; all the Ctrl+C work is on the catching side.
- **Throwing side for SIGHUP/SIGTERM: a real change is needed.** utop installs its own
  handler for `SIGHUP` and `SIGTERM` that raises a *custom* exception,
  [`Term signo`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop_main.ml#L1459-L1470),
  from inside the signal handler. `Term` is not `Sys.Break`, so OxCaml will **not**
  make it asynchronous automatically: under the new model an ordinary
  exception escaping a signal handler terminates the program. To keep raising it from
  the handler it must be thrown with the new throwing primitive (`Sys.raise_async`).
  (See the note below on whether utop even needs to raise it at all.)
- **The one `Sys.Break` handler that is *not* affected** is the one in
  [`read_phrase`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop_main.ml#L700-L716)
  (`| Sys.Break -> … "Interrupted." … read_phrase term`). That `Sys.Break` is **not**
  the signal-handler one: while utop is sitting at the prompt waiting for input, the
  terminal library (lambda-term) handles the Ctrl+C *key* itself and does a plain
  `raise Sys.Break` on the ordinary code path — it is a normal synchronous exception,
  caught normally, and is unaffected by the change. (Confirmed by reading lambda-term:
  the Ctrl+C key is bound to a `Break` action that does `raise Sys.Break`, while the
  SIGINT *signal* is what `Sys.catch_break` turns into a `Sys.Break`.)
- **Only two asynchronous exceptions are actually in play:** `Sys.Break` (Ctrl+C
  during evaluation, the bug above) and `Stack_overflow` (utop evaluates arbitrary
  user code and runs the OCaml type-checker, so deep recursion is realistic).
  `Out_of_memory` from the GC becomes fatal under the design doc; utop has no catch
  site for it anyway. There are **no** `Gc.finalise`, no memprof callbacks, no
  `setitimer`/`Unix.alarm` timers anywhere in utop's own code.
- **A small amount of clean-up gets skipped on its way out**, but most of it is
  either restored at the start of the next phrase or backstopped by `at_exit` — see
  the table. The user-visible damage is the session dying, not corrupted on-disk
  state. The one piece of state that genuinely matters — the terminal mode — is
  restored by lambda-term, not by utop, and survives because lambda-term tears the
  terminal down as the exception unwinds out of `Lwt_main.run` (worth confirming on
  the OxCaml runtime; see open questions).
- **Code that needs the new wrapper (`Sys.with_async_exns`):** the two evaluation
  catch-alls
  ([console loop `with exn ->`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop_main.ml#L770-L802),
  [Emacs `process_checked_phrase`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop_main.ml#L986-L1002)),
  which today catch a Ctrl+C and let the loop continue. Where exactly to put the
  wrapper is the design call.
- **utop is a long-lived interactive loop, not a one-shot command.** The "the process
  is exiting anyway" reasoning does *not* apply: the loop is supposed to survive a
  Ctrl+C and serve the next phrase. That is precisely why the bug matters — the
  recovery that keeps the session alive is the thing that stops working.
- Everything else checked (`Stack_overflow`, `Out_of_memory`, the SIGHUP/SIGTERM
  handler's reach, the Emacs SIGINT-ignore save/restore, the formatter save/restore
  helpers, `use_output`'s temp-file removal, history save, the pure `with _ ->`
  guards) is clear, low-severity, or already backstopped.

## Which exceptions are affected

### `Sys.Break` is asynchronous during evaluation (`Sys.catch_break true`)

[`common_init`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop_main.ml#L1410-L1470)
runs `Sys.catch_break true` (line 1418), which in the standard library is
`Sys.set_signal Sys.sigint (Signal_handle (fun _ -> raise Break))` — i.e. utop
installs a SIGINT handler that raises `Sys.Break` from inside the handler. That is
exactly the asynchronous-exception pattern. The comment right above it —
"Make sure SIGINT is catched while executing OCaml code." — says what it is for: a
Ctrl+C while a user expression is running.

Today the raised `Sys.Break` surfaces at the next safe point inside `execute_phrase`,
and the catch-all `with exn ->` around the evaluation in
[`loop term`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop_main.ml#L770-L802)
catches it. The code there even comments "The only possible errors are directive
errors" — the authors did not single Ctrl+C out, because today a `Sys.Break` is just
another exception this arm happens to catch. It is turned into an error message and
the loop continues to the next prompt.

Two consequences under OxCaml:
1. **Throwing side — already handled.** Because utop uses the standard
   `Sys.catch_break true`, OxCaml already makes the `Sys.Break` escaping that handler
   an asynchronous exception. utop needs no code change to throw it the new way.
2. **Catching side — the bug.** The asynchronous `Sys.Break` does **not** match the
   `with exn ->` around the evaluation. It escapes `loop term` and propagates outward
   to `main_internal`, which prints `Fatal error: exception …` and exits 2. The
   read-eval-print loop dies on Ctrl+C instead of recovering. A `Sys.with_async_exns`
   wrapper is needed to turn the asynchronous exception back into an ordinary one so
   the existing catch-all fires as it does today.

I verified the present-day behaviour empirically on stock OCaml 5.4.1: a program that
runs `Sys.catch_break true` and then loops in a long computation under a plain
`try … with exn -> …` *does* catch the SIGINT-driven `Sys.Break` and continue. That
is the behaviour utop relies on and that the new model removes.

### Ctrl+C at the prompt is a *different*, unaffected `Sys.Break`

While utop is waiting for you to type (inside the lambda-term read-line engine, run
via `Lwt_main.run` in
[`read_phrase`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop_main.ml#L700-L716)),
Ctrl+C is handled as a *key*, not via the SIGINT signal: lambda-term's key binding for
Ctrl+C does a plain `raise Sys.Break` on the ordinary Lwt code path. utop's
`read_phrase` catches that with `| Sys.Break -> … read_phrase term`, prints
"Interrupted." and re-prompts. Because that `Sys.Break` is a normal synchronous raise
(not from a signal handler), it stays a normal exception under OxCaml and this handler
keeps working. So the prompt-level "clear the current line" behaviour is unaffected;
only the *during-evaluation* Ctrl+C regresses.

### `Term` from the SIGHUP/SIGTERM handler is a custom exception from a signal handler

[`common_init`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop_main.ml#L1459-L1470)
installs `Sys.Signal_handle (fun signo -> raise (Term signo))` for `SIGHUP` and
`SIGTERM`, where `Term` is utop's
[own exception](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop_main.ml#L27)
(also exported in
[`uTop_main.mli`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop_main.mli#L13)).
Under OxCaml a *custom* (non-`Sys.Break`) exception escaping a signal handler does not
become asynchronous automatically and would terminate the program rather
than propagate as an exception. So if utop wants to keep raising `Term` from the
handler it must throw it with `Sys.raise_async`.

That said, `Term` is **never caught by name anywhere in utop** — there is no
`with Term …` in the codebase. Today it just unwinds to `main_internal`'s catch-all,
which prints "Fatal error: exception …" and exits 2. So the handler exists only to
turn SIGHUP/SIGTERM into "print a message and exit 2" instead of the default
terminate-silently. Two reasonable fixes: raise it via `Sys.raise_async` (preserves
today's message + exit code, needs a wrapper to catch it cleanly), or drop the custom
exception and let the default signal action terminate the process. Either way this is
a deliberate small decision, not a mechanical port.

### `Stack_overflow`

utop runs arbitrary user code and the OCaml type-checker, so a stack overflow during
evaluation or type-checking is realistic. utop has **no** `with Stack_overflow ->`
anywhere; today a stack overflow during evaluation is caught by the same evaluation
`with exn ->` and reported like any other error, and the loop continues. Under OxCaml
`Stack_overflow` becomes asynchronous and bypasses that arm exactly as `Sys.Break`
does — so it joins `Sys.Break` in killing the session instead of being reported. The
fix is the same wrapper; see the table.

## Clean-up that could be skipped

All sites below are reached by an asynchronous `Sys.Break` (Ctrl+C during evaluation)
or `Stack_overflow`. Verdicts take account of utop being a long-lived interactive loop.

| Site | Clean-up | Verdict | Why |
|------|----------|---------|-----|
| [Console eval `with exn ->`, `uTop_main.ml:770-802`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop_main.ml#L770-L802) and [Emacs `process_checked_phrase`, `:986-1002`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop_main.ml#L986-L1002) | catch the exception, print it as an error, **let the loop continue to the next prompt** | **GENUINE BUG** | This is the recovery that keeps the session alive. An asynchronous `Sys.Break`/`Stack_overflow` skips this arm, escapes `loop`, is not matched by the `LTerm_read_line.Interrupt` handler, and reaches `main_internal` → `Fatal error` + `exit 2`. utop dies on Ctrl+C-during-eval instead of returning to the prompt. This is a *catch-to-decide* site (it decides "report and keep looping"), so the fix is a `Sys.with_async_exns` wrapper, not a clean-up bracket. |
| [`uTop.ml:155-184` `collect_formatters`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop.ml#L155-L184), [`:186-214` `discard_formatters`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop.ml#L186-L214) | `with exn -> restore (); raise exn` — restore the global `Format` formatters' output functions and margins | safe today, only by luck (fragile) | These wrap output redirection during type-checking/printing. An asynchronous exception skips `restore ()`, leaving `Format.std_formatter`/`err_formatter` pointing at a buffer or with output suppressed. But under the bug above the session is dying anyway (the exception reaches `main_internal` and exits), so the stale formatter state is never observed. Becomes a real, observed corruption the moment the session is made to recover (i.e. once the wrapper above is added) — so convert it then. |
| [`uTop.ml:375-390` `check_phrase`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop.ml#L375-L390) | `with exn -> … Toploop.toplevel_env := env; Btype.backtrack snap; …` — restore the toplevel environment and type-checker snapshot after a failed check | safe today, only by luck (fragile) | This runs during *reading* (inside `read_phrase`/`Lwt_main.run`), where the live `Sys.Break` is the synchronous lambda-term key one (unaffected). The `Sys.catch_break` SIGINT handler is also armed here, so a SIGINT delivered mid-check could raise an asynchronous `Sys.Break` that skips the env/snapshot restore, leaving the type-checker's global environment inconsistent for the next phrase. Low day-one risk (the type-check window is short and Ctrl+C at the prompt normally arrives as the lambda-term key), but it restores genuinely sticky global state, so harden it alongside the eval wrapper. |
| [`uTop.ml:796-809` `use_output`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop.ml#L796-L809) | `Misc.try_finally ~always:(remove temp file)` | safe today, only by luck (low) | The `#use_output` directive shells out and removes a temp file in `~always`. An asynchronous `Sys.Break` during the command skips the removal, leaking one temp file in the system temp dir. Wasted disk at worst; not a broken invariant. (And it is reached via `execute_phrase`, so today it dies anyway under the bug above.) |
| [`uTop.ml:683-687` stash directive](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop.ml#L674-L687) | `with exn -> close_out oc; raise exn` — close the stash file | safe today, only by luck (low) | Runs as a `#utop_stash`/`#utop_save` directive during evaluation. An asynchronous exception skips `close_out`, leaking a file descriptor and possibly an unflushed file. Reached via `execute_phrase`, so today the session dies anyway; the fd is reclaimed at process exit. Low severity for a one-shot directive; matters only once recovery is restored. |
| [`uTop_main.ml:934-942` Emacs `read_line`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop_main.ml#L934-L942) | `Sys.signal sigint Signal_ignore` … `with exn -> Sys.set_signal sigint behavior; raise exn` — restore the SIGINT handler after reading a command line | safe today, only by luck (low) | While reading an Emacs command, SIGINT is set to **ignore**, so *no* `Sys.Break` is raised in this window — the only asynchronous exception that can hit it is `Stack_overflow`, and that would skip the restore, leaving SIGINT ignored. But a `Stack_overflow` here also escapes to `main_internal` and exits, so the un-restored handler is moot. Convert for hygiene when restoring recovery. |

## Why this is subtle

1. **Which handler catches Ctrl+C today is decided by where the program happens to be.**
   The SIGINT-driven `Sys.Break` fires at some arbitrary safe point, so whichever
   matching handler is innermost at that instant catches it. During evaluation that is
   the eval catch-all; at the prompt it is (a different, key-driven) `Sys.Break` in
   `read_phrase`. The evaluation catch-all was written as a generic "report errors and
   keep going" arm precisely because Ctrl+C can land anywhere inside the user's code.
2. **There are two different `Sys.Break`s that look identical in the source.** The one
   in `read_phrase` (prompt-level, from a lambda-term key) is a normal synchronous
   raise and keeps working; the one the eval catch-all relies on (from the
   `Sys.catch_break` SIGINT handler) becomes asynchronous and stops being caught. A
   reader who only greps for `Sys.Break` sees the `read_phrase` handler and might
   conclude utop is fine — but the affected catch is the *unnamed* `with exn ->` around
   evaluation, which never mentions `Sys.Break` at all.
3. **The handlers form a chain, and the asynchronous exception skips all but the
   outermost.** A Ctrl+C deep inside a running expression unwinds, innermost-out:
   ```
   Sys.with_async_exns wrapper                                   <- to be added (turn asynchronous -> ordinary)
     main_internal  with exn -> "Fatal error"; exit 2            (uTop_main.ml:1531) <- today: never reached on Ctrl+C; under OxCaml: this is what fires -> session dies
       main_aux  try loop term with LTerm_read_line.Interrupt -> ()  (uTop_main.ml:1514) <- does NOT match Sys.Break
         loop term  (try execute_phrase … with exn -> print error)   (uTop_main.ml:770-802) <- TODAY catches Sys.Break, prints, CONTINUES loop
           formatter save/restore, temp-file/fd clean-up           (uTop.ml) <- restore + re-raise, all skipped
   ```
   Today the innermost eval catch-all wins and the loop survives. Under OxCaml every
   frame up to `main_internal` is skipped, so the outermost "Fatal error + exit" wins.
4. **The damage is "the loop stops", not "state is corrupted".** Because the session
   exits, the skipped clean-up (formatter state, a temp file, an fd) is mostly moot
   *today*. It only becomes observable once the wrapper restores recovery — at which
   point those fragile sites must be converted so the now-surviving session does not
   accumulate stale formatter state, leaked temp files, or an ignored SIGINT handler.

## How to fix it

Use `Sys.with_async_exns` to convert the asynchronous exception back into an ordinary
one at the point where utop wants to *recover* from it, and an interruption-safe
clean-up helper (`new_protect`, which runs the finaliser and then re-throws the
interruption unchanged) for the restore-and-re-raise sites. The fixes split by what
each site needs from the exception:

| Site kind | Sites | Fix | Difficulty |
|-----------|-------|-----|------------|
| Catches the exception to decide what to do next — *consumes* the Ctrl+C/`Stack_overflow` to report it and keep looping | the console eval catch-all ([`uTop_main.ml:770-802`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop_main.ml#L770-L802)) and the Emacs one ([`:986-1002`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop_main.ml#L986-L1002)) | wrap the **evaluation step** in `Sys.with_async_exns` so an asynchronous `Sys.Break`/`Stack_overflow` turns back into an ordinary exception that the existing `with exn ->` catches — the loop then reports it and continues exactly as today | small, but placement is the design call |
| Resource / state restore — only needs "the finaliser must run" | the two formatter save/restore wrappers ([`uTop.ml:155-184`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop.ml#L155-L184), [`:186-214`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop.ml#L186-L214)), the env/snapshot restore in `check_phrase` ([`:375-390`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop.ml#L375-L390)), `use_output`'s temp-file removal ([`:796-809`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop.ml#L796-L809)), the stash `close_out` ([`:683-687`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop.ml#L683-L687)), the Emacs SIGINT restore ([`uTop_main.ml:934-942`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop_main.ml#L934-L942)) | `new_protect` so the restore/removal runs even when an asynchronous exception unwinds, then re-throw the interruption unchanged | mechanical |
| Custom exception raised from a signal handler | the `Term` SIGHUP/SIGTERM handler ([`uTop_main.ml:1459-1470`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop_main.ml#L1459-L1470)) | raise `Term` via `Sys.raise_async` (and add a wrapper if you want to keep printing the message + exit cleanly), or drop the custom exception and let the default signal action terminate | small decision |

**Caveat — how the clean-up helper re-throws.** A `new_protect` that runs the restore
and then re-throws the *asynchronous* exception frees the resource but keeps the
exception flying past downstream ordinary handlers (you still need the explicit
`Sys.with_async_exns` at the eval step for the loop to actually catch it and continue).
A `new_protect` that instead turned it back into an *ordinary* exception would silently
re-enable utop's many `with exn ->`/`with _ ->` arms to swallow a future asynchronous
exception — against the model. So: re-throw the interruption unchanged in the clean-up
helpers, and convert to ordinary only at the one explicit `Sys.with_async_exns` placed
at the evaluation step.

## Resolved during this pass

- **`Sys.Break` really is asynchronous in utop during evaluation** — confirmed via
  `Sys.catch_break true`
  ([`uTop_main.ml:1418`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop_main.ml#L1417-L1418)),
  which installs a SIGINT handler that raises. The handler that catches it today is the
  unnamed `with exn ->` around `execute_phrase`
  ([`:770-802`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop_main.ml#L770-L802)),
  not the named `read_phrase` handler.
- **The prompt-level `Sys.Break` is a separate, unaffected exception** — confirmed by
  reading lambda-term: the Ctrl+C key is bound to a `Break` action that does
  `raise Sys.Break` on the ordinary code path (it is not raised from a signal handler).
  So `read_phrase`'s `| Sys.Break ->` handler
  ([`:700-716`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop_main.ml#L700-L716))
  keeps working.
- **Throwing side for Ctrl+C needs no change** — OxCaml already makes the `Sys.Break`
  raised by the standard `Sys.catch_break` an asynchronous exception. (This makes the
  recovery bug a *present-day* bug on OxCaml, not a future risk.)
- **`Term` is a custom exception from a signal handler and is never caught by name** —
  confirmed there is no `with Term` anywhere; it only unwinds to `main_internal`'s
  catch-all. It needs `Sys.raise_async` (or removal).
- **There are no `Gc.finalise`, no memprof callbacks, no `setitimer`/`Unix.alarm`
  timers** in utop's own code — the only signal sources are `Sys.catch_break`
  (SIGINT), the `Term` handler (SIGHUP/SIGTERM), and the Emacs `Signal_ignore` for
  SIGINT (which raises nothing).
- **History save is backstopped by `at_exit`** — `init_history` registers the save via
  `Lwt_main.at_exit`
  ([`uTop_main.ml:49`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop_main.ml#L47-L64)),
  and `Lwt_main.at_exit` is implemented as an ordinary OCaml `at_exit` hook that drains
  the Lwt exit hooks. I verified on stock OCaml 5.4.1 that `at_exit` runs on an uncaught
  exception (including the `Sys.Break`/`Stack_overflow` case). So even when utop dies on
  the bug above, history is still saved — *if* OxCaml's uncaught-asynchronous path runs
  `at_exit` (one runtime confirmation).
- **utop is a long-lived interactive loop** — so the bug is a behaviour regression
  (loss of recovery), and the fragile clean-up sites become real once recovery is
  restored. The "process is exiting anyway" exemption only excuses them *today*, while
  the bug is unfixed and the session dies regardless.

## Open questions (design calls / runtime confirmations)

- **Where to put the `Sys.with_async_exns` wrapper (the real design call).** The
  natural place is around the evaluation step in each loop —
  `execute_phrase` in the console
  [`loop`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop_main.ml#L770-L802)
  and in the Emacs
  [`process_checked_phrase`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop_main.ml#L986-L1002)
  — so that an asynchronous `Sys.Break`/`Stack_overflow` becomes an ordinary exception
  that the existing catch-all handles and the loop continues. A wrapper placed too high
  (e.g. around the whole `loop term`) would convert the exception but then unwind out of
  the loop, which does not restore recovery — it must sit *inside* the loop body, below
  the eval catch-all, so the loop iterates again.
- **What to do with the `Term` SIGHUP/SIGTERM handler** — raise it via `Sys.raise_async`
  (and add a small wrapper to keep the "print + exit 2" behaviour), or drop the custom
  exception entirely. A decision, not a mechanical port.
- **OxCaml confirmations** (need the runtime, not the source): (1) does `at_exit` run on
  asynchronous-uncaught termination (so the history save survives if the session dies)?
  (2) does the terminal mode get restored when an asynchronous exception unwinds out of
  `Lwt_main.run` — lambda-term restores the terminal as part of tearing the read-line
  engine down; confirm that teardown still runs (or that utop's exit path restores it)
  under the new model, otherwise a Ctrl+C-killed session could leave the terminal in raw
  mode.

## Checked and fine

- **`Stack_overflow`** — listed as asynchronous by the doc; realistic in utop (user
  code + type-checker recursion). No `with Stack_overflow ->` anywhere, so no dedicated
  recovery is lost, but it joins `Sys.Break` in bypassing the evaluation catch-all and
  killing the session. Covered by the same eval wrapper. On its way out it skips the
  same fragile clean-up sites — same "safe today only by luck" analysis.
- **`Out_of_memory`** — the design doc makes GC out-of-memory fatal; utop has zero
  catch sites for it, so no recovery path is broken. The synchronous kind (e.g. a bad
  `Array.make` size in user code) stays an ordinary exception caught by the eval
  catch-all as today.
- **The `read_phrase` `| Sys.Break ->` handler**
  ([`uTop_main.ml:700-716`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop_main.ml#L700-L716))
  — catches the *key-driven* `Sys.Break` from lambda-term (ordinary synchronous raise),
  not the signal one. Unaffected.
- **The Emacs SIGINT-ignore window**
  ([`uTop_main.ml:934-942`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop_main.ml#L934-L942))
  — SIGINT is set to `Signal_ignore` while reading, so no `Sys.Break` is raised there;
  only the save/restore clean-up could be skipped (by `Stack_overflow`), and that is
  moot while the session is dying. Listed in the table for hygiene.
- **The pure `with _ -> None` guards** in the rewrite-rule machinery
  ([`uTop_main.ml:598`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop_main.ml#L595-L599),
  [`:627`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop_main.ml#L621-L629),
  [`:1463`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop_main.ml#L1460-L1466))
  — these turn any failure into a default (`None`, or "signal not supported"). They
  release no resource and restore no state; an asynchronous exception simply skips the
  default and propagates, which is acceptable.
- **The Emacs output-copy `Thread.create`**
  ([`uTop_main.ml:924`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop_main.ml#L920-L928))
  — a background thread that copies ocaml output to Emacs; it installs no signal
  handler and raises nothing asynchronous. Not in scope.
- **`Misc.try_finally` in `use_output`** — covered in the table (temp-file leak, low
  severity). It is the compiler-libs helper; utop only supplies the body and finaliser.
- **The classic (non-tty) path** — when stdin/stdout are not a tty, utop hands off to
  `Toploop.loop` ([`uTop_main.ml:1518-1523`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop_main.ml#L1518-L1523)),
  the stock OCaml toplevel loop. Any Ctrl+C-recovery question there belongs to the
  standard toplevel, not to utop's own code.
- **`expunge/modules.ml` `Fun.protect`** — a build-time helper that closes a file it
  read; not part of the shipped interactive loop and not reached by an asynchronous
  exception during a session. Out of scope.

## Recommendations

In priority order. The Ctrl+C throwing side already works (`Sys.catch_break` yields an
asynchronous `Sys.Break` on OxCaml), so the headline work is one wrapper plus the
SIGHUP/SIGTERM decision.

1. **Restore Ctrl+C recovery — the one genuine bug.** Wrap the **evaluation step** in
   each loop in `Sys.with_async_exns`: around `execute_phrase` in the console
   [`loop`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop_main.ml#L770-L802)
   and in the Emacs
   [`process_checked_phrase`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop_main.ml#L986-L1002),
   placed *inside* the loop body and *below* the existing `with exn ->` so the
   asynchronous `Sys.Break`/`Stack_overflow` becomes ordinary, the catch-all reports it,
   and the loop iterates again — exactly today's behaviour. This is the design call
   (where, precisely) and the fix for the bug at once.

2. **Decide the `Term` SIGHUP/SIGTERM handler**
   ([`uTop_main.ml:1459-1470`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop_main.ml#L1459-L1470)):
   either raise `Term` via `Sys.raise_async` (preserving the "print + exit 2" path,
   with a wrapper to catch it) or remove the custom exception and let the default
   signal action terminate. Required because a custom exception from a signal handler
   otherwise terminates the program under the new model.

3. **Convert the fragile clean-up sites to `new_protect`** once recovery is restored:
   the two formatter save/restore wrappers
   ([`uTop.ml:155-184`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop.ml#L155-L184),
   [`:186-214`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop.ml#L186-L214)),
   the `check_phrase` env/snapshot restore
   ([`:375-390`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop.ml#L375-L390)),
   the stash `close_out`
   ([`:683-687`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop.ml#L683-L687)),
   `use_output`'s temp-file removal
   ([`:796-809`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop.ml#L796-L809)),
   and the Emacs SIGINT restore
   ([`uTop_main.ml:934-942`](https://github.com/ocaml-community/utop/blob/8cd6716d0f90efd67da90afa5bc31c7abe9a7a57/src/lib/uTop_main.ml#L934-L942)).
   Re-throw the interruption *unchanged* (do not turn it into an ordinary exception).
   These are moot while the session dies; they matter the moment step 1 lets the
   session survive.

4. **Confirm on the OxCaml runtime** (not derivable from source): (a) `at_exit` runs on
   asynchronous-uncaught termination, so the history save survives a killed session;
   (b) the terminal mode is restored when an asynchronous exception unwinds out of
   `Lwt_main.run` (lambda-term teardown), so a Ctrl+C-killed session does not leave the
   terminal in raw mode.

**Sequencing note:** step 1 is the substantive fix and restores today's behaviour on
its own. Step 2 is an independent small decision. Step 3 only pays off after step 1
(before it, the skipped clean-up is unobservable because the session exits). **Do not**
rebuild the clean-up helpers to turn the asynchronous exception into an ordinary one:
utop has many downstream `with exn ->`/`with _ ->` arms that would then silently
swallow a future Ctrl+C, defeating the model.

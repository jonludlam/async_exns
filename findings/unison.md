# Unison — async-exceptions initial triage

Repo: `bcpierce00/unison` @ `91421d0617b0fb543c0eee51bcb4d4791d8b0631` (shallow clone in `./unison`).

This is a first pass looking for code that works correctly today but would break under
OxCaml's new rules for "asynchronous exceptions".

## What an asynchronous exception is

A few kinds of event can interrupt an OCaml program at almost any moment: pressing Ctrl+C,
a time-limit going off, or the program running out of stack space. Today they surface as
ordinary exceptions, so a normal `try … with` catches them. OxCaml changes this: these
asynchronous exceptions now travel a separate route, an ordinary `try … with` no longer
catches them, and unless the program wraps the right region in a new special handler
(`Sys.with_async_exns`, which turns an asynchronous exception back into an ordinary one
at that point) the exception flies straight past every normal handler and either reaches
that wrapper or stops the program.

## Bottom line

- **Unison turns Ctrl+C into `Sys.Break`, but in two stages, and only the second stage is
  asynchronous.** In the text interface, the first one or two Ctrl+C presses run a signal
  handler that merely sets a flag (`Abort.all ()`) — a *cooperative* request that the
  transport engine notices at its own checkpoints and turns into an ordinary `Util.Transient`
  exception. Only the **third (and later) Ctrl+C** actually `raise Sys.Break` *from inside
  the signal handler*
  ([`uitext.ml:919-925`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/uitext.ml#L919-L936)),
  to force immediate termination. That forced `Sys.Break` is the genuine asynchronous
  exception. (`Sys.Break` is *also* raised the ordinary way from non-handler code — see
  below — and those raises are unaffected.)
- **Throwing side: nothing to do for Ctrl+C.** Unison installs its own `SIGINT` handler
  that ends in `raise Sys.Break`
  ([`uitext.ml:946`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/uitext.ml#L937-L959)),
  and it also calls `Sys.catch_break true`. OxCaml already arranges for a `Sys.Break`
  escaping a signal handler to become an asynchronous exception, so no code change is
  needed to throw it the new way.
- **Only `Sys.Break` is actually affected; the residual is `Stack_overflow`.** "Out of
  stack" (`Stack_overflow`) can always happen in a deeply recursive program like Unison,
  but it has **no recovery catch sites**. "Out of memory" (`Out_of_memory`) is named in one
  place ([`uitext.ml:1759`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/uitext.ml#L1752-L1762))
  only inside a *pattern match on a stored value*, not a live catch. There are **no
  `Gc.finalise` calls in the synced core**, no memprof callbacks, and **no `setitimer`/
  `Unix.alarm` timers**. The other signal handlers (`SIGUSR1` log-rotate
  [`trace.ml:137-143`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/ubase/trace.ml#L137-L144),
  `SIGUSR2` safe-stop
  [`uitext.ml:1359-1373`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/uitext.ml#L1359-L1373),
  `SIGPIPE` ignore
  [`remote.ml:33`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/remote.ml#L31-L34))
  **do not raise** — they close a channel, write to a pipe / set a flag, or are ignored.
- **Unison's own clean-up primitives already do NOT run on `Sys.Break` — by design.** This
  is the single most important fact about Unison for this analysis. Its three clean-up
  combinators only catch Unison's *own* cooperative exceptions:
  - `Util.unwindProtect` / `Util.finalize`
    ([`util.ml:226-244`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/ubase/util.ml#L226-L244))
    catch **only `Util.Transient`** (`with Transient _ as e -> … ; raise e`).
  - `Remote.Thread.unwindProtect`
    ([`remote.ml:683-697`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/remote.ml#L681-L699))
    catches **only `Util.Transient` / `Util.Fatal`**, and is Lwt-based.

  A `Sys.Break` (or `Stack_overflow`, or `Util.Fatal` for the first two) already skips all
  of these *today*. So the on-disk-consistency clean-up that matters does **not** rely on
  catching the asynchronous exception, and the new model does not change its behaviour.
  This is why Unison is largely a non-target.
- **The only clean-up combinator that does catch `Sys.Break` today is `Copy.protect`**
  ([`copy.ml:24-35`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/copy.ml#L24-L36))
  (`with e -> close fds / delete temp; raise e`). It closes file descriptors and removes a
  partial temp file. Under the new model a forced `Sys.Break` would skip it. But (a) the
  file descriptors are independently tracked and closed when the connection is torn down
  (`Remote.resourceWithConnCleanup`,
  [`copy.ml:253-262`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/copy.ml#L253-L262)),
  and (b) the temp file is named with a distinctive `.unison.` prefix
  ([`os.ml:43-44`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/os.ml#L43-L44))
  that Unison ignores and that is harmless to leave behind. So this is **safe today only by
  luck (fragile)**, not a genuine on-disk bug.
- **On-disk consistency does not depend on catching the interruption.** The dangerous step
  — replacing a file by a two-stage rename through a temp file — is protected by a *commit
  log* on disk (`DANGER.README`), written *before* the renames and deleted *after*
  ([`files.ml:30-86`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/files.ml#L30-L86),
  [`files.ml:313-342`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/files.ml#L313-L342)).
  If Unison is killed between the two renames, the next run sees the leftover
  `DANGER.README` and **refuses to proceed** until the user checks the files
  ([`processCommitLog`, files.ml:70-79](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/files.ml#L70-L79)).
  This protection is a *file on disk*, not an exception handler, so it survives an
  asynchronous `Sys.Break` (indeed it survives `kill -9`). The archive itself is committed
  by a careful multi-phase write-scratch-then-flip
  ([`commitUpdates`, update.ml:2626-2660](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/update.ml#L2626-L2660)),
  and only *after* all transfers finish, so an interruption either leaves the old archive
  intact or is caught by the next run's crash-recovery.
- **The archive lock is backstopped by `at_exit`.** A skipped `unlockArchives` on
  `Sys.Break` is covered by `at_exit releaseHeldLocks`
  ([`update.ml:946-950`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/update.ml#L944-L950)),
  which runs on uncaught `Sys.Break` (verified, see below), plus a connection-close handler.
- **Code that catches `Sys.Break` to decide what to do next (needs `Sys.with_async_exns`).**
  These stop firing under the new model:
  - The text UI top level
    ([`uitext.ml:1733-1748`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/uitext.ml#L1710-L1748))
    `with Sys.Break -> terminate ()` — restores the terminal and exits cleanly with the
    interrupted message.
  - The transfer summary
    ([`uitext.ml:1083-1098`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/uitext.ml#L1080-L1098))
    `with e -> … (match e with Sys.Break -> "interrupted" …); raise e` — prints
    "Synchronization interrupted by user request" then re-raises.
  - The signal-handler-restore wrapper `stopAtIntr`
    ([`uitext.ml:937-959`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/uitext.ml#L937-L959))
    `with e -> restoreSig (); raise e` — re-installs the previous `SIGINT`/`SIGTERM`
    handlers on the way out.
  - The socket-server loop
    ([`remote.ml:2320-2329`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/remote.ml#L2310-L2330))
    `with Sys.Break -> unixSocketCleanup ()` — removes the Unix-domain socket file.
- **Long-lived modes analysed separately (below): repeat/watch terminates on Ctrl+C
  rather than recovering, so it gains nothing extra; the persistent socket server is the
  one place where a skipped `Sys.Break` cleanup leaves a file on disk.**
- Everything else checked (`Stack_overflow`, `Out_of_memory`, the GTK GUI's
  cooperative-only cancellation, the non-raising signal handlers, the ordinary
  `raise Sys.Break` sites) is clear or low-severity.

## Which exceptions are affected

### `Sys.Break` — asynchronous only when forced from the signal handler

Unison's text UI installs its own `SIGINT` handler `ctrlCHandler`
([`uitext.ml:924-936`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/uitext.ml#L919-L959), via
`stopAtIntr`) around the whole transfer phase. Its behaviour is graduated:

1. **First / second Ctrl+C** → `Abort.all ()`
   ([`abort.ml:59`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/abort.ml#L59))
   sets a flag and prints a message; **nothing is raised from the handler**. The transport
   engine notices the flag at its own checkpoints (`Abort.check` / `Abort.checkAll`,
   [`abort.ml:65-75`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/abort.ml#L65-L75))
   and raises an ordinary `Util.Transient "Aborted by user request"` — a *cooperative*
   cancellation along the normal path, unaffected by the change.
2. **Third+ Ctrl+C** → `raise Sys.Break`
   ([`uitext.ml:920`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/uitext.ml#L919-L922))
   *from inside the handler*. This is the genuine asynchronous exception: today it surfaces
   at the next safe point and one of the `with Sys.Break` / catch-all handlers above catches
   it; under the new model none of them do, and it goes to the nearest `Sys.with_async_exns`
   or stops the program.

So the *graceful* Ctrl+C path (presses 1–2) is cooperative and unaffected; the *forced*
Ctrl+C path (press 3+) is the one asynchronous exception that changes behaviour.

`Sys.Break` is *also* raised the **ordinary** way from non-handler code — the interactive
"quit" menu entries
([`uitext.ml:807`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/uitext.ml#L804-L807),
[`:1236`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/uitext.ml#L1233-L1236),
[`:1620`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/uitext.ml#L1618-L1620)),
the "stop repeat mode" re-raise
([`uitext.ml:1147`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/uitext.ml#L1147)),
and the raw-terminal read of a literal Ctrl+C byte
([`uitext.ml:188`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/uitext.ml#L180-L189)).
These are ordinary synchronous raises from normal code; catching them is **unaffected** by
the change, but they share the catch sites below.

### Unison's clean-up combinators already ignore the interruption

The reason almost nothing breaks: Unison's transport and archive code does not use a
general "run this finaliser whatever happens" bracket. It uses combinators scoped to its
*own* recoverable exceptions:

- `Util.finalize` / `Util.unwindProtect`
  ([`util.ml:226-244`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/ubase/util.ml#L226-L244))
  — `with Transient _ as e -> cleanup; raise e`. A `Sys.Break`, a `Stack_overflow`, or even
  a `Util.Fatal` already flies past these untouched, *today*. (The `.mli` comment
  [`util.mli:17-24`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/ubase/util.mli#L17-L24)
  says "the above two exceptions", i.e. `Transient` and `Fatal`, but the implementation only
  matches `Transient`; either way `Sys.Break` is not caught.)
- `Remote.Thread.unwindProtect`
  ([`remote.ml:683-697`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/remote.ml#L681-L699))
  — Lwt-based, catches only `Util.Transient`/`Util.Fatal`. The per-item transport clean-up
  ([`transport.ml:283`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/transport.ml#L279-L309))
  goes through this. A `Sys.Break` is not in scope here today.

Because these never ran on `Sys.Break` to begin with, the new model leaves them exactly as
they are. The on-disk-consistency guarantees are carried instead by the commit-log file and
the multi-phase archive commit (below), which are independent of exception handling.

### On-disk consistency is carried by files on disk, not by handlers

- **File replacement** (the only step that can leave a replica half-written): Unison renames
  the freshly-transferred temp file into place. When two renames are needed (the
  "moveFirst" case), it writes a `DANGER.README` commit log first, does the two renames,
  then deletes the log
  ([`files.ml:313-342`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/files.ml#L313-L342)).
  The `Util.finalize` there only clears the log on a `Transient`; on a `Sys.Break` (or any
  hard kill) the log is **deliberately left behind**, and the next run's `processCommitLog`
  ([`files.ml:70-79`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/files.ml#L70-L79))
  raises a fatal error telling the user to inspect the files. So an interruption mid-rename
  is *detected*, not silently corrupting — and this is true today and under the new model
  alike.
- **Archive commit** runs only *after* all transfers are done
  ([`uitext.ml:1108-1109`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/uitext.ml#L1108-L1109)),
  and writes to a scratch file, verifies all roots' checksums match, then flips scratch→main
  ([`commitUpdates`, update.ml:2626-2660](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/update.ml#L2626-L2660)).
  An interruption leaves the old archive intact (worst case the next run does crash-recovery
  / a full re-scan), never a half-written main archive.
- **Archive lock** is released by `at_exit releaseHeldLocks`
  ([`update.ml:946-950`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/update.ml#L944-L950))
  and by the connection-close handler
  ([`update.ml:957-971`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/update.ml#L957-L971)),
  so a skipped `unlockArchives` is covered. (And if it weren't, a stale lock is *detected*
  on the next run, not silently ignored.)

### Long-lived modes

Unison has three relevant run shapes:

1. **One-shot sync** (default, and the GTK GUI single run). Covered above: a forced Ctrl+C
   skips the terminal-restore / summary handlers and exits; on-disk state is protected by
   the commit log + multi-phase archive commit + `at_exit` lock release.
2. **Repeat / watch mode** (`-repeat`, `-repeat watch`). The repeat loops
   (`synchronizeUntilDone` / `synchronizePathsFromFilesystemWatcher`,
   [`uitext.ml:1434-1508`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/uitext.ml#L1434-L1509))
   have **no per-iteration `try … with`**; they loop in process but a `Sys.Break` from any
   iteration propagates up to the single top-level `with Sys.Break -> terminate ()`
   ([`uitext.ml:1734`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/uitext.ml#L1733-L1734)).
   The "restart on any error" arm
   ([`uitext.ml:1739-1747`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/uitext.ml#L1739-L1747))
   is **disabled by default** (`noRepeat = true || …`,
   [`uitext.ml:1700-1705`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/uitext.ml#L1699-L1705))
   and in any case explicitly excludes `Sys.Break` (`breakRepeat`,
   [`uitext.ml:1752-1762`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/uitext.ml#L1752-L1762)).
   So **Ctrl+C is meant to *stop* repeat/watch mode, not be recovered from** — the only loss
   under the new model is that the top-level `terminate ()` (terminal restore + clean exit
   message) stops firing, i.e. the same one design call as one-shot mode.
3. **Persistent socket server** (`unison -socket <port>`,
   [`remote.ml:2300-2330`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/remote.ml#L2300-L2330)).
   This is genuinely long-lived and *does* catch `Sys.Break` to clean up:
   `with Sys.Break -> unixSocketCleanup ()` removes the listening Unix-domain socket file.
   Under the new model that handler stops firing, so an asynchronous `Sys.Break` leaves the
   **socket file on disk**, which can block the next server start. This is the one
   long-lived-mode site where a skipped clean-up has an observable on-disk effect.

## Clean-up that could be skipped

Verdicts assume the genuine asynchronous exception is a forced `Sys.Break` (3rd+ Ctrl+C) or
`Stack_overflow`.

| Site | Cleanup | Verdict | Why |
|------|---------|---------|-----|
| [`util.ml:226-244`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/ubase/util.ml#L226-L244) `Util.unwindProtect` / `Util.finalize` | `with Transient _ -> cleanup; raise e` | not affected | Catches **only** `Util.Transient`. A `Sys.Break`/`Stack_overflow` already skips these today, so behaviour is unchanged by the new model. All the archive/file/merge clean-up built on `finalize` ([`files.ml:323`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/files.ml#L323), [`files.ml:1101`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/files.ml#L1099-L1101)) inherits this. On-disk consistency is instead carried by the commit log and multi-phase archive commit. |
| [`remote.ml:683-697`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/remote.ml#L681-L699) `Remote.Thread.unwindProtect` | `with Transient/Fatal -> cleanup; reraise` (Lwt) | not affected | Catches only `Transient`/`Fatal`, and is Lwt-based — a `Sys.Break` does not travel this path. The per-item transport clean-up ([`transport.ml:283`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/transport.ml#L279-L309)) and archive-lock release ([`update.ml:2519`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/update.ml#L2519), [`:2630`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/update.ml#L2630)) inherit this; the lock is additionally backstopped by `at_exit`. |
| [`copy.ml:24-35`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/copy.ml#L24-L36) `Copy.protect` / `lwt_protect` | `with e -> close fds / delete temp; raise e` (catches everything) | safe today, only by luck (fragile) | The one combinator that *does* catch `Sys.Break` today. Used in `copyContents` ([`copy.ml:467-486`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/copy.ml#L460-L486)) etc. to close transfer fds and drop a partial temp. A forced `Sys.Break` would skip it, but the fds are independently closed by `Remote.resourceWithConnCleanup` ([`copy.ml:261-262`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/copy.ml#L253-L262)) and the temp file has a distinctive ignored `.unison.` name ([`os.ml:43-44`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/os.ml#L43-L44)), harmless to leave. No on-disk corruption; at worst a leftover temp file the next run ignores. |
| [`remote.ml:2320-2329`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/remote.ml#L2310-L2330) socket server | `with Sys.Break -> unixSocketCleanup ()` (delete socket file) | safe today, only by luck (long-lived) | In the persistent `-socket` server, this removes the listening socket file on Ctrl+C. Under the new model it stops firing, so the socket file is left on disk and may block the next server start. The only long-lived-mode site with an observable on-disk effect. |
| [`uitext.ml:937-959`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/uitext.ml#L937-L959) `stopAtIntr` | `with e -> restoreSig (); raise e` (re-install prev SIGINT/SIGTERM) | benign (process is exiting) | Restores the previous signal handlers after the transfer phase. A forced `Sys.Break` skips it, but it only matters if the program keeps running — and a forced Ctrl+C is exactly the case where it does not (it propagates to `terminate ()` → `exit`). No observable effect. |
| [`uitext.ml:1080-1098`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/uitext.ml#L1080-L1098) `doTransp` summary | `with e -> log "interrupted"; reraise` | benign (diagnostic only) | Prints "Synchronization interrupted by user request" then re-raises. Skipping it loses only the log line; no resource or on-disk effect. |
| [`util.ml:280-291`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/ubase/util.ml#L280-L291) `Util.blockSignals` | `with e -> restoreMask (); raise e` | benign (process is exiting) | Restores the signal mask around the brief ssh-spawn / sigusr2-setup critical sections. A skipped restore in a dying process is moot; both uses are short and not on the steady-state path. |

## Why this is subtle

1. **Which handler catches a forced Ctrl+C today is unpredictable.** The forced `Sys.Break`
   fires at some arbitrary safe point, so whichever matching `with Sys.Break` / catch-all
   handler happens to be innermost at that instant catches it. The handlers are written
   defensively precisely because Ctrl+C can land anywhere.
2. **Graceful vs forced Ctrl+C is the key split, and only the forced path is asynchronous.**
   The first two presses are cooperative (`Abort.all ()` → checkpoint → `Util.Transient`,
   the normal path), and remain so. Only the third+ press raises an asynchronous `Sys.Break`
   from the handler.
3. **The handlers form a clean-up-then-decide chain.** A forced Ctrl+C deep in a transfer
   runs, innermost-out:
   ```
   Sys.with_async_exns wrapper                              <- to be added
     uitext.ml:1734  with Sys.Break -> terminate ()         <- restore terminal + clean exit  (decide)
       uitext.ml:1098  doTransp summary (log "interrupted") <- diagnostic, then re-raise
         uitext.ml:954  stopAtIntr (restore signal handlers)<- re-install handlers, then re-raise
           Copy.protect (close fds / delete temp)           <- cleanup, then re-raise (the one bracket that catches Break)
   ```
   The `Util.finalize` / `Remote.Thread.unwindProtect` brackets are **not** in this chain for
   `Sys.Break`: they only catch `Util.Transient`/`Util.Fatal`, so they are transparent to it
   today and under the new model alike.

The change splits in two:

- **Independent of where the wrapper goes:** the `Copy.protect`, `stopAtIntr`, summary, and
  `blockSignals` frames between the forced Break and the wrapper are skipped no matter
  where the wrapper sits. Per the table, none of these is an on-disk-consistency bug — they
  are fd/temp/handler/mask housekeeping that is either independently backstopped or moot in
  a dying process.
- **Depends on where the wrapper goes:** which top-level `Sys.Break` decision still runs —
  the text UI's `terminate ()` (terminal restore + exit message) and the socket server's
  `unixSocketCleanup ()`. Placing a `Sys.with_async_exns` at each top level restores them.

## How to fix it

The clean-up that protects on-disk state (commit log, multi-phase archive commit, `at_exit`
lock release) is independent of exception handling and needs no change. The work is on the
handful of sites that catch `Sys.Break` to *decide what to do next*, plus the one bracket
that catches everything.

| Site kind | Sites | Fix | Difficulty |
|-----------|-------|-----|------------|
| Resource clean-up that catches everything — only needs "the finaliser must run" | `Copy.protect` / `lwt_protect` ([`copy.ml:24`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/copy.ml#L24-L36)) | rebuild as an interruption-safe bracket (`new_protect`) that runs the clean-up then re-raises the interruption unchanged — callers (`copyContents`, the rsync paths) inherit it | mechanical, one definition |
| Catches `Sys.Break` to decide what to do next | text UI top level ([`uitext.ml:1734`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/uitext.ml#L1733-L1734)); socket server ([`remote.ml:2325`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/remote.ml#L2320-L2329)); and, if their messages are worth keeping, the summary ([`uitext.ml:1088`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/uitext.ml#L1083-L1098)) and `stopAtIntr` ([`uitext.ml:955`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/uitext.ml#L954-L958)) | wrap the relevant region in `Sys.with_async_exns` so the forced `Sys.Break` becomes an ordinary one and the existing handler fires | design call (where to put each wrapper) |

`new_protect`'s shape: `new_protect : init:(unit -> 'a) -> body:('a -> 'b) ->
finaliser:('a -> unit) -> 'b`, which runs the finaliser even when an asynchronous exception
unwinds, then re-raises it.

**Caveat — how the bracket re-raises.** A `new_protect` for `Copy.protect` should re-raise
the *asynchronous* exception unchanged (free the fd / drop the temp, keep the Break flying);
you then add an explicit `Sys.with_async_exns` at the top level to get the clean
`terminate ()`. A version that turned the interruption back into an *ordinary* exception
would re-enable Unison's downstream `with Sys.Break` / catch-all arms swallowing a future
Ctrl+C — against the model. Best: interruption-safe `Copy.protect` + one explicit
`with_async_exns` per top level.

## Resolved during this pass

- **Forced Ctrl+C is the only asynchronous `Sys.Break`; graceful Ctrl+C is cooperative.**
  Confirmed by reading `ctrlCHandler`/`sigtermHandler`
  ([`uitext.ml:919-936`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/uitext.ml#L919-L936)):
  presses 1–2 call `Abort.all ()` (flag → checkpoint → `Util.Transient`, normal path); only
  press 3+ raises `Sys.Break` from the handler. The single most important fact for Unison.
- **Throwing side needs no change** — OxCaml already makes a `Sys.Break` escaping a signal
  handler asynchronous, and Unison both uses `Sys.catch_break true` and its own handler that
  raises `Sys.Break`.
- **The clean-up combinators ignore `Sys.Break` by design** — `Util.unwindProtect`/
  `Util.finalize` catch only `Util.Transient`
  ([`util.ml:226-244`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/ubase/util.ml#L226-L244));
  `Remote.Thread.unwindProtect` only `Transient`/`Fatal` and is Lwt-based. They never ran on
  `Sys.Break`, so the new model leaves them unchanged. `Copy.protect` is the lone bracket
  that catches everything.
- **On-disk consistency does not rely on catching the interruption** — verified the
  `DANGER.README` commit log ([`files.ml:30-86`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/files.ml#L30-L86),
  [`:313-342`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/files.ml#L313-L342))
  and the write-scratch-then-flip archive commit
  ([`update.ml:2626-2660`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/update.ml#L2626-L2660)).
  Both are crash-safe by file-on-disk design, surviving any interruption (even `kill -9`).
- **`at_exit` runs on uncaught `Sys.Break`** — verified by experiment on stock OCaml 5.x
  (the `at_exit` callback fired; exit code 2). So `releaseHeldLocks`
  ([`update.ml:950`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/update.ml#L946-L950))
  releases archive locks even on an uncaught forced Ctrl+C *if OxCaml's asynchronous-uncaught
  path also runs `at_exit`* (one thing to confirm on the runtime).
- **No timers, no GC finalisers in the core, no memprof** — no `setitimer`/`Unix.alarm`;
  the only `Gc.finalise` is in the Solaris fsmonitor helper, not the synced core; the other
  signal handlers do not raise.
- **Repeat/watch mode is meant to stop on Ctrl+C, not recover** — the restart arm is
  disabled by default and excludes `Sys.Break` (`breakRepeat`,
  [`uitext.ml:1752-1762`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/uitext.ml#L1752-L1762)).

## Open questions (design calls / runtime confirmations)

- **Where to put the `Sys.with_async_exns` wrappers (the design call).** At minimum one each
  at the text-UI top level (around `synchronizeUntilDone ()` / the `start2` body,
  [`uitext.ml:1710-1748`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/uitext.ml#L1710-L1748))
  so the existing `with Sys.Break -> terminate ()` fires (terminal restore + clean exit), and
  one around the socket-server accept loop
  ([`remote.ml:2320-2329`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/remote.ml#L2310-L2330))
  so `unixSocketCleanup ()` removes the socket file. Optionally lower wrappers if the
  "interrupted" summary line and signal-handler restore are wanted.
- **OxCaml confirmation** (needs the runtime, not the source): does `at_exit` run on
  asynchronous-uncaught termination (so `releaseHeldLocks` and the log flush survive)?

## Checked and fine

- **`Stack_overflow`** — listed as asynchronous by the doc; Unison is deeply recursive
  (archive trees, reconciliation), but has **no** catch sites. Uncaught/fatal today, stays
  fatal. On its way out it skips the same `Copy.protect`/handler frames as a forced Break —
  same "safe today only by luck" analysis, minus any recovery path; on-disk state is still
  protected by the commit log / archive commit / `at_exit`.
- **`Out_of_memory`** — appears only in a *pattern match on a stored value* (`breakRepeat`,
  [`uitext.ml:1759`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/uitext.ml#L1752-L1762)),
  not a live catch. The doc's "OOM from the GC becomes fatal" breaks no recovery path here.
- **GTK GUI** ([`uigtk3.ml`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/uigtk3.ml))
  — uses **no `Sys.Break` / `catch_break` / signal handlers**; interruption is the Cancel
  button calling `Abort.all ()`
  ([`uigtk3.ml:4256`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/uigtk3.ml#L4256)),
  i.e. cooperative cancellation on the normal path. Its `Sys.Break` references
  ([`uicommon.ml:390`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/uicommon.ml#L388-L394))
  are only for formatting an error string. Out of scope for the change.
- **`SIGUSR1` log-rotate handler** ([`trace.ml:130-143`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/ubase/trace.ml#L130-L144))
  — closes the log channel; does **not** raise.
- **`SIGUSR2` safe-stop handler** ([`uitext.ml:1359-1373`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/uitext.ml#L1359-L1373))
  — sets a flag and closes a pipe end to wake an Lwt sleep; does **not** raise. (This is the
  graceful-shutdown-between-syncs mechanism, the cooperative analogue of the new model.)
- **`SIGPIPE`** ([`remote.ml:33`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/remote.ml#L31-L34))
  — set to `Signal_ignore`; no handler runs.
- **Ordinary `raise Sys.Break` from menus / raw-input / repeat-stop**
  ([`uitext.ml:188`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/uitext.ml#L180-L189),
  [`:807`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/uitext.ml#L804-L807),
  [`:1147`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/uitext.ml#L1147),
  [`:1236`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/uitext.ml#L1233-L1236),
  [`:1620`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/uitext.ml#L1618-L1620))
  — these are synchronous raises from ordinary code, caught normally; **unaffected** by the
  change. (They do share the top-level catch sites, which is why the wrapper there still
  needs to admit ordinary `Sys.Break` too — `Sys.with_async_exns` does, it only *adds* the
  asynchronous ones.)
- **`fsmonitor` Solaris `Gc.finalise`** ([`watcher.ml:85`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/fsmonitor/solaris/watcher.ml#L85))
  — in the optional file-watcher helper, frees a C event object; not in the synced core, and
  a `free` finaliser does not raise.

## Recommendations

In priority order. The throwing side already works (`Sys.catch_break` / the custom handler's
`raise Sys.Break` already become asynchronous on OxCaml), and on-disk consistency does not
depend on catching the interruption, so the work is small.

1. **Add a `Sys.with_async_exns` wrapper at each top level (the one design call).**
   - Text UI: wrap the `start2` body / `synchronizeUntilDone ()`
     ([`uitext.ml:1710-1748`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/uitext.ml#L1710-L1748))
     so the existing `with Sys.Break -> terminate ()` runs (terminal restore + clean exit
     message), in one-shot and repeat/watch alike.
   - Socket server: wrap the accept loop
     ([`remote.ml:2320-2329`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/remote.ml#L2310-L2330))
     so `unixSocketCleanup ()` removes the listening socket file on Ctrl+C — the one
     long-lived-mode site with an observable on-disk effect.

2. **Make `Copy.protect`/`lwt_protect` interruption-safe**
   ([`copy.ml:24-35`](https://github.com/bcpierce00/unison/blob/91421d0617b0fb543c0eee51bcb4d4791d8b0631/src/copy.ml#L24-L36))
   — rebuild as a `new_protect`-style bracket that closes the fd / drops the partial temp
   even on an asynchronous exception, re-raising it unchanged. Low stakes (the fds are
   independently cleaned at connection close and the temp is harmless), but good hygiene and
   the one bracket whose "luck" runs out if a second asynchronous exception enters scope.

3. **Nothing needed for the clean-up combinators or on-disk consistency** —
   `Util.unwindProtect`/`Util.finalize`/`Remote.Thread.unwindProtect` already only run on
   `Util.Transient`/`Util.Fatal`, and the commit log + multi-phase archive commit + `at_exit`
   lock release carry crash-safety independently of exceptions.

4. **Confirm on the OxCaml runtime** (not derivable from source): does `at_exit` run on
   asynchronous-uncaught termination (so `releaseHeldLocks` and the log flush survive)?

**Sequencing note:** (1) restores today's graceful-interrupt behaviour and is the substantive
change; (2) is independent hygiene; (3) is a no-op (already correct); (4) gates how much (1)
buys you. **Do *not*** make `Copy.protect` turn the interruption back into an ordinary
exception — Unison has downstream `with Sys.Break` / catch-all arms that would then silently
re-enable Ctrl+C swallowing, against the model.

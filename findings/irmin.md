# Irmin — async-exceptions initial triage

Repo: `mirage/irmin` @ `7a09a06fff67bc4981faca36a332c51fc16e819e` (shallow clone in `./irmin`).

This is a first pass looking for code in Irmin that works correctly today but breaks
under OxCaml's new rules for "asynchronous exceptions". Irmin is a Git-like content
store: a library plus an on-disk backend (`irmin-pack`), a Git backend, a CLI, and an
optional network server.

## What an asynchronous exception is

A few kinds of event can interrupt an OCaml program at almost any moment: running out
of stack space, pressing Ctrl+C, or an exception thrown from a signal handler, a GC
finaliser, or a memprof callback. Today these surface as ordinary exceptions, so a
normal `try … with` catches them. OxCaml changes this: these asynchronous exceptions
now travel a separate route, an ordinary `try … with` no longer catches them, and
unless the program wraps the right region in a new special handler
(`Sys.with_async_exns`, which turns an asynchronous exception back into an ordinary one
so normal handlers see it again) the exception flies straight past every normal handler
and either reaches that wrapper or stops the program.

## Bottom line

- **Irmin installs nothing that produces an asynchronous exception of its own.** There
  is no `Sys.catch_break`, no `Sys.set_signal`/`Sys.signal` with an OCaml handler that
  raises, no `Stdlib.Gc.finalise`, no memprof callback, and no `setitimer`/`Unix.alarm`
  anywhere in Irmin's own code. So Irmin produces **no `Sys.Break`** and raises nothing
  from a finaliser or signal handler.
- **The only asynchronous exception that can actually reach Irmin's code is
  `Stack_overflow`** (plus `Out_of_memory` only in its synchronous, non-GC form — the
  design doc makes GC out-of-memory fatal). Irmin walks deep, possibly cyclic object
  graphs (commit history, trees, lowest-common-ancestor search), so `Stack_overflow`
  is realistic.
- **Throwing side: nothing to do.** Irmin raises no exception from any signal handler,
  finaliser, or memprof callback, so there is nothing to convert to the new throwing
  primitive (`Sys.raise_async`). And because there is no `Sys.Break` in Irmin, the
  "`Sys.catch_break` comes for free" rule is simply not engaged here. Any Ctrl+C
  handling belongs to the application that embeds Irmin, not to Irmin.
- **There are zero places that catch `Stack_overflow` or `Out_of_memory`.** Every
  error path in Irmin is written to catch *specific, named, synchronous* exceptions
  (`Unix.Unix_error`, `Sys_error`, `Errors.Pack_error`, `Irmin.Closed`,
  `Irmin_pack.RO_not_allowed`) and convert them into result values, or it re-raises
  anything else. So `Stack_overflow` is uncaught and fatal **today**, and stays fatal
  under OxCaml — no recovery path regresses in the core library or the on-disk backend.
- **The on-disk backend is crash-consistent by design, and that design protects it
  here too.** `irmin-pack` commits a write by writing data first and then atomically
  swapping a small *control file* into place with a rename
  ([`control_file.ml:336-348`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/irmin-pack/unix/control_file.ml#L336-L348),
  [`io.ml:266-270`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/irmin-pack/unix/io.ml#L266-L270)).
  The control file is the single commit point; until the rename lands, the store still
  reads as the previous consistent state. An interruption part-way through a write
  therefore loses at most the latest, not-yet-committed data — it does **not** corrupt
  the store. This is the same guarantee the backend already gives against a power-cut,
  so an asynchronous `Stack_overflow` mid-write is no worse.
- **The clean-up that an asynchronous `Stack_overflow` would skip lives behind a small
  number of brackets, and on inspection each is either in-memory-only (moot once the
  process dies) or backstopped.** The on-disk backend funnels resource clean-up through
  two helpers, `Errors.finalise` and `Errors.finalise_exn`
  ([`errors.ml:22-35`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/irmin-pack/unix/errors.ml#L22-L35)),
  and the write path through `batch`
  ([`store.ml:404-444`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/irmin-pack/unix/store.ml#L404-L444)).
  These are the high-leverage points to harden (details below), but none is a
  day-one on-disk bug for a one-shot program.
- **The long-lived network server (`irmin-server`) is *not* a confirmed day-one change —
  it depends on Lwt's exception filter, and on current Lwt there is no change.** Its
  per-request loop wraps each request in `Lwt.catch` and would, in principle, recover from
  a failed request and keep serving
  ([`server.ml:99-135`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/irmin-server/unix/server.ml#L99-L135)).
  But `Lwt.catch` only catches an exception that passes Lwt's `Exception_filter`, and the
  **default filter (`handle_all_except_runtime`, verified in the Lwt source) excludes
  `Stack_overflow`** — so on current Lwt the server **already does not recover** from a
  stack-overflowing request, and the new model changes nothing. It would only be a genuine
  change on a Lwt whose default filter catches `Stack_overflow` (older Lwt, or a host that
  sets `handle_all`). Confirm `irmin-server`'s pinned Lwt before treating this as real
  (see "The network server" below). Either way the per-request clean-up is fragile and
  worth hardening.
- **Everything else checked** (`Out_of_memory`, the store's own `Gc` module — which is
  not OCaml's `Gc.finalise`, the `Lwt_unix.on_signal` handlers in the server, the
  `Lwt.finalize` sites in the core and CLI, the `at_exit` GC-process killer, the
  `libirmin` C-binding wildcard catches, the `irmin-git` backend, the JavaScript
  client) is clear or low-severity.

## Which exceptions are affected

### Irmin produces no asynchronous exception of its own

A search of the whole tree for the usual sources finds none in Irmin's own code:

- **No `Sys.catch_break`, no OCaml signal handler that raises.** The only signal
  handling anywhere is in the network server, where
  [`server.ml:331-339`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/irmin-server/unix/server.ml#L331-L339)
  registers SIGINT and SIGTERM handlers with `Lwt_unix.on_signal`. Those handlers run
  **on the ordinary Lwt event-loop path** (Lwt turns the signal into an event; it does
  not raise from inside the OS signal handler), and the handler bodies merely unlink the
  socket and `exit 0`. So even this is not an exception raised through OCaml frames.
- **No `Stdlib.Gc.finalise`, no memprof, no timers.** The store's own
  `Gc.finalise`/`Gc.finalise_exn` (e.g.
  [`store.ml:281-297`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/irmin-pack/unix/store.ml#L281-L297),
  [`gc.ml:247`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/irmin-pack/unix/gc.ml#L247))
  is Irmin's *store* garbage collector — the operation that reclaims unreachable
  objects from the pack file. It is a normal function on the normal path; it has nothing
  to do with OCaml's `Gc.finalise` callback mechanism. The only use of the real `Gc`
  module is read-only: `gc_stats.ml` calls `Stdlib.Gc.quick_stat` to read counters
  ([`gc_stats.ml:137-140`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/irmin-pack/unix/gc_stats.ml#L137-L140)).
  No finaliser is ever installed.

So the throwing side needs no work: there is no custom exception thrown from a handler
or finaliser to convert to `Sys.raise_async`, and no `Sys.Break` to worry about.

### `Stack_overflow` is the whole story, and it is already uncaught everywhere

Irmin's error handling is built on **result types and narrowly-typed catches**, never
catch-alls that would absorb a runtime exception. The on-disk backend's `catch` helper
turns only its own named exceptions into results and lets everything else through:

```
let catch f =
  try Ok (f ()) with
  | Pack_error e -> Error (e : base_error :> [> t ])
  | RO_not_allowed -> Error `Ro_not_allowed
  | Closed -> Error `Closed
```
([`errors.ml:127-131`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/irmin-pack/unix/errors.ml#L127-L131)).
The GC-launch wrapper does the same and explicitly re-raises the rest:
`| exn -> raise exn`
([`store.ml:597-607`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/irmin-pack/unix/store.ml#L597-L607)).
There is **no** `with Stack_overflow ->`, no `with Out_of_memory ->`, and no bare
`with _ ->` in the core library, the on-disk backend, or the Git backend. A
`Stack_overflow` is therefore uncaught and fatal today; under OxCaml it remains
uncaught and fatal. No recovery behaviour regresses.

### How an asynchronous `Stack_overflow` leaves a write part-way

The write path is `batch`
([`store.ml:404-444`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/irmin-pack/unix/store.ml#L404-L444)).
It sets an in-memory flag `during_batch <- true`, runs the user's function with
`Lwt.try_bind`, and in **both** the success and failure continuations clears the flag
and flushes:

- `on_success` clears `during_batch` and flushes
  ([`store.ml:419-425`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/irmin-pack/unix/store.ml#L419-L425));
- `on_fail` clears `during_batch`, flushes, and re-raises
  ([`store.ml:427-442`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/irmin-pack/unix/store.ml#L427-L442)).

`Lwt.try_bind` keys `on_fail` off the body *resolving to a failed promise* on the
normal path. An asynchronous `Stack_overflow` raised inside the body does not become a
failed promise — it bypasses `on_fail`, so `during_batch` is left `true` and the trailing
flush is skipped. But `during_batch` is purely in-memory state, and a skipped flush only
means the latest buffered data is not written. Because the on-disk commit point is the
control-file rename that the flush would have performed, **the store on disk stays at
its last consistent state** — exactly as if the machine had lost power at the same
instant. Nothing is corrupted; at most the most recent un-committed writes are lost.

## Clean-up that could be skipped

All the sites below run clean-up via an `Errors.finalise`/`finalise_exn`/`Lwt.try_bind`/
`Lwt.finalize` arm that an asynchronous `Stack_overflow` would bypass. Verdicts assume a
one-shot program (CLI command or library call inside a host app); the long-lived server
is treated separately below.

| Site | Cleanup | Verdict | Why |
|------|---------|---------|-----|
| [`store.ml:404-444`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/irmin-pack/unix/store.ml#L404-L444) `batch` | `Lwt.try_bind` whose `on_fail` clears `during_batch` and flushes | safe today, only by luck | An asynchronous `Stack_overflow` in the body skips `on_fail`, leaving the in-memory `during_batch` flag stuck `true` and the final flush un-done. The flag is in-memory only (gone when the process dies); the skipped flush loses only not-yet-committed data and cannot corrupt the store, because the on-disk commit point is the atomic control-file rename. No on-disk damage. Becomes a real concern only if the same repository handle is reused after recovering from the overflow (see the server, below). |
| [`errors.ml:22-25`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/irmin-pack/unix/errors.ml#L22-L25) `Errors.finalise` | runs `finaliser res` after the body returns | safe today, only by luck | This is the success-only finaliser used to close short-lived read handles, e.g. the legacy-file readers `read_offset_from_legacy_file` / `read_version_from_legacy_file` ([`file_manager.ml:650-672`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/irmin-pack/unix/file_manager.ml#L650-L672)). It does not even run the finaliser on a *synchronous* exception today (it only wraps the success path), so an asynchronous `Stack_overflow` skipping it changes nothing about the synchronous-failure case. The skipped close leaks one file descriptor, released by the OS at process exit. |
| [`errors.ml:28-35`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/irmin-pack/unix/errors.ml#L28-L35) `Errors.finalise_exn` | `try … finaliser (Some res) with exn -> finaliser None; raise exn` | safe today, only by luck → fragile | The proper clean-up bracket: it *does* run the finaliser on a synchronous exception and re-raise. An asynchronous `Stack_overflow` bypasses the `with exn ->` arm, so the finaliser is skipped. Its callers are inside the forked GC worker ([`gc_worker.ml:229`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/irmin-pack/unix/gc_worker.ml#L229), [`:274`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/irmin-pack/unix/gc_worker.ml#L274), [`:297`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/irmin-pack/unix/gc_worker.ml#L297)) and the GC `finalise_exn` ([`store.ml:281-297`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/irmin-pack/unix/store.ml#L281-L297)). The GC writes its result to its own files and the parent re-derives state from the control file, so a skipped finaliser there means at most a leftover temp file / fd, not store corruption. Worth converting for hygiene; not a day-one bug. |
| [`store.ml:446-455`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/irmin-pack/unix/store.ml#L446-L455) `close` | cancels a running GC, then closes file-manager / branch / in-memory stores in sequence | safe today, only by luck | This is a plain sequence of close steps, not a bracket — if a `Stack_overflow` lands mid-sequence the later closes are skipped, leaking file descriptors. Those are released by the OS at process exit. It would matter only if a host application caught the overflow (it cannot under the new model — there is no catch site) and kept running with a half-closed repo. No on-disk effect: closing does not change the committed state. |
| [`commit.ml:558-564`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/irmin/commit.ml#L558-L564) `Lwt.finalize` in the lowest-common-ancestor search | finaliser only logs elapsed time | low severity (diagnostic only) | The finaliser writes a debug log line; skipping it has no resource or correctness impact. (This is also one of the deep graph walks where a `Stack_overflow` is most plausible.) |
| [`cli.ml:627-633`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/irmin-cli/cli.ml#L620-L635) `Lwt.finalize` | closes the `.dot` output file | safe today, only by luck (CLI, one-shot) | A `Stack_overflow` while writing the `.dot` graph skips `close_out`; the file descriptor is released at process exit and the half-written file is a throwaway export. CLI command, no persistent state. |

## Why this is subtle

1. **The asynchronous exception fires at an arbitrary safe point.** A `Stack_overflow`
   during a deep graph walk (history traversal, tree folding, ancestor search) surfaces
   wherever the stack happens to run out, so it can land in the middle of a write, a
   close, or a GC step. The clean-up brackets above are written defensively precisely
   because the failure can land anywhere.
2. **The brackets form a clean-up-then-(in some cases)-convert chain.** A `Stack_overflow`
   deep inside a `batch` body would, today, run innermost-out:
   ```
   Lwt.try_bind on_fail (store.ml:427)         <- clear during_batch, flush, re-raise
     Errors.finalise_exn (errors.ml:28)        <- run a finaliser, re-raise   (only if one is on the stack)
       store close path (store.ml:446)         <- close fds                   (only if reached on the way out)
   ```
   Under OxCaml every one of these frames is skipped — the exception goes straight past
   them to wherever the host application has a `Sys.with_async_exns` boundary, or it
   stops the program.
3. **The on-disk design absorbs the damage that would otherwise matter.** The reason
   "skip the flush / skip the close" is not a corruption bug is that the durable commit
   point is the atomic control-file rename, written *after* the data. Skipping the
   trailing flush is indistinguishable from a power loss at the same moment, which the
   format already tolerates.
4. **The split that does matter is one-shot vs long-lived.** For a CLI command or a host
   app that lets the overflow terminate the process, the only thing the skipped clean-up
   loses is in-memory flags and OS-reclaimed file descriptors. For the long-lived network
   server, which *catches and recovers* today, the same skipped clean-up accumulates
   across the process lifetime — and the recovery itself stops happening (see below).

## The network server: version-dependent, and *not* a day-one change on current Lwt

`irmin-server` runs a per-connection request loop that wraps each request in
`Lwt.catch` with a final wildcard arm
([`server.ml:99-135`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/irmin-server/unix/server.ml#L99-L135)):

```
Lwt.catch
  (fun () -> … run the command …)
  (function
    | Error.Error s -> … reply with error, continue …
    | End_of_file   -> … client disconnected …
    | exn           -> … log, reply with the error string, continue …)
>>= fun () -> loop repo conn client info
```

It is tempting to say "today a `Stack_overflow` in a request is caught here and the
server keeps serving, and under OxCaml it would bypass the arm and stop the server."
**But whether `Lwt.catch` catches a `Stack_overflow` at all depends on Lwt's exception
filter, and on current Lwt it does not** — so there is most likely no day-one change to
preserve here:

> **What Lwt's filter actually does.** Lwt's `catch`/`try_bind`/`finalize` only catch an
> exception that passes `Lwt.Exception_filter.run`. Reading the Lwt source we have checked
> out (`lwt/src/core/lwt.ml`), the **default filter is `handle_all_except_runtime`, which
> returns `false` for `Stack_overflow` and `Out_of_memory`** (`let v = ref
> handle_all_except_runtime`). (The `lwt.mli` doc comment that calls `handle_all` the
> default is stale — the code says otherwise.) `irmin-server` never calls
> `Lwt.Exception_filter.set`, so on this Lwt its `| exn ->` arm **already does not catch a
> `Stack_overflow`** — the server already does not recover from a stack-overflowing
> request today, so the new model changes nothing about that.

So the verdict here is **version-dependent and must be confirmed against the actual Lwt
`irmin-server` is built against**:
- On Lwt with the `handle_all_except_runtime` default (what we verified): the per-request
  `Lwt.catch` does not catch `Stack_overflow` today, the server already dies on such a
  request, and there is **no day-one behaviour change** — the skipped clean-up is then in
  the same "safe today only by luck / fragile" category as the rest.
- Only on a Lwt whose default filter catches `Stack_overflow` (an older Lwt predating the
  exception filter, or a host that calls `Lwt.Exception_filter.set handle_all`) would the
  server recover today and lose that recovery under OxCaml — and then the skipped
  per-request clean-up would also accumulate across the server's lifetime.

This is exactly the kind of cross-dependency assumption to verify rather than assert: the
"genuine server bug" only exists for a Lwt configuration that, by default, current Lwt is
not. Confirm `irmin-server`'s pinned Lwt and its filter before treating this as real.

To preserve the current behaviour, the request loop needs a `Sys.with_async_exns`
boundary that turns the asynchronous `Stack_overflow` back into an ordinary one *before*
it reaches the `Lwt.catch`, placed around the per-request body so the existing `| exn ->`
arm catches it as today. Where exactly to put it (per request, vs around the whole
connection) is the design call.

(`Out_of_memory` is less of a concern: the design doc makes GC out-of-memory fatal, so
the only `Out_of_memory` left is the synchronous kind — e.g. a bad size to `Bytes.create`
— which stays an ordinary exception and is still caught by the `| exn ->` arm. Only
`Stack_overflow` regresses.)

## How to fix it

| Site kind | Sites | Fix | Difficulty |
|-----------|-------|-----|------------|
| Server's per-request loop — *only if* it recovers from `Stack_overflow` today, which on current Lwt it does **not** (the default filter excludes `Stack_overflow`); applies only on a Lwt that catches it | the per-request body in `irmin-server` ([`server.ml:99-135`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/irmin-server/unix/server.ml#L99-L135)) | *if* recovery is wanted and the Lwt filter catches `Stack_overflow`: a `Sys.with_async_exns` wrapper around the request body so the existing `Lwt.catch` `\| exn ->` arm catches it as today | conditional — verify the Lwt filter first |
| Resource / state clean-up that only needs "the finaliser must run" | `Errors.finalise`/`finalise_exn` ([`errors.ml:22-35`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/irmin-pack/unix/errors.ml#L22-L35)), the `batch` `on_fail` flush/flag-reset ([`store.ml:427-442`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/irmin-pack/unix/store.ml#L427-L442)), the `close` sequence ([`store.ml:446-455`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/irmin-pack/unix/store.ml#L446-L455)) | an interruption-safe clean-up helper (`new_protect`, which runs the finaliser and then re-throws the interruption unchanged) — fix the *helpers*; their callers inherit it | mechanical |

`new_protect` shape: `init:(unit -> 'a) -> body:('a -> 'b) -> finaliser:('a -> unit) -> 'b`.

**Caveat — how the helper re-throws.** A `new_protect` that runs the clean-up and then
re-throws the *asynchronous* exception unchanged frees the resource but keeps the
exception flying past downstream ordinary handlers (you still need the explicit
`Sys.with_async_exns` if you want the server's `Lwt.catch` to convert it to an error
reply). A `new_protect` that instead turns it back into an *ordinary* exception would
silently re-enable every downstream catch to swallow a future asynchronous exception
against the model. So: re-throw the interruption unchanged in the helpers, and convert
to ordinary only at the one explicit `Sys.with_async_exns` in the server loop.

`new_protect`'s `init`/`finaliser` shape also closes acquire→enter races that the current
helpers leave open — e.g. `Errors.finalise` only wraps the success path, so a synchronous
exception between opening the handle and entering the body already leaks the handle today.

## Resolved during this pass

- **Irmin installs no asynchronous-exception source of its own** — confirmed by a
  tree-wide search: no `Sys.catch_break`, no `Sys.set_signal`/`Sys.signal` with a raising
  OCaml handler, no `Stdlib.Gc.finalise`, no `Memprof`, no `setitimer`/`Unix.alarm`. The
  only signal handling is the server's `Lwt_unix.on_signal` SIGINT/SIGTERM hooks
  ([`server.ml:331-339`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/irmin-server/unix/server.ml#L331-L339)),
  which run on the event-loop path and only `exit 0`.
- **The store's `Gc` is not OCaml's `Gc.finalise`** — it is Irmin's pack-file garbage
  collector, a normal function ([`gc.ml:247`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/irmin-pack/unix/gc.ml#L247));
  the only real `Gc` use is the read-only `Stdlib.Gc.quick_stat`
  ([`gc_stats.ml:137-140`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/irmin-pack/unix/gc_stats.ml#L137-L140)).
- **`Stack_overflow`/`Out_of_memory` have zero catch sites** — every error path catches
  only specific named synchronous exceptions and converts them to results, or re-raises
  the rest ([`errors.ml:127-131`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/irmin-pack/unix/errors.ml#L127-L131),
  [`store.ml:597-607`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/irmin-pack/unix/store.ml#L597-L607)).
  So no recovery path regresses in the library or the on-disk backend.
- **The on-disk backend is crash-consistent via an atomic control-file rename**, so a
  skipped flush mid-write loses at most uncommitted data and cannot corrupt the store
  ([`control_file.ml:336-348`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/irmin-pack/unix/control_file.ml#L336-L348),
  flush ordering in [`file_manager.ml:121-188`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/irmin-pack/unix/file_manager.ml#L121-L188)).
- **`at_exit` runs on an uncaught `Stack_overflow`** — verified by experiment on stock
  OCaml 5.x (a program dying from runaway recursion still ran its `at_exit` callback;
  exit code 2). This matters because the GC subprocess killer is registered with
  `at_exit` ([`async.ml:43`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/irmin-pack/unix/async.ml#L43)):
  even if a parent process dies of an uncaught asynchronous exception, the registered
  hook kills the spawned GC child PIDs — *if OxCaml's uncaught path runs `at_exit`* (one
  runtime confirmation).
- **The GC runs in a forked subprocess** ([`async.ml:59-77`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/irmin-pack/unix/async.ml#L59-L77)),
  so a `Stack_overflow` inside the worker dies in the child, not the parent; the child
  uses `Unix._exit` to skip `at_exit`. The parent re-derives state from the control file,
  so a dead worker does not corrupt the store.

## Open questions (design calls / runtime confirmations)

- **Whether to keep the network server's recover-and-continue behaviour, and where to
  put the wrapper.** If `irmin-server` should keep surviving a `Stack_overflow` in one
  request, it needs a `Sys.with_async_exns` boundary around the per-request body
  ([`server.ml:99-135`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/irmin-server/unix/server.ml#L99-L135))
  so the asynchronous `Stack_overflow` is turned back into an ordinary one and the
  existing `Lwt.catch` `| exn ->` arm catches it. This is the one real design decision.
  Coupled with it: if the server keeps running, the per-request clean-up (`batch`,
  `Errors.finalise_exn`, file handles) must also be made interruption-safe so it does not
  accumulate.
- **Dependency on Lwt's exception filter — verify against the pinned Lwt.** Whether the
  server recovers from a `Stack_overflow` today (and therefore whether there is any change
  to preserve) hinges entirely on Lwt's `Exception_filter`. In the Lwt source we checked
  out (`lwt/src/core/lwt.ml`) the **default is `handle_all_except_runtime`, which excludes
  `Stack_overflow`/`Out_of_memory`** — so on that Lwt `Lwt.catch` does **not** catch a
  stack overflow, the server already dies on such a request, and there is **no day-one
  change** (recommendation 1 is moot). The server would only have recover-and-continue
  behaviour to lose on a Lwt whose default catches `Stack_overflow` — an older Lwt
  predating the filter, or a host that calls `Lwt.Exception_filter.set handle_all`. Action:
  check the exact Lwt `irmin-server` is built against and its filter setting before
  treating the server case as a real change. (Note: an earlier draft of this report
  asserted the default was `handle_all`; that was wrong — corrected against the source.)
- **OxCaml confirmation** (needs the runtime, not the source): (1) does `at_exit` run on
  asynchronous-uncaught termination, so the GC-process killer in
  [`async.ml:43`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/irmin-pack/unix/async.ml#L43)
  still fires; (2) whether a `Stack_overflow` inside a `Lwt_preemptive`/threaded context
  used by a host application kills just that thread or the whole process under the new
  model.

## Checked and fine

- **`Stack_overflow`** — the one realistic asynchronous exception (deep graph walks).
  Zero catch sites in the core library and on-disk backend, so it is already uncaught and
  fatal today and stays that way; no recovery regresses there. On its way out it skips the
  clean-up brackets in the table above, but for a one-shot program those are in-memory
  flags and OS-reclaimed file descriptors, and the on-disk store stays consistent. The
  only recover-path that regresses is the network server (handled above).
- **`Out_of_memory`** — the design doc makes GC out-of-memory fatal, so only the
  synchronous kind remains; Irmin has zero catch sites for it, so nothing regresses, and
  the server's `| exn ->` arm still catches the synchronous kind.
- **Signals** — the only handlers are the server's `Lwt_unix.on_signal` SIGINT/SIGTERM
  hooks ([`server.ml:331-339`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/irmin-server/unix/server.ml#L331-L339)),
  which run on the event-loop path and only unlink the socket and `exit 0`. They do not
  raise through OCaml frames. No `Sys.Break`, no `Sys.catch_break`.
- **`Gc.finalise` / memprof / timers** — none of OCaml's. The store's `Gc` is its own
  pack-file collector; the only `Stdlib.Gc` use is read-only stat reading.
- **The on-disk consistency primitives** — atomic control-file rename
  ([`control_file.ml:336-348`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/irmin-pack/unix/control_file.ml#L336-L348)),
  data-before-control flush ordering
  ([`file_manager.ml:121-188`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/irmin-pack/unix/file_manager.ml#L121-L188)),
  and `Io.close` marking the handle closed even if `Unix.close` fails
  ([`io.ml:148-158`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/irmin-pack/unix/io.ml#L148-L158)).
  A skipped flush mid-write is equivalent to a power loss and is already tolerated.
- **`io.ml:103` `with _ -> `No_such_file_or_directory`** — wraps only `Unix.stat` inside
  `classify_path` ([`io.ml:96-103`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/irmin-pack/unix/io.ml#L96-L103)).
  A `Stack_overflow` during a single `stat` syscall is implausible; under the new model it
  would correctly propagate instead of being mis-classified as a missing file. No effect
  in practice.
- **`libirmin` C-binding wildcard catches** ([`config.ml:37`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/libirmin/config.ml#L37)
  and the `with _ -> null config` / `with _ -> false` arms at
  [`:49`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/libirmin/config.ml#L49),
  [`:58`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/libirmin/config.ml#L58),
  etc.) — these turn any failure during config parsing into a null/false return for C
  callers. A `Stack_overflow` there is implausible (small string/option work); today it
  would be swallowed and a null returned to C, under the new model it would propagate. If
  anything this is *safer*. Low severity, but worth a note since `libirmin` is a public C
  ABI and these are the only catch-alls that would change a returned value.
- **`Lwt.finalize` in the core / CLI** — `commit.ml:558` (timing log only),
  `cli.ml:627` (closes a throwaway `.dot` file). Diagnostic / one-shot; no persistent
  impact.
- **`irmin-git` backend** — no signal handlers, no finalisers, no `Stack_overflow`/
  `Out_of_memory` catches, no catch-alls. Inherits the same `Stack_overflow`-only picture
  as the core; its on-disk safety is delegated to the underlying Git library.
- **The JavaScript client** (`irmin-client/jsoo`) — has its own `Timeout` exception
  ([`irmin_client_jsoo.ml:160`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/irmin-client/jsoo/irmin_client_jsoo.ml#L160)),
  but a JavaScript build has no Unix signals and `Stack_overflow` is the only runtime
  concern; the `Timeout` value is matched on the normal path, not raised from a handler.
  Out of scope.
- **The various `Timeout` exceptions** (`conn_intf.ml:29`, the server/client `IO.ml`
  aliases of `Lwt_unix.Timeout`) — these are raised on the normal Lwt path by the
  ordinary timeout machinery, not from a signal handler or the runtime. They are ordinary
  cooperative exceptions and are unaffected by the change.

## Recommendations

In priority order. Irmin's signal discipline already matches the new model (it raises no
asynchronous exception of its own), and its error handling already lets the runtime
exceptions through everywhere, so there is no confirmed day-one change and the substantive
work is small.

1. **First, settle whether the network server even has behaviour to preserve — verify its
   Lwt's exception filter.** On the Lwt we checked (default `handle_all_except_runtime`,
   which excludes `Stack_overflow`), `irmin-server`'s per-request `Lwt.catch`
   ([`server.ml:99-135`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/irmin-server/unix/server.ml#L99-L135))
   **already does not recover** from a stack-overflowing request, so there is nothing to
   preserve and no wrapper is needed. *Only if* `irmin-server` is built against a Lwt whose
   default catches `Stack_overflow` (or a host sets `handle_all`) does it recover today —
   in which case decide whether to keep that, and if so wrap the per-request body in
   `Sys.with_async_exns` so the existing `| exn ->` arm catches it as today (and make the
   per-request clean-up in (2) interruption-safe so it doesn't accumulate). This is the
   design call, gated on the filter check.

2. **Make the on-disk backend's clean-up helpers interruption-safe.** Reimplement
   `Errors.finalise`/`finalise_exn`
   ([`errors.ml:22-35`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/irmin-pack/unix/errors.ml#L22-L35))
   as interruption-safe `new_protect` helpers (re-throw the asynchronous exception
   unchanged — do **not** turn it into an ordinary one), and similarly harden `batch`'s
   flag-reset/flush ([`store.ml:427-442`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/irmin-pack/unix/store.ml#L427-L442))
   and the `close` sequence ([`store.ml:446-455`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/irmin-pack/unix/store.ml#L446-L455)).
   This buys little for a one-shot program (the on-disk store is consistent regardless and
   fds are reclaimed at exit), but it is the correct hygiene and is *required* if the
   server in (1) is kept alive, so the per-request resources do not accumulate.

3. **Nothing to throw the new way.** Irmin raises no exception from any signal handler,
   finaliser, or memprof callback, so there is no `Sys.raise_async` work, and there is no
   `Sys.Break` to handle (any Ctrl+C handling belongs to the embedding application).

4. **Confirm on the OxCaml runtime** (not derivable from source): (1) `at_exit` runs on
   asynchronous-uncaught termination so the GC-process killer
   ([`async.ml:43`](https://github.com/mirage/irmin/blob/7a09a06fff67bc4981faca36a332c51fc16e819e/src/irmin-pack/unix/async.ml#L43))
   still fires; (2) whether a `Stack_overflow` in a threaded/`Lwt_preemptive` context
   used by a host kills just that thread or the whole process.

**Sequencing note:** (1) is the only change that preserves a currently-observable
behaviour and is the single real decision; (2) is hardening that is optional for one-shot
use but coupled to (1) for the server; (3) is a no-op. **Do *not*** rebuild the clean-up
helpers so they turn the asynchronous exception into an ordinary one: that would re-enable
the server's downstream `| exn ->` arm (and any host catch-alls) to swallow a future
asynchronous exception, against the model — convert to ordinary only at the one explicit
`Sys.with_async_exns` in the server loop.

**A note on what is Irmin's concern vs the host's.** Irmin is a library. It produces no
asynchronous exception itself; the only one that can reach its code is `Stack_overflow`
from its own deep recursion. Whether that `Stack_overflow` is *caught* anywhere, and
whether a Ctrl+C ever becomes a `Sys.Break`, is entirely up to the application that embeds
Irmin. Irmin's job under the new model is narrow: keep its own clean-up brackets working
when an asynchronous exception passes through (recommendation 2), and, for the one
long-lived component it ships itself — the network server — decide whether to keep
recovering (recommendation 1).

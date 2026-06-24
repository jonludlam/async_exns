# Merlin and ocaml-lsp-server — async-exceptions impact analysis

Repos analysed (one report, two tightly-coupled tools — ocaml-lsp builds on
merlin):

- `ocaml/merlin` @ `6b3613715f92ecb466fe0377473201e382368edd` (shallow clone in `./merlin`).
- `ocaml/ocaml-lsp` @ `b90cbe7cdfb096eaf32df92b8db6681c428a5643` (shallow clone in `./ocaml-lsp`).

These are the OCaml editor back-ends: **merlin** answers code-intelligence
queries (type-under-cursor, completion, jump-to-definition, …), and
**ocaml-lsp-server** (the `ocamllsp` binary) wraps merlin as a Language Server
Protocol service for editors. Both run as **long-lived background processes**
that serve many requests and are expected to recover from a failed request and
keep serving — that fact drives most of the analysis below.

An asynchronous exception is an event — running out of stack,
pressing Ctrl+C, a timer firing, the garbage collector calling a finaliser —
that can surface at almost any point in the program. Today these asynchronous
exceptions travel through ordinary `try … with` handlers, so a normal handler
(especially a catch-all `with _ ->`) catches them. Under the new model they
travel a separate route: an ordinary `try … with` no longer catches them. There
is a new wrapper, `Sys.with_async_exns`, that turns one of these asynchronous
exceptions back into an ordinary one inside the region it wraps, so the normal
handlers can see it again; and a new function, `Sys.raise_async`, for code that
*deliberately* throws an asynchronous exception from a signal handler or
finaliser. Below these are called "asynchronous exceptions".

## Bottom line

- **The only asynchronous exception that can actually reach either program is
  "out of stack" (`Stack_overflow`).** Neither tool turns Ctrl+C into an
  exception, neither uses timers/alarms that raise, and no finaliser or signal
  handler of theirs throws. So there is **no `Sys.Break`** and **no custom
  asynchronous exception** in scope — just `Stack_overflow`, which the runtime
  can raise out of the deeply-recursive type-checking and analysis code that
  both tools spend almost all their time in.
- **Nothing to do on the throwing side.** Neither tool raises an exception from
  a signal handler, a finaliser, or a memprof callback, so there is nothing to
  convert to the new `Sys.raise_async` way of throwing. (merlin's one finaliser
  and ocaml-lsp's one signal handler do not raise — see below.)
- **Both already keep signals from becoming exceptions.** merlin ignores SIGPIPE
  (and optionally SIGINT/SIGUSR1/SIGHUP); ocaml-lsp **blocks** SIGCHLD on a
  dedicated thread and consumes it as an event, ignores SIGPIPE, and leaves
  SIGINT/SIGTERM at their default (which simply ends the process, raising
  nothing). This is exactly the discipline the new model wants, and it is why
  the only asynchronous exception that can reach either tool is `Stack_overflow`.
- **Genuine bug — merlin's long-lived `server` mode corrupts itself after one
  `Stack_overflow`.** Each query runs inside a bracket that swaps the OCaml
  compiler's global mutable state in, and on the way out swaps it back and clears
  an "in use" flag, using `Fun.protect ~finally`
  ([`local_store.ml:52-59`](https://github.com/ocaml/merlin/blob/6b3613715f92ecb466fe0377473201e382368edd/src/ocaml/utils/local_store.ml#L52-L59)).
  Today a `Stack_overflow` inside a query runs that `~finally` on the way out, so
  the flag is cleared and the **next** query works. Under the new model the
  asynchronous `Stack_overflow` skips the `~finally`: the "in use" flag stays
  set, and **every later query in that server process** trips the guard
  `if Local_store.is_bound () then failwith "…another instance is already in
  use"`
  ([`mocaml.ml:15-24`](https://github.com/ocaml/merlin/blob/6b3613715f92ecb466fe0377473201e382368edd/src/kernel/mocaml.ml#L15-L24))
  and fails. The server process does not die (its per-request catch-all keeps it
  alive), so the editor sees the merlin backend return an error for everything
  until the server times out and is replaced. **GENUINE BUG (long-lived mode).**
- **Recovery is lost — both tools' per-request catch-all stops catching, so the
  backend dies instead of answering an error.** Both are structured around a
  loop that runs one request, catches *any* exception it raises, turns it into an
  error response, and goes back to wait for the next request:
  - merlin `server`/`single`:
    [`new_merlin.ml:90-150`](https://github.com/ocaml/merlin/blob/6b3613715f92ecb466fe0377473201e382368edd/src/frontend/ocamlmerlin/new/new_merlin.ml#L90-L150)
    catches a failed command and returns it as a JSON `"exception"`/`"error"`
    value; the server's outer
    [`ocamlmerlin_server.ml:10-24`](https://github.com/ocaml/merlin/blob/6b3613715f92ecb466fe0377473201e382368edd/src/frontend/ocamlmerlin/ocamlmerlin_server.ml#L10-L24)
    `| exception exn -> log; close_with (-1)` keeps the loop going.
  - merlin `old-protocol`:
    [`old_merlin.ml:72-92`](https://github.com/ocaml/merlin/blob/6b3613715f92ecb466fe0377473201e382368edd/src/frontend/ocamlmerlin/old/old_merlin.ml#L72-L92)
    `merlin_loop`'s `| exception exn -> output (Exception exn); merlin_loop`
    reports and loops.
  - ocaml-lsp: every request handler runs inside `Fiber.map_reduce_errors
    ~on_error:(turn the exception into a JSON-RPC error response)`
    ([`jsonrpc_fiber.ml:215-255`](https://github.com/ocaml/ocaml-lsp/blob/b90cbe7cdfb096eaf32df92b8db6681c428a5643/jsonrpc-fiber/src/jsonrpc_fiber.ml#L215-L255)),
    and the loop
    ([`jsonrpc_fiber.ml:184-199`](https://github.com/ocaml/ocaml-lsp/blob/b90cbe7cdfb096eaf32df92b8db6681c428a5643/jsonrpc-fiber/src/jsonrpc_fiber.ml#L184-L199))
    serves the next message.
  Today a `Stack_overflow` in a request is caught by these arms, reported, and
  the backend keeps serving. Under the new model the asynchronous
  `Stack_overflow` slips past them and **terminates the whole backend process**,
  losing every other in-flight and future request — a real behaviour change for
  a long-lived server.
- **Safe today only by luck (fragile):** a handful of small clean-up/restore
  brackets in merlin that run a finaliser then re-throw —
  `New_merlin.with_wd`'s `Fun.protect ~finally:(Sys.chdir back)`
  ([`new_merlin.ml:178-186`](https://github.com/ocaml/merlin/blob/6b3613715f92ecb466fe0377473201e382368edd/src/frontend/ocamlmerlin/new/new_merlin.ml#L178-L186)),
  `Mppx.with_include_dir`'s restore-then-reraise
  ([`mppx.ml:5-22`](https://github.com/ocaml/merlin/blob/6b3613715f92ecb466fe0377473201e382368edd/src/kernel/mppx.ml#L5-L22)),
  `Std.file_contents`' close-then-reraise
  ([`std.ml:833-845`](https://github.com/ocaml/merlin/blob/6b3613715f92ecb466fe0377473201e382368edd/src/utils/std.ml#L833-L845)),
  `File_cache.read`' drop-entry-then-reraise
  ([`file_cache.ml:61-76`](https://github.com/ocaml/merlin/blob/6b3613715f92ecb466fe0377473201e382368edd/src/utils/file_cache.ml#L61-L76)).
  In ocaml-lsp the clean-up is funnelled through the shared `Fiber.finalize` /
  `Fiber.Mutex.with_lock` / `Stdune.Exn.protect` helpers (used at
  [`ocamlformat.ml:53-68`](https://github.com/ocaml/ocaml-lsp/blob/b90cbe7cdfb096eaf32df92b8db6681c428a5643/ocaml-lsp-server/src/ocamlformat.ml#L53-L68),
  [`rpc.ml:193-203`](https://github.com/ocaml/ocaml-lsp/blob/b90cbe7cdfb096eaf32df92b8db6681c428a5643/lsp-fiber/src/rpc.ml#L193-L203),
  [`import.ml:24-26`](https://github.com/ocaml/ocaml-lsp/blob/b90cbe7cdfb096eaf32df92b8db6681c428a5643/ocaml-lsp-server/src/import.ml#L24-L26),
  [`fiber_io.ml:40-47`](https://github.com/ocaml/ocaml-lsp/blob/b90cbe7cdfb096eaf32df92b8db6681c428a5643/lsp-fiber/src/fiber_io.ml#L40-L47)).
  An asynchronous `Stack_overflow` skips each of these; in a long-lived process
  the skipped work (restore the working directory / compiler paths, close a file
  descriptor, unlock a fiber mutex, drop a stale cache entry) accumulates.
- **Cooperative cancellation is not affected.** ocaml-lsp's `$/cancelRequest`
  works by *firing a cancel token* the handler polls
  ([`rpc.ml:339-345`](https://github.com/ocaml/ocaml-lsp/blob/b90cbe7cdfb096eaf32df92b8db6681c428a5643/lsp-fiber/src/rpc.ml#L339-L345)),
  and `workspace_symbol.ml`'s `Cancelled` is an ordinary exception raised at a
  cooperative check
  ([`workspace_symbol.ml:212-256`](https://github.com/ocaml/ocaml-lsp/blob/b90cbe7cdfb096eaf32df92b8db6681c428a5643/ocaml-lsp-server/src/workspace_symbol.ml#L212-L256)).
  Neither is an asynchronous exception; both are unchanged by the new model.
- Everything else checked (`Out_of_memory`, `Gc.finalise`, memprof, the SIGPIPE
  ignores, the no-op SIGCHLD handler, the merlin reader subprocess finaliser, the
  vendored compiler's `Sys.Break` print) is clear.

## Which exceptions are affected

### Signals never become exceptions (why only `Stack_overflow` can reach these tools)

**merlin** never installs a signal handler that raises. The server only ignores
SIGPIPE so a disconnecting client cannot kill it
([`ocamlmerlin_server.ml:59`](https://github.com/ocaml/merlin/blob/6b3613715f92ecb466fe0377473201e382368edd/src/frontend/ocamlmerlin/ocamlmerlin_server.ml#L59)).
The `old-protocol` front-end ignores SIGUSR1/SIGPIPE/SIGHUP
([`old_merlin.ml:95-99`](https://github.com/ocaml/merlin/blob/6b3613715f92ecb466fe0377473201e382368edd/src/frontend/ocamlmerlin/old/old_merlin.ml#L95-L99))
and offers a `-ignore-sigint` flag that sets SIGINT to *ignore*
([`old_merlin.ml:36-40`](https://github.com/ocaml/merlin/blob/6b3613715f92ecb466fe0377473201e382368edd/src/frontend/ocamlmerlin/old/old_merlin.ml#L36-L40))
— i.e. SIGINT is either left at its default (ends the process) or ignored;
nothing raises `Sys.Break`. There is **no `Sys.catch_break`** anywhere in
merlin's own code. So Ctrl+C is never delivered as an exception, and there is no
throwing-side change to make for it.

**ocaml-lsp** runs on the lev-fiber event loop, which uses the
"block-and-consume" discipline: it **blocks** SIGCHLD (and an internal stop
signal) with `Unix.sigprocmask SIG_BLOCK` and consumes them on a dedicated
thread via `Thread.wait_signal`, turning each into an event sent to the loop;
the SIGCHLD handler it installs is a no-op, and SIGPIPE is ignored
([`lev_fiber.ml:15-57`](https://github.com/ocaml/ocaml-lsp/blob/b90cbe7cdfb096eaf32df92b8db6681c428a5643/submodules/lev/lev-fiber/src/lev_fiber.ml#L15-L57)).
SIGINT and SIGTERM are left at their default disposition — they end the process
without raising anything. So no signal becomes an exception in ocaml-lsp either.

That leaves **`Stack_overflow`** as the single asynchronous exception either tool
can actually encounter. Both spend essentially all their time in the OCaml
type-checker, parser and analysis passes (deeply recursive over the syntax tree
and type graph), so a stack overflow on a pathological input is the realistic
trigger.

### merlin: the per-query compiler-state bracket (the genuine bug)

merlin isolates each query by giving it a fresh snapshot of the OCaml compiler's
global mutable state. `Mpipeline.with_pipeline`
([`mpipeline.ml:104-106`](https://github.com/ocaml/merlin/blob/6b3613715f92ecb466fe0377473201e382368edd/src/kernel/mpipeline.ml#L104-L106))
calls `Mocaml.with_state`, which first checks an "in use" flag and refuses to run
if it is already set, then runs the query through `Local_store.with_store`
([`mocaml.ml:15-24`](https://github.com/ocaml/merlin/blob/6b3613715f92ecb466fe0377473201e382368edd/src/kernel/mocaml.ml#L15-L24)).
`Local_store.with_store` sets the flag, swaps the snapshot's values into the
global refs, and uses `Fun.protect ~finally:(…)` to swap them back out and clear
the flag on the way out
([`local_store.ml:52-59`](https://github.com/ocaml/merlin/blob/6b3613715f92ecb466fe0377473201e382368edd/src/ocaml/utils/local_store.ml#L52-L59)).

The flag (`global_bindings.is_bound`) is **process-global** and lives across
queries. In merlin's `server` mode one process answers many queries in a loop,
so if an asynchronous `Stack_overflow` skips the `~finally`, the flag is never
cleared and every subsequent query hits the `failwith "…another instance is
already in use"` guard. The server stays up (its catch-all turns the `failwith`
into an error response), but it is now permanently broken for that session.

### Both: the per-request catch-all (recovery that stops working)

Each tool's request loop is built so that one failed request becomes an error
response and the loop continues:

- merlin (new front-end): the command action runs inside `match … with |
  exception exn -> (turn into a JSON error)`
  ([`new_merlin.ml:114-150`](https://github.com/ocaml/merlin/blob/6b3613715f92ecb466fe0377473201e382368edd/src/frontend/ocamlmerlin/new/new_merlin.ml#L114-L150)),
  and the server's `process_client` wraps that in `| exception exn -> log;
  close_with (-1)` before looping
  ([`ocamlmerlin_server.ml:10-52`](https://github.com/ocaml/merlin/blob/6b3613715f92ecb466fe0377473201e382368edd/src/frontend/ocamlmerlin/ocamlmerlin_server.ml#L10-L52)).
- merlin (old front-end): `merlin_loop`'s `| exception exn -> output (Exception
  exn); merlin_loop`
  ([`old_merlin.ml:72-92`](https://github.com/ocaml/merlin/blob/6b3613715f92ecb466fe0377473201e382368edd/src/frontend/ocamlmerlin/old/old_merlin.ml#L72-L92)).
- ocaml-lsp: each request handler runs inside `Fiber.map_reduce_errors
  ~on_error:(make a JSON-RPC error response)`
  ([`jsonrpc_fiber.ml:215-255`](https://github.com/ocaml/ocaml-lsp/blob/b90cbe7cdfb096eaf32df92b8db6681c428a5643/jsonrpc-fiber/src/jsonrpc_fiber.ml#L215-L255));
  notifications similarly through `Fiber.collect_errors`
  ([`jsonrpc_fiber.ml:256-270`](https://github.com/ocaml/ocaml-lsp/blob/b90cbe7cdfb096eaf32df92b8db6681c428a5643/jsonrpc-fiber/src/jsonrpc_fiber.ml#L256-L270)),
  with a per-request error/finalize step in the LSP layer
  ([`rpc.ml:160-205`](https://github.com/ocaml/ocaml-lsp/blob/b90cbe7cdfb096eaf32df92b8db6681c428a5643/lsp-fiber/src/rpc.ml#L160-L205)).
  The fiber error machinery only catches what the fiber scheduler's task wrapper
  caught for it (an ordinary `try … with exn -> …` around each task), which an
  asynchronous `Stack_overflow` bypasses.

These are *code that catches the exception to decide what to do next* (report it,
keep the server alive). Today they catch `Stack_overflow`; under the new model
they do not, so the backend terminates instead of recovering.

### Cooperative cancellation and the `Cancelled` exception (not affected)

ocaml-lsp implements LSP `$/cancelRequest` by firing a `Fiber.Cancel` token that
the running handler observes at cooperative points
([`rpc.ml:339-345`](https://github.com/ocaml/ocaml-lsp/blob/b90cbe7cdfb096eaf32df92b8db6681c428a5643/lsp-fiber/src/rpc.ml#L339-L345),
[`rpc.ml:160-205`](https://github.com/ocaml/ocaml-lsp/blob/b90cbe7cdfb096eaf32df92b8db6681c428a5643/lsp-fiber/src/rpc.ml#L160-L205)).
The `Cancelled` exception in workspace-symbol search is a plain exception raised
when a polled flag is set and caught right there
([`workspace_symbol.ml:212-256`](https://github.com/ocaml/ocaml-lsp/blob/b90cbe7cdfb096eaf32df92b8db6681c428a5643/ocaml-lsp-server/src/workspace_symbol.ml#L212-L256)).
Both are ordinary, deliberate, synchronous mechanisms — they are not asynchronous
exceptions and the new model leaves them exactly as they are.

## Clean-up that could be skipped

All sites below run a finaliser / restore / unlock and then re-throw (or rely on
the fiber error machinery), via an arm that an asynchronous `Stack_overflow`
bypasses. Because both tools are long-lived servers, the "the process is exiting
anyway" reasoning does **not** apply: skipped clean-up accumulates over the
process's lifetime, and a skipped recovery arm kills the backend instead of
answering an error.

| Site | Clean-up | Verdict | Why |
|------|----------|---------|-----|
| **merlin** [`local_store.ml:52-59`](https://github.com/ocaml/merlin/blob/6b3613715f92ecb466fe0377473201e382368edd/src/ocaml/utils/local_store.ml#L52-L59) `with_store` (the per-query state bracket, guarded by [`mocaml.ml:15-24`](https://github.com/ocaml/merlin/blob/6b3613715f92ecb466fe0377473201e382368edd/src/kernel/mocaml.ml#L15-L24)) | `Fun.protect ~finally`: swap compiler global state back out, clear the "in use" flag | **GENUINE BUG (server mode)** | The "in use" flag is process-global and outlives the query. A skipped `~finally` leaves it set, so every later query in the same `server` process fails the `failwith "…already in use"` guard. The server stays up but is broken for the session until it times out and is replaced. On a one-shot `single` invocation the process exits, so it is harmless there — but `server` mode is the normal editor configuration. |
| **merlin** [`new_merlin.ml:178-186`](https://github.com/ocaml/merlin/blob/6b3613715f92ecb466fe0377473201e382368edd/src/frontend/ocamlmerlin/new/new_merlin.ml#L178-L186) `with_wd` | `Fun.protect ~finally:(fun () -> Sys.chdir old_wd)` | safe today, only by luck (fragile) | Restores the process working directory after running a query in the client's directory. A skipped restore leaves the server's working directory pointing at the last client's directory; the next query resets it again at its start, so in practice it is overwritten before being read — but it is a process-global side effect skipped on the asynchronous path. |
| **merlin** [`mppx.ml:5-22`](https://github.com/ocaml/merlin/blob/6b3613715f92ecb466fe0377473201e382368edd/src/kernel/mppx.ml#L5-L22) `with_include_dir` | `try … with e -> restore (); raise e` (restore `Clflags.include_dirs` / `hidden_include_dirs`) | safe today, only by luck (fragile) | Restores two compiler global include-path refs around PPX rewriting. Set fresh at the start of each rewrite, so a skipped restore is overwritten before being read — benign in practice, but a process-global skipped on the asynchronous path. |
| **merlin** [`std.ml:833-845`](https://github.com/ocaml/merlin/blob/6b3613715f92ecb466fe0377473201e382368edd/src/utils/std.ml#L833-L845) `file_contents` | `try … with exn -> close_in_noerr ic; raise exn` | safe today, only by luck (fragile) | A skipped `close_in_noerr` leaks one file descriptor. Over a long-lived server's lifetime, repeated overflows while reading source files could accumulate leaked descriptors. Low severity (a `Stack_overflow` purely inside a file read is unlikely). |
| **merlin** [`file_cache.ml:61-76`](https://github.com/ocaml/merlin/blob/6b3613715f92ecb466fe0377473201e382368edd/src/utils/file_cache.ml#L61-L76) `read` | `try … with exn -> Hashtbl.remove cache filename; raise exn` | safe today, only by luck (fragile) | Drops a half-populated cache entry on failure. A skipped removal can only leave a *stale-but-valid* prior entry (the failing read never inserts), so at most a stale cache read; the cache also re-checks file identity. Low severity. |
| **ocaml-lsp** [`ocamlformat.ml:53-68`](https://github.com/ocaml/ocaml-lsp/blob/b90cbe7cdfb096eaf32df92b8db6681c428a5643/ocaml-lsp-server/src/ocamlformat.ml#L53-L68), [`rpc.ml:193-203`](https://github.com/ocaml/ocaml-lsp/blob/b90cbe7cdfb096eaf32df92b8db6681c428a5643/lsp-fiber/src/rpc.ml#L193-L203), [`import.ml:24-26`](https://github.com/ocaml/ocaml-lsp/blob/b90cbe7cdfb096eaf32df92b8db6681c428a5643/ocaml-lsp-server/src/import.ml#L24-L26), [`fiber_io.ml:40-47`](https://github.com/ocaml/ocaml-lsp/blob/b90cbe7cdfb096eaf32df92b8db6681c428a5643/lsp-fiber/src/fiber_io.ml#L40-L47) — `Fiber.finalize` / `Stdune.Exn.protect` / `Fiber.Mutex.with_lock` | run the finaliser (close fd / pipe, unlock fiber mutex, remove pending entry) then re-raise | safe today, only by luck (fragile) | These funnel essentially all of ocaml-lsp's resource clean-up through the shared fiber/stdune bracket helpers. An asynchronous `Stack_overflow` skips the wrapper's `with exn ->`/`~finally`, so the finaliser does not run — a leaked descriptor/pipe, a held fiber mutex, or a stale pending-request entry. In a long-lived server these accumulate rather than being moot at exit. The fix is in the shared helpers, not these call sites. |
| **ocaml-lsp** [`rpc.ml:160-205`](https://github.com/ocaml/ocaml-lsp/blob/b90cbe7cdfb096eaf32df92b8db6681c428a5643/lsp-fiber/src/rpc.ml#L160-L205) `to_jsonrpc` per-request error/finalize | `Fiber.with_error_handler ~on_error:(remove pending; reraise)` + `Fiber.finalize ~finally:(remove pending)` | safe today, only by luck (fragile) | Removes the request from the pending table when it errors or completes. A skipped removal leaks one pending-request entry per overflow; harmless in isolation but accumulates in a long-lived process and is bypassed on the asynchronous path. |

## Why this is subtle

1. **Today's catching is unpredictable in principle.** An asynchronous
   `Stack_overflow` fires at whatever safe point the stack happens to run out,
   and is caught by whichever catch-all handler is innermost at that instant. The
   request loops are written defensively for exactly this reason — they assume
   any request can blow up anywhere and must be turned into an error without
   taking the server down.

2. **The handlers form a clean-up-then-recover chain.** A `Stack_overflow` deep
   inside a query passes, innermost-out, through:

   ```
   merlin server:
     ocamlmerlin_server.ml:22  | exception exn -> log; close_with (-1)   <- keep the server loop alive (recover)
       new_merlin.ml:114       | exception exn -> JSON "exception"/"error" <- turn into an error response
         local_store.ml:56     Fun.protect ~finally (clear "in use" flag)  <- restore global state  (skipped -> the bug)

   ocaml-lsp:
     jsonrpc_fiber.ml:184      loop ()                                     <- serve the next request (recover)
       jsonrpc_fiber.ml:219    Fiber.map_reduce_errors ~on_error           <- turn into a JSON-RPC error response
         rpc.ml:193            Fiber.finalize ~finally (remove pending)     <- per-request clean-up (skipped)
   ```

   Under the new model the asynchronous `Stack_overflow` skips **all** of these
   frames: no per-request clean-up, no error response, and the backend process
   terminates instead of looping.

The change splits in two:

- **Independent of where any wrapper goes:** every finaliser/restore between the
  overflow and any wrapper is skipped no matter what — the compiler-state flag is
  left set, the pending entry is left in the table, the fd is left open. Whether
  each *matters* is per-site (the table above): the merlin compiler-state flag is
  a genuine session-breaking bug in server mode; the rest are fragile but
  practically overwritten or merely leaked.
- **Dependent on where a wrapper goes:** whether the request loop recovers at
  all. With no `Sys.with_async_exns` in the loop, an asynchronous
  `Stack_overflow` ends the backend; with one per request, the existing
  report-and-continue machinery runs as today.

## How to fix it

The fixes split by what each site needs from the exception.

| Site kind | Sites | Fix | Difficulty |
|-----------|-------|-----|------------|
| Catches the exception to decide what to do next (report it, keep serving) — must keep recovering | merlin [`ocamlmerlin_server.ml:10-52`](https://github.com/ocaml/merlin/blob/6b3613715f92ecb466fe0377473201e382368edd/src/frontend/ocamlmerlin/ocamlmerlin_server.ml#L10-L52) / [`old_merlin.ml:72-92`](https://github.com/ocaml/merlin/blob/6b3613715f92ecb466fe0377473201e382368edd/src/frontend/ocamlmerlin/old/old_merlin.ml#L72-L92); ocaml-lsp request loop [`jsonrpc_fiber.ml:184-255`](https://github.com/ocaml/ocaml-lsp/blob/b90cbe7cdfb096eaf32df92b8db6681c428a5643/jsonrpc-fiber/src/jsonrpc_fiber.ml#L184-L255) | wrap the **per-request** body in `Sys.with_async_exns`, turning an asynchronous `Stack_overflow` back into an ordinary one so the existing report-and-continue arms fire and the backend survives | small, but a real design choice (one boundary per request) |
| Resource / global-state / lock clean-up — only needs "the finaliser must run" | merlin `with_store` (via `Fun.protect`), `with_wd`, `with_include_dir`, `file_contents`, `File_cache.read`; ocaml-lsp's `Fiber.finalize` / `Fiber.Mutex.with_lock` / `Stdune.Exn.protect` helpers | an interruption-safe version of the bracket (`new_protect`) that runs the finaliser and then re-throws the asynchronous exception unchanged — fix the *helpers* and the callers inherit it | mechanical |

For both tools the per-request `Sys.with_async_exns` boundary is the high-leverage
change: it restores today's "one bad request becomes an error, the server keeps
running" behaviour for `Stack_overflow`. Placing it **per request** (not once
around the whole server) is what preserves the existing behaviour — a boundary
that high would convert the overflow back to ordinary but still skip the
per-request clean-up below it, so put it just outside each request's handler.

**Caveat — how the helper re-throws.** An interruption-safe bracket that runs the
finaliser and re-throws the *asynchronous* exception unchanged frees the resource
but keeps the exception travelling the asynchronous path (so you still need the
explicit per-request `Sys.with_async_exns` to recover). A version that instead
turned it back into an *ordinary* exception inside the helper would silently
re-enable every downstream catch-all (`with _ ->`) — of which merlin's analysis
code has many — to swallow a future asynchronous exception, against the model.
Best approach: interruption-safe helpers that re-throw the asynchronous exception
unchanged, plus one explicit `Sys.with_async_exns` per request.

## Resolved during this pass

- **Signals never become exceptions in either tool** — merlin only ignores
  signals (SIGPIPE/SIGUSR1/SIGHUP, optional SIGINT-ignore); ocaml-lsp blocks
  SIGCHLD on a dedicated thread and consumes it as an event, ignores SIGPIPE, and
  leaves SIGINT/SIGTERM at default. Confirms there is no `Sys.Break` and nothing
  to convert on the throwing side.
- **The only asynchronous exception in scope is `Stack_overflow`** — no
  `Sys.catch_break`, no `setitimer`/`Unix.alarm` timeout exception, no custom
  asynchronous exception in either repo's own code.
- **The merlin reader-subprocess finaliser does not raise** — `Mreader_extend`
  registers `Gc.finalise stop_finalise`
  ([`mreader_extend.ml:52`](https://github.com/ocaml/merlin/blob/6b3613715f92ecb466fe0377473201e382368edd/src/kernel/mreader_extend.ml#L52)),
  which calls `Extend_driver.stop` — that only does `close_out_noerr` /
  `close_in_noerr` / `Unix.waitpid`
  ([`extend_driver.ml:40-44`](https://github.com/ocaml/merlin/blob/6b3613715f92ecb466fe0377473201e382368edd/src/extend/extend_driver.ml#L40-L44)),
  none of which raises in normal operation. So the doc's "an exception escaping a
  finaliser terminates the program" rule is inert here.
- **`at_exit` runs on an uncaught `Stack_overflow`** — verified empirically on
  stock OCaml 5.4.1 (a self-recursion overflow still ran its `at_exit` callback;
  exit code 2). Neither tool relies on `at_exit` for request-level clean-up (the
  one `at_exit` in merlin's tree closes an index file in the separate
  `ocaml-index` tool,
  [`granular_marshal.ml:65-70`](https://github.com/ocaml/merlin/blob/6b3613715f92ecb466fe0377473201e382368edd/src/index-format/granular_marshal.ml#L65-L70),
  not the server loop), so this is a backstop only for whole-process exit, not
  for the per-session corruption above.
- **Cooperative cancellation is unaffected** — `$/cancelRequest` fires a cancel
  token, and `workspace_symbol.ml`'s `Cancelled` is a plain synchronous
  exception; neither is asynchronous.

## Still open (design calls / runtime confirmations)

- **Where to put the per-request `Sys.with_async_exns` boundary.** For merlin:
  just inside `process_client` / `merlin_loop`, around the single command
  evaluation, so the existing error-response arm fires and the server keeps
  looping. For ocaml-lsp: around each request/notification handler in the
  jsonrpc-fiber loop
  ([`jsonrpc_fiber.ml:184-270`](https://github.com/ocaml/ocaml-lsp/blob/b90cbe7cdfb096eaf32df92b8db6681c428a5643/jsonrpc-fiber/src/jsonrpc_fiber.ml#L184-L270)),
  so a `Stack_overflow` in one handler becomes that request's error response and
  the loop serves the next message. Placing it too high (once around the whole
  server) would lose the per-request clean-up below it.
- **OxCaml runtime confirmation** (not derivable from source): does a
  `Stack_overflow` that occurs *inside a worker thread* (ocaml-lsp uses a merlin
  worker thread and a signal-watcher thread; merlin's reader runs a subprocess)
  terminate just that thread/domain or the whole process? This affects whether
  the per-request boundary alone is sufficient or whether the worker threads also
  need wrapping.

## Obviously fine (gaps checked and cleared)

- **`Out_of_memory`** — no catch sites in either tool's own code. The doc's
  "out-of-memory from the GC becomes fatal" breaks no recovery path here.
- **`Stack_overflow` dedicated catches** — there are **no** handlers that match
  `Stack_overflow` by name; it is only caught by the catch-all request arms
  above. So there is no specific recovery logic to break — only the generic
  report-and-continue behaviour, analysed above.
- **`Gc.finalise`** — merlin's one finaliser (reader subprocess) does not raise;
  ocaml-lsp registers none in its own code.
- **memprof callbacks** — none in either tool's own code.
- **Signal handlers** — merlin's are all *ignore*; ocaml-lsp's lone installed
  handler (SIGCHLD) is a no-op. None raises.
- **Wildcard `with _ ->` handlers in merlin's analysis code** (completion,
  type-enclosing, locate, type-utils, etc.) — these catch *ordinary* failures of
  a single analysis step and fall back; under the new model an asynchronous
  `Stack_overflow` simply stops being swallowed by them and propagates to the
  per-request boundary, which is the desired direction. They are not clean-up
  sites and need no change.
- **Vendored OCaml compiler `Sys.Break` print** —
  [`oprint.ml:872`](https://github.com/ocaml/merlin/blob/6b3613715f92ecb466fe0377473201e382368edd/src/ocaml/typing/oprint.ml#L872)
  prints "Interrupted." on `Sys.Break`; this is dead with respect to merlin,
  which drives no interactive toplevel that raises `Sys.Break`. Inert.
- **No JavaScript build** — both are native Unix/Windows binaries; there is no
  js_of_ocaml target to consider.

## Recommendations

In priority order. Both tools' signal handling already matches the new model, so
there is no throwing-side work; the substantive work is preserving long-lived
server recovery and the per-query state isolation.

1. **Add a per-request `Sys.with_async_exns` boundary in each request loop**
   (merlin
   [`ocamlmerlin_server.ml:10-52`](https://github.com/ocaml/merlin/blob/6b3613715f92ecb466fe0377473201e382368edd/src/frontend/ocamlmerlin/ocamlmerlin_server.ml#L10-L52)
   / [`old_merlin.ml:72-92`](https://github.com/ocaml/merlin/blob/6b3613715f92ecb466fe0377473201e382368edd/src/frontend/ocamlmerlin/old/old_merlin.ml#L72-L92);
   ocaml-lsp
   [`jsonrpc_fiber.ml:184-270`](https://github.com/ocaml/ocaml-lsp/blob/b90cbe7cdfb096eaf32df92b8db6681c428a5643/jsonrpc-fiber/src/jsonrpc_fiber.ml#L184-L270)),
   so an asynchronous `Stack_overflow` in one request is turned back into an
   ordinary exception, caught by the existing report-and-continue arm, and the
   backend keeps serving instead of dying. This is the one real behaviour change
   for these long-lived servers; pair it with (2)/(3) so per-request resources
   are released on the way out.

2. **Fix the merlin per-query state bracket (the genuine bug).** Make
   `Local_store.with_store`
   ([`local_store.ml:52-59`](https://github.com/ocaml/merlin/blob/6b3613715f92ecb466fe0377473201e382368edd/src/ocaml/utils/local_store.ml#L52-L59))
   run its `~finally` (clear the "in use" flag, swap state back) even when an
   asynchronous `Stack_overflow` unwinds — i.e. an interruption-safe `new_protect`
   in place of `Fun.protect`. Without this, a single overflow permanently breaks
   every later query in a merlin `server` process (the editor's normal mode).

3. **Make the remaining clean-up helpers interruption-safe (hygiene).** merlin's
   `with_wd`, `with_include_dir`, `file_contents`, `File_cache.read`; ocaml-lsp's
   shared `Fiber.finalize` / `Fiber.Mutex.with_lock` / `Stdune.Exn.protect`
   helpers (which cover essentially all of its resource clean-up — fix the helper,
   not the call sites). Have them re-throw the asynchronous exception unchanged,
   **not** convert it to an ordinary one (which would re-enable merlin's many
   downstream `with _ ->` arms to swallow a future asynchronous exception).

4. **Nothing to throw the new way** — neither tool raises an exception from a
   signal handler, finaliser, or memprof callback, so there is no
   `Sys.raise_async` work.

5. **Confirm on the OxCaml runtime**: whether a `Stack_overflow` inside a worker
   thread (ocaml-lsp's merlin worker / signal-watcher threads) terminates just
   that thread or the whole process — this decides whether the per-request
   boundary alone suffices or the worker threads also need wrapping.

**Sequencing note:** (1) and (2) are the substantive fixes and are independent of
each other (one preserves recovery, the other prevents per-session corruption);
(3) is hygiene that also makes the luck explicit; (4) is a no-op confirmation;
(5) gates how much (1) buys for the threaded paths. **Do not** rebuild the
clean-up helpers to convert the asynchronous exception into an ordinary one —
merlin's analysis code is full of downstream catch-all handlers that would then
silently swallow a future asynchronous exception, defeating the model.

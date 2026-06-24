**Summary**

The treatment of “asynchronous exceptions” in OxCaml is different from OCaml.  We call an exception “asynchronous” if it arises when a program is executing code in the runtime system (including certain user-written code such as custom block finalizers and signal handlers).  In OCaml, any safe point (e.g. an allocation directly in OCaml code) may emit such exceptions, meaning that they manifest themselves in user code at seemingly random points.

This isn’t compatible with Flambda 2, where allocations are treated as pure.  We don’t want to treat them as operations which may have control flow effects such as the raising of exceptions: it would be overly heavyweight and lead to unnecessary compiler code and IR term complexity.  It follows that changes to the asynchronous exception model are needed, because otherwise, an exception handler might not be correctly installed at the time when an allocation raises such an exception.

Despite being convenient for handling Ctrl+C (see below), the existing semantics can also cause bad behaviour of user programs, as the following example due to Frisch shows:  
	**`try`**  
  `42, Hashtbl.find t x`  
**`with`** `Not_found ->`  
  `key_not_in_table ()`  
It looks as if the exception handler will only be reached if the given key is not in the table, but in fact, it can also be reached if the allocation of the pair on the second line triggers a GC which in turn calls a finaliser (or signal handler) which raises `Not_found`.

In the past, an analogous example demonstrated that pressing Ctrl+C at an (in)opportune moment in the middle of a Coq proof would cause it to succeed, even if false\! 

We suspect users are aware that wildcard patterns in with clauses can cause Sys.Break to be erroneously caught, but fewer are aware that exception handlers can also match exceptions which were not raised by the body of the `try` clause.

**What we would like to do**

There is a wide subject area dealing with this sort of problem, including correct implementation of things such as `bracket` (c.f. Haskell), which would also require masking of asynchronous exceptions.  We’re not proposing to do anything as complex as that right now, although we do need to deal with Fun.protect (see below).  The main aim is to upstream a minimal solution, which can be extended in the future, but that immediately solves the problem of the current semantics being fundamentally incompatible with Flambda 2\.  This also conveniently resolves the problem of bad behaviour in user programs.

The downside is that some small changes are required to user code, as discussed below, but this seems unavoidable.  \[We will augment this paragraph after the two investigation points below have been done.\]

**How the OxCaml implementation works**

Asynchronous exceptions fall into the following cases:

* The GC running out of memory (the case currently handled by raising the `Out_of_memory` exception)  
* Exceptions raised in C finalisers for custom blocks  
* Exceptions raised in OCaml signal handlers  
  * This often includes `Sys.Break` for handling Ctrl+C via a `SIGINT` handler  
* Exceptions raised in memprof callbacks  
* The `Stack_overflow` exception (typically raised by the runtime’s C signal handler for `SIGSEGV`).

The solution adopted in OxCaml is surprisingly lightweight.  Parallel to the existing exception raising path, which starts via the current trap handler stored in the domain state, we add a new asynchronous exception path.  Many of the code changes required are just bookkeeping of that.  Then, we allow exceptions to be raised down the asynchronous path (using `caml_raise_async`), and provide a means for installing a handler to catch asynchronous exceptions and turn them into normal ones.  This is called `Sys.with_async_exns`.

**Breaking change: catching of asynchronous exceptions**

Code needing to catch asynchronous exceptions must explicitly call `Sys.with_async_exns`, around a sufficiently large scope of code, *otherwise any asynchronous exception raised will terminate the program* (just like a normal uncaught exception).  In particular code which catches `Sys.Break`, via a signal handler for `SIGINT`, needs to be updated.  We have not found this to be a problem with utop, for example, which relied on the old behaviour.

As an example, for catching Ctrl+C (given an appropriate OCaml signal handler already installed), one might do:  
**`try`**  
  `Sys.with_async_exns (fun () ->`  
    `...`  
    `try`  
      `(* slow computation, imagine Ctrl+C pressed here *)`  
    `with exn ->`  
      `(* exception handler E *)`  
  `)`  
**`with`** `Sys.Break ->`  
  `stop ()`  
In this example it is guaranteed that “exception handler E” will never be called when Ctrl+C is pressed.  We will instead jump directly to the outer handler that calls `stop`.  Note that the outer handler can still receive normal exceptions as well: all that `Sys.with_async_exns` is doing is turning any asynchronous exceptions into normal ones.

**Investigation point 1:** find where `Sys.with_async_exns` would need to be inserted in a variety of large programs, including Rocq, Frama-C and Alt-Ergo.  (Amongst other things, look for `Sys.Break`, and any catching of “timeout” exceptions; together with catch-all exception handlers around large outer loops.)

**Fun.protect**

There is a potentially difficult discussion to be had concerning `Fun.protect`.  The consensus on the Jane Street side is that this function should be left as-is and should not catch asynchronous exceptions; instead, a second version of the function should be provided which does.

It should be noted that `Fun.protect` is not a completely race-free implementation at present, nor will the proposed second version of such function be, although it should be an improvement.

**Raising of asynchronous exceptions**

Currently in OxCaml, we would like to ensure that the asynchronous exception model is only used for the places which really require it, so for the moment it is restricted to `Sys.Break` and `Stack_overflow` exceptions.  (Restricting the available asynchronous exceptions also makes it easier to think about in the case where they are raised recursively, e.g. a signal handler running out of stack space).  Any other exceptions raised from a finaliser, signal handler or memprof callback will terminate the program.  

In OxCaml, `Stack_overflow` is raised from the runtime, which calls a distinguished `caml_raise_async` function.  `Sys.Break` is raised from an OCaml signal handler; the runtime then sees that a permitted (non-async) exception is escaping from such a handler, and turns it into an asynchronous one.

We think the above is too restrictive for upstream.  An acceptable solution may be to remove the restriction on which exceptions can be raised, but insist that any asynchronous exception is raised by a new function (maybe called `Sys.raise_async`), e.g. from a user-defined signal handler.  Any normal exception raises from finalisers, memprof callbacks and signal handlers would always terminate the program.  This is a clean solution but does require more changes to user code, however we suspect that in general there will be fewer places to change to add `Sys.raise_async` than to add `Sys.with_async_exns`.

**Investigation point 2:** find where `Sys.raise_async` would need to be inserted in the same set of apps as above.  (Find all OCaml finalisers, memprof callbacks and OCaml signal handlers.  Look for things such as “timeout” exceptions.)

**Out\_of\_memory is now a fatal error when raised from the GC**

There is one place in the runtime where an exception used to be raised where now a fatal error is produced, namely the “out of memory” case in the GC.  Thus the `Out_of_memory` exception is only now raised in synchronous cases, such as providing an erroneous size to `Array.make`.  We have previously spoken to Damien about this change and he seemed on board with it.

**Size of change**

This is not a large patch.  It has been well-tested (in production for several years for the x86-64 version) and comes with good test cases.  In terms of implementation there are some fairly straightforward refactoring changes/addition to the “fail” files in the runtime etc, plus modifications to the runtime assembly language files to maintain the asynchronous exception stack.  Then there is the addition of `Sys.with_async_exns` to the stdlib.  There only exist assembly implementations for x86-64 and arm64, but it shouldn’t take long to port to the others, and we would expect to have done that prior to posting a PR upstream.

**Code in oxcaml**

**Main body of work:** port the following to upstream.  It is likely that Claude Code can do a pretty good job at this, including writing the assembly code for the currently-unsupported architectures.

The following five changesets, listed newest first, contain the code and tests (in `testsuite/tests/async-exns/`):

`30a3da9166 Async exns x effects (#2455)`  
`510e7b55bc Better handling of multiple simultaneous async exns (#2453)`  
`cd10a70aa4 Async exceptions implementation for arm64 (#2450)`  
`909abb16ff Asynchronous exceptions for 5.x runtime (#2007)`  
`a77f0dd658 Improve the semantics of asynchronous exceptions (new simpler version) (#802)`

The following exist as an artifact of the old flambda-backend repo structure (which had a git subtree looking like upstream) and should be ignored:

`6a6c53e4f9 flambda-backend: Better handling of multiple simultaneous async exns (#2453)`  
`e539d07578 flambda-backend: Async exceptions implementation for arm64 (#2450)`  
`fbeafb9a0f flambda-backend: Asynchronous exceptions for 5.x runtime (#2007)`  
`9469765e42 flambda-backend: Improve the semantics of asynchronous exceptions (new simpler version) (#802)`

The following PRs mention “async exceptions” in some way but are actually probably unrelated:

`43d8f9e6fe Fix async exception handling inside Domain.spawn`  
`17d657c3b8 Rename caml_handle_gc_interrupt_no_async_exception into caml_handle_gc_interrupt`  
`7c3ed63812 try to ensure asynchronous exceptions can't happen from signal handlers in caml_alloc functions called by C`  

# Experiment

The task is to do some code reading. I would like to determine where we're going to have to make changes to some large OCaml projects as a result of the "Asynchronous Exceptions" work that's outlined in the document "OxCaml asynchronous exceptions.md" in this directory. 

I would like to start with a single project, and I'm interested in finding areas that will need to be changed - in particular areas that are currently sound, but will break due to the new semantics. I'm thinking of things like invariants that will no longer hold, or resources that might be leaked, and so on. The fundamental problem is that in the new world a block of code with an exception handler either runs to completion, or enters the exception handler. A wildcard `with` statement can therefore be used to restore state, tidy up, etc. In the new world, this will no longer be true.



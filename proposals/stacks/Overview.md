# First-Class Stacks

There are a number of control-flow patterns that go beyond the classic "last-in first-out" form.
Coroutining, iterators based on yield, asynchronous I/O and, more exotically, delimited continuations, are all examples of non-linear control flow.
These patterns are becoming increasingly important in applications that are intended to be dynamically responsive.

While it is possible to emulate many of these control-flow patterns using standard core WASM features, there is significant cost in doing so &mdash; in terms of run-time efficiency, code size, and composability.

At the same time, the space of control-flow patterns is sufficiently diverse that it may not be appropriate to design special mechanisms for each of them.
Even focusing on coroutining, for example, there are at least two common patterns of coroutines&mdash;so-called symmetric and asymmetric coroutines&mdash;with significantly different implementation requirements.
Furthermore, the details of how data is exchanged between coroutines also varies greatly; reflecting choices made at the language-design level as well as application design.

This proposal focuses on a suite of low-level mechanisms that would allow language implementers to build different variations on *cooperative* multi-tasking.
Specifically, we focus on enabling the execution of WebAssembly programs on multiple stacks that a WebAssembly application can switch between.

Although important, we do not directly provide the mechanisms necessary to exchange messages between coroutines, nor do we "take a stand" on issues such as whether to support asymmetric vs. symmetric coroutines.
Instead we provide the primitives that&mdash;in conjunction with other functionality WebAssembly already provides&mdash;enables the language implementer to develop their own mechanisms.
We do, however, establish the framework for communicating the status of a coroutine when switching to another stack. That status must encode whether the coroutine is terminating and *may* encode values as part of that event.

In this document we explain the main aspects of the design for *multiple stacks* and illustrate them in the context of a few applications, including how to provide cooperative lightweight threads, how to implement asynchronous I/O, and how one can support delimited continuations.

## Design: Multiple Stacks

At the core of this proposal is the ability to create and switch between multiple stacks.
Our running example will be a lightweight thread manager, in which each lightweight thread has its own stack, and the manager switches between these stacks (and its own).

### Leaf References

A stack has two endpoints: its *root* (i.e. initial frame), and its *leaf* (i.e. active frame).
Both prior proposals for stack switching have used references to the *root* of the stack, and implemented switching through *yielding* and *resuming*.
But the root of a stack does not know where its leaf is, and when you resume to a stack its the leaf rather than the root that tells you what to update the stack-pointer and instruction-pointer registers to.
As such, both prior proposals implemented stack yielding by walking the stack to find its root and store a pointer to the current leaf in the root.
All in all, this made stack switching more expensive than it needs to be.

Here we instead propose to use references to the *leaf* of the stack.
And rather than yielding and resuming stacks, we simply switch stacks.
In a stack switch, the current stack transfers control to a given leaf reference for some stack and hands off the reference to the current stack's own leaf in the process.
This conceptually amounts to simply swapping the current contents of the stack- and instruction-pointer registers with whatever registers are (together) holding the leaf reference of the target stack.
In particular, there is no need to search the stack to find the root of the current stack.

We give these leaf-reference values the type `leafref`.

#### Linearity

Once someone switches to the leaf reference of some stack, it is important that no one else switch to that same leaf reference.
The reason is that the stack changes immediately when it is switched to, invalidating the stack- and code-pointer that comprised the leaf reference.
For this reason, `leafref` must be a (nearly) *linear* type so that its values cannot be duplicated.
This requires some clarifications for and variations to a few instructions.
In particular, various `get` and `set` instructions would be disallowed for linear types, and instead one would use `get_clear` (which replaces the read variable's contents with null) and `set_cleared` (which traps if the variable's contents were not null).
But overall, most of WebAssembly is unphased.

### Stack Switching

The key operation in this proposal is the stack switch.
When a stack switch occurs, one generally provides values for the target stack to resume execution with.
The question is what should the types of these values be?
There seems to be no good universal answer, and having each `leafref` *statically* specify the expected types would both complicate the type system and restrict usage patterns.
So instead we use a *dynamic* convention, repurposing events provided by exception handling.
That is, the stack giving control specifies an event tag and arguments corresponding to that event tag, and the stack receiving control checks that event tag to determine what to do.

This is achieved by the following instruction:
```
stack.switch $event : [t* leafref] -> unreachable
```
The `leafref` on the stack is the stack to switch to (trapping if that `leafref` is null).
The `$event` specifies the event to use for the transfer and must have type `[t* leafref]`.
The values for the `t*` of the event are given by the `t*` on the stack.
The value of the `leafref` of the event is the leaf reference of the current stack that is giving up control.

This leaves the stack in a halted state, waiting for control to eventually be transferred back.
When control is transferred back, some event is supplied.
This event is received using existing (revised) exception-handling mechanisms.

Although the most straightforward way to implement event handling is using a stack walk, typically the relevant event handler for a `stack.switch` will be in the immediate scope, so engines will likely want to optimize their implementation by directing control straight to that handler.

>Can we require that the event handler is syntactically local?

The following illustrates how a lightweight thread could yield control to a central thread manager by fetching that `leafref` and switching to it with the `$thread_yielded` event:
```
(event $thread_yielded (param leafref))
(event $resume (param))
(global $manager (param leafref))

(func $yield_current_thread
  (block $resumed
    (try
      (stack.switch $thread_yielded (global.get_clear $manager))
    catch $resume $resumed
    )
  )
)
```

>Here we have assumed that whenever the thread manager transfers control to a thread, its `leafref` is stored into the global variable `$manager`. A more sophisticated system &mdash; involving communication channels for example &mdash; would use a different bookkeeping mechanism.

Note that the `$resume` event does not have a `leafref` in its payload. This is because the thread manager uses a more advanced stack-switching instruction to resume a thread.

### Advanced Stack Switching

With `stack.switch`, the stack receiving control is always responsible for storing the `leafref` of the stack yielding control.
However, often the stack yielding control is the one that better knows what to do with its own `leafref`.
To support this pattern, we provide the instruction:

```
stack.switch_call $func $event : [ti* leafref] -> unreachable
```

where `event $event : [to*]` and `func $func : [ti* leafref] -> [to*]`. I.e., the event does not reference the `leafref` of the yielding stack.

This instruction switches control to the given `leafref` but has the receiving stack immediately call `$func` with the given arguments and the `leafref` for the *yielding* stack.
When `$func` returns, its result is then used to determine the arguments for the `$event` that the stack is waiting for.

With this we can use the following events and functions
```
(type $product ...)
(event $work_completed (param $product))

(func $resume_thread (param $thread leafref)
  (global.set_cleared $manager (local.get $thread))
)
(func $complete_work (param $work $product) (param $thread leafref) (result $product)
  (local.get $work)
)
```
so that when the manager resumes a thread it can do so with simply `stack.switch_call $resume $resume_thread`, which takes care of updating the global `$manager` variable, and when a thread completes it can do so with simply `stack.switch_call $complete_work $work_completed`.

Note that `$complete_work` implicitly drops the given `leafref`.

Because stack references are linear values, the stack is cleaned up by the engine once the stack frame it is stored in is cleaned up, so `$complete_work` implicitly performs this clean up before returning.
(The stack itself might contain references to other stacks within its stack frames, so this might proceed recursively.)
Unlike other memory management however, linearity enables this process to be done at a deterministic point in the execution, so systems that are sensitive to memory or timing can be ensured to have consistent behavior.

Note that both `$resume_thread` and `$complete_work` are very simple.
As such, `stack.switch_call` for these functions does not actually need to be implemented with a true function call.
Rather, these functions can be thought of as specifying, in a safe manner, the bookkeeping that should be done *during* the stack switch.

### Stack Creation, Part 1

One of our invariants is that a `leafref` always references an *attached* stack.
That is, if one were to walk the stack from the leaf, one would always reach a special *grounding* frame created by the host.
This grounding frame is responsible in particular for handling traps, whether by terminating the thread executing the trapping stack or by moving to the next event in the queue or by reporting an error and so on.
The point is, only the host can create grounding frames, and the semantics of these grounding frames depend on context, so we provide no means to create leaf references out of thin air.
Instead, a module that needs this functionality must import a function from the host, which the host can then instantiate with whatever is appropriate for the module instance's execution environment.

For our running implementation of lightweight threads, each thread stack needs such a grounding frame.
As such, the module imports a `$new_stack` function from the host, which it uses each time it needs to spawn a new thread.
```
(import "host" "new_stack" (func $new_stack (result leafref)))
(type $task ...)
(table $thread_pool 1000 leafref)
(global $current i32 (i32.const 999))
(func $find_noncurrent_null_in_thread_pool (result i32) ...)

(func $spawn_thread (param $work $task) (result i32) (local $id i32) (local $stack leafref)
  (local.set $id (call $find_noncurrent_null_in_thread_pool))
  (local.set_cleared $stack (call $new_stack))
  (local.set_cleared $stack (stack.extend $thread_rout (local.get $id) (local.get $work) (local.get_clear $stack)))
  (table.set_cleared $thread_pool (local.get $id) (local.get_clear $stack))
  (local.get $id)
)
```

The `$spawn_thread` function finds an unused identifier for the thread, uses the imported `$new_stack` function to create a new attached stack, then *extends* that stack with a custom frame, and finally adds the thread to the pool, returning its identifier.

The extension step is critical.
Realize that all the imported `$new_stack` function does is provide a `leafref` for some new stack, one that is conceptually blank besides its grounding frame.

In order to give this stack some functionality specific to the application at hand, one must extend the given stack with a special kind of call frame specific to the application at hand.
Because this concept is inspired by coroutines and typically is used to set up the root of the stack, we call this special kind of function a `rout`.
The following is an example `rout` for a lightweight thread:
```
(event $abort : [stackref])
(event $thread_aborted : [])
(func $do_work (param $task) (result $product) ...)
(func $finish_aborting (param $thread leafref)
  ;; implicitly cleanup the thread stack now that we're done with it
)

(rout $thread_rout (param $id i32) (param $input $task) (local $output $product)
  (block $aborted
    (try
      (block $resumed
        (try
          stack.start
        catch $resume $resumed
        )
      ) ;; $resumed
      (local.set $output (call $do_work (local.get $input))
      (stack.switch_call $work_completed $complete_work (local.get $output) (global.get_clear $manager))
    catch $abort $aborted
    catch_all
      unreachable
    )
  ) ;; $aborted : [leafref]
  (stack.switch_call $thread_aborted $finish_aborting)
)
```

A `rout` is similar to a `func` in many ways. There are a few differences: the first instructions of the body must be instructions like `block` and `try` that do nothing besides set up control flow, followed by `stack.start`. This sets up a stack frame that is awaiting an event and set to handle that event.
Another difference is that the type signature of a `rout` never has a return type - because it does not return a value.

The instruction `stack.extend $rout $event? : [ti* leafref] -> [leafref]`, where `$rout` is a `rout (param ti*) (result to*)` and `$event` (if specified) is an `event (param to*)`, extends the given stack with the frame for `$rout` with the given values as its arguments.
The resulting stack is waiting at the instruction `stack.start : [] -> unreachable` for an event.
(`stack.start` is only usable at the near-start of a `rout`.)
If an `$event` is specified, then once the stack is switched to and handled by the `rout` such that it returns, the returned values are forwarded to the `leafref` the `rout` was attached to with the specified event.
If no `$event` is specified, then the `rout` traps if it returns.
Any unhandled exceptions thrown from the `rout` are forwarded to the `leafref` the `rout` was attached to.

So if one calls an imported function `$new_stack : [] -> [stackref]` and then executes `(stack.extend $thread_rout (i32.const 237) $work)`, that altogether creates a new stack that is
1. waiting for its first `$resume` event,
2. upon which it will call `$do_thread_work` with the given `$work`,
3. which in turn can yield control to the thread manager via `(call $yield_current_thread)` (defined above),
4. and after all the work is complete it will return control to the thread manager, providing the result of the work via the event `$work_completed`.

### Stack Cleanup

Notice that `$thread_rout` also handles an `$abort` event, which provides a `leafref` in its payload.
This is used to unwind the stack of a thread that has been aborted.
Although discarding a leaf reference implicitly causes the stack to be cleaned up, along with any stacks that it contains references to, this process does nothing more than memory management.
That is, besides side channels like timing behavior, this process has no visible semantic effect.
But the thread being aborted may be using various application-level resources that need to be freed, and as such one wants to run the unwinding code (e.g. destructors) on its stack.

Because we build upon exception-handling events, this can be done simply with the following function (requiring no new instructions):
```
(func $abort_thread (param $id i32)
  (block $aborted
    (try
      (stack.switch $abort (table.get_clear $thread_pool (local.get $id)))
    catch $thread_aborted $aborted
    catch_all
      unreachable
    )
  )
)
```
Note that the `$yield_current_thread` function does not handle the `$abort` event.
As such, when a yielded thread stack is switched to, the event gets thrown as an exception.
The expectation is that, except for its root `$thread_rout`, the thread will not catch the `$abort` event.
This in turn causes all of the unwinding code on its stack to execute.
As for `$thread_rout`, the switch in `$abort_thread` packages the `leafref` of whoever called `$abort_thread`, which `$thread_rout` finally extracts and switches control back to (implicitly cleaning up the last remnants of the aborted thread's stack in the process).

### Application&mdash;Lightweight Threads

We now illustrate the last few pieces for implementing cooperative lightweight threads.

```
(table $intermediate_results 1000 $product)
```

Our module uses a table to store the results of intermediate threads.
In our example, results are never null (i.e. null indicates the thread has not completed its work).

```
(func $find_next_nonnull_in_thread_pool (param i32) (result i32) ...)

(func $main (param $work $task) (result $product) (local $primary i32)
  (local.set $primary (call $spawn_thread $work))
  (block $finished
    (loop $round_robin
      (global.set $current (call $find_next_nonnull_in_thread_pool (global.get $current)))
      (table.set $intermediate_results (global.get $current) (externref.null))
      (block $completed
        (block $yielded
          (try
            (stack.switch_call $resume $resume_thread (table.get_clear $thread_pool (global.get $current)))
          catch $thread_yielded $yielded
          catch $work_completed $completed
          catch_all
            unreachable
          )
        ) ;; $yielded : [leafref]
        (table.set_cleared $thread_pool (global.get $current))
        (br $round_robin)
      ) ;; $completed : [$product]
      (table.set $intermediate_results (global.get $current))
      (br_if $finished (i32.eq (global.get $current) (local.get $primary)))
      (br $round_robin)
    )
  ) ;; $finished
  (block $done
    (loop $cleanup
      (global.set $current (call $find_next_nonnull_in_thread_pool (local.get $primary)))
      (br_if $done (i32.eq (global.get $current) (local.get $primary)))
      (table.set $intermediate_results (global.get $current) (externref.null))
      (call $abort_thread (global.get $current))
    )
  ) ;; $done
  (table.get $intermediate_results (local.get $primary))
)
```

The main program first spawns a "primary" thread with the work at hand.
Then it round robins through the threads until the primary thread completes.
If a thread yields, its `leafref` is stored into the `$thread_pool` table.
If a thread completes, the result is stored into the `$intermediate_results` table, and the cycle guarantees that every thread that depends on that work has a chance to fetch it from the table before it is cleared.
After the primary thread finishes, all the threads are aborted before returning the final result in order to run any unwinding code (e.g. destructors) on their stacks.

Lastly, for completeness we illustrate how threads split up and rejoin work in this simplified example.
We emphasize that this is not the only way this can be done&mdash;the instructions provided support many alternatives&mdash;this just happens to be how the toy example works.
```
(func $divide_and_conquer (param $work1 $task) (param $work2 $task) (result $product $product)
    (local $id1 i32) (local $id2 i32) (local $prod1 $product) (local $prod2 $product)
  (local.set $id1 (call $spawn_thread (local.get $work1)))
  (local.set $id2 (call $spawn_thread (local.get $work2)))
  (loop $waiting
    (block $resumed
      (try
        (call $yield_current_thread)
      catch $resume $resumed
      )
    ) ;; $resumed
    (if (ref.is_null (table.get $intermediate_results (local.get $id1)))
    then
    else
      (local.set $prod1 (table.get $intermediate_results))
    )
    (if (ref.is_null (table.get $intermediate_results (local.get $id2)))
    then
    else
      (local.set $prod2 (table.get $intermediate_results))
    )
    (br_if $waiting (ref.is_null (local.get $prod1)))
    (br_if $waiting (ref.is_null (local.get $prod2)))
  )
  (local.get $prod1) (local.get $prod2)
)
```

And with that, we have efficient lightweight threads without any stack walking (except to cleanup/unwind stacks) or any cyclic garbage collection or reference counting.

## Design: Detachable Stacks

So far all stacks have been *attached*, meaning they have a grounding frame provided by the host and can be directly switched to via their `leafref`.
But sometimes a portion of a stack needs to be detached so that it can be saved for later and run on a different stack.
Our running example for this will be asynchronous I/O, in which its useful to detach the portion of the stack belonging to the WebAssembly application at hand so that it can be run later after an asynchronous action has completed.

### Stack Creation, Part 2

The proposal never lets one detach a stack arbitrarily.
Instead, one creates detached stacks that they can attach to and detach from the current stack.
Whereas we provide no instruction for creating *attached* stacks, since these need a grounding frame that only the host has the appropriate context to create, we do provide the following instruction for creating *detached* stacks, which later can be attached to attached stacks with grounding frames:
```
stack.create : [] -> [stackref]
```
This instruction introduces the type `stackref`, which conceptually is a pair of references to both the root and the leaf of a detached stack.
The root is waiting for a `leafref` to attach the stack to.
The leaf is set up to just forward whatever event is received to whatever the root is attached to.
In other words, the newly created stack is empty, at least at the moment.

Just like with leaf references, we can extend a `stackref` with a `rout` to give it some application-specific functionality.
The instruction:

```
stack.extend_leaf $rout $event? : [ti* stackref] -> [stackref]
```

where `$rout` is a `rout (param ti*) (result to*)` and `$event` (if specified) is an `event (param to*)`, extends the given stack with the frame for `$rout` with the given values as its arguments.
Note that this instruction consumes and produces a `stackref`.
This is because, like `leafref`, the `stackref` type is linear.

### Stack Composition

Because a `stackref` is detached, we can compose them with various stack references.
One way to do so is to generalize `rout` a bit.
In particular, in place of `stack.start`, we allow a `rout` to use `stack.attach_clear $local`, where `$local` is a local variable of type `stackref`.
This instruction indicates to initially configure the `rout` to have the `stackref` in the local variable be attached (and clears the content of the variable).
So any event received by the `rout` goes to the leaf of the given `stackref`, and escaping events thrown in the given `stackref` propagate to the `rout`.

As an example, the following illustrates how to compose two `stackref`s together:
```
(rout $attach_rout (param $inner stackref)
  (stack.attach_clear $inner)
)
(func $compose (param $outer stackref) (param $inner stackref) (result stackref)
  (stack.extend_leaf $attach_rout (local.get_clear $inner) (local.get_clear $outer))
)
```

We can also attach a `stackref` to the *current* stack.
This is done using the instruction:

```
stack.attach_switch $event : [t* stackref] -> unreachable
```

where the specified event `$event : [t*]` is used to transfer control to the leaf of the given `stackref` after attaching it.
This will be particularly useful for resuming computation on a `stackref` that was saved earlier using the following detaching process.

### Stack Decomposition

For detaching stacks, we expand upon stack inspection, i.e. the first phase of two-phase exception handling.
We describe stack inspection in more detail below, but in short one inspects the stack with the instruction

```
inquire $dispatch_tag : [ti*] -> [to*]
```
where `$dispatch_tag` is a dispatch tag, i.e. a generalization of the function-type signature used in `call_indirect`, of type `[ti*] -> [to*]`.
Handlers on the stack take the form of a `respond $dispatch_tag` block whose body responds to the inquiry with instructions that generate outputs matching the types specified by the dispatch tag.

Normally `respond $dispatch_tag` would follow a `try` block, per standard exception handling, but in this proposal we also allow it to follow `stack.attach_clear`.
Furthermore, when used in this manner, the body of the `respond` block can use a special `stack.detach` instruction:

```
stack.detach $responder $rout $label : [ti* tl*] -> unreachable
```
The `$responder` indicates which containing `respond $dispatch_tag` block to detach up to, where `$dispatch_tag` must have output type `[to*]`.
The `$rout` is a `rout` of type `[ti*] -> [to*]`; it is put on the detached stack in order to handle the stack-switching event, and is configured so that its returned values provide the response for the `respond` block.
Lastly, control is transferred to the label `$label` *outside* of the `respond` block, where `$label` has type `[tl* stackref]`&mdash;the final `stackref` is then the reference to the stack that was just detached.
The unfortunate complexity of the instruction is due to the fact that the content of the current stack needs to be examined by the `$responder` in order to determine *if* the stack should be detached (and the `$label` specifies how), but the rest of the response must not depend on the current stack because it might be concluded after being attached to a completely different stack (so the `$rout` specifies how to conclude the response).

### Application&mdash;Async/Await

Now we put the pieces together to illustrate how a program written in a synchronous style can be made asynchronous using detachable stacks.
First, suppose the main function of the program is:

```
(func $main (param externref) (result externref) ...)
```
operating on `externref` for the sake of simplicity.
Second, suppose `$main` and its callees use

```
(call $await) : [externref] -> [externref]`
```
to extract (and possibly wait for) the value of what is conceptually a promise.

Using this proposal, we can implement `$await` within WebAssembly:
```
(dispatch_tag $awaiting (param externref) (result externref))

(func $await (param $promise externref) (result externref)
  (inquire $awaiting (local.get $promise))
)
```
This starts a stack inspection, with the intent that it will cause the current stack to be detached and registered with the `externref` promise, eventually responding with the value the promise resolves to.

For that to happen, an `$awaiting` handler on the stack needs to actually detach the stack and such.
The following is one example of such a handler:
```
(event $fulfilled (param externref))
(event $finished (param externref))
(import (func $create_promise (param externref stackref) (result externref)))

(rout $resumer (result externref)
  (block $resumed
    (try
      stack.start
    catch $fulfilled $resumed
    )
  ) ;; $resumed : [externref]
)
(func $fulfill (export "fulfill") (param $stack stackref) (param $value externref) (result externref)
  (block $finish
    (try
      (block $yielded
        (stack.attach_switch $fulfilled (local.get $value) (local.get_clear $stack))
        (respond $responder $awaiting ;; [externref] -> [externref]
          (stack.detach $responder $resumer $yielded)
        )
      ) ;; $yielded : [externref stackref]
      (return (call $create_promise))
    catch $finished $finish
    )
  ) ;; $finish : [externref]
)
```
The `$fulfill` function attaches the given `stackref` and switches to it with the `$fulfill` event, which is used to indicate that the value the stack was waiting for has been provided (as the payload of the event).
That stack might finish, indicating the `$main` program is all done, in which case it "returns" with the `$finished` event and the result of the payload.
So `$fulfill` catches that event and returns its payload.
However, the stack might also end up waiting for some other promise.
In this case, `$responder` detaches the stack, using `$resumer` to respond with the payload of any future `$fulfill` event, and then hands off the promise and the detached stack to `$yielded`.
This in turn creates a promise, returning it as the (incomplete) result of `$fulfill`, using the imported function `$create_promise`.
That imported function calls `then` on the given promise in JavaScript, handing it JavaScript functions that either take the `stackref` and the resolved value and call `$fulfill` with them or take the `stackref` and the error and call `$reject` with them, defined below.

```
(import (event $extern_exn (param externref)))

(func $reject (export "reject") (param $stack stackref) (param $error externref) (result externref)
  (block $finish
    (try
      (block $yielded
        (stack.attach_switch $extern_exn (local.get $error) (local.get_clear $stack))
        (respond $responder $awaiting ;; [externref] -> [externref]
          (stack.detach $responder $resumer $yielded)
        )
      ) ;; $yielded : [externref stackref]
      (return (call $create_promise))
    catch $finished $finish
    )
  ) ;; $finish : [externref]
)
```
This is nearly identical to `$fulfill` except that it switches with the `$extern_exn` event rather than `$fulfilled`.
Because `$extern_exn` is not handled by `$resumer`, it will be thrown as an exception on the stack.
The intent is for `$extern_exn` to be instantiated with the event for JavaScript exceptions, so that this amounts to throwing a JavaScript exception on the stack, which is exactly what would have happened had the promise been resolved synchronously.

Lastly, to tie it all together, here is the exported asynchronous version of the main program:
```
(rout $main_rout (result externref)
  (block $started
    (try
      stack.start
    catch $fulfilled $started
    )
  ) ;; $started : [externref]
  (call $main)
)
(func $main_async (export "main") (param $input externref) (result externref) (local $stack stackref)
  (local.set_cleared $stack (stack.create))
  (local.set_cleared $stack (stack.extend_leaf $main_rout $finished (local.get_clear $stack)))
  (call $fulfill (local.get_clear $stack) (local.get $input))
)
```
The asynchronous version essentially creates a `stackref` and initializes it to run `$main` and "return" the result with the `$finished` event.
Then it kicks off the main program, now on its own stack, calling `$fulfill` to pass in the value that the program is waiting for as well as set up all the infrastructure for detaching the stack and creating promises as needed.

And with that, we have an efficient implementation of the async/await pattern, with the only reliance on the JS API being to register the handlers on the given promise, and with the property that each JS event runs to completion (assuming it terminates).

## Design: Stack Inspection

The previous example has very coarse delimiting of stacks, such that its `respond` for `stack.attach` always detaches upon the right inquiry.
Furthermore, the stacks are always used just once.
On the other hand, languages with tagged forms of reusable delimited continuations need the additional flexibility that `respond` blocks provide (in order to *dynamically* check tags), and they need an ability to duplicate stacks.
We purposely do not provide a primitive for stack duplication, both because we are concerned duplicating stacks of programs not designed for such functionality could create vulnerabilities, and because the process is rather complex with many seemingly arbitrary design choices to be made such that it is unclear it can be done in a suitably agnostic manner.
So instead here we demonstrate how stack inspection can be used to implement tagged reusable delimited continuations.

## Summary

This proposal outlines a suite of features that can be used to implement various patterns of non-sequential control flow: coroutines (both symmetric and asymmetric), support for asynchronous I/O, delimited continuations and many more.

The key abstractions revolve around the concept of a stack as a manageable entity and switching control between stacks.

In order to support these, we identify two main gaps in the WebAssembly architecture: support for stack inspections and support for linearly typed variables. We explicate them here to the extent necessary for our primary use-case; but they deserve independent proposals as they have other applications.

## Appendix: Listing of new features

The features core to this proposal are those directly supporting first-class stacks.
However, those features rely on support for linear types and stack inspection.
Linear types and stack inspection each have utility beyond first-class stacks and so might be better factored out into separate proposals that would consider how best to design their features in a broader context.
Here we summarize the new instructions/constructs introduced above and how they fall into these three categories.

### First-Class Stacks

#### Types

* `leafref`: reference to an attached stack

* `stackref`: reference to a detached stack

### Forms

* `rout`: variant of `func` using `stack.start`

#### Instructions

* `stack.attach_clear`

```
stack.attach_clear $local
```

* `stack.attach_switch`

```
stack.attach_switch $event : [t* stackref] -> unreachable
```

* `stack.create`

```
stack.create : [] -> [stackref]
```

* `stack.extend`

```
stack.extend $rout : [t* stackref] -> [stackref]
```

* `stack.extend_leaf`

```
stack.extend_leaf $rout $event? : [ti* stackref] -> [stackref]
```

* `stack_detach`

```
stack.detach $responder $rout $label : [ti* tl*] -> unreachable
```

* `stack.start` (usable only in specific locations within a `rout`)

```
stack.start : [] -> unreachable
```

* `stack.switch`

```
stack.switch $event : [t* leafref] -> unreachable
```

* `stack.switch_call`

```
stack.switch_call $func $event : [ti* leafref] -> unreachable
```

### Linear Types

These instructions are used to support linear types.
They all either clear (i.e. set to `null`) a store after getting its value, or check that a store is cleared before setting it to a value.

* `global.get_clear`

```
global.get_clear $global : [] -> [t]
```

* `global.set_cleared`

```
global.set_cleared $global : [t] -> []
```

* `local.get_clear`

```
local.get_clear $local : [] -> [t]
```

* `local.set_cleared`

```
local.set_cleared $local : [t] -> []
```

* `table.get_clear`

```
table.get_clear $table : [i32] -> [t]
```

* `table.set_cleared`

```
table.set_cleared $table : [i32 t] -> []
```

### Stack Inspection

We list only a few of the features that we would need from a complete suite of operations for stack inspection.

* `inquire`

`inquire` searches the stack for a handler for a given call tag and invokes that handler with the additional arguments provided.

```
inquire $call_tag : [ti*] -> [to*]
```

* `respond`

The `respond` form establishes a handler for a given call tag.

```
respond $label instr* end
```

## Appendix: Frequently Asked Questions (FAQ)

### How does this relate to other proposals?

There are three prior proposals-of-sorts on this topic, each of which had significant influence on the design we developed:
1. [Andreas Rossberg's presentation at the Feb 2020 In-Person CG Meeting](https://github.com/WebAssembly/meetings/blob/master/main/2020/presentations/2020-02-rossberg-continuations.pdf) inspired our heavy use of events
2. [Ross Tate's primer on low-level stack primitives](https://github.com/WebAssembly/exception-handling/issues/105) inspired our considerations for stack walking
3. [Alon Zakai's and Ingvar Stepanyan's proposal for await](https://github.com/WebAssembly/design/issues/1345) inspired our attention to host needs regarding *attached* stacks

One major difference between this proposal and all priors is our introduction of `leafref`.
All prior proposals would require most applications to use a stack walk to switch between stacks.
But looking at related systems, we found that `leafref` enabled a much more efficient stack-switching mechanism for applications more centered around cooperative multi-tasking.

Another major difference is our use of linear types.
Each prior proposal implicitly relied on automatic memory management (and finalizers) to clean up stacks.
Furthermore, each prior proposal implicitly baked in some locking and/or invalidating policy to prevent multiple accesses to the same stack.
By using linear types, our instructions can be implemented more efficiently, knowing they have the only reference to the stack, and the application can have more control to design and implement its own policies for managing a linear resource.

Lastly, our introduction of `rout` enables more efficient initialization, extension, and composition of stacks *without* requiring switching to the stack being modified and back.

In short, we believe the prior proposals were all headed in the direction of this proposal.
We have just examined a wider range of applications in more depth, and in doing came across opportunities to break higher-level operations down into orthogonal lower-level operations.

On that note, some of the more complex aspects of this proposal are due to WebAssembly's bundling of code-state, stack-state, and syntactic nesting.
For example, we found it useful to realize that a label specifies both a code pointer *and* a stack pointer.
Similarly, a branch jumps to the code pointer *and* cleans up the stack below the given stack pointer.
The fact that labels and the like are established through syntactic nesting is what necessitated `rout` as a distinct concept from `func`.
This is not to say this bundling is a fault in the design of WebAssembly, since in many ways we benefited from it, but it is important for understanding the rationale behind the design of this proposal and why certain instructions/constructs could not be decomposed further.

### What are the key dependencies?

The entire proposal is dependent on [exception handling](https://github.com/WebAssembly/exception-handling/), and the `stackref` portion is dependent on stack inspection.
There is no dependency on [garbage collection](https://github.com/WebAssembly/exception-handling/), though garbage collection would have to be amended to accommodate linear types.

### What are the implications for JavaScript and browser interoperability?

This proposal was designed to not introduce any significant complications with JavaScript interoperability.
Browsers already need to deal with computations being suspended midflight with their stacks put aside until later due to preemptive multi-tasking; this proposal just enables that suspension to occur (cooperatively) within a thread.
Yes, that includes switching between stacks while processing a message on the event loop, but by not providing a way to create a `leafref` out of thin air, (and presumably by not allowing stacks to be passed within messages to other threads,) the design ensures that all these stacks are anchored to the event loop.
As such, there is no way to use this proposal to somehow escape the event loop.

Of course, just like JavaScript programs can run an infinite while loop, WebAssembly programs can switch between stacks infinitely.
That is, just like JavaScript developers, any design using this proposal needs to be conscious of the run-to-completion model.
For example, our thread-manager example does not complete until the work is completely done, but it would be easy to modify it to have the scheduler return to the caller once in a while, relying on the caller to schedule an event that will call back into the instance and resume the scheduler.

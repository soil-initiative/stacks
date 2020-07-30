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

In this document we explain the main aspects of the design for *first-class stacks* and illustrate them in the context of a few applications, including how to provide cooperative lightweight threads, how to implement asynchronous I/O, and how one can support multi-shot tagged delimited continuations.

## Design: Multiple Stacks

At the core of this proposal is the ability to create and switch between multiple stacks.
Our running example will be a lightweight thread manager, in which each lightweight thread has its own stack, and the manager switches between these stacks (and its own).

### Stack References

A stack has two endpoints: its *root* (i.e. initial frame), and its *leaf* (i.e. active frame).
In this proposal, a `stackref` is a reference to the *leaf* of a stack.
Consequently, we can switch between stacks by simply swapping the current contents of the stack- and instruction-pointer registers with whatever registers are (together) holding the leaf reference of the stack being switched to.
In particular, there is no need to search the stack to find the root of the current stack in order to *detach* it, a topic we discuss later.

#### Linearity

Once someone switches to some stack, it is important that no one else switch to that same stack.
The reason is that the stack changes immediately when it is switched to, invalidating the stack- and code-pointer that comprised the reference.
For this reason, `stackref` must be a (nearly) *linear* type so that its values cannot be duplicated.
This requires some clarifications for and variations to a few instructions.
In particular, various `get` and `set` instructions would be disallowed for linear types, and instead one would use `get_clear` (which replaces the read variable's contents with null) and `set_cleared` (which traps if the variable's contents were not null).
But overall, most of WebAssembly is unphased.
(Although linear types are difficult to use in type systems for high-level languages, they align well with type systems for low-level languages.)

### Stack Switching

The key operation in this proposal is the stack switch.
When a stack switch occurs, one generally provides values for the target stack to resume execution with.
The question is what should the types of these values be?
There seems to be no good universal answer, and having each `stackref` *statically* specify the expected types would both complicate the type system and restrict usage patterns.
So instead we use a *dynamic* convention, repurposing events provided by exception handling.
That is, the stack giving control specifies an event and arguments corresponding to that event, and the stack receiving control checks that event to determine what to do.

This is achieved by the following instruction:
```
stack.switch $event : [t* stackref] -> unreachable
```
The `stackref` is the stack to switch to (trapping if that `stackref` is null).
The `$event` specifies the event to use for the transfer and must have type `[t* stackref]`.
The values for the `t*` of the event are given by the `t*` on the stack.
The value of the `stackref` of the event is the reference to the current stack that is giving up control.

This leaves the stack that was switched from in a halted state, waiting for control to eventually be transferred back.
When control is transferred back, some event is supplied.
This event is received using existing exception-handling mechanisms.
There is currently discussion on how to revise the exception-handling proposal to better support two-phase stack unwinding (and similarly this proposal), so for this document we suppose a `try instr*` block can be followed by multiple `catch $event $label` instructions indicating to transfer control to `$label` on `$event` exceptions.

Although the most straightforward way to implement event handling is using a stack walk, typically the relevant event handler for a `stack.switch` will be in the immediate scope, so engines will likely want to optimize their implementation by directing control straight to that handler.

The following illustrates how a lightweight thread could yield control to a central thread manager by fetching that `stackref` and switching to it with the `$thread_yielded` event:
```
(event $thread_yielded (param stackref))
(event $resume (param))
(global $manager (param stackref))

(func $yield_current_thread
  (block $resumed
    (try
      (stack.switch $thread_yielded (global.get_clear $manager))
    catch $resume $resumed
    )
  )
)
```

Here we have assumed that whenever the thread manager transfers control to a thread, its `stackref` is stored into the global variable `$manager`.
A more sophisticated system&mdash;involving communication channels for example&mdash;would use a different bookkeeping mechanism.

Note that the `$resume` event does not have a `stackref` in its payload.
This is because the thread manager uses a more advanced stack-switching instruction to resume a thread.

### Advanced Stack Switching

With `stack.switch`, the stack receiving control is always responsible for storing the `stackref` of the stack yielding control.
However, often the stack yielding control is the one that better knows what to do with its own `stackref`.
To support this pattern, we provide the instruction:
```
stack.switch_call $func $event? : [ti* stackref] -> unreachable
```
where `func $func : [ti* stackref] -> [to*]` and `event $event : [to*]` (if specified).
I.e., the event does not reference the `stackref` of the yielding stack.

This instruction switches control to the given `stackref` but has the receiving stack immediately call `$func` with the given arguments *and* the `stackref` for the *yielding* stack.
When `$func` returns, its result is then used to determine the arguments for the `$event` that the stack is waiting for.
If no `$event` is specified, the return traps.

With this we can use the following events and functions
```
(type $product ...)
(event $work_completed (param $product))

(func $resume_thread (param $thread stackref)
  (global.set_cleared $manager (local.get $thread))
)
(func $complete_work (param $work $product) (param $thread stackref) (result $product)
  (local.get $work)
)
```
so that when the manager resumes a thread it can do so with simply `stack.switch_call $resume_thread $resume`, which takes care of updating the global `$manager` variable, and when a thread completes it can do so with simply `stack.switch_call $complete_work $work_completed`.

Note that `$complete_work` implicitly drops the given `stackref`.
Because stack references are linear values, the stack is cleaned up by the engine once the stack frame it is stored in is cleaned up, so `$complete_work` implicitly performs this clean up before returning.
(The stack itself might contain references to other stacks within its stack frames, so this might proceed recursively.)
Unlike other memory management however, linearity enables this process to be done at a deterministic point in the execution, so systems that are sensitive to memory or timing can be ensured to have consistent behavior.
This pattern is common, so we include the following instruction as a shorthand for switching and dropping the yielding stack (so that `$event` just has payload `[t*]`):
```
stack.switch_drop $event : [t* stackref] -> unreachable
```

Note that both `$resume_thread` and `$complete_work` are very simple.
As such, `stack.switch_call` for these functions does not actually need to be implemented with a true function call.
Rather, these functions can be thought of as specifying, in a safe manner, the bookkeeping that should be done *during* the stack switch.

### Stack Creation

One of our invariants is that a `stackref` always references a *complete* stack.
That is, if one were to walk the stack from the leaf, one would always reach a special *grounding* frame created by the host.
This grounding frame is responsible in particular for handling traps, whether by terminating the thread executing the trapping stack or by moving to the next event in the queue or by reporting an error and so on.
The point is, only the host can create grounding frames, and the semantics of these grounding frames depend on context, so we provide no means to create stack references out of thin air.
Instead, a module that needs this functionality must import a function from the host, which the host can then instantiate with whatever is appropriate for the module instance's execution environment.

For our running implementation of lightweight threads, each thread stack needs such a grounding frame.
As such, the module imports a `$new_stack` function from the host, which it uses each time it needs to spawn a new thread.
```
(import "host" "new_stack" (func $new_stack (result stackref)))
(type $task ...)
(table $thread_pool 1000 stackref)
(global $current i32 (i32.const 999))
(func $find_noncurrent_null_in_thread_pool (result i32) ...)

(func $spawn_thread (param $work $task) (result i32) (local $id i32) (local $stack stackref)
  (local.set $id (call $find_noncurrent_null_in_thread_pool))
  (local.set_cleared $stack (call $new_stack))
  (local.set_cleared $stack (stack.extend $thread_rout (local.get $id) (local.get $work) (local.get_clear $stack)))
  (table.set_cleared $thread_pool (local.get $id) (local.get_clear $stack))
  (local.get $id)
)
```

The `$spawn_thread` function finds an unused identifier for the thread, uses the imported `$new_stack` function to create a new complete stack, then *extends* that stack with a custom frame, and finally adds the thread to the pool, returning its identifier.

The extension step is critical.
Realize that all the imported `$new_stack` function does is provide a `stackref` for some new stack, one that is conceptually blank besides its grounding frame.

In order to give this stack some functionality specific to the application at hand, one must extend the given stack with a special kind of call frame specific to the application at hand.
Because this concept is inspired by coroutines and typically is used to set up the root of the stack, we call this special kind of function a `rout`.
The following is an example `rout` for a lightweight thread:
```
(event $abort : [stackref])
(event $thread_aborted : [])
(func $do_work (param $task) (result $product) ...)

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
      (stack.switch_drop $work_completed (local.get $output) (global.get_clear $manager))
    catch $abort $aborted
    )
  ) ;; $aborted : [stackref]
  (stack.switch_drop $thread_aborted)
)
```

A `rout` is similar to a `func` but with one critical difference: the first instructions of the body must be instructions like `block` and `try` that do nothing besides set up control flow, followed by `stack.start`.
This sets up a stack frame that is awaiting an event and set to handle that event.

To add a `rout` stack-frame to a stack, one uses the following instruction:
```
stack.extend $rout $event? : [ti* stackref] -> [stackref]
```
where `$rout` is a `rout (param ti*) (result to*)` and `$event` (if specified) is an `event (param to*)`, extends the given stack with the frame for `$rout` with the given values as its arguments.
The resulting stack is waiting for an event at the following special instruction that is only usable at the near-start of a `rout`:
```
stack.start : [] -> unreachable
```
If an `$event` is specified, then once the stack is switched to and handled by the `rout` such that it returns, the returned values are forwarded to the `stackref` the `rout` was attached to with the specified event.
If no `$event` is specified, then the `rout` traps if it returns.
Any unhandled exceptions thrown from the `rout` are forwarded to the `stackref` the `rout` was attached to.

So if one calls an imported function `$new_stack : [] -> [stackref]` and then executes `(stack.extend $thread_rout (i32.const 237) $work)`, that altogether creates a new stack that is
1. waiting for its first `$resume` event,
2. upon which it will call `$do_thread_work` with the given `$work`,
3. which in turn can yield control to the thread manager via `(call $yield_current_thread)` (defined above),
4. and after all the work is complete it will return control to the thread manager, providing the result of the work via the event `$work_completed`.

### Stack Cleanup

Notice that `$thread_rout` also handles an `$abort` event, which provides a `stackref` in its payload.
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
    )
  )
)
```
Note that the `$yield_current_thread` function does not handle the `$abort` event.
As such, when a yielded thread stack is switched to, the event gets thrown as an exception.
The expectation is that, except for its root `$thread_rout`, the thread will not catch the `$abort` event.
This in turn causes all of the unwinding code on its stack to execute.
As for `$thread_rout`, the switch in `$abort_thread` packages the `stackref` of whoever called `$abort_thread`, which `$thread_rout` finally extracts and switches control back to (implicitly cleaning up the last remnants of the aborted thread's stack in the process).

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
            (stack.switch_call $resume_thread $resume (table.get_clear $thread_pool (global.get $current)))
          catch $thread_yielded $yielded
          catch $work_completed $completed
          )
        ) ;; $yielded : [stackref]
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
If a thread yields, its `stackref` is stored into the `$thread_pool` table.
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

## Design: Composable Stacks

So far each stack has been independent, but in many applications stacks are chained together to form a conceptual whole.
In particular, if an exception is thrown at the end of this chain, the exception is propagated through the entire chain until it finds a handler.
So although these stacks are allocated separately, they are made to appear as one stack.
Our running example for composable stacks will be asynchronous I/O.

### Redirection

We provide stack composition by redirecting stack *walks*, the technique used by engines to implement features such as [exception handling](https://github.com/WebAssembly/exception-handling/) and [stack inspection](https://github.com/WebAssembly/design/issues/1356).
We do so with the following instruction:
```
stack.redirect $local instr* end : [ti*] -> [to*]
```
where `local $local : stackref` and `instr* : [ti*] -> [to*]`.
This instruction executes `instr*` but redirects an exceptions thrown therein to the `stackref` in `$local` at the time the exception reaches the `stack.redirect`.
Stack inspections are similarly redirected.
If `$local` is null at the relevant time, then no redirection occurs.

Given that `stack.redirect` does nothing upon entry, like `block` and `try`, it is allowed to preceed `stack.start` in a `rout`.

### Stack Inspection

For most applications of stack inspection, one needs some form of stack inspection.
In particular, applications typically set up the redirection in the root frame of the stack, but need the leaf frame to have some way to get and change the `stackref` being redirected to.
This involves walking the stack (while keeping it in tact) to find the root frame and then executing relevant code to access and mutate the `$local` being used for redirection in the root frame.

Stack inspection has yet to be designed.
It generally requires some way to functions registered with frames on the stack that have the ability to inspect and update the stack frame they are registered with.
For now we assume we have the following instructions for stack inspection (using the [Call Tags](https://github.com/WebAssembly/call-tags) proposal):
* `call_stack $call_tag : [ti*] -> [to*]`
  * where `call_tag $call_tag : [ti*] -> [to*]`, specifying the inputs and outputs of the inspection
  * looks up the stack for an `answer $call_tag` and calls it
* `answer $call_tag instr1* within instr2* end`
  * where `call tag $call_tag : [ti*] -> [to*]`
  * and `instr1* : [ti*] -> [to*]`
  * executes (and inherits the type of) `instr2*`, during which answering any `call_stack $call_tag` with `instr1*`

### Application&mdash;Async/Await

Now we put the pieces together to illustrate how a program written in a synchronous style can be made asynchronous using stack composition.
At a high-level, every external entry point to the WebAssembly program allocates a new internal stack to run the program on and then uses redirection to make it appear as if the program is running on the original stack.
Whenever the program has to wait for some promise, it removes the redirection, registers the internal stack as part of a listener on the promise, and returns control to the original stack.
When the promise resolves, the internal stack sets up a redirection to the promise's stack, and continues executing the program.

First, suppose the synchronous program has the following as an entry point:
```
(func $entry (param f64) (result f64) ...)
```
operating on `f64` for the sake of simplicity.

Then the asynchronous program would have the following as the corresponding export:
```
(import (func $new_stack (result stackref)))
(import (func $create_promise (param externref stackref) (result externref)))
(import (func $f64_externref (param f64) (result externref)))

(event $returning (param f64))
(event $awaiting (param externref stackref))

(func $entry_async (export "entry") (param $input f64) (result externref)
  (block $returned
    (block $awaited
      (try
        (stack.switch_call $entry_root (local.get $input) (call $new_stack))
      catch $awaiting $awaited
      catch $returning $returned
      )
    ) ;; $awaited : [externref stackref]
    (return (call $create_promise))
  ) ;; $returned : [f64]
  (call $f64_externref)
)
```
Note that the first thing this function does is create a new stack (via `call $new_stack`).
Then it switches to that new stack and calls `$entry_root` (defined below) on it.
`$entry_root` essentially calls `$entry` (on this new stack).
If that call completes, it switches control back to the original stack using the `$returning` event.
`$entry_async` then catches that event and converts the returned `f64` to an `externref`.
If on the other hand the call in `$entry_root` has to await for some promise, it switches control back to the original stack using the `$awaiting` event.
The payload of this event is the promise to await for and the stack that is waiting for the resolution.
`$entry_async` catches that event and calls `$create_promise` (defined below) to return the desired promise to whomever called the (asynchronous) exported "entry" function.

The `$entry_root` function that runs on the newly created stack is defined as follows:
```
(import $externexn (param externref))

(call_tag $await (param externref) (result externref))
(event $resolving (param externref stackref))
(event $rejecting (param externref stackref))

(func $entry_root (param $input f64) (param $stack stackref) (local $output f64)
  (stack.redirect $stack
    (answer $await ;; [externref] -> [externref]
      (block $resolved
        (block $rejected
          (try
            (let (local $promise externref)
              (stack.switch $awaiting (local.get $promise) (local.get_clear $stack))
            )
          catch $resolving $resolved
          catch $rejecting $rejected
          )
        ) ;; $rejected : [externref stackref]
        (local.set_cleared $stack)
        (throw $externexn)
      ) ;; $resolved : [externref stackref]
      (local.set_cleared $stack)
    within
      (local.set $output (call $entry (local.get $input)))
    )
  )
  (stack.switch_drop $returning (local.get $output) (local.get_clear $stack))
)
```
The first instruction `$entry_root` really executes is the call to `$entry`.
However, it does so `within` an `answer $await`.
This `answer` enables `$entry` and its callees to use `call_stack $await : [externref] -> [externref]` to get (and wait for) the resolution of a promise.
The `answer` works by switching control back to the original stack, handing it the promise and the current stack (that is still in tact).
That `stack.switch` is configured to catch the `$resolving` event that is used to switch control back to `$entry_root`.
The payload of this event is the `externref` value the promise resolved to along with the stack that the promise is being resolved on.
That stack is stored in the local `$stack` variable, and the `externref` resolution is returned as the result of the `$await` inspection.
If instead the `$rejecting` event is caught, then the local `$stack` variable is similarly stored, and the contain `externref` error-value is thrown as an `$externref`, which is intended to be the event for specifically JavaScript exceptions.
Note that whenever control is inside the call to `$entry`, stack walks are being redirected to `$stack`.
This ensures that any (JavaScript) exceptions that occur during this call are redirected to the appropriate original stack.

All this is predicated on the existence of some imported `$create_promise : [externref, stackref] -> [externref]` function.
This function is defined in JavaScript as follows:
```
(promise, stack) => promise.then((x) => module_instance.resolve(x, stack),
                                 (e) => module_instance.reject(e, stack))
```
which in turn calls back into the module through two exported functions that are in charge of returning control to the given stack in the appropriate manner.

The "resolve" function below is used when the awaited promise resolves to a value.
Note that it looks very similar to `$entry_main` except that it uses the provided stack, rather than a new stack, and it uses the `$resolving` event expected by the code in the `answer $await` above.
```
(func (export "resolve") (param $resolution externref) (param $stack stackref) (result externref)
  (block $returned
    (block $awaited
      (try
        (stack.switch $resolving (local.get $resolution) (local.get_clear $stack))
      catch $awaiting $awaited
      catch $returning $returned
      )
    ) ;; $awaited : [externref stackref]
    (return (call $create_promise))
  ) ;; $returned : [f64]
  (call $f64_externref)
)
```

The "reject" function is used with the awaited promise results in an error.
Note that it looks very similar to "resolve"; the only difference is the event that is used to switch control.
The `answer $await` above will turn the given `$rejecting` event into an `$externexn` exception.
This is intended to be instantiated with whatever event represents specifically JavaScript exceptions.
As such, the thrown exception will get handled by the code in `$entry` just like any other JavaScript exception.
```
(func (export "reject") (param $error externref) (param $stack stackref) (result externref)
  (block $returned
    (block $awaited
      (try
        (stack.switch $rejecting (local.get $error) (local.get_clear $stack))
      catch $awaiting $awaited
      catch $returning $returned
      )
    ) ;; $awaited : [externref stackref]
    (return (call $create_promise))
  ) ;; $returned : [f64]
  (call $f64_externref)
)
```

And with that, we have an efficient implementation of the async/await pattern, with the only reliance on the JS API being to register the handlers on the given promise, and with the property that each JS event runs to completion (assuming it terminates).

### Variant

Many WebAssembly modules are compiled with the assumption that only one stack is live in any particular instance of the module at a time.
For this common case, there is a more efficient way to support asynchronous I/O if we consider a variant of `stack.redirect`.
Currently, `stack.redirect` specifies a local variable to use to determine where to redirect to when a stack walk occurs.
We could more generally have `stack.redirect` specify a computation to run to determine where to redirect to, as follows:
```
stack.redirect_to instr1* then instr2* within instr3* end : [ti*] -> [to*]
```
* where `instr1* : [] -> [stackref]` specifies the instructions to run to determine where to redirect to
* where `instr2* : [stackref] -> []` specifies the instructions to run after the redirection as done (returning the `stackref` that was redirected to)
* and `instr3*` : [ti*] -> [to*]` are the instructions whose stack walks get redirected.

Using this, `stack.redirect $local instr* end` is the special case `stack.redirect_to (local.get_clear $local) then (local.set_cleared $local) within instr* end`.
But now we can support other redirection patterns, such as getting/setting a mutable field of some heap reference, getting/setting an entry in a table, or getting/setting a *global* variable.
This last case is potentially quite useful to efficiently support asynchronous I/O.

If we are willing to assume we have a single live stack, then we can use global variables to store the relevant "host" stack and "application" stack:
```
(global $hoststack stackref)
(global $appstack stackref)
```
This means that, rather than having to do a stack inspection to update who the application stack should redirect to, we can use `stack.redirect_to` to make the application stack just always redirect to whatever stack is stored in `$hoststack`:
```
(func $entry_root (param $input f64) (param $stack stackref) (local $output f64)
  (global.set_cleared $hoststack (local.get $stack))
  (stack.redirect_to
    (global.get_clear $hoststack)
  then
    (global.set_clear $hoststack)
  within
    (local.set $output (call $entry (local.get $input)))
  )
  (stack.switch_drop $returning (local.get $output) (global.get_clear $hoststack))
)
```

Then rather than having programs await promises by performaing `call_stack $await`, we instead have them simply perform `call $await` using the following function:
```
(func $await (param $promise externref) (result externref)
  (block $resolved
    (block $rejected
      (try
        (stack.switch $awaiting (local.get $promise) (global.get_clear $hoststack))
      catch $resolving $resolved
      catch $rejecting $rejected
      )
    ) ;; $rejected : [externref stackref]
    (global.set_cleared $hoststack)
    (throw $externexn)
  ) ;; $resolved : [externref stackref]
  (global.set_cleared $hoststack)
)
```
This simply transfers control back to the `$hoststack`, but with the `$awaiting` event rather than the `$returning` event, and then restores the `$hoststack` variable when control as transferred back.
This is much faster than before because it no longer involves a stack inspection.

Lastly, we update the exported functions to use the new convention, namely storing the awaiting stack in the `$appstack` global variable rather than as part of the continuation:
```
(import (func $new_stack (result stackref)))
(import (func $create_promise (param externref) (result externref)))
(import (func $f64_externref (param f64) (result externref)))

(event $returning (param f64))
(event $awaiting (param externref stackref))

(func $entry_async (export "entry") (param $input f64) (result externref)
  (block $returned
    (block $awaited
      (try
        (stack.switch_call $entry_root (local.get $input) (call $new_stack))
      catch $awaiting $awaited
      catch $returning $returned
      )
    ) ;; $awaited : [externref stackref]
    (global.set_cleared $appstack)
    (return (call $create_promise))
  ) ;; $returned : [f64]
  (call $f64_externref)
)

(func (export "resolve") (param $resolution externref) (result externref)
  (block $returned
    (block $awaited
      (try
        (stack.switch $resolving (local.get $resolution) (global.get_clear $appstack))
      catch $awaiting $awaited
      catch $returning $returned
      )
    ) ;; $awaited : [externref stackref]
    (global.set_cleared $appstack)
    (return (call $create_promise))
  ) ;; $returned : [f64]
  (call $f64_externref)
)

(func (export "reject") (param $error externref) (result externref)
  (block $returned
    (block $awaited
      (try
        (stack.switch $rejecting (local.get $error) (global.get_clear $appstack))
      catch $awaiting $awaited
      catch $returning $returned
      )
    ) ;; $awaited : [externref stackref]
    (return (call $create_promise))
  ) ;; $returned : [f64]
  (call $f64_externref)
)
```
and the revise the imported function `$create_promise : [externref] -> [externref]` similarly as follows:
```
(promise) => promise.then((x) => module_instance.resolve(x),
                          (e) => module_instance.reject(e))
```

Beyond providing a useful optimization, this variant illustrates the substantial flexibility this proposal provides applications for determining how best to implement stack-based features according to their specific circumstances.

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

* `stackref`: reference to a complete stack

### Forms

* `rout`: variant of `func` using `stack.start`

#### Instructions

##### `stack.extend`

```
stack.extend $rout $event? : [ti* stackref] -> [stackref]
```
where `rout $rout : [ti*] -> [to*]` and `event $event : [to*]` (if specified)

Adds a stack frame to the given `stackref`, returning the reference to newly extended stack (and trapping if the given `stackref` is null).
The data content of the stack frame as given by the `ti*` values.
The code content of the stack frame (i.e. the address of the handler code) is given by the `$rout`, specifically by the location of `stack.start` in the definition of `$rout`.
Because the given `stackref` was expecting an event, the optional `$event` specifies what event to use if/when the `$rout` returns to it.
If no event is specified, then the `$rout` traps if/when it attempts to return to its parent stack frame.

##### `stack.redirect`

```
stack.redirect $local instr* end : [ti*] -> [to*]
```
where `local $local : stackref` and `instr* : [ti*] -> [to*]`

During the execution of `instr*`, whenever a stack walk reaches `stack.redirect`, the stack walk is redirected to the `stackref` in `$local` at that time.
If that value is null, then the stack walk continues up the current stack.
Otherwise, the value `$local` is set to null (to prevent the `stackref` from being switched to while it is being walked).
When the stack walk completes, `$local` is set to its former value (trapping if `$local` is not null).

As a technical note, if a second stack walk reaches `stack.redirect` while it is already redirecting a stack walk, then the second stack walk is redirected to the same stack.
This can only happen if the first stack walk initiates the second stack walk, so this is a bit of a corner case.
That fact guarantees, though, that the second stack walk cannot complete before the first stack walk completes.

##### `stack.start`

```
stack.start : [] -> unreachable
```
(usable only in specific locations within a `rout`)

This special instruction serves as a marker indicating where a `rout` starts.
It can only be used within a `rout` and can only be preceded by a few kinds of instructions, e.g. `block`, `try`, and `stack.redirect`.
It throws whatever event is used to transfer control to the stack that the `rout` was mounted onto.

##### `stack.switch`

```
stack.switch $event : [t* stackref] -> unreachable
```
where `event $event : [t* stackref]`

Switches control to the given `stackref` (trapping if null), providing the specified `$event` with the given `t*` as all but the last value in its payload.
The last value of the payload is the reference to the stack that yielded control (*not* the value of the given `stackref`).

When the stack that yielded control is switched back to with some event, that event (*not* `$event`) is thrown from `stack.switch`.

##### `stack.switch_call`

```
stack.switch_call $func $event? : [ti* stackref] -> unreachable
```
where `func $func : [ti* stackref] -> [to*]` and `event $event : [to*]` (if specified)

Switches control to the given `stackref` (trapping if null) and then calls `$func` on that stack, passing the given `ti*` as all but the last argument to the function.
The last argument is the reference to the stack that yielded control (*not* the value of the given `stackref`).
Because the given `stackref` was expecting an event, the optional `$event` specifies what event to use if/when the `$func` returns to it.
If no event is specified, then the `$func` traps if/when it attempts to return to its parent stack frame.

When the stack that yielded control is switched back to with some event, that event (*not* `$event`) is thrown from `stack.switch_call`.

##### `stack.switch_drop`

```
stack.switch_drop $event : [t* stackref] -> unreachable
```
where `event $event : [t*]`

Switches control to the given `stackref` (trapping if null), providing the specified `$event` with the given `t*` as its payload.
But before that event is handled on the given `stackref`, the stack that yielded control is first cleaned up.

### Linear Types

These instructions are used to support linear types.
They all either clear (i.e. set to `null`) a store after getting its value, or check that a store is cleared before setting it to a value.

##### `global.get_clear`

```
global.get_clear $global : [] -> [t]
```

##### `global.set_cleared`

```
global.set_cleared $global : [t] -> []
```

##### `local.get_clear`

```
local.get_clear $local : [] -> [t]
```

##### `local.set_cleared`

```
local.set_cleared $local : [t] -> []
```

##### `table.get_clear`

```
table.get_clear $table : [i32] -> [t]
```

##### `table.set_cleared`

```
table.set_cleared $table : [i32 t] -> []
```

### Stack Inspection

We list only the features that we would need from a complete suite of operations for stack inspection.

##### `answer`

```
answer $call_tag instr1* within instr2* end : [ti2*] -> [to2*]
```
where `call_tag $call_tag : [ti1*] -> [to1*]` and `instr1* : [ti1*] -> [to1*]` and `instr2* : [ti2*] -> [to2*]`

Executes `instr2*`, during which answering any `call_stack $call_tag` that reaches this `answer` by executing `instr1*`.

##### `call_stack`

```
call_stack $call_tag : [ti*] -> [to*]
```
where `call_tag $call_tag : [ti*] -> [to*]`

Looks up the current stack for an `answer $call_tag` and executes it with the given `ti*` as inputs, returning here with the `to*` it outputs.

## Appendix: Frequently Asked Questions (FAQ)

### How does this relate to other proposals?

There are three prior proposals-of-sorts on this topic, each of which had significant influence on the design we developed:
1. [Andreas Rossberg's presentation at the Feb 2020 In-Person CG Meeting](https://github.com/WebAssembly/meetings/blob/master/main/2020/presentations/2020-02-rossberg-continuations.pdf) inspired our heavy use of events
2. [Ross Tate's primer on low-level stack primitives](https://github.com/WebAssembly/exception-handling/issues/105) inspired our considerations for stack walking
3. [Alon Zakai's and Ingvar Stepanyan's proposal for await](https://github.com/WebAssembly/design/issues/1345) inspired our attention to host needs regarding *grounding* frames

One major difference between this proposal and all priors is our `stackref` values being *complete* stacks.
This enables us to quickly switch between stacks, whereas all prior proposals required employing a stack walk to first detach a stack *slice* and then attaching a different stack slice.
Looking at related systems, we found that `stackref` enabled a much more efficient stack-switching mechanism for applications more centered around cooperative multi-tasking, and instead broke down the process of detaching and attaching stack slices into the combination of stack inspection and stack-walk redirection.

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

The entire proposal is dependent on [exception handling](https://github.com/WebAssembly/exception-handling/), and the composable-stacks portion is dependent on stack inspection.
There is no dependency on [garbage collection](https://github.com/WebAssembly/exception-handling/), though garbage collection would have to be amended to accommodate linear types.

### What are the implications for JavaScript and browser interoperability?

This proposal was designed to not introduce any significant complications with JavaScript interoperability.
Browsers already need to deal with computations being suspended midflight with their stacks put aside until later due to preemptive multi-tasking; this proposal just enables that suspension to occur (cooperatively) within a thread.
Yes, that includes switching between stacks while processing a message on the event loop, but by not providing a way to create a `stackref` out of thin air, (and presumably by not allowing stacks to be passed within messages to other threads,) the design ensures that all these stacks are grounded to the event loop.
As such, there is no way to use this proposal to somehow escape the event loop.

Of course, just like JavaScript programs can run an infinite while loop, WebAssembly programs can switch between stacks infinitely.
That is, just like JavaScript developers, any design using this proposal needs to be conscious of the run-to-completion model.
For example, our thread-manager example does not complete until the work is completely done, but it would be easy to modify it to have the scheduler return to the caller once in a while, relying on the caller to schedule an event that will call back into the instance and resume the scheduler.

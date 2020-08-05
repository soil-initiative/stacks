# Case Study: Cont with headers

Here we consider how this proposal supports Multicore OCaml's implementation strategy for the (revised) `cont` design in [WebAssembly/design#1359](https://github.com/WebAssembly/design/issues/1359) using headers for each continuation.

## Background

A key detail of given implementation strategy is the assumption that one can easily determine the address of the root of a continuation from the address of its leaf.
This assumption is reasonable, but it should be made explicit since it is an implementation constraint.
What is important is that under this assumption we can provide a variant of stack inspection, which is also how we can make expose the benefits of this assumption to WebAssembly in a composable manner that does not expose underlying implementation details.
This variant differs from normal stack inspection only in terms of how it is implemented, meaning there is no semantic difference, just performance tradeoffs.

Normal stack inspection identifies marks lazily.
That is, stack marks (e.g. `answer`s) are found on demand by walking through the frames of the stack and checking whether return addresses are within certain code ranges.
This is ideal useful for frequent marks that are typically iterated through during a relevant stack inspection, like those for stack tracing, linear-memory garbage collection, or debugging.

But for infrequent marks for, especially ones where a stack inspection typically stops at the first relevant mark, it can be useful to track marks eagerly.
That is, every time control enters a marked code region, a table is updated to indicate the current stack frame as providing the most-immediate relevant handler for that mark.
The stack frame stores the old handler (if any) so that the table can be restored when control exits the marked code region.
This table is specific to the current stack, so the ideal place to store it is above the root of the stack *if you can find the root of a stack conveniently from its leaf*.

So for this case study, we will preface stack-inspection instructions with `eager` to indicate when to update/access this table.

## Events

This translation is specifically optimized towards all event handlers using continuations.
For each (bidirectional) event `$evt : [tp*] -> [tr*]`, this translation has a corresponding (unidirectional/exception) `(event $inputs<$evt> (param tp*))`, and we use the (outdated) `exnref` type to package exception events with payloads as memory-managed values using an instruction `exnref.new`.
The overhead from packaging and unpackaging these values, however, should be viewed as being due to impedance mismatch rather than fundamental.

Note that we will be using `<...>` as templating notation, though purely as a pedagogical device rather than as part of any proposal.

## Types

The type `(cont ([tp*] -> [tr*]))` translates to the type `$cont = ref (struct $header (mut stackref))`, where `$header = ref (struct (mut funcref) (mut $header) (mut stackref))`.
The first field indicates the "header" of the continuation, and the second field indicates the `stackref` of the continuation.

The header is comprised of three fields.
When a continuation is detached, these fields are all null.
They are placeholders for information about the parent continuation that it gets attached to.
The `funcref` field implements the event-handler information (which in a real translation be some sort of data structure, but a `funcref` will do for our illustrative purposes).
The `$header` field is the parent's own header (to be used when a suspension event is not supposed to be handled according to the `funcref`).
The `structref` field is the parent's stack (to be used when a suspension event is supposed to be handled according to the `funcref`, or when the continuation returns or throws an exception).

Note that `$cont` has no type parameters.
We found that using input types `tp*` constrained stack-switching patterns.
So a `(cont ([tp*] -> [tr*]))` corresponds to a stack that is waiting for a specific "resuming" input-only event `(event $resuming<tp*> (param tp*))`.
We also do not use return types, since "returning" from one stack to another is just a special case of a stack switch that cleans up the former stack (i.e. `stack.switch_drop`).
So a `(cont ([tp*] -> [tr*]))` corresponds to a stack that "ends" with a specific "returning" input-only event with payload `tr*`, which we will denote with `(event $returning<tr*> (param tr*))`.

## Creating a continuation

When we need a "canonical" call tag for a signature, we will denote it with `$canon_tag<tp*;tr*> : [tp*] -> [tr*]`.

To support the following function for stack creation:
```
cont.new : [(ref ([tp*] -> [tr*]))] -> [(cont ([tp*] -> [tr*]))]
```
we use the following (assuming `$new_stack : [] -> [stackref]` has been imported):
```
(struct.new_default $header)
(let (local $h $header)
  (struct.new $cont
    (local.get $h)
    (stackref.extend $cont_rout<tp*;tr*> (local.get $h) (call $new_stack))
  )
)
```
where the `rout` template `$cont_rout<tp*;tr*>` is defined as follows:
```
(call_tag $get_header (result $header))

(rout $cont_rout<tp*;tr*> (param $h $header) (param $f funcref)
  (stack.redirect_to ;; redirect any thrown exceptions to the parent
    (struct.get_clear 2 (local.get $h)) ;; [] -> [stackref]
  then
    (struct.set_cleared 2 (local.get $parent)) ;; [stackref] -> []
  within
    (eager answer $get_header ;; [] -> [$header]
      (local.get $h)
    within
      (block $resumed
        (try
          stack.start
        catch $resuming<tp*> $resumed
        )
      ) ;; $resumed : [tp*]
      (call_funcref $canon_tag<tp*;tr*> $f) ;; [tp*] -> [tr*]
    )
  )
  (stack.switch_drop $returning<tr*> (struct.get_clear 2 (local.get $h)))
)
```

`$cont_rout` has a lot of machinery because a `cont` is many concepts bundled together.

First, note that `stack.start` is set up to receive an `$resuming<tp*>` event.
Once that happens, `$cont_rout` will then proceed to call the given function reference.
After that call completes, `$cont_rout` then switches back to the parent stack with the `$returning<tr*>` event.

The entire time, `$cont_rout` is set up to forward any exceptions that get thrown to the parent stack.
This is done using a more general variant of `stack.redirect` that is presented [here](https://github.com/soil-initiative/stacks/pull/9/files), which in particular allows one to use a mutable field rather than a local variable to determine where to redirect to.

Most importantly for this translation, `$cont_rout` immediately sets up an `eager` [answer](https://github.com/WebAssembly/design/issues/1356) to `$get_header` (and there will never be other answers to `$get_header` in this stack).
This means that, upon creation of this new continuation, there will be a fast way to access its `$header` any time during its execution.
This provides the critical functionality Multicore OCaml uses in its implementation of continuations.

## Suspending a continuation

The `cont` design suspends the current continuation with the following instruction:
```
cont.suspend $evt : [tp*] -> [tr*]
```
where `event $evt : [tp*] -> [tr*]`.
Note that this instruction has no direct access to where to suspend to.
That information is implicit and has to be found somewhere.
In particular, it depends on the evaluation context that the instruction is executed in.

Because of this, we translate this instruction to the following:
```
(block $resumed ;; [tp*] -> [tr*]
  (exnref.new $inputs<$evt>)
  (eager call_stack $get_header)
  (try
    (call $suspend)
    (unreachable)
  catch $resuming<tp*> $resumed
  end)
)
```
where
```
(call_tag $handle (param exnref $header))

(func $suspend (param $e exnref) (param $h $header)
  (loop $search ;; [] -> []
    (call_funcref $handle (local.get $e) (local.get $h) (struct.get 0 (local.get $h)))
    (local.set $h (struct.get 1 (local.get $h)))
    (br $search (local.get $result))
  )
)
```
This translation makes it clear that the first thing a suspension involves is a stack inspection before there can be an actual stack switch.
But in this translation, that stack inspection is `eager` and effectively jumps to the root of the continuation to fetch its `$header`.
From then on, that `$header` is used to implement the suspension via a call to `$suspend`, which eventually throws the `$resuming<tp*>` exception when the continuation is resumed.

`$suspend` uses the information in the `$header` to then determine which ancestor to suspend to.
In particular, the `$header` is a linked list, and `$suspend` iterates through this linked list.
In each iteration, it passes the inputs to the event at hand to the `funcref` in the `$header`.
The expectation is that this `funcref` will perform the actual stack switching if the event matches, letting the eventual `$resuming<tp*>` switch-back event be propagated up the stack as an exception.
But if the event doesn't match, then the `funcref` simply returns, prompting the search to continue.

## Resuming a continuation

The `cont` design resumes a continuation on top of the current continuation with the following instruction:
```
cont.resume (event $evt $label)* : [tp* (cont ([tp*] -> [tr*]))] -> [tr*]
```
where `event $evt : [ti*] -> [to*]` and `label $label : [ti* (cont ([to*] -> [tr*])]` for each event-label pair in the given list.
This resumes the given continuation, which is then detached if suspended with an event in the given list, at which point the appropriate label is given the inputs to the event along with the suspended continuation.

We can translate this to the following, for simplicity presenting the case of a single event-label pair:
```
(block $returned ;; [tp* $cont] -> [tr*]
  (let (local $c $cont)
    (struct.get 0 (local.get $c))
    (struct.get_clear 1 (local.get $c))
    (let (local $h $header) (local $s stackref)
      (struct.set 0 (local.get $h) (ref.func $handler<$evt>))
      (struct.set 1 (local.get $h (eager call_stack $get_header)))
      (block $detached ;; [tp*] -> [ti* stackref]
        (try
          (local.get $h)
          (local.get_clear $s)
          (stack.switch_call $resume_set_parent<tp*> $resuming<tp*>)
        catch $detaching<$evt> $detached
        catch $returning<tr*> $returned
        )
      ) ;; $detached : [ti* stackref]
      (local.set_cleared $s)
      (struct.new $cont (local.get $h) (local.get_clear $s))
      (br $label)
    )
  )
) ;; $returned : [tr*]
```
where
```
(event $detaching<$evt> (param ti* stackref)) ;; where $evt : [ti*] -> [to*]
(func $handler<$evt*> (param $e exnref) (param $h $header) (tag $handle)
  (block $missed
    (block $handled
      (br_on_exn $inputs<$evt> (local.get $e))
      (br $missed)
    ) ;; $handled : [to*] where $evt : [ti*] -> [to*]
    (struct.set 0 (local.get $h) (funcref.null))
    (struct.set 1 (local.get $h) ($header.null))
    (struct.get_clear 2 (local.get $h))
    (stack.switch $detaching<$evt>)
  )* ;; repeat for each $evt
)
(func $resume_set_parent<tp*> (param $p tp)* (param $h $header) (param $s stackref) (result tp)*)
  (struct.set_cleared 2 (local.get $h) (local.get_clear $s))
  (local.get $p)*
)
```

The instructions for `cont.resume` first configure the header of the child continuation.
They set the `funcref` to refer to `$handler<evt*>`, which checks if the `evtref` corresponds to one of the events in the list and in which case switches after clearing out the given header.
And they set the `$header` to the header of the current continuation, again using an `eager` stack inspection to effectively jump straight to the root and fetch it.
After configuring the header, they then switch to the child continuation, storing the `stackref` of the parent configuration into its header in the process.
If that child detaches, its header and its new `stackref` are packaged into a new continuation and forwarded to the specified `$label`.

## Aborting a continuation

When one no longer needs a continuation, one often explicitly "aborts" it in order to run unwinders on the stack to release resources.
The `cont` design does this with the following instruction:
```
cont.throw $exn : [te* (cont ([tp*] -> [tr*]))] -> [tr*]
```
Ideally this instruction would also specify an event-label list for any events the aborting continuation might suspend with, either during aborting or after catching the given exception.
But the `cont` design seems to consider only the simplest case here.

We can translate this to the following:
```
(block $returned ;; [te* $cont] -> [tr*]
  (let (local $c $cont)
    (struct.get 0 (local.get $c))
    (struct.get_clear 1 (local.get $c))
    (let (local $h $header) (local $s stackref)
      (struct.set 0 (local.get $h) (ref.func $handler<>))
      (struct.set 1 (local.get $h (eager call_stack $get_header)))
      (try
        (local.get $h)
        (local.get_clear $s)
        (stack.switch_call $abort_set_parent<te*> $exn)
      catch $returning<tr*> $returned
      )
    )
  )
) ;; $returned : [tr*]
```
where
```
(func $abort_set_parent<te*> (param $p te)* (param $h $header) (param $s stackref) (result te)*)
  (struct.set_cleared 2 (local.get $h) (local.get_clear $s))
  (local.get $p)*
)
```

Note that we use the empty `$handler<>` and similarly do not handle any `$detaching<...>` switch events because they will never occur.

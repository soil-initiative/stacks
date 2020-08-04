# Case Study: Cont using stack inspection

Here we consider how this proposal supports the concepts of the `cont` design in [WebAssembly/design#1359](https://github.com/WebAssembly/design/issues/1359) using a stack-inspection-based implementation.

## Background

Before we can discuss how we support the `cont` design, we first need to discuss some of the problems with the `cont` design.
Consider the following snippet from the example thread scheduler:
```
(block $on_event (result (evtref))
  (cont.resume $on_event (event $yield $spawn) (call $dequeue))
  (br $l)
)
(switch-on-evtref
  (case $yield ;; cont on stack
    (call $enqueue)
  )
  (case $fork ;; proc and cont on stack
    (cont.new) (call $enqueue)
    (call $enqueue)
  )
  (default ;; evtref on stack
    (cont.forward)
  )
)
```
Note that this design *always* "detaches" the continuation for every relevant event, runs some code to figure out if the detachment was actually wanted, and if not it "reattaches" the continuation and continues searching for the correct detachment point.
In this toy example, whether to detach can be determined based on purely static information, i.e. the event tag.
But in more advanced applications, one needs to consider dynamic information in this process as well.
Detaching and reattaching involves a reasonable amount of paperwork, so it would be better to do this computation of whether to detach *before* detaching.

Because of that, we found it better to put the code for whether to detach into the continuation potentially being detached.
It also avoids the need to allocate an `evtref` in the heap just to store information that will be discarded almost immediately.

So if at certain points the translation we provide seems a little convoluted, that is because the design we are translating is not aligned well with efficient implementation.

### Fixing the design and example

It is worth noting that the design of `cont.forward` is broken.
There is nothing specifying which label should specify how to handle further events spurred by the reattached continuation in the future.
The `cont` design seems to assume here that the `evtref` will always be forwarded in a particular fashion that the design in no way guarantees.
One could return the `evtref` out of the thread-scheduling function entirely and `cont.forward` it in an entirely different context where the label `$on_event` is no longer valid.
So, to make the `cont` design actually work, `cont.forward` needs to specify a label and list of events just like `cont.resume`.
Also, the continuation inside `cont.forward` can still later return after its (forwarded) event is handled and the continuation resumed, so it needs a return type rather than `unreachable`.
Then, to fix this example, we have to introduce a loop:
```
(block $on_event (result (evtref))
  (cont.resume $on_event (event $yield $spawn) (call $dequeue))
  (br $l)
)
(loop $on_event
  (switch-on-evtref
    (case $yield ;; cont on stack
      (call $enqueue)
    )
    (case $spawn ;; proc and cont on stack
      (cont.new) (call $enqueue)
      (call $enqueue)
    )
    (default ;; evtref on stack
      (cont.forward $on_event (event $yield $spawn))
    )
  )
  (br $l)
)
```

Alternatively, it could be that `cont.forward` is not a renaming of `cont.reyield` from the in-person meeting and instead is meant to drop the current continuation and then forward the event to its parent.
But there is no discussion on how one would unwind the current continuation.
Given that the instruction is not even reachable in the only example provided, we have little information with which to work with.

## Events

Ideally an event `(event $evt (param tp*) (result tr*))` in the `cont` design would translate to a [call tag](https://github.com/WebAssembly/call-tag): `(call_tag $evt (param tp*) (resulttr*))`, and similarly the instruction `cont.suspend $evt : [tp*] -> [tr*]` would translate to `call_stack $evt`.
This would open up the possibility that the relevant `answer` for the `$evt` could be more efficiently provided without any stack switch at all.
But that possibility is currently inexpressible in the `cont` design, illustrating one of its shortcomings.
Furthermore, the `br_on_evt` and `evtref` aspect of the `cont` design makes it poorly aligned with stack inspection in the future.

So instead, for each (bidirectional) event `$evt : [tp*] -> [tr*]`, our translation has corresponding (unidirectional/exception) events `$inputs<$evt> : [tp*]` and `$outputs<$evt> : [tr*]`, and we use the (outdated) `exnref` type to package exception events with payloads as memory-managed values using an instruction `exnref.new`.

### Suspending a continuation

The `cont` design suspends the current continuation with the following instruction:
```
cont.suspend $evt : [tp*] -> [tr*]
```
where `event $evt : [tp*] -> [tr*]`.
Note that this instruction has no direct access to where to suspend to.
That information is implicit and has to be found somewhere.
In particular, it depends on the evaluation context that the instruction is executed in.

Because of this, we translate this instruction to the following (using the call tag `$suspend : [exnref] -> [exnref]`):
```
(block $resumed ;; [tp*] -> [tr*]
  (exnref.new $inputs<$evt>)
  (call_stack $suspend)
  (br_on_exn $outputs<$evt> $resumed)
  (unreachable)
)
```
This translation makes it clear that a suspension involves a stack inspection before there can be an actual stack switch.
In the rest of the translation, we will see that this inspection finds the root of the current continuation, whereupon it switches to the continuation's parent.

## Types

The type `(cont ([tp*] -> [tr*]))` translates to the type `$cont = ref (struct $hook (mut stackref))`, where `$hook = ref (struct (mut stackref))`.
The first field indicates the "parent" reference to update when one "attaches" the continuation, and the second field indicates the `stackref` to transfer control to.

Note that `$cont` has no type parameters.
We found that using input types `tp*` constrained stack-switching patterns.
So a `(cont ([tp*] -> [tr*]))` corresponds to a stack that is waiting for a specific "attaching" input-only event `(event $attaching (param exnref))`, where the `exnref` is in turn a "resuming" event `(event $resuming<tp*> (param tp*))`.
We also do not use return types, since a return is implemented as a stack switch that cleans up the former stack (i.e. `stack.switch_drop`).
So a `(cont ([tp*] -> [tr*]))` corresponds to a stack that "ends" with a specific "returning" input-only event with payload `tr*`, which we will denote with `(event $returning<tr*> (param tr*))`.

Note that, to bridge this typed-untyped mismatch, we will be using this templating notation `<...>` a bunch, though purely as a convenience rather than part of any proposal.

### Creating a new continuation

When we need a "canonical" call tag for a signature, we will denote it with `$canon<tp*;tr*>`.

To support the following function for stack creation:
```
cont.new : [(ref ([tp*] -> [tr*]))] -> [(cont ([tp*] -> [tr*]))]
```
we use the following (assuming `$new_stack : [] -> [stackref]` has been imported):
```
(struct.new $hook)
(let (local $h $hook)
  (struct.new $cont
    (local.get $h)
    (stackref.extend $cont_rout<tp*;tr*> (local.get $h) (call $new_stack))
  )
)
```
where the `rout` template `$cont_rout<tp*;tr*>` is defined as follows:
```
(call_tag $canon<tp*;tr*> (param tp*) (result tr*))
(call_tag $forward (param exnref $hook stackref))
(event $detaching : [exnref stackref])

(rout $cont_rout<tp*;tr*> (param $parent $hook) (param $f funcref)
  (stack.redirect_to ;; redirect any thrown exceptions to the parent
    (struct.get_clear 0 (local.get $parent)) ;; [] -> [stackref]
  then
    (struct.set_cleared 0 (local.get $parent)) ;; [stackref] -> []
  within
    (block $resumed
      (block $attached
        (try
          stack.start
        catch $attaching $attached
        )
      ) ;; $attached : [exnref]
      (br_on_exn $resuming<tp*> $resumed)
      (unreachable)
    ) ;; $resumed : [tp*]
    (answer $suspend ;; [exnref] -> [exnref]
      (block $attached
        (try
          (struct.get_clear 0 (local.get $parent))
          (stack.switch $detaching)
        catch $attaching $attached
        )
      ) ;; $attached : [exnref]
    within
      (answer $forward ;; [exnref $hook stackref] -> []
        (struct.get_clear 0 (local.get $parent))
        (stack.switch_call $detach_forward $detaching)
      within
        (call_funcref $canon<tp*;tr*> $f) ;; [tp*] -> [tr*]
      )
    )
  )
  (stack.switch_drop $returning<tr*> (struct.get_clear 0 (local.get $parent)))
)

(func $detach_forward (param $e exnref) (param $h $hook) (param $leaf stackref) (param $self stackref) (result exnref stackref)
  (struct.set_cleared 0 (local.get $hook) (local.get_clear $self))
  (local.get $e)
  (local.get $leaf)
)
```

`$cont_rout` has a lot of machinery because a `cont` is many concepts bundled together.

First, note that `stack.start` is set up to receive an `$attaching` event.
Once that happens, `$cont_rout` will then proceed to call the given function reference.
After that call completes, `$cont_rout` then switches back to the parent stack with the `$returning<tr*>` event.

The entire time, `$cont_rout` is set up to forward any exceptions that get thrown to the parent stack.
This is done using a more general variant of `stack.redirect` that is presented [here](https://github.com/soil-initiative/stacks/pull/9/files), which in particular allows one to use a mutable field rather than a local variable to determine where to redirect to.

During the call to the function reference, an `answer` to `$suspend` is set up, as is an `answer` to `$forward`.
These answers switch control back to the parent using the `$detaching` event, whose payload indicates the suspending event and the continuation to later resume.
The answer to `$suspend` uses `stack.switch` to provide itself as the continuation to later resume.
The answer to `$forward` uses `stack.switch_call $detach_forward` to set the forwarding stack as the parent in the given `$hook` and then provides the given `stackref` as the continuation to later resume.
The answer to `$suspend` waits for the `$attaching` event to provide the answer to the inquiry.
The answer to `$forward` waits for no events because it is expected to be called in a context that knows what events to handle.
(Interestingly, this is the first example we have where a stack switch does not have the handlers for intended *non-aborting* events specified in the immediate scope.)

## Resuming a continuation

The `cont` design resumes a continuation on top of the current continuation with the following instruction:
```
cont.resume $label (event $evt*) : [tp* (cont ([tp*] -> [tr*]))] -> [tr*]
```
where `label $label : [(evtref tr*)]`.
The list of events `$evt*` is provided to enable some static filtering on the event before detaching.
We will put this aside for now and discuss it later when we discuss other optimizations.

The label `$label` expects an `evtref tr*`.
We translate this type to `$evtref = ref (struct exnref $cont)`.

We can translate this to the following:
```
(block $returned ;; [tp* $cont] -> [tr*]
  (let (local $c $cont)
    (block $detached
      (try
        (exnref.new $resuming<tp*>)
        (struct.get 0 (local.get $c))
        (struct.get_clear 1 (local.get $c))
        (stack.switch_call $attach_set_parent $attaching)
      catch $detaching $detached
      catch $returning<tr*> $returned
      )
    ) ;; $detached : [exnref stackref]
    (let (local $e exnref) (local $s stackref)
      (struct.new $evtref (local.get $e)
                          (struct.new $cont (struct.get 0 (local.get $c)))
                                            (local.get_clear $s))
      (br $label)
    )
  )
) ;; $returned : [tr*]
```
where
```
(func $attach_set_parent (param $e exnref) (param $parent $hook) (param $s stackref) (result exnref))
  (struct.set_cleared (local.get $parent) (local.get_clear $s))
  (local.get $e)
)
```

These instructions switch control to the `stackref` representing the continuation, using `stack.switch_call $attach_set_parent` to set the parent of the continuation to be the current stack.
They then wait for that `stackref` to either "detach" (via the `$detaching` event) or to "return" (via the `$returning<tr*>` event).
Note that the handler for detaching branches to `$label` and makes the allocation of the `evtref` explicit.

## Handling an event

After the continuation has been detached, the `cont` design switches on its detachment event with the following instruction:
```
br_on_evt $label (event $evt) : [(evtref t*)] -> [(evtref t*)]
```
where `event $evt : [tp*] -> [tr*]` and `label $label : [tp* (cont ([tr*] -> [t*]))]`.

We can translate this to the following:
```
(block $mismatched ;; [$evtref] -> [$evtref]
  (let (local $e $evtref)
    (block $matched
      (br_on_exn $matched $inputs<$evt> (struct.get 0 (local.get $e)))
      (local.get $e)
      (br $mismatched)
    ) ;; $matched : [tp*]
    (struct.get 1 (local.get $e))
    (br $label)
  )
) ;; $mismatched : [$evtref]
```

## Forwarding an event

The `cont` design expects a specific pattern: detach when an event occurs, examine the event, forward it if is not an event you are looking for.
The final step is done with the following (fixed) instruction:
```
cont.forward $label (event $evt*) : [(evtref tr*)] -> [tr*]
```
where `label $label : [(evtref tr*)]`.

We can translate this to the following:
```
(block $returned ;; [$evtref] -> [tr*]
  (let (local $e $evtref)
    (block $detached
      (try
        (call_stack $forward (struct.get 0 (local.get $e))
                             (struct.get 0 (struct.get 1 (local.get $e)))
                             (struct.get_clear 1 (struct.get 1 (local.get $e))))
      catch $detaching $detached
      catch $returning<tr*> $returned
      )
    ) ;; $detached : [exnref stackref]
    (let (local $ex exnref) (local $s stackref)
      (struct.new $evtref (local.get $ex)
                          (struct.new $cont (struct.get 0 (struct.get 1 (local.get $e)))
                                            (local.get_clear $s))
      (br $label)
    )
  )
) ;; $returned : [tr*]
```
This illustrates that `cont.forward` does a stack inspection to find the current continuation's parent to forward the event to.
It also illustrates how we make use of the fact that a `call_stack` can throw exceptions just like a normal `call`.
In particular, the handler for `$detaching` should be specified here, rather than within an `answer` to `$forward`, because it (after some bookkeeping) jumps to a locally specified label.

## Aborting a continuation

When one no longer needs a continuation, one often explicitly "aborts" it in order to run unwinders on the stack to release resources.
The `cont` design does this with the following instruction:
```
cont.throw $exn : [te* (cont ([tp*] -> [tr*]))] -> [tr*]
```
Unfortunately, it is not entirely clear what the semantics of this instruction is.
It is clearly meant to throw an `$exn` in the continuation, but what happens if that exception is caught and later the continuation suspends with some event?
As with `cont.forward`, there should generally be a label to specify what the event handler should be, but based on discussion it sounds like the expectation is that all events will always be forwarded.

Fortunately, it is fairly easy for this proposal to support all of variations of what its semantics should be, but for now we will go with expectation that all events are always forwarded:
```
(block $returned ;; [te* $cont] -> [tr*]
  (let (local $c $cont)
    (block $detached
      (try
        (struct.get 0 (local.get $c))
        (struct.get_clear 1 (local.get $c))
        (stack.switch_call $abort_set_parent<te*> $exn)
      catch $detaching $detached
      catch $returning<tr*> $returned
      )
    ) ;; $detached : [exnref stackref]
    (loop $detached
      (let (local $e exnref) (local $s stackref)
        (try
          (call_stack $forward (local.get $e) (struct.get 0 (local.get $c)) (local.get_clear $s))
        catch $detaching $detached
        catch $returning<tr*> $returned
        )
      )
    )
  )
) ;; $returned : [tr*]
```
where
```
(func $abort_set_parent<te*> (param $p* te*) (param $parent $hook) (param $s stackref) (result te*))
  (struct.set_cleared (local.get $parent) (local.get_clear $s))
  (local.get $p*)
)
```

Notice the `loop`.
This illustrates that even though this stack is not handling any of the events, it does have to repeatedly forward them until eventually the continuation terminates.

## Optimizing by filtering events

Lastly, one item we overlooked is the use of an event list to filter detachments based on at least static information.
We can implement this optimization by adding a `(mut funcref)` field to the `$hook` structure.
Whenever a stack sets itself as a parent in such a `$hook`, it would also set this field to a reference to the filtering function to use.
The `answer` for `$suspend` and for `$forward` would then first execute this function on the available `exnref`.
If the filter excludes the event at hand, then they would instead redirect a `call_stack $suspend` or `call_stack $forward` to the parent stack.

# Case Study: Cont using thread-local storage

Here we consider how this proposal supports the concepts of the `cont` design in [WebAssembly/design#1359](https://github.com/WebAssembly/design/issues/1359) using thread-local storage.
That is, rather than use stack inspection to find a handler on demand, this translation uses thread-local storage to eagerly track where the current handler for each event is.
Of course, this assumes WebAssembly has been extended with thread-local storage, which we suppose is provided with the following instructions:
* `tls.get $event : [] -> [t*]` where `event $event : [t*]`
* `tls.put $event : [t*] -> []` where `event $event : [t*]`

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
  (case $spawn ;; proc and cont on stack
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
    (case $fork ;; proc and cont on stack
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

An event `(event $evt (param tp*) (result tr*))` in the `cont` design translates to a [call tag](https://github.com/WebAssembly/call-tag): `(call_tag $evt (param tp*) (resulttr*))`.

For this translation, we will have an `(event $handler<$evt> (param $hook))` used to track the current handler of an event in thread-local storage, where `$hook = ref (struct (mut stackref))`.
The understanding is that the `stackref` in this `$hook` is expecting an `(event $detaching : [exnref funcref stackref])` where
* the `exnref` in the payload is an `(event $inputs<$evt> (param tp*))`,
* the `funcref` in the payload is expecting a `(call_tag $get_handler_tag (result stackref))`,
* and the `stackref` in the payload is expecting an `(event $attaching : [exnref])`, where the `exnref` is an `(event $resuming<tr*> (param tr*))`.

Note that we are using `<...>` as a templating notation, though purely as a convenience rather than part of any proposal.

### Suspending a continuation

The `cont` design suspends the current continuation with the following instruction:
```
cont.suspend $evt : [tp*] -> [tr*]
```
where `event $evt : [tp*] -> [tr*]`.
Note that this instruction has no direct access to where to suspend to.
That information is implicit and has to be found somewhere.
While that information conceptually depends on the evaluation context that the instruction is executed in, in this translation it has been stored in thread-local storage.

Because of this, we translate this instruction to the following:
```
(block $resumed ;; [tp*] -> [tr*]
  (block $attached
    (try
      (exnref.new $inputs<$evt>)
      (ref.func $get_handler<$evt>)
      (struct.get_clear 0 (tls.get $handler<$evt>))
      (stack.switch $detaching)
    catch $attaching $attached
    )
  ) ;; $attached : [exnref]
  (br_on_exn $resuming<tr*>)
  (unreachable)
) ;; $resumed : [tr*]
```
where
```
(func $get_handler<$evt> (result stackref)
  (struct.get_clear 0 (tls.get $handler<$evt>))
)
```
This translation makes it clear that using thread-local storage significantly facilitates continuation suspension, though by pushing the costs of maintaining this storage elsewhere.

## Types

The type `(cont ([tp*] -> [tr*]))` translates to the type `$cont = ref (struct $hook (mut stackref))`.
The first field indicates the "parent" reference to update when one "attaches" the continuation, and the second field indicates the `stackref` to transfer control to.

Note that `$cont` has no type parameters.
We found that using input types `tp*` constrained stack-switching patterns.
So a `(cont ([tp*] -> [tr*]))` corresponds to a stack that is waiting for a specific "resuming" input-only event with payload `tp*`, which we will denote with `(event $resuming<tp*> (param tp*))`.
We also do not use return types, since a return is implemented as a stack switch that cleans up the former stack (i.e. `stack.switch_drop`).
So a `(cont ([tp*] -> [tr*]))` corresponds to a stack that "ends" with a specific "returning" input-only event with payload `tr*`, which we will denote with `(event $returning<tr*> (param tr*))`.

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
      (br_on_exn $resuming<tp*>)
      (unreachable)
    ) ;; $resumed : [tp*]
    (call_funcref $canon<tp*;tr*> $f) ;; [tp*] -> [tr*]
  )
  (stack.switch_drop $returning<tr*> (struct.get_clear 0 (local.get $parent)))
)
```

`$cont_rout` has less machinery in this translation because the mechanisms for finding handlers has been pushed elsewhere.

First, note that `stack.start` is set up to receive a `$resuming` event.
Once that happens, `$cont_rout` will store the former stack as its parent and then proceed to call the given function reference.
After that call completes, `$cont_rout` then switches back to the parent stack with the `$returning` event.

The entire time, `$cont_rout` is set up to forward any exceptions that get thrown to the parent stack.
This is done using a more general variant of `stack.redirect` that is presented [here](https://github.com/soil-initiative/stacks/pull/9/files), which in particular allows one to use a mutable field rather than a local variable to determine where to redirect to.

## Resuming a continuation

The `cont` design resumes a continuation on top of the current continuation with the following instruction:
```
cont.resume $label (event $evt*) : [tp* (cont ([tp*] -> [tr*]))] -> [tr*]
```
where `label $label : [(evtref tr*)]`.
The list of events `$evt*` is provided to enable some static filtering on the event before detaching.
In this translation, that list is used to update the handlers in thread-local storage.

The label `$label` expects an `evtref tr*`.
We translate this type to `$evtref = ref (struct exnref funcref $cont)`.

We can translate the simple case where `$evt*` is just a single event `$evt : [ti*] -> [to*]` to the following:
```
(block $returned ;; [tp* $cont] -> [tr*]
  (tls.get $handler<$evt>) ;; get the old handler
  (let (local $c $cont) (local $old $hook)
    (block $detached
      (try
        (tls.set $handler<$evt> (struct.get 0 (local.get $c)))
        (try
          (exnref.new $resuming<tp*>)
          (struct.get 0 (local.get $c))
          (struct.get_clear 1 (local.get $c))
          (stack.switch_call $attach_set_parent $attaching)
        unwind
          (tls.set $handler<$evt> (local.get $old))
        )
      catch $detaching $detached
      catch $returning<tr*> $returned
      )
    ) ;; $detached : [exnref funcref stackref]
    (let (local $e exnref) (local $f funcref) (local $s stackref)
      (struct.new $evtref (local.get $e)
                          (local.get $f)
                          (struct.new $cont (struct.get 0 (local.get $c))
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
It is straightforward to generalize this translation to a proper list of events `$evt*`.

These instructions switch control to the `stackref` representing the continuation, using `stack.switch_call $attach_set_parent` to set the parent of the continuation to be the current stack.
They then wait for that `stackref` to either "detach" (via the `$detaching` event) or to "return" (via the `$returning<tr*>` event).
Note that the handler for detaching branches to `$label` and makes the allocation of the `evtref` explicit.

The key change in this translation is the update to the thread-local storage.
This sets the `$hook` in the `$cont` as the handler for `$evt` (and more generally as the handler of each event in `$evt*`).
The `stack.switch_call $attach_set_parent` then sets the `stackref` in that `$hook` to be the current stack, i.e. the stack that is executing `cont.resume`.
The use of `unwind` then ensures that the handler for `$evt` (and more generally the handler of each event in `$evt*`) is restored to its former value after the continuation returns, detaches, or throws an exception.
This translation illustrates how this implementation strategy pushes the work to attachment/detachment.

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
    (struct.get 2 (local.get $e))
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

We can translate the simple case where `$evt*` is just a single event `$evt : [ti*] -> [to*]` to the following:
```
(block $returned ;; [$evtref] -> [tr*]
  (let (local $e $evtref)
    (block $attached
      (try
        (struct.get 0 (local.get $e))
        (struct.get 1 (local.get $e))
        (call_funcref $get_handler_tag (struct.get 1 (local.get $e)))
        (stack.switch $detaching)
      catch $attaching $attached
      )
    ) ;; $attached : [exnref]
  )
  (tls.get $handler<$evt>) ;; get the old handler
  (let (local $ex exnref) (local $old $hook)
    (block $detached
      (try
        (tls.set $handler<$evt> (struct.get 0 (struct.get 2 (local.get $e)))
        (try
          (local.get $ex)
          (struct.get 0 (struct.get 2 (local.get $e)))
          (struct.get_clear 1 (struct.get 2 (local.get $e)))
          (stack.switch_call $attach_set_parent $attaching)
        unwind
          (tls.set $handler<$evt> (local.get $old))
        )
      catch $detaching $detached
      catch $returning<tr*> $returned
      )
    ) ;; $detached : [exnref funcref stackref]
    (let (local $e exnref) (local $f funcref) (local $s stackref)
      (struct.new $evtref (local.get $e)
                          (local.get $f)
                          (struct.new $cont (struct.get 0 (local.get $c))
                                            (local.get_clear $s))
      (br $label)
    )
  )
) ;; $returned : [tr*]
```
It is straightforward to generalize this translation to a proper list of events `$evt*`.

This translation is essentially a suspend followed by a resume.
Unlike with the stack-inspection translation, this translation is unable to implement `cont.forward` in a way that lets the subsequent `cont.resume` jump straight to the continuation that executed `cont.suspend`.
The reason is that we have to appropriately update the thread-local storage to register the forwarding continuation as a handler for the relevant events.
Thus resuming a continuation can involve a lot of switching and bookkeeping for applications that need to forward frequently (say because they use dynamic information to determine handlers).
This is possibly the most substantial cost of this implementation strategy, beyond the costs associated with thread-local storage (and assuming its existence) in general.

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
    (try
      (struct.get 0 (local.get $c))
      (struct.get_clear 1 (local.get $c))
      (stack.switch_call $abort_set_parent<te*> $exn)
    catch $returning<tr*> $returned
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

Unlike the other instructions, this translation does not need to handle the `$detaching` event.
The reason is that the relevant `$hook` is never registered as a handler of any events in thread-local storage, and as such the connection will never be detached.

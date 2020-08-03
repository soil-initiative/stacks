# Single-Stack Async/Await using `redirect_to`

Many WebAssembly modules are compiled with the assumption that only one stack is live in any particular instance of the module at a time.
For this common case, there is a more efficient way to support asynchronous I/O as discussed in the [Overview](../Overview.md) if we utilize `stack.redirect_to`.
Whereas `stack.redirect` specifies a local variable to use to determine where to redirect to when a stack walk occurs, `stack.redirect_to` specifies a computation to run to determine where to redirect to.
In particular, this enables an application to use a *global* variable to store where to redirect to.

If the application using asynchronous I/O is willing to assume it has only one stack live at a time, then it can use global variables to store the relevant "host" stack and "application" stack:
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

Then rather than having programs await promises by performing `call_stack $await`, we instead have them simply perform `call $await` using the following function:
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
and then revise the imported function `$create_promise : [externref] -> [externref]` similarly as follows:
```
(promise) => promise.then((x) => module_instance.resolve(x),
                          (e) => module_instance.reject(e))
```

Beyond providing a useful optimization, this variant illustrates the substantial flexibility this proposal provides applications for determining how best to implement stack-based features according to their specific circumstances.

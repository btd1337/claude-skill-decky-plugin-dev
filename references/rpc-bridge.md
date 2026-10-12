# Decky backend/frontend RPC bridge

## How `call()` maps onto Python method parameters

`@decky/api`'s `call(methodName, ...args)` passes each JS argument you give it as a
*separate positional argument* to the matching Python method (everything after `self`).
Crucially: when frontend code calls `call('method', { foo, bar })` with **one object**, Python
receives that whole object as the *first* positional parameter - not as keyword arguments
matching the object's keys.

Most backend methods in a well-built plugin therefore use a shared helper to accept either
calling convention, since different call sites end up passing either a single params object or
separate positional values:

```python
def _extract_params(self, first_arg, **kwargs) -> Dict[str, Any]:
    """Decky may pass all params as a single dict in first_arg, or as
    separate positional args - accept either."""
    if isinstance(first_arg, dict):
        return first_arg
    result = {'id': str(first_arg)} if first_arg is not None else {}
    result.update(kwargs)
    return result
```

Used like:

```python
async def set_item_value(self, id_or_params, value: str = None) -> Dict[str, bool]:
    params = self._extract_params(id_or_params, value=value)
    item_id = str(params.get('id', ''))
    value = params.get('value')
    ...
```

## The failure mode when a method *doesn't* use this pattern

Any backend method that declares multiple **required** positional parameters with no
defaults - e.g. a simple `async def log_message(self, level: str, text: str)` helper that
skips the `_extract_params` pattern because it "doesn't need it" - breaks if called from JS
with a single object argument. Python raises "missing required positional argument" *inside
Decky's own RPC dispatch*, before the method body ever executes.

If that failing call isn't `await`ed and wrapped in `try`/`catch` on the JS side (a common
pattern for fire-and-forget logging calls), the failure is **completely silent**: no log line,
no visible error, nothing reaches the backend at all. A plugin's own frontend-to-backend
logging helper is a very easy place for this to happen, since it's often written early,
quickly, without the `_extract_params` pattern used everywhere else - and a diagnostic log
call through it that appears to simply do nothing may actually never be reaching the Python
method at all.

**Fix / lesson**: before calling a 2+ parameter backend method from JS, check whether that
specific method uses the `_extract_params` pattern (or otherwise accepts a single dict). If it
doesn't, call it with separate positional arguments matching its signature exactly:

```ts
call('log_message', 'info', 'some message')   // correct for a method with no _extract_params
// NOT:
call('log_message', { level: 'info', text: 'some message' })  // silently fails
```

## Payload-nesting mismatches between frontend and backend

When you add a new field to a payload that already flows through multiple layers - e.g. the
frontend collects a new flag per item, puts it inside a nested `item_data` object alongside
other existing fields, and sends that whole object to a sync endpoint - it's very easy to read
it back at the wrong nesting level on the backend without any error:

```python
# Frontend actually sends: { id, item_data: { some_existing_field, new_flag }, ... }

# BUG: reads from the wrong level, silently gets False/missing every time
new_flag = bool(params.get('new_flag', False))

# FIX: read from where it actually lives
item_data = params.get('item_data', {})
new_flag = bool(item_data.get('new_flag', False))
```

Both ends can be individually "correctly implemented" and the feature will still silently not
work, because nothing throws - the field just reads as its default. When a feature that
touches a multi-layer payload "doesn't work" with no error, grep the *exact* JS object literal
being sent and the *exact* Python dict path being read, side by side, rather than assuming the
field names "line up" just because the same key string appears somewhere in scope on both
sides.

## Debugging a payload mismatch

Add a temporary diagnostic log on *both* sides of the boundary at once:

1. In JS, right before the `call(...)`, log the exact object you're about to send.
2. In Python, at the very top of the method body, log the exact `params`/`kwargs` received.

Comparing these two directly is faster than staring at either side alone, since the bug is as
likely to be a shape mismatch between the two as a logic error in either one.

Remember: a fire-and-forget `call(...)` (no `await`, no `.catch()`) turns any backend exception
into an invisible unhandled promise rejection. For debugging, always `await` the call and
surface the result/error - otherwise a genuinely-failing diagnostic call can look identical to
one that worked but found nothing.

# itertools.concat(*...iters*)

Concatenate multiple iterables to a single iterable. Basically, this:

```js
export function *concat(...iters) {
    for (let i = 0; i < iters.length; i++) yield* iters[i]
}
```

### Why?

1. We already have `Array.prototype.concat`, but this is still useful for generic iterables.

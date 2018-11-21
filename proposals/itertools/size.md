# itertools.size(*iter*)

Basically `Array.prototype.length`, just generalized to iterables.

```js
export function size(iter) {
    let i = 0
    for (const item of iter) i++
    return i
}
```

### Why?

1. We already have `Array.prototype.length`, but this is can still be useful for generic iterables.
1. For many built-in collections like `Map` and `Set`, this can be determined in constant time.

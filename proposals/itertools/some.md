# itertools.some(*...iters*)

Check if a condition holds for at least one item of an iterable. Basically, this:

```js
const apply = Reflect.apply

export function some(iter, func, thisValue = void 0) {
    let i = 0
    for (const item of iter) {
        const index = i++
        if (apply(func, thisValue, [item, index])) return true
    }
    return false
}
```

### Why?

1. We already have `Array.prototype.some`, but this is still useful for generic iterables.

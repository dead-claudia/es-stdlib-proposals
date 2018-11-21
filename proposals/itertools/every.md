# itertools.every(*...iters*)

Check if a condition holds for all items of an iterable. Basically, this:

```js
const apply = Reflect.apply

export function every(iter, func, thisValue = void 0) {
    let i = 0
    for (const item of iter) {
        const index = i++
        if (!apply(func, thisValue, [item, index])) return false
    }
    return true
}
```

### Why?

1. We already have `Array.prototype.every`, but this is still useful for generic iterables.

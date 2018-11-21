# itertools.map(*iter*, *func* [ , *thisValue* ])

Basically `Array.prototype.map`, just generalized to iterables.

```js
const apply = Reflect.apply

export function *map(iter, func, thisValue = void 0) {
    let i = 0
    for (const item of iter) {
        const index = i++
        yield apply(func, thisValue, [item, index])
    }
}
```

### Why?

1. We already have `Array.prototype.map`, but this is even *more* useful for generic iterables.

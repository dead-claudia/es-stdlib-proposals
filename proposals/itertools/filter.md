# itertools.filter(*iter*, *func* [ , *thisValue* ])

Basically `Array.prototype.filter`, just generalized to iterables.

```js
const apply = Reflect.apply

export function *filter(iter, func, thisValue = void 0) {
    let i = 0
    for (const item of iter) {
        const index = i++
        if (apply(func, thisValue, [item, index])) yield item
    }
}
```

### Why?

1. We already have `Array.prototype.filter`, but this is even *more* useful for generic iterables.

# itertools.min(*iter* [ , *func* ])

Find the min value of a list of objects, optionally read by a `func` key. The value itself is returned, not necessarily the return value of `func`. It'd look close to this:

```js
function min(iter, func = void 0) {
    if (func == null) func = x => x
    if (typeof func !== "function") {
        throw new TypeError("`func` is not a function!")
    }
    let maxItem, maxTest
    for (const item of iter) {
        const test = +func(item)
        if (maxTest == null || maxTest < test) {
            maxTest = test
            maxItem = item
        }
    }
    return maxItem
}
```

### Why?

1. `Math.min(...iter)` allocates its argument, creating a memory concern for larger iterables. This, conversely, has almost zero overhead.
1. `Math.min(...iter)` can hit stack space limits for large iterables in implementations that allocate arguments on the heap.
1. This is a good candidate for vectorized code gen if the iterable is an integer array or similar.

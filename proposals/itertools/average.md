# itertools.average(*iter* [ , *func* ])

Find the average value of a list of objects, optionally read by a `func` key. The value itself is returned, not necessarily the return value of `func`. It'd look close to this:

```js
function average(iter) {
    let result = 0
    let i = 0
    // Have to use an incremental average here
    for (const item of iter) {
        result += (item - result) / ++i
    }
    return result
}
```

### Why?

1. It's not uncommon common to need to calculate an average of a list of elements.
1. It's not obvious how to do it with potentially infinite iterators.
1. The naïve version of `(a[0] + ... + a[n - 1]) / a.length` risks overflow issues.
1. It's [trivially vectorized](https://godbolt.org/z/9SCQ3g) for numeric arrays.
    - It took some finagling to get Clang to generate the correct assembly here. For some reason, it forgot it could use `vmovupd` to load a contiguous vector when auto-vectorizing the loop, so it instead just generated a bunch of `vmovsd`/`vdivsd`/`vaddsd` instructions. GCC [just generates crap code](https://godbolt.org/z/Qs_1Op), so I suspect it's mostly just compiler optimizer bugs.
1. [Lodash has it](https://lodash.com/docs#mean)
1. There's some language precedent:
    - Java via [`IntStream.average`](https://docs.oracle.com/javase/9/docs/api/java/util/stream/IntStream.html#average--) and friends
    - Python via [`statistics.mean`](https://docs.python.org/3/library/statistics.html#statistics.mean)

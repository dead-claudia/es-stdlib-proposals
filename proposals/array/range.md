# Array.range(*start* , *end* [ , *step* ] [ , *func* ]), %TypedArray%.range(*start* , *end* [ , *step* ] [ , *func* ])

Basically this:

```js
Array.range = TypedArray.range = function (
    start, end, step = undefined; func = undefined
) {
    start = +start; end = +end
    if (func == null && typeof step === "function") { func = step; step = undefined }
    if (func != null && typeof func !== "function") throw new TypeError()

    const ret = new this[Symbol.species](end - start)

    if (func != null) {
        for (let i = start; i < end; i++) ret[i] = func(i)
    } else {
        for (let i = start; i < end; i++) ret[i] = i
    }

    return ret
}
```

### Why?

- I find myself commonly doing `Array.from({length: i}, (_, i) => ...)`, but no engine optimizes that pattern. It's not a super easy pattern to detect and leverage in general because of the lambda, and no engine I'm aware of checks from callees what variables are used in functions. (V8 implemented `.map`/etc. optimization by allowing bailouts within builtins.)
- It's sometimes clearer to do things like `for (const i of Array.range(start, end - offset))` rather than use an extra variable.
- It's an obvious case for code gen here.
    - It's embarassingly parallel apart from generating the indices themselves.
    - You can go straight to a "typed" integer/float array even for `Array.range` (as long as the prototype isn't modified).
    - The allocation can be elided if used as the collection for `for ... of`, something a JIT compiler could detect.
- It's been suggested [a lot](https://www.google.com/search?q=site%3Aesdiscuss.org+array+range) on ES Discuss.
- Language precedent:
    - Python via its *very* commonly used [`range`](https://docs.python.org/3/library/stdtypes.html#typesseq-range)
    - Ruby via [`Integer#upto`](https://ruby-doc.org/core-2.5.0/Integer.html#method-i-upto)
    - C# via [`Enumerable.Range`](https://docs.microsoft.com/en-us/dotnet/api/system.linq.enumerable.range)
    - Haskell via the native syntax `[start .. end]`
    - Rust via [`(start .. end)`](https://doc.rust-lang.org/std/ops/struct.Range.html)
    - Clojure via [`range`](https://clojuredocs.org/clojure.core/range)
- Library precedent:
    - [Lodash](https://lodash.com/docs#range) and [Underscore](https://underscorejs.org/#range) both have `_.range` that's mostly this.
    - [Ramda](https://ramdajs.com/) has [`R.range`](https://ramdajs.com/docs/#range) that's basically this, just with an explicit from/to.

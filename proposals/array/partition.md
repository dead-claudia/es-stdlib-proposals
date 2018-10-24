# Array.prototype.partition(*func* [ , *thisValue* ]), %TypedArray%.prototype.partition(*func* [ , *thisValue* ])

Basically a combined `filter` + `reject`, just returning a pair of two arrays where the first are the ones `func` returned truthy on, the second where it returned falsy on. An engine would do better to do that in a single pass rather than two, since it's faster. It'd look something close to this:

```js
Array.prototype.partition = function (func, thisValue) {
    let pass = []
    let fail = []
    for (let i = 0; i < this.length; i++) {
        let item = this[i++]
        ;(func.call(thisValue, item, i, this) ? pass : fail).push(item)
    }
    return [pass, fail]
}
```

### Why?

1. A very large number of `filter`/`reject` cases reduce to this. Any time you see `x.filter(x => cond(x))` followed by `x.filter(x => !cond(x))`, you're looking at something better written with a partition (it's DRYer without losing readability).
1. Underscore has had it for [a few years at this point](https://github.com/jashkenas/underscore/commit/407d027ccac09ab8f6ddfa1cd5a81ee9516e5d4b), and it was requested as far back as [2011](https://github.com/jashkenas/underscore/issues/132). Lodash added it [shortly thereafter](https://github.com/lodash/lodash/commit/5f02c336e7d584393e98ac35f80efb1d70a3ab98).
1. Several languages already offer something similar:
    - Ruby via [`Enumerable#partition`](https://ruby-doc.org/core-2.5.0/Enumerable.html#method-i-partition)
    - Clojure via [`clojure.core/partition`](https://clojuredocs.org/clojure.core/partition)
    - Kotlin via [`Array<T>.partition`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/partition.html), etc.
    - Rust via [`Iterator<T>.partition`](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.partition)
    - Haskell via [`Data.List.partition`](https://hackage.haskell.org/package/base-4.11.1.0/docs/Data-List.html#v:partition)
    - Swift and C++ have in-place partitions via [`Array<T>.partition(by:)`](https://developer.apple.com/documentation/swift/array/3017524-partition) and [`partition` in `<algorithm>`](http://www.cplusplus.com/reference/algorithm/partition/) respectively, where the return value is the start of the second partition.

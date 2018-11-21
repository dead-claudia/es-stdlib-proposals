# itertools.zip(*...iters*)

Zip multiple iterables to a single iterable of tuples, ending when the first one ends. Basically, something to the effect of this:

```js
export function *zip(...iters) {
    const iterables = iters.map(iter => iter[Symbol.iterator]())
    while (true) {
        const results = iterables.map(iter => iter.next())
        if (results.some(result => result.done)) return
        yield results.map(result => result.value)
    }
}
```

### Why?

1. It's sometimes useful to combine two sequences into a sequence of pairs.
1. [Underscore](http://underscorejs.org/#zip), [Lodash](https://lodash.com/docs#zip), and [Ramda](http://ramdajs.com/docs/#zip) all three have had it for pretty much the whole time they've existed, just for arrays only.
1. For dense arrays and typed arrays, this can be *seriously* optimized with code gen.
1. Several languages already offer something similar:
    - Python via [`zip`](https://docs.python.org/3/library/functions.html#zip)
    - Ruby via [`Array#zip`](https://ruby-doc.org/core-2.5.0/Array.html#method-i-zip)
    - Clojure via [`clojure.core/zip`](https://clojuredocs.org/clojure.core/zip)
    - Kotlin via [`Array<T>.zip`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/zip.html), etc.
    - C# via [`IEnumerable<T>.zip`](https://docs.microsoft.com/en-us/dotnet/api/system.linq.enumerable.zip?view=netframework-4.7.2), although it requires a third mapper argument
    - Rust via [`Iterator<T>.zip`](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.zip)
    - Haskell via [`Data.List.zip`](https://hackage.haskell.org/package/base-4.11.1.0/docs/Data-List.html#v:zip)
    - Swift via [`zip(_:_:)`](https://developer.apple.com/documentation/swift/1541125-zip)

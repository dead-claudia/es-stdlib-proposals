# Array.prototype.sortBy(*key* [ , ...*args* ])

- When no comparator is passed, this would be equivalent to `array.sort((a, b) => SortCompare(a[key], b[key]))` where `SortCompare` is defined [here](https://tc39.github.io/ecma262/#sec-sortcompare), but it would also factor in the function change above.
- When a comparator is passed, this would be equivalent to `array.sort((a, b) => comparator(a[key], b[key]))`, where `comparator` is the comparator itself.
- When *key* is an iterable, it's assumed to be an iterable of indices to access.
- When *key* is a function, it's used to get the sort key itself.

### Why?

1. Most use cases for the comparator are really just wanting to sort by a possibly nested property.
1. Some languages already offer something similar:
    - Python via [`list.sort(key='foo')`](https://docs.python.org/3/library/stdtypes.html#list.sort)
    - Ruby via [`Enumerable#sort_by`](https://ruby-doc.org/core-2.5.0/Enumerable.html#method-i-sort_by)
    - Clojure via [`clojure.core/sort-by`](http://clojure.github.io/clojure/clojure.core-api.html#clojure.core/sort-by)
    - Java via [`Arrays.sort`](https://docs.oracle.com/javase/9/docs/api/java/util/Arrays.html#sort-T:A-java.util.Comparator-)
    - Rust via [`[T].sort_by_key`](https://doc.rust-lang.org/std/primitive.slice.html#method.sort_by_key)

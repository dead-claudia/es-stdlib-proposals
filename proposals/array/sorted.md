# Immutable sorting

- Array.prototype.sorted(*compareType* [ , *reserved1* [ , *reserved2* ] ])
- Array.prototype.sorted(*comparator*)
- Array.prototype.sortedBy(*key* [ , ...*args* ])
- %TypedArray%.prototype.sorted(*comparator*)
- %TypedArray%.prototype.sortedBy(*key* [ , ...*args* ])

Basically a non-destructive sort that creates a new array in the process. Each of these mirrors the mutating version:

- Array.prototype.sort(*comparator*), %TypedArray%.prototype.sort(*comparator*) &rarr; Array.prototype.sorted(*comparator*), %TypedArray%.prototype.sorted(*comparator*)
- Array.prototype.sort(*compareType* [ , *reserved1* [ , *reserved2* ] ]) &rarr; Array.prototype.sorted(*compareType* [ , *reserved1* [ , *reserved2* ] ]), %TypedArray%.prototype.sorted(*compareType* [ , *reserved1* [ , *reserved2* ] ])
- Array.prototype.sortBy(*key* [ , ...*args* ]), %TypedArray%.prototype.sortBy(*key* [ , ...*args* ]) &rarr; Array.prototype.sortedBy(*key* [ , ...*args* ]), %TypedArray%.prototype.sortedBy(*key* [ , ...*args* ])

### Why?

1. In my experience, it's rare to want to sort an array without also cloning it first. I've found myself do `array.slice().sort()` or `array.concat().sort()` much more frequently than destructive sorting, with the only exception being in performance-sensitive algorithms.
1. Some languages already offern something similar:
    - Python via [`sorted(iter)`](https://docs.python.org/3/library/functions.html#sorted)
    - Ruby via [`Array#sort`](https://ruby-doc.org/core-2.5.0/Array.html#method-i-sort)
    - Clojure via [`clojure.core/sort`](https://clojuredocs.org/clojure.core/sort) (when the operand isn't a native array)
    - Java via [`Stream<T>.sorted()`](https://docs.oracle.com/javase/9/docs/api/java/util/stream/Stream.html#sorted--) (for streams only)

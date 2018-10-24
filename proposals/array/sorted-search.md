# Sorted search

- Array.prototype.sortedSearch(*compareType* [ , *reserved1* [ , *reserved2* ] ])
- Array.prototype.sortedSearch(*comparator*)
- Array.prototype.sortedSearchBy(*key* [ , ...*args* ])
- %TypedArray%.prototype.sortedSearch(*comparator*)
- %TypedArray%.prototype.sortedSearchBy(*key* [ , ...*args* ])

Search an array for a value using a sorted search and return its index, or -1 if it doesn't exist. The name is chosen to not imply binary search (the most common algorithm), since faster algorithms (*O*(√(log n))) exist for integer-only arrays, if extra allocation is permitted, and binary search is slower than iterative searching for smaller arrays and segments.

Observably, this *must* follow binary search, though - getters should be able to verify this. The comparison works similarly to the normal `.sort` and `.sorted` variants (which actually compute a new array).

### Why?

1. Sometimes, you need a sorted set, but iteration speed is more important than "does X exist in Y". This combined with `Array.prototype.insert` as proposed here addresses issues with that.
1. If you need a sorted set, but you have a lot of data, an array is much lighter than a binary search tree.
1. Several languages already offer this:
    - Python via [its `bisect` module](https://docs.python.org/3.6/library/bisect.html#module-bisect)
    - Ruby via [`Array#bsearch`](https://ruby-doc.org/core-2.2.0/Array.html#method-i-bsearch) indirectly
    - Java via [its various `Array.binarySearch(T[], T)` methods](https://docs.oracle.com/javase/9/docs/api/java/util/Arrays.html)
    - Rust via [`Vec<T>::binary_search(&self, &T)`](https://doc.rust-lang.org/beta/std/vec/struct.Vec.html#method.binary_search) and friends
    - C via [`bsearch(key, base, nel, width, compar)`](http://pubs.opengroup.org/onlinepubs/9699919799/functions/bsearch.html)
    - C++ via [`std::binary_search(ForwardIt, ForwardIt, const T&)`](https://en.cppreference.com/w/cpp/algorithm/binary_search)

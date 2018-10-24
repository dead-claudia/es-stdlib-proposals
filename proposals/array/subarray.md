# Array.prototype.subarray(*from* [ , *to* ])

Return an array view of the array from *from* to *to*. It works similarly to `%TypedArray%.prototype.subarray`, just generalized to array-likes and not bound to a buffer.

Within subarray views:

- Their [[Prototype]] is set to %ArrayPrototype%.
- `Array.isArray` returns `true` for them.
- The view's `.length` is a non-configurable getter with no associated setter.
- A `TypeError` is raised if you attempt to set an index out of the subarray's bounds.
- As a result, `push`, `shift`, `insert`, and friends are not permitted.
- They carry no other marking of their view-ness, but they *do* reference the backing array directly.

### Why?

1. This could be used with methods like `map`, `filter`, and `sorted` to operate more efficiently over an interval.
1. It exists for consistency with `TypedArray.prototype.subarray`, which does the same thing.
1. Java has [`List<T>.sublist`](https://docs.oracle.com/javase/9/docs/api/java/util/List.html#subList-int-int-), which works similarly.
1. This is simpler than adding support for offsets to everything.

Alternatively, this could become syntax and absorb [this proposal](https://github.com/tc39/proposal-slice-notation). In that event, it'd work even more closely to how Python arrays do, and it'd be easier for engines to optimize (fewer assumptions).

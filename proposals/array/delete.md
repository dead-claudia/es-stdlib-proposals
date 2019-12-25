# Array.prototype.delete(*item* [ , *startOffset* [ , *endOffset* ] ])

This removes the first occurrence of `item` from `this`, optionally starting from `startOffset` and ending at `endOffset`, returning `true` if the item was found and `false` if it was not. This is roughly sugar for this, but the algorithm can be substantially optimized.

```js
Array.prototype.delete = function (item, startOffset = 0, endOffset = this.length) {
    for (let i = startOffset; i < endOffset; i++) {
        if (i in this && SameValueZero(this[i], item)) {
            this.splice(i, 0)
            return true
        }
    }
    return false
}
```

### Why?

It's pretty common to want to remove something from an array. And in some cases, an array is actually *better* than a set:

- If it's almost always small and never contains strings (when the constant factor of hashing dominates)
- If it's rarely tested or removed from, but frequently iterated.
- If memory usage is a larger concern than computational complexity.

In this case, having it built-in allows a few other optimizations that aren't always possible, like using vector instructions for the whole thing when 1. the array is dense and 2. you don't need to dereference the array's elements to check them. There's also other language precedent:

- Python via [`MutableSequence.remove`](https://docs.python.org/3/library/stdtypes.html#mutable-sequence-types)
- C# via [`System.Collections.ArrayList.Remove`](https://docs.microsoft.com/en-us/dotnet/api/system.collections.arraylist.remove)
- Rust recently via [`Vec::remove_item`](https://doc.rust-lang.org/std/vec/struct.Vec.html#method.remove_item)
- Ruby via [`Array#delete`](https://docs.ruby-lang.org/en/2.5.0/Array.html#method-i-delete)
- Java via [`java.util.ArrayList.remove`](https://docs.oracle.com/javase/10/docs/api/java/util/ArrayList.html#remove(java.lang.Object))

### Optimization

There's a few points of optimization:

- When the array's specialized and the item can't be converted to the type the array's specialized for, no search is required and it can immediately return.
- When the array's unspecialized and `SameValueZero` is simple pointer equality, you can search for it by basically the same process as `indexOf`, just with a final index to search.
- For most searches that involve objects, numbers, booleans, `null`, and `undefined`, it's easily vectorized. A single branch is required in case a possible match is detected, but that's usually comparatively rare.

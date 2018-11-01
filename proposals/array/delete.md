# Array.prototype.delete(*item* [ , *all* [ , *startOffset* [ , endOffset ] ] ])

This removes the first occurrence of `item` from `this` (or all occurrences if `all` is truthy), optionally starting from `startOffset`, returning `true` if the item was found and `false` if it was not. This is mostly sugar for this, but the algorithm can be substantially optimized.

```js
// Sugar for this
Array.prototype.delete = function (item, all = false, startOffset = undefined, endOffset = this.length) {
    let index = this.indexOf(item, startOffset)
    if (index < 0 || index >= endOffset) return false
    if (all) {
        do {
            this.splice(item, 1)
            index = this.indexOf(item, index)
        } while (index >= 0 && index < endOffset)
    } else {
        this.splice(item, 1)
    }
    return true
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
- Rust recently via [`Vec::remove_item`](https://doc.rust-lang.org/std/vec/struct.Vec.html#method.remove_item).
- Ruby via [`Array#delete`](https://docs.ruby-lang.org/en/2.5.0/Array.html#method-i-delete)

# Array.prototype.deleteAll(*item* [ , *startOffset* [ , *endOffset* ] ])

This removes all occurrences of `item` from `this`, optionally starting from `startOffset` and ending at `endOffset`, returning the number of items removed. This is sugar for roughly this, but the algorithm can be substantially optimized.

```js
Array.prototype.deleteAll = function (item, startOffset = 0, endOffset = this.length) {
    let oldLength = this.length
    let targetOffset = startOffset
    for (let i = startOffset; i < endOffset; i++) {
        if (i in this && !SameValueZero(this[i], item)) {
            this[count] = this[i]
            count++
        }
    }
    let diff = endOffset - targetOffset
    this.splice(targetOffset, diff)
    return diff
}
```

### Why?

It's pretty common to want to remove something from an array. And in some cases, an array is actually *better* than a set:

- If it's almost always small and never contains strings (when the constant factor of hashing dominates)
- If it's rarely tested or removed from, but frequently iterated.
- If memory usage is a larger concern than computational complexity.

Also, Lodash's `_.compact(array)`, which removes all `null`s and `undefined`s from the array, is as simple as `array.deleteAll(null); array.deleteAll(undefined)`

In this case, having it built-in allows a few other optimizations that aren't always possible, like using vector instructions for the whole thing when 1. the array is dense and 2. you don't need to dereference the array's elements to check them. There's some language and library precedent for this and related:

- Ruby via [`array.delete_if { |i| item == i }`](https://docs.ruby-lang.org/en/2.5.0/Array.html#method-i-delete_if)
- Java via [`collection.removeIf(item::equals)`](https://docs.oracle.com/javase/10/docs/api/java/util/ArrayList.html#removeIf(java.util.function.Predicate))
- Ramda via [`R.without([item], array)`](https://ramdajs.com/docs/#without) - although that function uses `Object.is` internally, it would otherwise work like this if it used `SameValueZero` instead:

    ```js
    R.without = function without(items, array) {
        if (arguments.length === 0) return without
        if (arguments.length === 1) return array => without(items, array)
        const result = array.slice()
        for (const item of items) result.deleteAll(item)
        return result
    }
    ```

### Optimization

There's a few points of optimization:

- When the array's specialized and the item can't be converted to the type the array's specialized for, no search is required and it can immediately return.
- For most searches that involve objects, numbers, booleans, `null`, and `undefined`, where the array is dense, `SameValueZero` is just pointer equality, and searching doesn't require unboxing, it's easily vectorized. Unlike with `Array.prototype.delete`, you can also do this entirely without branching in the loop, by doing the following:
    - Before the vectorized loop, load the lookahead vector.
    - In the vectorized loop:
        - Set the current vector to the next vector.
        - Set the next vector to the lookahead vector.
        - Load the lookahead vector at the read index to be the next vector after the current vector.
        - Compare all lanes in the current vector with the value to remove, generating a mask of bits where each bit is set if the comparison evaluated to true.
        - Set the increment size to the number of ones in the generated mask.
        - Compress the current vector with the resulting mask (like via x86 `vcompressq`).
        - Invert the mask and move the ones in it to the low N bits, where N is the number of bits set.
        - Expand the next vector (like via x86 `vexpandq`) to move the low N lanes to be the high N lanes.
        - Blend the current vector into the next vector, where the low N lanes are of the current vector and the remaining lanes are the next vector. (In x86, this could just do `vpblendmq` with the same mask used to expand.)
        - Store the next vector at the write index.
        - Increment the write index by the increment size generated previously.
    - In the scalar fallback loop:
        - Load the value at the read index.
        - Compare with the value to remove.
        - Save the result to a register, 1 if failure, 0 if success.
        - Store the value at the write index.
        - Increment the write index by the comparison result.
    - Note: the vectorized loop requires the data in the arrays to generally be 16-bit aligned to work (both SSE and NEON require this).
        - Most implementations need this anyway just so they can tag their pointers to handle unboxed integers. So this is unlikely to be an issue.

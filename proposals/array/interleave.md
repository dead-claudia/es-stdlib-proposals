# Array.prototype.interleave(*other*), %TypedArray%.prototype.interleave(*other*)

This cycles through each array, emitting their `i`th entry until it hits the end. Technically, you could do it via `flatten(zip(arrays))`, but this generates a lot of overhead. It also doesn't provide obvious semantics for things like template tags (think: `const interpolate = (strings, ...args) => Array.interleave(strings, args).join("")`), because most `zip` implementations don't include partial or sparse arrays after one array terminates. A semi-optimized implementation would look like this:

```js
Array.prototype.interleave = function (other) {
    const thisLen = getLength(this)
    const otherLen = getLength(other)
    const minLen = Math.min(thisLen, otherLen)
    const result = new Array(thisLen + otherLen).fill()

    // Write all of `this` first
    emitOffset(result, this, thisLen, 0, minLen)

    // Write all of `other` second
    // Move the last iteration in the first loop to the second - it's simpler that way
    emitOffset(result, other, otherLen, 1, minLen - 1)

    return result
}

function emitOffset(result, source, len, offset, breakpoint) {
    let i = 0, j = offset
    while (i < breakpoint) { result[j] = source[i]; i++; j += 2 }
    while (i < len) { result[j] = source[i]; i++; j++ }
}

// Basically `? ToLength(? Get(? ToObject(value), "length"))`
function getLength(value) {
    if (value != null) {
        const l = +value.length
        if (l === l) {
            if (l > 9007199254740991) return 9007199254740991
            if (l >= 1) return floor(l)
        }
    }
    return 0
}
```

A typed array implementation would look similar. Note that this could be easily vectorized when both arrays are dense and of compatible types, and with typed arrays, you could fully generalize it based on target/source byte count. You could also use `memcpy` for the final parts.

### Why?

1. It'd make a lot of working with template tags ***way*** easier. Currently, I do a lot of the manual equivalent of `strs.interleave(args).join("")` or `strs.interleave(args.map(...)).join("")` already when implementing template tags, and as common as this need is, this would be very invaluable.
1. Consider how often people need to do this: it ended up as two Ruby questions on SO, [one with the wrong answer of `a.zip(b).flat`](https://stackoverflow.com/questions/7312713/merge-and-interleave-two-arrays-in-ruby) and [one with the right answer](https://stackoverflow.com/questions/3587227/how-to-interleave-arrays-of-different-length-in-ruby), and I suspect others have run into similar issues.

I'm currently unaware of language precedent here outside the case of similarly-sized arrays, but given the broad immediate application with template tags, I'm not sure language precedent is necessary here.

### Open questions

- Should this be extended to N arrays? One could read all lengths up front and then all entries of each list, to enable an implementation with O(array count) memory overhead and little computational overhead. If this were added, it'd be better to do `Array.interleave(...arrays)` instead of the above method. However, I'm not sure it's common enough of a need to merit inclusion into core, especially given the complexity of it.

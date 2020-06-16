# Array.prototype.set(*array* [ , *offset* [ , *start* [ , *end* ] ] ])

This assigns to `this`, starting at `offset`, the items in `array` from `start` to `end`. I included more spec-like text modeled after `Array.prototype.copyWithin` and `%TypedArray%.prototype.set` to make it a little clearer what I'm looking for.

The defaults for each of these are as follows:

- *offset*: `0`
- *start*: `0`
- *end*: `target.length`

It'd be implemented roughly as this:

```js
const copyWithin = Function.call.bind(Function.call, [].copyWithin)

Array.prototype.set = function (
    array, offset = 0, start = 0, end = array.length
) {
    offset = Math.floor(offset)
    start = Math.floor(start)
    end = Math.floor(end)
    if (offset < 0) throw new RangeError("negative offset")
    if (start > end) throw new RangeError("negative length")

    // This is an obvious candidate for `std::memmove`
    if (this === source) {
        copyWithin(this, offset, start, end)
    } else {
        const length = this.length
        while (offset < length && start < end) {
            this[offset++] = source[start++]
        }
    }
}
```

### Why?

For `Array.prototype.set`, consider the surprisingly frequent usage of [`System.arraycopy`](https://docs.oracle.com/javase/9/docs/api/java/lang/System.html) within Java circles. In performance-sensitive code when you need to copy items across two arrays, it'd be nice to have a native JIT primitive that can do it in a very highly vectorized fashion. (In fact, I'd say most of my simple loops are literally just copying between arrays.) Such a method already exists in typed arrays, but it'd be nice to have that parity be moved over to normal arrays, too, since most normal JS code (even perf-sensitive code) can't lower *all* operations into plain numbers. As for other precedent:

- JS already has `%TypedArray%.prototype.set`, and the non-typed-array variant is almost literally what's proposed here minus typed array-specific coercions and the extra parameters. (It's complete with Get and Set calls in the spec.)
- I've seen `Object.assign(array, values)` in the wild more than once, even though it's clearly wrong.
- Python has the `s[i:j] = t` idiom, which replaces one sublist with another (an extended version of our `.splice`). You can have a more exact analogue via `s[i:j] = t[k:k+(j-i)]`, and I've seen similar code out in the wild before.
- C# has [`System.Array.Copy`](https://docs.microsoft.com/en-us/dotnet/api/system.array.copy) (which is effectively Java's `System.arraycopy`), [`System.Buffer.BlockCopy`](https://docs.microsoft.com/en-us/dotnet/api/system.buffer.blockcopy) (equivalent specialized for primitives), and [`System.Array.CopyTo`](https://docs.microsoft.com/en-us/dotnet/api/system.array.copyto) for the common case of copying a smaller array's contents into a larger array.
- Rust has [`Vec::copy_from_slice`](https://doc.rust-lang.org/std/vec/struct.Vec.html#method.copy_from_slice).
- OCaml has [`Array.blit`](https://caml.inria.fr/pub/docs/manual-ocaml/libref/Array.html) that's effectively Java's `System.arraycopy`.
- C++ has [`std::copy`](http://www.cplusplus.com/reference/algorithm/copy/), which operates on both pointer offsets and iterators.
- C, of course, has [`memcpy`](www.cplusplus.com/reference/clibrary/cstring/memcpy/) and [`memmove`](www.cplusplus.com/reference/clibrary/cstring/memmove/), which are commonly used for this purpose.
- There's other examples in other languages, too (like [Julia](https://docs.julialang.org/en/v1/base/arrays/index.html#Base.copyto!-Tuple{AbstractArray,CartesianIndices,AbstractArray,CartesianIndices})), but you get the point.

## %TypedArray%.prototype.set(*overloaded* [ , *offset* [ , *start* [ , *end* ] ] ])

Update the offset handling to work similarly to my proposed `Array.prototype.set`. Note that the *offset* is the same as what's currently in the spec, so the risk of breakage I suspect would be rather low.

### Why?

I find it surprising that they're *not* there in the typed array variant, and I feel it limits its utility by quite a bit. Engines already have to use a start offset just to perform the algorithm, and it's pretty trivial to go from source end offset to target end offset (it's just a couple subtracts), so it shouldn't be that hard to allow users to provide info to hook into it. Also, most the above precedent for `Array.prototype.set` contains such functionality.

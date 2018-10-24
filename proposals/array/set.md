# Array.prototype.set(*array* [ , *srcOffset* [ , *destOffset* [ , *count* ] ] ])

This assigns to `this`, starting at `destOffset`, the first `count` items in `array` starting from `srcOffset`. I included more spec-like text modeled after `Array.prototype.copyWithin` and `%TypedArray%.prototype.set` to make it a little clearer what I'm looking for.

(The parameter order is to provide sane defaults - the common case for Java's `System.arraycopy` uses `srcPos = 0` and `len = src.length`.)

1. Let *O* be ? ToObject(**this** value).
1. Let *A* be ? ToObject(*array*).
1. Let *destLen* be ? ToLength(? Get(*O*, **"length"**)).
1. Let *srcLen* be ? ToLength(? Get(*A*, **"length"**)).
1. If *srcOffset* is **undefined**, then let *srcOffset* be 0. Else, let *srcOffset* be ? ToLength(*srcOffset*).
1. If *destOffset* is **undefined**, then let *destOffset* be 0. Else, let *destOffset* be ? ToLength(*destOffset*).
1. If *count* is **undefined**, then let *count* be *destLen*. Else, let *count* be ? ToLength(*count*).
1. If *count* < 0, throw a **RangeError** exception.
1. Let *srcEnd* be min(*srcLen*, *srcOffset* + *count*).
1. Let *destEnd* be min(*destLen*, *destOffset* + *count*).
1. Let *i* be *srcOffset*.
1. Let *j* be *destOffset*.
1. Repeat, while *i* < *srcEnd* and *j* < *destEnd*:
    1. Let *fromKey* be ! ToString(*i*).
    1. Let *toKey* be ! ToString(*j*).
    1. Let *fromPresent* be ? HasProperty(*A*, *fromKey*).
    1. If *fromPresent* is **true**, then:
        1. Let *value* be ? Get(*A*, *fromKey*).
        1. Perform ? CreateDataPropertyOrThrow(*O*, *toKey*, *value*).
    1. Else *fromPresent* is **false**:
        1. Perform ? DeletePropertyOrThrow(*O*, *toKey*).
    1. Increase *i* by 1.
    1. Increase *j* by 1.
1. Return **undefined**.

The steps roughly equate to this after some optimization:

```js
Array.prototype.set = function (
    source, srcOffset = 0, destOffset = 0, count = target.length
) {
    if (count < 0) throw new RangeError("negative count")
    count = Math.min(
        count,
        Math.max(0, this.length - srcOffset),
        Math.max(0, source.length - destOffset)
    )

    // This is an obvious candidate for `std::memmove`
    if (this === source) {
        this.copyWithin(srcOffset, destOffset, count)
    } else {
        for (let i = 0; i < count; i++) {
            this[destOffset++] = source[srcOffset++]
        }
    }
}
```

### Why?

For `Array.prototype.set`, consider the surprisingly frequent usage of [`System.arraycopy`](https://docs.oracle.com/javase/9/docs/api/java/lang/System.html) within Java circles. In performance-sensitive code when you need to copy items across two arrays, it'd be nice to have a native JIT primitive that can do it in a very highly vectorized fashion. Such a method already exists in typed arrays, but it'd be nice to have that parity be moved over to normal arrays, too, since most normal JS code (even perf-sensitive code) can't lower *all* operations into plain numbers. As for other precedent:

- JS already has `%TypedArray%.prototype.set`.
- I've seen `Object.assign(array, values)` in the wild more than once, even though it's clearly wrong.
- Python has the `s[i:j] = t` idiom, which replaces one sublist with another (an extended version of our `.splice`).
- C# has [`System.Array.Copy`](https://docs.microsoft.com/en-us/dotnet/api/system.array.copy) (which is effectively Java's `System.arraycopy`), [`System.Buffer.BlockCopy`](https://docs.microsoft.com/en-us/dotnet/api/system.buffer.blockcopy) (equivalent specialized for primitives), and [`System.Array.CopyTo`](https://docs.microsoft.com/en-us/dotnet/api/system.array.copyto) for the common case of copying a smaller array's contents into a larger array.
- Rust has [`Vec::copy_from_slice`](https://doc.rust-lang.org/std/vec/struct.Vec.html#method.copy_from_slice).
- OCaml has [`Array.blit`](https://caml.inria.fr/pub/docs/manual-ocaml/libref/Array.html) that's effectively Java's `System.arraycopy`.
- C++ has [`std::copy`](http://www.cplusplus.com/reference/algorithm/copy/), which operates on pointer offsets only.
- C, of course, has [`memcpy`](www.cplusplus.com/reference/clibrary/cstring/memcpy/).

## %TypedArray%.prototype.set(*overloaded* [ , *srcOffset* [ , *destOffset* [ , *count* ] ] ])

Update the offset handling to work similarly to my proposed `Array.prototype.set`. Note that the *srcOffset* is functionally the same as the existing second *offset* parameter within the spec, so it's not super breaking.

### Why?

I find it surprising that they're *not* there in the typed array variant, and I feel it limits its utility by quite a bit. It's something engines already have to have just to perform the algorithm, so it shouldn't be that hard to allow users to provide info to hook into it.

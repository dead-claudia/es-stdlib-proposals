# Proposal of various array additions

Arrays are incredibly useful. They make working with structured data much easier. But there are a few things I'd like to see. Each of them are detailed with their related algorithm and rationale.

## Array.prototype.delete(*item* [ , *startOffset* [ , *all* ] ])

This removes the first occurrence of `item` from `this` (or all occurrences if `all` is truthy), optionally starting from `startOffset`, returning `true` if the item was found and `false` if it was not. This is mostly sugar for this, but the algorithm can be substantially optimized:

```js
// Sugar for this
Array.prototype.delete = function (item, startOffset = undefined, all = false) {
    let index = this.indexOf(item, startOffset)
    if (index < 0) return false
    if (all) {
        do {
            this.splice(item, 1)
            index = this.indexOf(item, index)
        } while (index >= 0)
    } else {
        this.splice(item, 1)
    }
    return true
}

// Expected scalar implementation
Array.prototype.delete = function (item, startOffset = undefined, all = false) -> Completion {
    const O = Object(this)
    const length = ToLength(this.length)
    startOffset = ToInteger(startOffset)
    all = Boolean(startOffset)
    if (startOffset < 0) startOffset = Math.min(0, startOffset + oldLength)

    for (let i = startOffset; i !== oldLength; i++) {
        const entry = this[i]
        if (SameValueZero(entry, item)) {
            let newLength = i++
            if (all) {
                while (i !== oldLength) {
                    const entry = this[i++]
                    if (!SameValueZero(entry, item)) this[newLength++] = entry
                }
            } else {
                while (i !== oldLength) this[newLength++] = this[i++]
            }

            this.length = newLength
            // For arrays, this step can be omitted
            if (!isNativeArray(this)) {
                for (let i = newLength; i < oldLength; i++) delete this[i]
            }
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
- Rust recently via [`Vec::remove_item`](https://doc.rust-lang.org/std/vec/struct.Vec.html#method.remove_item).
- Ruby via [`Array#delete`](https://docs.ruby-lang.org/en/2.5.0/Array.html#method-i-delete)

## Array.prototype.set(*array* [ , *srcOffset* [ , *destOffset* [ , *count* ] ] ])

This assigns to `this`, starting at `destOffset`, the first `count` items in `array` starting from `srcOffset`. I included more spec-like text modeled after `Array.prototype.copyWithin` and `TypedArray.prototype.set` to make it a little clearer what I'm looking for.

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

- JS already has `TypedArray.prototype.set`.
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

## Array.prototype.reject(*func* [ , *thisValue* ])

Basically the same as `Array.prototype.filter`, but with a negated `func`. It could be fully polyfilled as this:

```js
const filter = Function.call.bind(Function.call, Array.prototype.filter)

Object.assign(Array.prototype, {
    reject(func, thisValue = void 0) {
        return filter(this, (...args) => !func.apply(thisValue, args))
    }
})
```

### Why?

1. It gets annoying having to do `.filter(x => !isEmpty(x))` when the inverse, `.filter(isNotEmpty)`, is much shorter to type.
1. We already have `.filter`. This is just a few small lines of code.
1. [Underscore](http://underscorejs.org/#reject), [Lodash](https://lodash.com/docs#reject), and [Ramda](http://ramdajs.com/docs/#reject) all three have had it for pretty much the whole time they've existed.
1. Most languages that have a `filter` method/function (e.g. most FP languages) or equivalent (e.g. Ruby/Smalltalk) have an inverse with that name.
    - Python is one of the few exceptions, with a `filter` global and an [`itertools.filterfalse`](https://docs.python.org/3/library/itertools.html#itertools.filterfalse) standard library export.

## Array.prototype.insert(*index*, ...*items*)

Basically the same as `Array.prototype.splice`, but with a couple differences:

1. No ability to remove items when replacing.
1. Returns the new length like `.push`, rather than an empty array.

### Why?

1. It'd be nice to have something more explicit than `array.splice(index, 0, ...items)`, with that magic-seeming 0.
1. Many languages already offer something like the above (and idiomatically prefer it), even though they have an equivalent to `.splice`:
    - Python via [`MutableSequence.insert`](https://docs.python.org/3/library/stdtypes.html#mutable-sequence-types) (`s.insert(i, x)` &harr; `s[i:i] = [x]`)
    - Ruby via [`Array#insert`](https://docs.ruby-lang.org/en/2.5.0/Array.html#method-i-insert) (`ary.insert(i, xs...)` &harr; `ary[i, i] = [xs...]`)
    - Java via [`ArrayList.add`](https://docs.oracle.com/javase/9/docs/api/java/util/ArrayList.html#add-int-E-) (`list.add(i, x)` &harr; `list.addAll(i, new T[] {x})`)

## Array.prototype.pushAll(*items*)

Basically sugar for this code:

```js
Array.prototype.pushAll = function (items) {
    let index = this.length
    for (const item of items) this[index++] = item
    this.length = index
    return this
}
```

### Why?

1. How often do you do `array.push(...entries)` or `array.push.apply(array, entries)`? It's no small amount, and it's very amenable to optimization.
    - For `this` and `items` being both dense arrays, this could be inlined to a simple `this.length += other.length` + conditional `realloc` + two `memcpy` calls, regardless of size.
1. When appending larger arrays and sequences (more than ~500K on FF's REPL, ~120K in Node's REPL), you can't do `array.push(...entries)` without running into a stack limit.
    - This doesn't come up very often in practice, but it has been hit before, and in recursive iteration of large trees and such, the problem surfaces quicker (each function call in V8 apparently takes about ~40 bytes base + 1 per given argument + 6 bytes overhead with misaligned arguments from testing).
1. Many languages already offer something similar:
    - Python via [`MutableSequence.extend` and `s += t`](https://docs.python.org/3.6/library/stdtypes.html#mutable-sequence-types).
    - Ruby via [`Array#concat`](https://ruby-doc.org/core-2.5.0/Array.html#method-i-concat)
    - Java via [`Collection.addAll`](https://docs.oracle.com/javase/9/docs/api/java/util/Collection.html#addAll-java.util.Collection-)
    - Rust via the [`std::iter::Extend<A>` trait](https://doc.rust-lang.org/std/iter/trait.Extend.html)

## Array.prototype.sort(*compareType* [ , *reserved1* [ , *reserved2* ] ])

- If *compareType* is **"numeric"**, they are compared by numeric value
- If *compareType* is **"lexicographic"**, they are compared lexicographically (the current default)
- If *compareType* is **"locale-sensitive"**. they are compared via `(a, b) => a.localeCompare(b, reserved1, reserved2)`

### Why?

1. When you want to sort an array of numbers, how often are you *really* expecting `[1, 10, 100, ..., 2, 20, 21, ...]`?
1. When you want to sort strings, lexicographically makes the most sense for the computer, but locale-sensitive makes the most sense for people. You shouldn't cross these up.
1. Starting with this, it could be a sufficient starting point for finally fixing this part of array handling. (We could also push people away from the existing zero-argument version.)

## Array.prototype.sortBy(*key* [ , ...*args* ])

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

## Array.prototype.sorted(*compareType* [ , *reserved1* [ , *reserved2* ] ]), Array.prototype.sorted(*comparator*), Array.prototype.sortedBy(*key* [ , ...*args* ]), TypedArray.prototype.sorted(*comparator*), TypedArray.prototype.sortedBy(*key* [ , ...*args* ])

Basically a non-destructive sort that creates a new array in the process. Each of these mirrors the mutating version:

- Array.prototype.sort(*comparator*), TypedArray.prototype.sort(*comparator*) &rarr; Array.prototype.sorted(*comparator*), TypedArray.prototype.sorted(*comparator*)
- Array.prototype.sort(*compareType* [ , *reserved1* [ , *reserved2* ] ]) &rarr; Array.prototype.sorted(*compareType* [ , *reserved1* [ , *reserved2* ] ]), TypedArray.prototype.sorted(*compareType* [ , *reserved1* [ , *reserved2* ] ])
- Array.prototype.sortBy(*key* [ , ...*args* ]), TypedArray.prototype.sortBy(*key* [ , ...*args* ]) &rarr; Array.prototype.sortedBy(*key* [ , ...*args* ]), TypedArray.prototype.sortedBy(*key* [ , ...*args* ])

### Why?

1. In my experience, it's rare to want to sort an array without also cloning it first. I've found myself do `array.slice().sort()` or `array.concat().sort()` much more frequently than destructive sorting, with the only exception being in performance-sensitive algorithms.
1. Some languages already offern something similar:
    - Python via [`sorted(iter)`](https://docs.python.org/3/library/functions.html#sorted)
    - Ruby via [`Array#sort`](https://ruby-doc.org/core-2.5.0/Array.html#method-i-sort)
    - Clojure via [`clojure.core/sort`](https://clojuredocs.org/clojure.core/sort) (when the operand isn't a native array)
    - Java via [`Stream<T>.sorted()`](https://docs.oracle.com/javase/9/docs/api/java/util/stream/Stream.html#sorted--) (for streams only)

## Array.prototype.sortedSearch(*value* [ , *comparator* ]) + other Array/TypedArray variants

Search an array for a value using a sorted search and return its index, or -1 if it doesn't exist. The name is chosen to not imply binary search (the most common algorithm), since faster algorithms (*O*(√(log n))) exist for integer-only arrays, if extra allocation is permitted, and binary search is slower than iterative searching for smaller arrays and segments.

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

## Array.prototype.partition(*func* [ , *thisValue* ])

Basically a combined `filter` + `reject`, just returning a pair of two arrays where the first are the ones `func` returned truthy on, the second where it returned falsy on. An engine would do better to do that in a single pass rather than two, since it's faster. It'd look something close to this:

```js
Array.prototype.partition = function (func, thisValue) {
    let pass = []
    let fail = []
    for (let i = 0; i < this.length; i++) {
        let item = this[i++]
        ;(func.call(thisValue, item, i, this) ? pass : fail).push(item)
    }
    return [pass, fail]
}
```

### Why?

1. A very large number of `filter`/`reject` cases reduce to this. Any time you see `x.filter(x => cond(x))` followed by `x.filter(x => !cond(x))`, you're looking at something better written with a partition (it's DRYer without losing readability).
1. Underscore has had it for [a few years at this point](https://github.com/jashkenas/underscore/commit/407d027ccac09ab8f6ddfa1cd5a81ee9516e5d4b), and it was requested as far back as [2011](https://github.com/jashkenas/underscore/issues/132). Lodash added it [shortly thereafter](https://github.com/lodash/lodash/commit/5f02c336e7d584393e98ac35f80efb1d70a3ab98).
1. Several languages already offer something similar:
    - Ruby via [`Enumerable#partition`](https://ruby-doc.org/core-2.5.0/Enumerable.html#method-i-partition)
    - Clojure via [`clojure.core/partition`](https://clojuredocs.org/clojure.core/partition)
    - Kotlin via [`Array<T>.partition`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/partition.html), etc.
    - Rust via [`Iterator<T>.partition`](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.partition)
    - Haskell via [`Data.List.partition`](https://hackage.haskell.org/package/base-4.11.1.0/docs/Data-List.html#v:partition)
    - Swift and C++ have in-place partitions via [`Array<T>.partition(by:)`](https://developer.apple.com/documentation/swift/array/3017524-partition) and [`partition` in `<algorithm>`](http://www.cplusplus.com/reference/algorithm/partition/) respectively, where the return value is the start of the second partition.

## Array.prototype.subarray(*from* [ , *to* ])

Return an array view of the array from *from* to *to*. It works similarly to `TypedArray.prototype.subarray`, just generalized to array-likes.

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

## get Array.prototype.reversed, get TypedArray.prototype.reversed

Return an array view of the array where all offsets are subtracted from the length to get the real index. In effect, this does the following:

- `.map`, `.filter`, and `.forEach` would iterate the backing array in reverse.
- `.lastIndexOf` and `.reduceRight` would be equivalent to `.reversed.indexOf` and `.reversed.reduce`.
- `.reversed.find` and `.reversed.findIndex` would find the *last* occurrence, not the first.
- `.sorted().reversed` would do a reverse sort.

Each instance would similarly inherit from the relevant prototype, but they would exist as exotic objects that delegate to their backing arrays. There are a few other particulars:

- If you wanted an actual reversed clone, you could do `.reversed.slice()` or `.slice().reversed`.
- It's a proper view, so changes to the backing array would propagate to the view.
- An engine might choose to optimize some of the view handling, such as with reversed slicing and reverse sorting.
- Reversed views are similar to subarray views in that they're visibly immutable and delegate everything to the backing array.

### Why?

1. It's broadly useful, and most of the time you want a mid-chain reversal, you don't really *need* to do it destructively.
1. It simplifies the concept of "find last item" to something you can just build from existing stuff.

## ArrayBuffer.from(*string* [ , *encoding* ])<br>DataView.prototype.getString(*offset* [ , *byteLength* [ , *encoding* ] ])<br>DataView.prototype.setString(*offset*, *string* [ , *encoding* ])<br>ArrayBuffer.prototype.transcode(*sourceEncoding*, *targetEncoding*)

- **`ArrayBuffer.from`**: Transcode the list of code units within *string* into a list of bytes encoded using *encoding* (default: **`"utf-8"`**), and return a new array buffer with its data set to those bytes.

- **`DataView.prototype.getString`**: Get the list of the first *byteLength* (default: **`this`**.[[ByteLength]] - *offset*, truncated at the end of the array buffer if necessary) bytes within **`this`**.[[ViewedArrayBuffer]] starting from *offset*, transcode it from *encoding* (default: **`"utf-8"`**) to code units, and return the resulting string.

- **`DataView.prototype.setString`**: Transcode the list of code units within *string* into a list of bytes encoded using *encoding* (default: **`"utf-8"`**), and write the first *byteLength* (default: length of *bytes*) bytes to **`this`**.[[ViewedArrayBuffer]], starting from *offset* (default: 0) and stopping after it either hits the end of the array buffer or runs out of code units.

- **`ArrayBuffer.prototype.transcode`**: Get the list of bytes within **`this`**.[[ArrayBufferData]], transcode it from *sourceEncoding* to code units and then from code units into *targetEncoding*, and then return a new array buffer with its data set to those bytes.

For all three, each of following encodings are required to be supported in both directions, but others may optionally be supported and not necessarily in both ways (like a theoretical read-only `"auto"` for UTF-8/UTF-16/etc. detection):

- **`"utf-8"`**
- [**`"wtf-8"`**](https://simonsapin.github.io/wtf-8/)
- **`"ascii"`**
- **`"utf-16-be"`**
- **`"utf-16-le"`** (alias: **`"utf-16"`**)
- **`"utf-32-be"`**
- **`"utf-32-le"`** (alias: **`"utf-32"`**)

Transcoding works like this:

- **`"utf-8"`**: From code units, convert from UTF-16 to code points, then to UTF-8, and then to bytes. From bytes, convert from UTF-8 to code points, then from code points to UTF-16, and then to code units.
- **`"wtf-8"`**: Transcode it according to the algorithm [described here](https://simonsapin.github.io/wtf-8/#converting-wtf-8-ill-formed-utf-16).
- **`"ascii"`**: From code points, mask off the top 9 bits from each code unit and emit the low 8 bits of each. To code units, mask to the low 7 bits, zero-extend from 8 to 16 bits, and emit it as code units.
- **`"utf-16-be"`**/**`"utf-16-le"`**: From code units, emit the raw code units in the relevant byte order. To code units, read the code units in the relevant byte order.
- **`"utf-32-be"`**/**`"utf-32-le"`**: From code units, convert from UTF-16 to code points, zero-extend from 21 to 32 bits, and emit the raw 32-bit code points in the relevant byte order, padded with zeroes. To code units, mask to the low 21 bits, convert to UTF-16, and emit the raw code units.

Rationale for each:

- **`"utf-8"`**: encoding for 99% of the web.
- **`"wtf-8"`**](https://simonsapin.github.io/wtf-8/: Similar to UTF-8, but losslessly encodes UTF-16 and is useful for internal representations that have to cross up UTF-8 and UTF-16/UCS-2.
- **`"ascii"`**: Many legacy formats require this.
- **`"utf-16-be"`**/**`"utf-16-le"`**/**`"utf-16"`**: JavaScript's native format is UTF-16.
- **`"utf-32-be"`**/**`"utf-32-le"`**/**`"utf-32"`**: UTF-32 requires no extra processing overhead, which is a good fit for many internal APIs where decoding overhead can be a problem. (This isn't likely in normal JS code, but could come up in WASM interop.)

Unknown encodings should cause a **`ReferenceError`** to be thrown, even on zero-length array buffers and strings. (This allows for feature detection.)

Notes:

- The first *byteLength* bytes in *bytes* within **`DataView.prototype.getString`**'s algorithm are not required to consitute a valid byte sequence according to *encoding* - it is delimited and truncated at the byte level, not the code point/character level.
- The methods are intentionally defined on `DataView`, not `Uint8Array`. If you wish to transcode such an array, create a temporary DataView wrapper for it and operate on *it*.
- The transcoded bytes written in **`DataView.prototype.transcode`**'s algorithm may be a different length of bytes than the originally encoded bytes.

### Why?

1. Engines can take advantage of their internal representation to do this extremely cheaply.
    - In particular `dataView.setString(offset, "string", "enc")` could be optimized to just a raw `memcpy` loop.
    - Also, `dataView.getString(offset, "enc")` could be optimized to a raw `memcpy` loop + cheap string wrapper if `"enc"` happens to match one of the implementation's internal encoding (like UTF-16 or ASCII). Engines already do this for static strings.
    - 99% of encoding-related copying and conversion can very easily be SIMD-accelerated.
1. Node.js (via [`Buffer`](https://nodejs.org/api/buffer.html)) and the DOM (via [`TextEncoder`](https://developer.mozilla.org/en-US/docs/Web/API/TextEncoder)) both provide means to do this.
1. Some complex string operations (like parsing) are much more efficient to do without the overhead of language strings.
1. This is useful for passing strings to and from exported methods in WASM and asm.js modules. (Most languages deal with strings in UTF-8.)

## Non-proposals

These are things I've been asked to promote, but there's reasons why I won't include them in this list.

- `Array.prototype.shuffle()` - The use case isn't general enough to merit being a new global in my opinion, especially considering the pseudo-random number generator complexity you need - most engines' `Math.random` implementations only have up to 128 bits of state, which means [they can only fairly shuffle arrays up to size 34](https://en.wikipedia.org/wiki/Fisher%E2%80%93Yates_shuffle#Pseudorandom_generators). This means that if you have an array of 100 items, a shuffle based on `Math.random` can only generate a **very** small fraction of the possibilities that array really has. (You *could* stretch that up to 170 using the [xorshift1024*](https://en.wikipedia.org/wiki/Xorshift#xorshift*) or a few thousand entries with a [Mersenne Twister](https://en.wikipedia.org/wiki/Mersenne_Twister) or related generator, but you'd need hardware cryptographic primitives for anything reliably better.)

    What really needs to happen is a third party implementation needs to be created that accounts for the fact you need enough PSRNG states to cover every permutation an array could have, and this requires quite a bit of non-trivial code. So far, neither [Lodash](https://github.com/lodash/lodash/blob/4.17.10/lodash.js#L6711) nor [Underscore](https://github.com/jashkenas/underscore/blob/master/underscore.js#L397) use an adequate PSRNG to do this, since they defer to `Math.random`.

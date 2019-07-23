# Array.prototype.equals(*other*), %TypedArray%.prototype.equals(*other*)

This checks if two arrays are of equal length and their entries are identical. Basically, this:

```js
Array.prototype.equal = function (other) {
    var length = this.length
    if (length !== other.length) return false
    for (var i = 0; i < length; i++) {
        if (this[i] !== other[i]) return false
    }
    return true
}
```

### Why?

It's not uncommon to want to compare arrays for equality, especially for things like diffing and memoization. It's also a *very* obvious case of specialized code gen for dense arrays:

- Integer, double, and object arrays can just read the inner data directly, skipping the overhead of accessing the array.
- Typed array code gen here doesn't need to care about the data size, just the byte length.
- This is basically `arr->size() == other->size() && memcmp(arr->data, other->data, arr->size()) == 0` in C (although compilers don't optimize it like they're supposed to - [GCC bug](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=52171), [LLVM patch/discussion](https://reviews.llvm.org/D31290)).

In x86 and x86-64, there is native support that makes this more compelling, similarly to why `Array.prototype.copyWithin` and its typed array equivalent exist:

```asm
; Return = rax
; Array 1 = rsi (clobbered)
; Array 2 = rdi (clobbered)
        mov rcx, [rsi+length_offset] ; Load own length
        mov rax, js_false            ; Set result to `false`, in case we jump
        cmp rcx, [rdi+length_offset] ; Compare other length
        jne next                     ; Jump if not equal
        cld                          ; Scan forward with `rep cmpsb`
        mov rsi, [rsi+data_offset]   ; Load own data
        mov rdi, [rdi+data_offset]   ; Load other data
        rep cmpsb                    ; Compare data
.next:  cmove rax, js_true           ; Set result to `true` if length + values equal
```

It's not quite that easy in ARM, but it's still not *bad* (this assumes 4-bit array length alignment):

```asm
; Return = r0
; Array 1 = r0
; Array 2 = r1
        MOV r0, r2 ; for easier return setting
        PUSH {r4,r5}
        LDR r3, r1, #length_offset
        LDR r4, r2, #length_offset
        MOV32 r0, #js_false
        CMP r3, r4
        BNE next
        LDR r1, r1, #data_offset
        LDR r2, r2, #data_offset
        B enter
.loop:  LDR r4, r1, r3
        LDR r5, r2, r3
        TEQ r4, r5
        BNE next
.enter: SUBS r3, r3, #1
        BNC loop
.pass:  MOV32 r0, #js_true
.next:  POP {r4,r5}
```

We also already have a *lot* of language precedent elsewhere - to name a few:

- Java via [`Arrays.equals`](https://docs.oracle.com/javase/9/docs/api/java/util/Arrays.html#equals-java.lang.Object:A-java.lang.Object:A-)
- Rust via implementing [`Eq`](https://doc.rust-lang.org/std/cmp/trait.Eq.html) for vectors and slices
- Ruby via [`Array#==`](https://docs.ruby-lang.org/en/2.5.0/Array.html#method-i-3D-3D) (Most other collection-heavy languages like Python, Julia, and Scala are similar)
- C via [`memcmp`](http://www.cplusplus.com/reference/cstring/memcmp/) (how often does *C* have something high-level that JS doesn't?)
- C++ via [`std::equal`](http://www.cplusplus.com/reference/algorithm/equal/)
- Clojure via [`=`](https://clojuredocs.org/clojure.core/=)
- Julia via [`Base.:==(x::Arr, y::Arr)`](https://github.com/JuliaLang/julia/blob/84024a1f44127c4993cc25f8629588f8807706c3/base/array.jl#L1400-L1409)

And of course, there's significant runtime/library precedent:

- Node.js has implemented [buffer equality](https://nodejs.org/api/buffer.html#buffer_buf_equals_otherbuffer) for a while (since v0.11.13), and it of course [delegates to the obvious `memcmp(...) == 0`](https://github.com/nodejs/node/blob/master/src/node_buffer.cc#L711-L725). This proposal is technically a non-breaking extension of that for typed arrays, so they could just remove their implementation once V8 supports it.
- [Lodash](https://lodash.com/docs#isEqual) and [Underscore](https://underscorejs.org/#isEqual) have `_.isEqual` for deep object comparisons, and this would accelerate those for a very common case. (Those are only two of [hundreds of npm packages](https://www.npmjs.com/search?q=equal) for this general kind of thing.)
- [`array-equal`](https://www.npmjs.com/package/array-equal) currently has about 1.5 million downloads a week and [`shallow-equal`](https://www.npmjs.com/package/shallow-equal) has about 350 thousand per week, but those are only two of [over a hundred](https://www.npmjs.com/search?q=array%20equal) dedicated specifically *for* shallow array equality.

I could go on for a while. JS *is* really the exception here, not the norm for languages of similar scale.

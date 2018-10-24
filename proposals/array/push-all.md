# Array.prototype.pushAll(*items*)

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

# Array.prototype.insert(*index*, ...*items*)

Basically the same as `Array.prototype.splice`, but with a couple differences:

1. No ability to remove items when replacing.
1. Returns the new length like `.push`, rather than an empty array.

### Why?

1. It'd be nice to have something more explicit than `array.splice(index, 0, ...items)`, with that magic-seeming 0.
1. Many languages already offer something like the above (and idiomatically prefer it), even though they have an equivalent to `.splice`:
    - Python via [`MutableSequence.insert`](https://docs.python.org/3/library/stdtypes.html#mutable-sequence-types) (`s.insert(i, x)` &harr; `s[i:i] = [x]`)
    - Ruby via [`Array#insert`](https://docs.ruby-lang.org/en/2.5.0/Array.html#method-i-insert) (`ary.insert(i, xs...)` &harr; `ary[i, i] = [xs...]`)
    - Java via [`ArrayList.add`](https://docs.oracle.com/javase/9/docs/api/java/util/ArrayList.html#add-int-E-) (`list.add(i, x)` &harr; `list.addAll(i, new T[] {x})`)

# Value matching

This would involve two things:

- `Object.matches(a, b)` - Whether two objects structurally match
- `[Symbol.matches](other, recurse)` - Whether an object considers itself to match another, accepts a function to recurse.

It'd all center around this basic algorithm, for Matches(*a*, *b*, *stack*):

1. If Type(*a*) is different from Type(*b*), return **`false`**.
1. If SameValueZero(*a*, *b*) is **`true`**, return **`true`**.
1. If Type(*a*) is Symbol, return SameValue(*a*.[[Description]], *b*.[[Description]]).
1. If Type(*a*) is not Object, return **`false`**.
1. For each *value* in *stack*:
    1. If SameValue(*a*, *value*), return **`false`**.
    1. If SameValue(*b*, *value*), return **`false`**.
1. If ? *a*.\[[GetPrototypeOf]]() is different from ? *b*.\[[GetPrototypeOf]](), return **`false`**.
1. Let *childStack* be the concatenation of *stack* and «*a*».
1. Let *matches* be ? GetMethod(*a*, @@matches).
1. If *matches* is not **`undefined`**, do:
    1. Let *recurse* be a function *F* that, when called with arguments *achild* and *bchild*, runs the following steps:
        1. Return ? Matches(*achild*, *bchild*, *F*.[[ChildStack]]).
    1. Set *recurse*.[[ChildStack]] to *childStack*.
    1. Return ? ToBoolean(? Call(*matches*, *a*, «*b*, *recurse*»)).
1. If IsCallable(*a*) is **`true`**, return **`false`**.
1. If IsCallable(*b*) is **`true`**, return **`false`**.
1. Let *aKeys* be ? EnumerableOwnProperties(*a*, key).
1. Let *bKeys* be ? EnumerableOwnProperties(*b*, key).
1. For each *key* in *akeys*, do:
    1. If *key* is not in *bkeys*, return **`false`**.
    1. If ? Matches(? Get(*a*, *key*), ? Get(*b*, *key*), *childStack*) is **`false`**, return **`false`**.
    1. Remove *key* from *bkeys*.
1. If *bkeys* is not empty, return **`false`**.
1. Return **`true`**.

Several native types would implement `Symbol.matches` as appropriate:

- Error.prototype[@@matches](*b*, *recurse*)
    1. Let *a* be **this** value.
    1. If Type(*a*) is not Object, throw a **TypeError** exception.
    1. If Type(*b*) is not Object, throw a **TypeError** exception.
    1. If IsCallable(*recurse*) is **`false`**, throw a **TypeError** exception.
    1. If *a* does not have a [[ErrorData]] internal slot, throw a **TypeError** exception.
    1. If *b* does not have a [[ErrorData]] internal slot, return **`false`**.
    1. Let *amessage* be ? ToString(? Get(*a*, **`"message"`**)).
    1. Let *bmessage* be ? ToString(? Get(*a*, **`"message"`**)).
    1. Return SameValue(*amessage*, *bmessage*).

- Number.prototype[@@matches](*b*, *recurse*)
    1. Let *a* be **this** value.
    1. If Type(*a*) is not Object, throw a **TypeError** exception.
    1. If Type(*b*) is not Object, throw a **TypeError** exception.
    1. If IsCallable(*recurse*) is **`false`**, throw a **TypeError** exception.
    1. If *a* does not have a [[NumberData]] internal slot, throw a **TypeError** exception.
    1. If *b* does not have a [[NumberData]] internal slot, return **`false`**.
    1. Return SameValueZero(*a*.[[NumberData]], *b*.[[NumberData]]).

- Date.prototype[@@matches](*b*, *recurse*)
    1. Let *a* be **this** value.
    1. If Type(*a*) is not Object, throw a **TypeError** exception.
    1. If Type(*b*) is not Object, throw a **TypeError** exception.
    1. If IsCallable(*recurse*) is **`false`**, throw a **TypeError** exception.
    1. If *a* does not have a [[DateValue]] internal slot, throw a **TypeError** exception.
    1. If *b* does not have a [[DateValue]] internal slot, return **`false`**.
    1. Return SameValueZero(*a*.[[DateValue]], *b*.[[DateValue]]).

- String.prototype[@@matches](*b*, *recurse*)
    1. Let *a* be **this** value.
    1. If Type(*a*) is not Object, throw a **TypeError** exception.
    1. If Type(*b*) is not Object, throw a **TypeError** exception.
    1. If IsCallable(*recurse*) is **`false`**, throw a **TypeError** exception.
    1. If *a* does not have a [[StringData]] internal slot, throw a **TypeError** exception.
    1. If *b* does not have a [[StringData]] internal slot, return **`false`**.
    1. Return SameValueZero(*a*.[[StringData]], *b*.[[StringData]]).

- RegExp.prototype[@@matches](*b*, *recurse*)
    1. Let *a* be **this** value.
    1. If Type(*a*) is not Object, throw a **TypeError** exception.
    1. If Type(*b*) is not Object, throw a **TypeError** exception.
    1. If IsCallable(*recurse*) is **`false`**, throw a **TypeError** exception.
    1. If *a* does not have a [[RegExpMatcher]] internal slot, throw a **TypeError** exception.
    1. If *b* does not have a [[RegExpMatcher]] internal slot, return **`false`**.
    1. If SameValue(*a*.[[OriginalSource]], *b*.[[OriginalSource]]) is **`false`**, return **`false`**.
    1. If SameValue(*a*.[[OriginalFlags]], *b*.[[OriginalFlags]]) is **`false`**, return **`false`**.
    1. If SameValue(? Get(*a*, **`"lastIndex"`**), ? Get(*b*, **`"lastIndex"`**)) is **`false`**, return **`false`**.
    1. Return **`true`**.

- Array.prototype[@@matches](*b*, *recurse*)
    1. Let *a* be **this** value.
    1. If Type(*a*) is not Object, throw a **TypeError** exception.
    1. If Type(*b*) is not Object, throw a **TypeError** exception.
    1. If IsCallable(*recurse*) is **`false`**, throw a **TypeError** exception.
    1. If ? IsArray(*a*) is **`false`**, throw a **TypeError** exception.
    1. If ? IsArray(*b*) is **`false`**, return **`false`**.
    1. Let *alength* be ? ToLength(? Get(*a*, **`"length"`**)).
    1. Let *blength* be ? ToLength(? Get(*b*, **`"length"`**)).
    1. If *alength* is different from *blength*, return **`false`**.
    1. Let *index* be 0.
    1. Repeat while *index* < *alength*:
        1. Let *avalue* be ? Get(*a*, ! ToString(*index*)).
        1. Let *bvalue* be ? Get(*b*, ! ToString(*index*)).
        1. If ? Call(*recurse*, **`undefined`**, «*avalue*, *bvalue*») is **`false`**, return **`false`**.
    1. Return **`true`**.

- %TypedArray%.prototype[@@matches](*b*, *recurse*)
    1. Let *a* be **this** value.
    1. If Type(*a*) is not Object, throw a **TypeError** exception.
    1. If Type(*b*) is not Object, throw a **TypeError** exception.
    1. If IsCallable(*recurse*) is **`false`**, throw a **TypeError** exception.
    1. If *a* does not have a [[ViewedArrayBuffer]] internal slot, throw a **TypeError** exception.
    1. If *b* does not have a [[ViewedArrayBuffer]] internal slot, return **`false`**.
    1. If *a*.[[TypedArrayName]] is different from *b*.[[TypedArrayName]], return **`false`**.
    1. Assert: *a*.[[ContentType]] is the same as *b*.[[ContentType]].
    1. Let *abuffer* be *a*.[[ViewedArrayBuffer]].
    1. Let *bbuffer* be *b*.[[ViewedArrayBuffer]].
    1. Let *aoffset* be *a*.[[ByteOffset]].
    1. Let *boffset* be *b*.[[ByteOffset]].
    1. Let *alength* be *a*.[[ArrayLength]].
    1. Let *blength* be *b*.[[ArrayLength]].
    1. If IsDetachedBuffer(*abuffer*), throw a **TypeError** exception.
    1. If IsDetachedBuffer(*bbuffer*), throw a **TypeError** exception.
    1. If SameValue(*alength*, *blength*) is **`false`**, return **`false`**.
    1. If SameValue(*abuffer*, *bbuffer*) is **`true`** and SameValue(*aoffset*, *boffset*) is **`true`**, return **`true`**.
    1. Let *elementType* be the Element Type value in [Table 62](https://tc39.es/ecma262/#table-the-typedarray-constructors) for *a*.[[TypedArrayName]].
    1. Let *elementSize* be the Element Type value in [Table 62](https://tc39.es/ecma262/#table-the-typedarray-constructors) for *a*.[[TypedArrayName]].
    1. Let *index* be 0.
    1. Repeat while *index* < *alength*:
        1. Let *avalue* be ? GetValueFromBuffer(*a*, ! ToString(*index*)).
        1. Let *bvalue* be ? Get(*b*, ! ToString(*index*)).
        1. Let *aposition* be (*index* × *elementSize*) + *aoffset*.
        1. Let *bposition* be (*index* × *elementSize*) + *boffset*.
        1. Let *avalue* be GetValueFromBuffer(*abuffer*, *aposition*, *elementType*, **`true`**, **Unordered**).
        1. Let *bvalue* be GetValueFromBuffer(*bbuffer*, *bposition*, *elementType*, **`true`**, **Unordered**).
        1. If SameValue(*avalue*, *bvalue*) is **`false`**, return **`false`**.
    1. Return **`true`**.
    1. Note: this generally works the same as [my proposed `this.equals(b)` for typed arrays](https://github.com/isiahmeadows/es-stdlib-proposals/blob/master/proposals/array/equals.md), except that if *b* isn't a typed array of the same type, it simply returns `false` instead.

- Map.prototype[@@matches](*b*, *recurse*)
    1. Let *a* be **this** value.
    1. If Type(*a*) is not Object, throw a **TypeError** exception.
    1. If Type(*b*) is not Object, throw a **TypeError** exception.
    1. If IsCallable(*recurse*) is **`false`**, throw a **TypeError** exception.
    1. If *a* does not have a [[MapData]] internal slot, throw a **TypeError** exception.
    1. If *b* does not have a [[MapData]] internal slot, return **`false`**.
    1. Let *aentries* be the List that is *a*.[[MapData]].
    1. Let *bentries* be the List that is *b*.[[MapData]].
    1. Let *acount* be 0.
    1. For each Record { [[Key]], [[Value]] } *ae* that is an element of *aentries*, in original key insertion order, do:
        1. If *ae*.[[Key]] is not **empty**, set *acount* to *acount* + 1.
    1. Let *bcount* be 0.
    1. For each Record { [[Key]], [[Value]] } *be* that is an element of *bentries*, in original key insertion order, do:
        1. If *be*.[[Key]] is not **empty**, set *bcount* to *bcount* + 1.
    1. If *acount* is different from *bcount*, return **`false`**.
    1. Let *visitedEntries* be an empty List.
    1. For each Record { [[Key]], [[Value]] } *ae* that is an element of *aentries*, in original key insertion order, do:
        1. If *ae*.[[Key]] is not **empty**, do:
            1. Let *skipNext* be **`false`**.
            1. For each Record { [[Key]], [[Value]] } *be* that is an element of *bentries*, in original key insertion order, do:
                1. If *be*.[[Key]] is not **empty** and *skipNext* is **`false`**, do:
                    1. If SameValue(*ae*.[[Key]], *be*.[[Key]]) is **`true`**, do:
                        1. Let *skipVisited* be **`false`**.
                        1. For each value *item* in *visited*, do
                            1. If SameValue(*item*, *be*.[[Key]]), set *skipVisited* to **`true`**.
                        1. If *skipVisited* is **`false`**:
                            1. Append *be*.[[Key]] to *visited*.
                            1. If ? Call(*recurse*, **`undefined`**, «*ae*.[[Value]], *be*.[[Value]]») is **`false`**, return **`false`**.
                            1. Set *skipNext* to **`true`**.
    1. Return **`true`**.

- Set.prototype[@@matches](*b*, *recurse*)
    1. Let *a* be **this** value.
    1. If Type(*a*) is not Object, throw a **TypeError** exception.
    1. If Type(*b*) is not Object, throw a **TypeError** exception.
    1. If IsCallable(*recurse*) is **`false`**, throw a **TypeError** exception.
    1. If *a* does not have a [[SetData]] internal slot, throw a **TypeError** exception.
    1. If *b* does not have a [[SetData]] internal slot, return **`false`**.
    1. Let *aentries* be the List that is *a*.[[SetData]].
    1. Let *bentries* be the List that is *b*.[[SetData]].
    1. Let *acount* be 0.
    1. For each *ae* that is an element of *aentries*, do:
        1. If *ae* is not **empty**, set *acount* to *acount* + 1.
    1. Let *bcount* be 0.
    1. For each *be* that is an element of *bentries*, do:
        1. If *be* is not **empty**, set *bcount* to *bcount* + 1.
    1. If *acount* is different from *bcount*, return **`false`**.
    1. Let *visitedEntries* be an empty List.
    1. For each *ae* that is an element of *aentries*, do:
        1. If *ae* is not **empty**, do:
            1. Let *skipNext* be **`false`**.
            1. For each *be* that is an element of *bentries*, do:
                1. If *be* is not **empty** and *skipNext* is **`false`**, do:
                    1. If SameValue(*ae*, *be*) is **`true`**, do:
                        1. Let *skipVisited* be **`false`**.
                        1. For each value *item* in *visited*, do
                            1. If SameValue(*item*, *be*), set *skipVisited* to **`true`**.
                        1. If *skipVisited* is **`false`**:
                            1. Append *be* to *visited*.
                            1. If ? Call(*recurse*, **`undefined`**, «*ae*, *be*») is **`false`**, return **`false`**.
                            1. Set *skipNext* to **`true`**.
    1. Return **`true`**.

- DataView.prototype[@@matches](*b*, *recurse*)
    1. Let *a* be **this** value.
    1. If Type(*a*) is not Object, throw a **TypeError** exception.
    1. If Type(*b*) is not Object, throw a **TypeError** exception.
    1. If IsCallable(*recurse*) is **`false`**, throw a **TypeError** exception.
    1. If *a* does not have a [[DataView]] internal slot, throw a **TypeError** exception.
    1. If *b* does not have a [[DataView]] internal slot, return **`false`**.
    1. If IsDetachedBuffer(*a*.[[ViewedArrayBuffer]]), throw a **TypeError** exception.
    1. If IsDetachedBuffer(*b*.[[ViewedArrayBuffer]]), throw a **TypeError** exception.
    1. If SameValue(*a*.[[ViewedArrayBuffer]], *b*.[[ViewedArrayBuffer]]) is **`false`**, return **`false`**.
    1. If SameValue(*a*.[[ByteOffset]], *b*.[[ByteOffset]]) is **`false`**, return **`false`**.
    1. If SameValue(*a*.[[ByteLength]], *b*.[[ByteLength]]) is **`false`**, return **`false`**.
    1. Return **`true`**.

### Why?

Lots of library precedent even in the JS world:

- [Lodash's `_.isEqual`](https://lodash.com/docs/4.17.15#isEqual)
- [Node has its own deep equality assertions](https://nodejs.org/api/assert.html#assert_assert_deepstrictequal_actual_expected_message)
- [The breadth and popularity of other deep equality modules on npm](https://www.npmjs.com/search?q=deep%20equal)
    - A standalone port of Node's `assert.deepEqual` that simply returns a boolean [was downloaded 6.8M times last week (as of 25 Dec 2019) with 1.9K public dependents on npm.](https://www.npmjs.com/package/deep-equal).
    - `fast-deep-equal`, a similar module but with support for typed arrays, maps, and sets, [was downloaded 14.1M times last week (as of 25 Dec 2019) with 577 dependents](https://www.npmjs.com/package/fast-deep-equal).
    - Even Lodash's `lodash.isequal` from its modularized build [was downloaded 2.7M times the past week (as of 25 Dec 2019) with 1.8K public dependents on npm](https://www.npmjs.com/package/lodash.isequal). For context, that's the [third most popular of all of Lodash's modularized packages as per npm](https://www.npmjs.com/search?q=keywords%3Alodash-modularized&ranking=popularity), second only to `lodash.merge` (recursive `Object.assign`, [2.4M downloads last week + 2.8K dependents](https://www.npmjs.com/package/lodash.merge) as of 25 Dec 2019) and `lodash.get` ([3.6M downloads last week + 2.8K dependents](https://www.npmjs.com/package/lodash.get) as of 25 Dec 2019).

Language precedent also exists, mostly in favor of making this the default:

- Ruby's `==` is deep by default, and is similarly pluggable.
- Python's `==` is specified in terms of [`__eq__`](https://docs.python.org/3/reference/datamodel.html#object.__eq__).
- Erlang's `==` is deep, but not pluggable.
- Java's `.equals` method is itself *frequently* overridden, and is used for similar effect.
    - Kotlin implements this automatically for data classes, along with `.hashCode()`, to delegate to its arguments. Also, `a == b` is spec'd to work like [`a == null ? b == null : a.equals(b)` in Java](https://kotlinlang.org/docs/reference/operator-overloading.html#equals).
    - Scala [spec's `a == b` similarly as `if (null.eq(this)) then null.eq(that) else this.equals(that)` where `a.eq(b)` is reference equality](https://www.scala-lang.org/files/archive/spec/2.13/12-the-scala-standard-library.html#root-classes).
    - Clojure's `=` [is similarly specified in terms of Java's `.equals` method, but also accounting for `nil`s](https://clojure.github.io/clojure/clojure.core-api.html#clojure.core/=).
- C++'s `==` for just about everything is value-based, and is commonly overloaded as such. Pointer equality is specifically only just that, equality with the values being the pointers themselves and not their dereferenced data, not any sort of special language feature.
    - Rust and other low-level languages are similar on this front.

### Alternatives

The first alternative is obvious, not doing anything. Libraries exist for this sort of thing already, obviously. But of course, I don't feel this is ideal, and so we can move on.

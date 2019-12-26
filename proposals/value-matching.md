# Value matching

This would involve two things:

- `Object.matches(a, b)` - Whether two objects structurally match.
- `[Symbol.matches](other, recurse)` - Whether an object considers itself to match another, accepts a function to recurse.

It'd all center around this basic recursive algorithm.

Object.matches(*a*, *b*):

1. Let *stack* be an empty List.
1. Return Matches(*a*, *b*, *stack*).

Matches(*a*, *b*, *stack*):

1. Assert: *a* is an ECMAScript object.
1. Assert: *b* is an ECMAScript object.
1. Assert: *stack* is a List.
1. Assert: *type* is either **shallow** or **deep**.
1. If MatchesPrimitive(*a*, *b*) evaluates to **`true`**, return **`true`**.
1. Assert: Type(*a*) is the same as Type(*b*).
1. If Type(*a*) is not Object, return **`false`**.
1. For each *value* in *stack*:
    1. If SameValue(*a*, *value*), return **`false`**.
    1. If SameValue(*b*, *value*), return **`false`**.
1. let *childStack* be the concatenation of *stack* and «*a*».
1. Let *aproto* be ? *a*.\[[GetPrototypeOf]]().
1. Let *bproto* be ? *b*.\[[GetPrototypeOf]]().
1. If *aproto* is different from *bproto*, return **`false`**.
1. Let *matches* be ? GetMethod(*a*, @@matches).
1. If *matches* is not **`undefined`**, do:
    1. Let *recurse* be a function *F* with an internal slot [[ChildStack]] that, when called with arguments *achild* and *bchild*, runs the following steps:
        1. Return Matches(*achild*, *bchild*, *F*.[[ChildStack]]).
    1. Set *recurse*.[[ChildStack]] to *childStack*.
    1. Return ? ToBoolean(? Call(*matches*, *a*, «*b*, *recurse*»)).
1. If IsCallable(*a*) is **`true`**, return **`false`**.
1. If IsCallable(*b*) is **`true`**, return **`false`**.
1. If *aproto* is neither **`null`** nor %Object.prototype%, return **`false`**.
1. Let *aKeys* be ? EnumerableOwnProperties(*a*, key).
1. Let *bKeys* be ? EnumerableOwnProperties(*b*, key).
1. For each *key* in *akeys*, do:
    1. If *key* is not in *bkeys*, return **`false`**.
    1. Let *achild* be ? Get(*a*, *key*).
    1. Let *bchild* be ? Get(*b*, *key*).
    1. If ? Matches(*avalue*, *bvalue*, *childStack*) is **`false`**, return **`false`**.
    1. Remove *key* from *bkeys*.
1. If *bkeys* is not empty, return **`false`**.
1. Return **`true`**.

> Note regarding the prototype check against %Object.prototype% and **`null`**: Most class instances that aren't effectively typed POJOs should be compared by identity, not key equality - they might encapsulate data external to the object (think: transaction handles). Examples in the language itself include promises (a future value) and array buffers (a pointer possibly allocated by the embedder itself). I don't go beyond just checking prototypes to detect it, however, since 1. it's impossible to polyfill and 2. it's akin to detecting the existence of private fields and general internal slots, something that breaks the hard encapsulation they were designed to have.

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
- React's [`React.PureComponent`](https://reactjs.org/docs/react-api.html#reactpurecomponent) and [`React.memo`](https://reactjs.org/docs/react-api.html#reactmemo) shallow-compare their old and new arguments internally, but this could be more precise and intelligent than that while still making it even more useful.

A *lot* of language precedent also exists, including in favor of making this the default:

- Ruby's `==` is deep by default, and is similarly pluggable.
- Python's `==` is specified in terms of [`__eq__`](https://docs.python.org/3/reference/datamodel.html#object.__eq__).
- Erlang's `==` is deep and by value, but not pluggable.
    - This is also very common among logic-based programming languages.
- Java's `.equals` method is itself *frequently* overridden, and is used for similar effect.
    - Kotlin [explicitly specifies `a == b` as sugar for `a?.equals(b) ?: b === null`](https://kotlinlang.org/docs/reference/operator-overloading.html#equals) where `===` is reference equality. Scala is similar in that it [defines `def ==(that) = if (null eq this) null eq that else this equals that` on `AnyRef`](https://www.scala-lang.org/files/archive/spec/2.13/12-the-scala-standard-library.html#root-classes), where `eq` is reference equality. (Scala defines `a op b` as sugar for `a.op(b)`, so it's methods all the way down.)
- Clojure: [`(= a b & more)` is value equality if either value is a primitive (including if it's a persistent collection), Java `a.equals(b)` for non-primitive objects](https://github.com/clojure/clojure/blob/ee3553362de9bc3bfd18d4b0b3381e3483c2a34c/src/jvm/clojure/lang/Util.java#L24-L132).
    - Groovy [appears to do similar for its `==` operator](https://github.com/apache/groovy/blob/master/src/main/java/org/codehaus/groovy/runtime/typehandling/DefaultTypeTransformation.java#L616-L655).
- C++'s `==` for just about everything is value-based, and is commonly overloaded as such. Pointer equality is specifically only just that, equality with the values being the pointers themselves and not their dereferenced data, not any sort of special language feature.
- Most languages with type classes (like Haskell) or equivalent constructs (like Rust) implement equality as syntactic sugar for invoking a specific method on a particular type class/trait/etc. dedicated to equality, and feature a separate operator for reference equality or some other form of runtime-level equality (if they target JS, the JVM, or similar).
    - Haskell: [its `Data.Eq` type class](https://hackage.haskell.org/package/base-4.12.0.0/docs/Data-Eq.html#t:Eq)
    - Rust: [its `std::cmp::PartialEq` trait](https://doc.rust-lang.org/std/cmp/trait.PartialEq.html)
    - PureScript: [its `Data.Eq` type class](https://pursuit.purescript.org/packages/purescript-prelude/4.1.0/docs/Data.Eq#t:Eq)
    - Clean: [its `Eq` type class from `StdClass` in `StdEnv`](https://cloogle.org/src/#StdEnv/StdClass;line=46)

And of course, [there's a highly-voted question on Stack Overflow from 2008 asking this very thing](https://stackoverflow.com/q/201183), and [a related one from the same year on arrays of objects](https://stackoverflow.com/q/27030). So this isn't even a recent development, and I would expect this to be *very* much appreciated.

### Alternatives

The first alternative is obvious, not doing anything. Libraries exist for this sort of thing already, obviously. But of course, I don't feel this is ideal for reasons explained above, and so we can move on.

You could also implement this as value-level only, making it not pluggable. While this could be useful in many circumstances, extensibility is important, and you wouldn't be able to make it work for custom collection types like [Immutable's various collections](https://immutable-js.github.io/immutable-js/docs/#/) or [Lodash's sequences](https://lodash.com/docs#lodash) - they both already support the ES iterable protocol. So in this case, we *want* to give them the ability to hook into it as they already do elsewhere.

You could implement this as a new operator, and language precedent *strongly* supports this, but this comes with its own caveats. First, you can't polyfill it, and so it becomes much harder for people to just start using it and experimenting with it. Second, the obvious choices of `==`/`!=` and `===`/`!==` are taken, so you can't use those without breaking web compatibility. And of most the alternatives that I could think of, like `a =:= b`/`a !:= b`, `a =~= b`/`a !~= b`, `a =~ b`/`a !~ b`, and `a ~= b`, some look awful, some produce potential ambiguities and require syntactic side-stepping just to remove the ambiguity, some have no obvious negation, among other things, and many of them have a mixture of these issues. Don't get me wrong - I'd ideally prefer an operator - but there's no clear operator to me to pick that signifies some form of loose equality and doesn't run into one of these issues, hence why I went with a function.

You could add `Object.prototype.equals`, `Date.prototype.equals`, and so on, and some libraries specify such a method (like [Immutable's `List`](https://immutable-js.github.io/immutable-js/docs/#/List/equals), but this could present web compatibility issues, and the semantics wouldn't always align with intuition, either.

- `Object.prototype.equals` - This is yet another thing that's exposed on all objects, including in the global scope. I highly doubt this is web-compatible to add.
- `Array.prototype.equals` - This could just as easily be [shallow equality](https://github.com/isiahmeadows/es-stdlib-proposals/blob/master/proposals/array/equals.md), so people would potentially be surprised when it isn't. Also, [this highly-rated SO answer from 2015 with a different conceptual `Array.prototype.equals`](https://stackoverflow.com/a/14853974) does something in the middle - it recurses into child arrays, but *not* other child objects. (And I suspect that's also [used in the wild more than just by a few developers](https://stackoverflow.com/q/26666513).)
- [Fantasy Land](https://github.com/fantasyland/fantasy-land), a popular and fairly established spec ([dating back to 2013](https://github.com/fantasyland/fantasy-land/tree/8586eb750c6fda88210ffc8302e44ce5fb7bf5e8)) among functional programmers primarily using JS, has [long had a `fantasy-land/equals` method](https://github.com/fantasyland/fantasy-land#setoid) ([first added in 2014](https://github.com/fantasyland/fantasy-land/tree/a5cd0d5474d93e7109b21bcd06d7a6c7431694cf)), but up until [this commit in 2016](https://github.com/fantasyland/fantasy-land/tree/53e75043253a8a9b9684af667c32dfc32fb005de), its methods *weren't* prefixed, and [Ramda still supports that today as a fallback](https://github.com/ramda/ramda/blob/master/source/internal/_equals.js#L50-L53). Since that spec requires both operands being of the same type, this would break that invariant that a lot of utilities in that ecosystem rely on.

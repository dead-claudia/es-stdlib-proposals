# Array.prototype.reject(*func* [ , *thisValue* ]), %TypedArray%.prototype.reject(*func* [ , *thisValue* ])

Basically the same as `Array.prototype.filter`, but with a negated `func`. It could be fully polyfilled as this:

```js
const filter = Function.call.bind(Function.call, Array.prototype.filter)
const apply = Reflect.apply

Object.assign(Array.prototype, {
    reject(func, thisValue = void 0) {
        return filter(this, function () {
            return !apply(func, thisValue, arguments)
        })
    }
})
```

### Why?

1. It gets annoying having to do `.filter(x => !isEmpty(x))` when the inverse, `.filter(isNotEmpty)`, is much shorter to type.
1. We already have `.filter`. This is just a few small lines of code.
1. [Underscore](http://underscorejs.org/#reject), [Lodash](https://lodash.com/docs#reject), and [Ramda](http://ramdajs.com/docs/#reject) all three have had it for pretty much the whole time they've existed.
1. Most languages that have a `filter` method/function (e.g. most FP languages) or equivalent (e.g. Ruby/Smalltalk) have an inverse with that name.
    - Python is one of the few exceptions, with a `filter` global and an [`itertools.filterfalse`](https://docs.python.org/3/library/itertools.html#itertools.filterfalse) standard library export.

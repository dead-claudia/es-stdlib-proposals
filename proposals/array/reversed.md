# get Array.prototype.reversed, get %TypedArray%.prototype.reversed

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

# Array.prototype.sort(*compareType* [ , *reserved1* [ , *reserved2* ] ])

- If *compareType* is **"numeric"**, they are compared by numeric value
- If *compareType* is **"lexicographic"**, they are compared lexicographically (the current default)
- If *compareType* is **"locale-sensitive"**. they are compared via `(a, b) => a.localeCompare(b, reserved1, reserved2)`

### Why?

1. When you want to sort an array of numbers, how often are you *really* expecting `[1, 10, 100, ..., 2, 20, 21, ...]`?
1. When you want to sort strings, lexicographically makes the most sense for the computer, but locale-sensitive makes the most sense for people. You shouldn't cross these up.
1. Starting with this, it could be a sufficient starting point for finally fixing this part of array handling. (We could also push people away from the existing zero-argument version.)

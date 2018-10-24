# Binary encoding, decoding, and transcoding

This would be a built-in module `import * as encodings from "encodings"`, exporting 4 methods:

- **encodings.makeArrayBuffer(*string* [ , *encoding* ])**: Transcode the list of code units within *string* into a list of bytes encoded using *encoding* (default: **`"utf-8"`**), and return a new array buffer with its data set to those bytes.

- **encodings.getString(*arrayBufferView*, *offset* [ , *byteLength* [ , *encoding* ] ])**: Get the list of the first *byteLength* (default: **`this`**.[[ByteLength]] - *offset*, truncated at the end of the array buffer if necessary) bytes within *arrayBufferView*.[[ViewedArrayBuffer]] starting from *offset*, transcode it from *encoding* (default: **`"utf-8"`**) to code units, and return the resulting string.

- **encodings.setString(*arrayBufferView*, *offset*, *string* [ , *encoding* ])**: Transcode the list of code units within *string* into a list of bytes encoded using *encoding* (default: **`"utf-8"`**), and write the first *byteLength* (default: length of *bytes*) bytes to *arrayBufferView*.[[ViewedArrayBuffer]], starting from *offset* (default: 0) and stopping after it either hits the end of the array buffer or runs out of code units.

- **encodings.transcode(*arrayBuffer*, *sourceEncoding*, *targetEncoding*)**: Get the list of bytes within *arrayBuffer*.[[ArrayBufferData]], transcode it from *sourceEncoding* to code units and then from code units into *targetEncoding*, and then return a new array buffer with its data set to those bytes.

### Required encodings

For all three, each of following encodings are required to be supported in both directions, but others may optionally be supported and not necessarily in both ways (like a theoretical read-only `"auto"` for UTF-8/UTF-16/etc. detection):

- **`"utf-8"`**
- [**`"wtf-8"`**](https://simonsapin.github.io/wtf-8/)
- **`"ascii"`**
- **`"utf-16-be"`**
- **`"utf-16-le"`**
- **`"utf-32-be"`**
- **`"utf-32-le"`**

Transcoding works like this:

- **`"utf-8"`**: From code units, convert from UTF-16 to code points, then to UTF-8, and then to bytes. From bytes, convert from UTF-8 to code points, then from code points to UTF-16, and then to code units.
- **`"wtf-8"`**: Transcode it according to the algorithm [described here](https://simonsapin.github.io/wtf-8/#converting-wtf-8-ill-formed-utf-16).
- **`"ascii"`**: From code points, mask off the top 9 bits from each code unit and emit the low 8 bits of each. To code units, mask to the low 7 bits, zero-extend from 8 to 16 bits, and emit it as code units.
- **`"utf-16-be"`**/**`"utf-16-le"`**: From code units, emit the raw code units in the relevant byte order. To code units, read the code units in the relevant byte order.
- **`"utf-32-be"`**/**`"utf-32-le"`**: From code units, convert from UTF-16 to code points, zero-extend from 21 to 32 bits, and emit the raw 32-bit code points in the relevant byte order, padded with zeroes. To code units, mask to the low 21 bits, convert to UTF-16, and emit the raw code units.

Rationale for each:

- **`"utf-8"`**: encoding for 99% of the web.
- [**`"wtf-8"`**](https://simonsapin.github.io/wtf-8/): Similar to UTF-8, but losslessly encodes UTF-16 and is useful for internal representations that have to cross up UTF-8 and UTF-16/UCS-2.
- **`"ascii"`**: Many legacy formats require this.
- **`"utf-16-be"`**/**`"utf-16-le"`**: JavaScript's native format is UTF-16.
- **`"utf-32-be"`**/**`"utf-32-le"`**: UTF-32 requires no extra processing overhead, which is a good fit for many internal APIs where decoding overhead can be a problem. (This isn't likely in normal JS code, but could come up in WASM interop.)

### Host encodings

In addition, there should exist a set of hooks for unspec'd types, so they can implement other required encodings as well. For example:

- Browsers need the ability to encode and decode [numerous uncommon and/or legacy encodings](https://html.spec.whatwg.org/multipage/parsing.html#character-encodings), like `"iso-8859-4"` (the default document encoding).
- Node supports [several encodings](https://nodejs.org/api/buffer.html#buffer_buffers_and_character_encodings),

The hooks included are:

- HostEncode(*string*, *from*) - This accepts a decoded string and returns a list of appropriately encoded bytes, or returns **`undefined`** if encoding with the encoding isn't supported.
- HostDecode(*bytes*, *to*) - This accepts a list of appropriately encoded bytes and returns a decoded string, or returns **`undefined`** if decoding with the encoding isn't supported.
- HostTranscode(*bytes*, *from*, *to*) - This transcodes a sequence of bytes from encoding *from* to encoding *to*, or returns **`undefined`** if transcoding from the first to second encoding isn't supported.

### Other notes

- Unknown encodings should cause a **`ReferenceError`** to be thrown, even on zero-length array buffers and strings. (This allows for feature detection.)
- The first *byteLength* bytes in *bytes* within **encodings.getString**'s algorithm are not required to consitute a valid byte sequence according to *encoding* - it is delimited and truncated at the byte level, not the code point/character level.
- The transcoded bytes written in **encodings.transcode**'s algorithm may be a different length of bytes than the originally encoded bytes.

### Why?

1. Engines can take advantage of their internal representation to do this extremely cheaply.
    - In particular `dataView.setString(offset, "string", "enc")` could be optimized to just a raw `memcpy` loop.
    - Also, `dataView.getString(offset, "enc")` could be optimized to a raw `memcpy` loop + cheap string wrapper if `"enc"` happens to match one of the implementation's internal encoding (like UTF-16 or ASCII). Engines already do this for static strings.
    - 99% of encoding-related copying and conversion can very easily be SIMD-accelerated.
1. Node.js (via [`Buffer`](https://nodejs.org/api/buffer.html)) and the DOM (via [`TextEncoder`](https://developer.mozilla.org/en-US/docs/Web/API/TextEncoder)) both provide means to do this.
1. Some complex string operations (like parsing) are much more efficient to do without the overhead of language strings.
1. This is useful for passing strings to and from exported methods in WASM and asm.js modules. (Most languages deal with strings in UTF-8.)

# Non-proposals

These are things I've been asked to promote, but there's reasons why I won't include them in this list.

## Array shuffling

The use case isn't general enough to merit being a new global in my opinion. It's a very niche use case that can usually be done in a dedicated library somewhere.

You need a far more complex pseudo-random number generator than `Math.random` - most engines' implementations only have up to 128 bits of state ([V8 uses xorshift128+](https://v8.dev/blog/math-random), for example), which means [they can only correctly shuffle arrays up to size 34](https://en.wikipedia.org/wiki/Fisher%E2%80%93Yates_shuffle#Pseudorandom_generators). This means that if you have an array of 100 items, a shuffle based on `Math.random` can only generate a **very** small fraction of the possibilities that array really has. (You *could* stretch that up to 170 using the [xorshift1024*](https://en.wikipedia.org/wiki/Xorshift#xorshift*) or a few thousand entries with a [Mersenne Twister](https://en.wikipedia.org/wiki/Mersenne_Twister) or related generator, but you'll struggle with larger arrays.) So far, neither [Lodash](https://github.com/lodash/lodash/blob/4.17.10/lodash.js#L6711) nor [Underscore](https://github.com/jashkenas/underscore/blob/master/underscore.js#L397) use an adequate PSRNG to do this, since they defer to `Math.random`.

What really needs to be created is a good userland implementation that accounts for the fact you need enough PSRNG states to cover every permutation an array could have. This basically means you have to use a cryptographically secure random number generator (preferably hardware-assisted) to hope to cover enough states for anything reasonably random. However, this *shouldn't* make it into the standard library because of the very poor cost/beneft ratio it has (technically complex, legally fraught, low payoff for most users).

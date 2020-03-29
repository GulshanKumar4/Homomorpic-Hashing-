# Homomorpic-Hashing
We can construct a very simple candidate for a homomorphic hash by using one very simple mathematical identity: the observation that g^(x0 * gx1 = gx0 + x1. So, for instance, 23 * 22 = 25. We can make use of this by the following procedure:

Pick a random number g
For each element x in our message, take gx. This is the hash of the given element.
Using the identity above, we can see that if we sum several message blocks together, we can compute their hash by multiplying the hashes of the individual blocks, and get the same result as if we 'hash' the sum. Unfortunately, this construction has a couple of obvious issues:

Our 'hash' really isn't - the hashes are way longer than the message elements themselves!
Any attacker can compute the original message block by taking the logarithm of the hash for that block. If we had a real hash with collisions, a similar procedure would let them generate a collision easily.
# A better hash with modular arithmetic
Fortunately, there's a way we can fix both problems in one shot: by using modular arithmetic. Modular arithmetic keeps our numbers bounded, which solves our first problem, while also making our attacker's life more difficult: finding a preimage for one of our hashes now requires solving the discrete log problem, a major unsolved problem in mathematics, and the foundation for several cryptosystems.

Here, unfortunately, is where the theory starts to get a little more complicated - and I start to get a little more vague. Bear with me.

First, we need to pick a modulus for adding blocks together - we'll call it q. For the purposes of this example, let's say we want to add numbers between 0 and 255, so let's pick the smallest prime greater than 255 - which is 257.

We'll also need another modulus under which to perform exponentiation and multiplication. We'll call this p. For reasons relating to Fermat's Little Theorem, this also needs to be a prime, and further, needs to be chosen such that p - 1 is a multiple of q (written q | (p - 1), or equivalently, p % q == 1). For the purposes of this example, we'll choose 1543, which is 257 * 6 + 1.

Using a finite field also puts some constraints on the number, g, that we use for the base of the exponent. Briefly, it has to be 'of order q', meaning that gq mod p must equal 1. For our example, we'll use 47, since 47257 % 1543 == 1.

So now we can reformulate our hash to work like this: To hash a message block, we compute gb mod p - in our example, 47b mod 1543 - where b is the message block. To combine hashes, we simply multiply them mod p, and to combine message blocks, we add them mod q.

Let's try it out. Suppose our message is the sequence [72, 101, 108, 108, 111] - that's "Hello" in ASCII. We can compute the hash of the first number as 4772 mod 1543, which is 883. Following the same procedure for the other elements gives us our list of hashes: [883, 958, 81, 81, 313].

We can now see how the properties of the hash play out. The sum of all the elements of the message is 500, which is 243 mod 257. The hash of 243 is 47243 mod 1543, or 376. And the product of our hashes is 883 * 958 * 81 * 81 * 313 mod 1543 - also 376! Feel free to try this for yourself with other messages and other subsets - they'll always match, as you would expect.

# A practical hash
Of course, our improved hash still has a couple of issues:

The domain of our input values is small enough that an attacker could simply try them all out to find collisions. And the domain of our output values is small enough the attacker could attempt to find discrete logarithms by brute force, too.
Although our hashes are shorter than they were without modular arithmetic, they're still longer than the input.
The first of these is fairly straightforward to resolve: we can simply pick larger primes for p and q. If we choose ones that are sufficiently large, both enumerating all inputs and brute force logarithm finding will become impractical.

The second problem is a little trickier, but not hugely so; we just have to reorganize our message a bit. Instead of breaking the message down into elements between 0 and q, and treating each of those as a block, we can break the message into arrays of elements between 0 and q. For instance, suppose we have a message that is 1024 bytes long. Instead of breaking it down into 1024 blocks of 1 byte each, let's break it down into, say, 64 blocks of 16 bytes. We then modify our hashing scheme a little bit to accommodate this:

Instead of picking a single random number as the base of our exponent, g, we pick 16 of them, g0 - g16. To hash a block, we take each number gi and raise it to the power of the corresponding sub-block. The resulting output is the same length as when we were hashing only a single block per hash, but we're taking 16 elements as input instead of a single one. When adding blocks together, we add all the corresponding sub-blocks individually. All the properties we had earlier still hold. Better, we've given ourselves another tuneable parameter: the number of sub blocks per block. This will be invaluable in getting the right tradeoff between security, granularity of blocks, and protocol overhead.

# Practical applications
What we've arrived at now is pretty much the construction described in the paper, and hopefully you can see how it would be applied to a system utilizing fountain codes. Simply pick two primes of about the right size - the paper recommends 257 bits for q and 1024 bits for p - figure out how big you want each block to be - and hence how many sub-blocks per block - and figure out a way for everyone to agree on the random numbers for g - such as by using a random number generator with a well defined seed value.

The construction we have now, although useful, is still not perfect, and has a couple more issues we should address. First of these is one you may have noticed yourself already: our input values pack neatly into bytes - integers between 0 and 255 in our example - but after summing them in a finite field, the domain has grown, and we can no longer pack them back into the same number of bits. There are two solutions to this: the tidy one and the ugly one.

The tidy one is what you'd expect: Since each value has grown by one bit, chop off the leading bit and transmit it along with the rest of the block. This allows you to transmit your block reasonably sanely and with minimal expansion in size, but is a bit messy to implement and seems - at least to me - inelegant.

The ugly solution is this: Pick the smallest prime number larger than your chosen power of 2 for q, and simply ignore or discard overflows. At first glance this seems like a terrible solution, but consider: the smallest prime larger than 2256 is 2256 + 297. The chance that a random number in that range is larger than 2256 is approximately 1 in 3.9 * 1074, or approximately one in 2247. This is way smaller than the probability of, say, two randomly generated texts having the same SHA-1 hash.

Thus, I think there's a reasonable argument for picking a prime using that method, then simply ignoring the possibility of overflows. Or, if you want to be paranoid, you can check for them, and throw out any encoded blocks that cause overflows - there won't be many of them, to say the least.

# Performance and how to improve it
Another thing you may be wondering about this scheme is just how well it performs. Unfortunately, the short answer is "not well". Using the example parameters in the paper, for each sub-block we're raising a 1024 bit number to the power of a 257 bit number; even on modern hardware this is not fast. We're doing this for every 256 bits of the file, so to hash an entire 1 gigabyte file, for instance, we have to compute over 33 million exponentiations. This is an algorithm that promises to really put the assumption that it's always worth spending CPU to save bandwidth to the test.

The paper offers two solutions to this problem; one for the content creator and one for the distributors.

For the content creator, the authors demonstrate that there is a way to generate the random constants g, used as the bases of the exponents using a secret value. With this secret value, the content creator can generate the hashes for their files much more quickly than without it. However, anyone with the secret value can also trivially generate hash collisions, so in such a scheme, the publisher must be careful not to disclose the value to anyone, and only distribute the computed constants gi. Further, the set of constants themselves aren't small - with the example parameters, a full set of constants weighs in at about the size of 4 data blocks. Thus, you need a good way to distribute the per-publisher constants in addition to the data itself. Anyone interested in this scheme should consult section C of the paper, titled "Per-Publisher Homomorphic Hashing".

For distributors, the authors offer a probabilistic check that works on batches of blocks, described in section D, "Computational Efficiency Improvements". Another easier to understand variant is this: Instead of verifying blocks individually as they arrive, accumulate blocks in a batch. When you have enough blocks, sum them all together, and calculate an expected hash by taking the product of the expected hashes of the individual blocks. Compute the composite block's hash. If it verifies, all the individual blocks are valid! If it doesn't, divide and conquer: split your batch in half and check each, winnowing out valid blocks until you're left with any invalid ones.

The nice thing about either of these procedures is that they allow you to trade off verification work with your vulnerability window. You can even dedicate a certain amount of CPU time to verification, and simply batch up incoming blocks until the current computation finishes, ensuring you're always verifying the last batch as you receive the next.

# Conclusion
Homomorphic Hashing provides a neat solution to the problem of verifying data from untrusted peers when using a fountain coding system, but it's not without its own drawbacks. It's complicated to implement and computationally expensive to compute, and requires careful tuning of the parameters to minimise the volume of the hash data without compromising security. Used correctly in conjunction with fountain codes, however, Homomorphic Hashing could be used to create an impressively fast and efficient content distribution network.

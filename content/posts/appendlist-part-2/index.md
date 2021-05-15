---
title: "AppendList in Rust, part 2: Optimizing the layout"
date: 2019-07-17
tags:
    - rust
series:
    - AppendList in Rust
series_weight: 2
---

At the end of [part 1](../appendlist-part-1), we had just about finished implementing `AppendList`, an array of arrays that let you push new elements on through an immutable reference.

It worked pretty well! There were a bunch of chunks that were all the same size, and it was pretty easy to figure out which index was in each chunk. But there's a downside: somebody needs to pick what chunk size to use. You could imagine this going wrong both ways: too small, and the overhead of keeping track of each chunk is pretty significant. Too big, and the unused space at the end of the last chunk is problematic.

We could just throw up our hands and let the end user pick, but that's not a great experience for them. We'd have to add a chunk size parameter to the `new` function, save that parameter somewhere (with some runtime cost),[^const-generics] and then reference that parameter whenever we need the chunk size.

[^const-generics]: The runtime cost could be lower, except that Rust still hasn't stabilized [const generics](https://github.com/rust-lang/rfcs/issues/2424), which would embed the chunk size information into the type. (Edit: These were stabilized in [Rust 1.51](https://blog.rust-lang.org/2021/03/25/Rust-1.51.0.html))

But as a developer, being forced to choose is annoying. We could pick a reasonable default, but then everybody would just go ahead and use that default and we would be right back where we started.

But there's another option. When you use a `Vec`, you don't generally have to worry about deciding how much memory to allocate. The `Vec` picks some small, reasonable minimum, then just doubles in size each time it needs more memory. This has the nice properties that reallocations are rare[^big-o-vec] and you never waste too much space.[^vec-waste]

[^big-o-vec]: Specifically, storing \\(n\\) elements requires \\(O(\\log n)\\) allocations, letting you amortize the occasional reallocation over the large number of elements and essentially acting like insertions are constant-time, even though they occasionally aren't.

[^vec-waste]: In the worst case (immediately after reallocating) storing \\(n\\) elements wastes \\(O(n)\\) elements worth of space. In the best case, immediately before reallocating, no space is wasted.

What if we took a similar approach and made each chunk twice the size of the last one? We would only need need \\(O(\\log n)\\) chunks rather than \\(O(n)\\), saving lots of allocations and getting better cache behavior as the chunks get bigger. This is the best of both worlds: small lists only waste a small amount of data, but big lists benefit from big chunks.

# What needs to change?

The code from the last post will _almost_ work. All of the translation from indices to chunk IDs is done by three functions:

* `chunk_size`, which determines the size of the chunk based on its ID
* `chunk_start`, which determines the first index contained based on a chunk's ID
* `index_chunk`, which determines the chunk ID that contains a particular index

Everything else can remain exactly the same. We just need to make sure that those few functions work exactly as expected. In particular, we need to make sure that a few things are true for every possible input:

* Each chunk starts where the last one ends
    * `chunk_start(n) + chunk_size(n) == chunk_start(n + 1)`
* Each index is contained within its chunk
    * `chunk_start(index_chunk(i)) <= i` and also
    * `chunk_start(index_chunk(i)) + chunk_size(index_chunk(i)) > i`
* The first chunk starts at index 0
    * `chunk_start(0) == 0`
* All chunks have a strictly positive size[^non-zero]
    * `chunk_size(n) > 0`

[^non-zero]: This is required based on the implementation of `push` from last time. You could imagine an alternate `push` implementation that allows zero-sized chunks, but it doesn't have practical value: a zero-sized chunk incurs some overhead without storing anything. Also, you would need to add a requirement in that case that more chunks eventually result in a longer list. For example, maybe that that the limit of `chunk_start` is unbounded as \\(n \\rightarrow \\infty\\). Of course, using `usize` means that there will never be negatives.

We can make each one into a test. If these tests pass, we can be fairly confident that the whole module works and is safe.[^pedantic] You can check out the tests (they're not particularly interesting) [in the Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=a83b41ba347bb07a6732463240761e44), and we can quickly verify that they pass using the uniform-sized chunk code we implemented last time.

[^pedantic]: I mean, somebody could do something nasty with global state that would make the tests pass but be incorrect, but any three pure functions that satisfy these rules will work. This is where functional programming can really shine.

```
running 5 tests
test test::chunks_start_where_last_ended ... ok
test test::all_chunks_have_positive_size ... ok
test test::first_chunk_starts_at_0 ... ok
test test::thousand_items ... ok
test test::index_in_correct_chunk ... ok
```

# Implementing `chunk_size`

Each chunk is twice the size of the previous one, except that the first chunk is `FIRST_CHUNK_SIZE`. Flipping that around, each chunk will be the first chunk size doubled `chunk_id` times. Because we're dealing with unsigned integers, a bitwise left shift by 1 (`<< 1` in Rust) doubles a value once, a shift by 2 doubles it twice, and a shift by \\(n\\) doubles it \\(n\\) times. Essentially, we're exploiting the fact that calculating powers of 2 in binary is as easy as calculating powers of 10 in decimal: just shift everything a few places! This makes `chunk_size` very simple:

```rust
fn chunk_size(chunk_id: usize) -> usize {
    FIRST_CHUNK_SIZE << chunk_id;
}
```

[Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=f0d7d209eb672c80b3d98467820cc668)

Let's try it...

```rust
test test::chunks_start_where_last_ended ... FAILED
```

Of course, it failed because `chunk_size` and `chunk_start` no longer match up.

# Implementing `chunk_start`

Figuring out where a chunk starts is equivalent to figuring out the total number of items in all preceding chunks. We could loop through the list of chunks, adding up each one's size to get the starting position of the next one. But is there a faster way?

Lets look at some example values when the `FIRST_CHUNK_SIZE` is 16.

Chunk ID | Size | Start
---------|------|------
0        | 16   | 0
1        | 32   | 16
2        | 64   | 48 (16 + 32)
3        | 128  | 112 (16 + 32 + 64)

A pattern! Each chunk starts at its own length minus the length of the first chunk. It turns out that that is holds for any `FIRST_CHUNK_SIZE` as long as the chunk sizes are doubling. Check out this footnote for a not-particularly-satisfying explanation.[^unsatisfying]

[^unsatisfying]: Because each chunk doubles the last one, it has just about the same size as all of the previous chunks added together. Imagine lining up each previous chunk next to the current chunk. For example, looking at chunk 3: it's 128 items long. Chunk 2 takes up takes up half of it with 64 items, chunk 1 takes up half of what remains with 32, and chunk 0 takes up half of whats left with 16. But there are 16 items unaccounted for. In a mathematical series, we would keep going: chunk -1 takes up half the slack with 8, then chunk -2, -3, -4, and so on, eventually finding that the infinite series converges to take up the full size. But we don't actually have these "negative" chunks! Instead, we just subtract the total size of all of those "negative" chunks from the size to get the starting position, and the negative chunks' total size is exactly the same as the `FIRST_CHUNK_SIZE`. This is kinda strange and not at all what I expected, so if you have a more intuitive explanation, I'd love to hear about it.

Happily, the actual code is super easy to write.

```rust
fn chunk_start(chunk_id: usize) -> usize {
    chunk_size(chunk_id) - FIRST_CHUNK_SIZE;
}
```

```
test test::chunks_start_where_last_ended ... ok
test test::index_in_correct_chunk ... FAILED
```

[Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=fece3065862917d11369df5bbf3a83e2)

Better! But still not there yet. Chunk starts and sizes line up with each other, but not indices.

# Implementing `index_chunk`

This is definitely the hardest one.

The chunk starting positions are an exponential function. We want the inverse of that function, which means we'll have to deal with some logarithms.

The functions we wrote for chunk size that we used before can be expressed as mathematical formulas. If the first chunk has size \\(C\\) and we care about chunk \\(n\\), the chunk has size \\(C \times 2^n\\) and it begins at index \\(C \times 2^n - C\\).

If we pick an index \\(i\\), we can figure out which chunk it should go in by setting \\(i\\) equal to the starting position of the chunk and solving for the chunk number. In general, the result we get is not an integer, but we know the chunk number is always an integer. We will have to round down because any index occuring partway through a chunk has the same chunk number as the beginning of the chunk.

The algebra isn't actually __that__ bad. We want to find the chunk \\(n\\) corresponding to index \\(i\\), so we set \\(i\\) equal to the chunk
start formula.

<span>
$$ i = C \times 2^n - C $$
</span>

Now factor out \\(C\\) and solve for \\(2^n\\).

<span>
$$ i = C \times (2^n - 1) $$
</span>

<span>
$$ \frac{i}{C} = 2^n - 1 $$
</span>

<span>
$$ \frac{i}{C} + 1 = 2^n $$
</span>

<span>
$$ \frac{i + C}{C} = 2^n $$
</span>

And then just take \\(\log_2\\) of both sides and simplify.

<span>
$$ \log_2{\frac{i + C}{C}} = \log_2{2^n} $$
</span>

<span>
$$ \log_2{(i + C)} - \log_2{C} = n $$
</span>

For any index that isn't the first index of a chunk, this formula will give you a fractional value for \\(n\\), the chunk ID. Like we said, fractional value should be rounded down, so the actual equation is the floor (rounded down value, \\(\lfloor x \rfloor\\)) of what we got before.

<span>
$$ \lfloor \log_2{(i + C)} - \log_2{C} \rfloor = n $$
</span>

Now, if we can guarantee that \\(C\\) is a power of 2 (and we can, we can set the first chunk size to any value), we break this down even further. This is allowed because if \\(C\\) is a power of 2, \\(\log_2{C}\\) will always be an integer, so it no longer matters if it gets rounded or not.

<span>
$$ \lfloor \log_2{(i + C)} \rfloor - \log_2{C} = n $$
</span>

After all that, it's as simple as:

```rust
fn index_chunk(index: usize) -> usize {
    floor_log2(index + FIRST_CHUNK_SIZE) - floor_log2(FIRST_CHUNK_SIZE)
}
```

Unfortunately, there's no direct `floor_log2` implementation on `usize`, but the standard library does have [`usize::leading_zeros()`](https://doc.rust-lang.org/std/primitive.usize.html), which gives the number of leading zeroes in the binary representation of a `usize`. This is actually pretty close to what we're looking for: a nonzero number \\(x\\) needs \\( \lfloor 1 + \log_2 x \rfloor \\) bits to be represented in binary, and the rest of the digits will be zeroes. So all we need to do is get the number of bits in the whole representation, then subtract the number of leading zero bits (and that extra 1).

```rust
#[inline]
const fn floor_log2(x: usize) -> usize {
    const BITS_PER_BYTE: usize = 8;

    BITS_PER_BYTE * std::mem::size_of::<usize>() - (x.leading_zeros() as usize) - 1
}
```

This one gets marked as `#[inline]` and `const` to help the compiler out in debug builds. `BITS_PER_BYTE` feels like it should be defined somewhere in `core` or `std`, but it somehow isn't. (Edit: [`usize::BITS`](https://doc.rust-lang.org/std/primitive.usize.html#associatedconstant.BITS) is now an experimental API.)

# And that's everything!

If you put all the code from the last two posts together, you end up essentially with the [appendlist](https://crates.io/crates/appendlist) crate.

The last question: is this data structure useful for anything? It's fairly niche. In general, you'd do better to store all of your objects in a `Vec` and keep track of the index to each one, just eating the resizing costs. Indexing into linear memory is cheap, so there's not much benefit to holding a reference over an index. If resizing is expensive, that might be a use case for a specialized linked list. If you can come up with a use for this data structure, let me know, I'd love to hear about it!

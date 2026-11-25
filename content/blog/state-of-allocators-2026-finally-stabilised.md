+++

title = "The State of Allocators in 2026 - Finally Stabilised!"
description = "Now that custom allocators are stabilised, what can we do with them?"
date = 2026-11-24

[taxonomies]
tags = ["rust"]
+++

It's only been a couple of months since [the last article](/blog/state-of-allocators-2026-part-2), and I am pleased to see that, after 10 years of limbo, allocators have finally been stabilised! While it _feels_ like the never type did steal a bit of thunder, rust `1.100` was nonetheless a [**monumental** release](https://blog.rust-lang.org/2026/11/12/Rust-1.100.0/). Congratulations on the team that shepherded this through.

So, now we have an MVP of the allocator API, what can we do with it? What does it get us besides closer feature parity with other system languages like C++ & Zig?

In this article I'll step through a few use cases that might be handy to the layman rust programmer, so hopefully you can see the benefit.

## Allocators can adjust when memory is freed

This was one of the leading cases for custom allocation. Dropping in rust is not free. There has been prior art in actually moving [memory to a dedicated thread to `Drop`](https://abrams.cc/rust-dropping-things-in-another-thread/) highly nested structures. This is actually only half a solution. You're still paying for the deallocation call costs, just it's not in the hot path anymore.

Instead, what if we knew that we would have a period of time in our code, where we're going to allocate a lot of stuff, do something with it, and then free everything? A good example of this is a web request, or a database query.

A request comes in, fills in some data structures, and then when the request finishes, the memory isn't needed anymore so it's dropped. This sort of "burst" pattern of allocation, where you allocate a lot up front, and then tear things down all at once at the end, is quite common.

For this we can use a bump allocator: reserve a big block of memory, and allocate from that, rather than paying the cost of talking to the system allocator for every allocation. Think of it like the [`BufReader`](https://doc.rust-lang.org/std/io/struct.BufReader.html) and [`BufWriter`](https://doc.rust-lang.org/std/io/struct.BufWriter.html) of std io, rather than make repeated syscalls to fetch data, we can _buffer_ that data in a big block, so that we can reduce the overhead considerably.

You want more memory? Just check if the size fits in our block, return that slice, increment our counter (bump!), and we're done. We can add more blocks as well, if we run out. This makes allocations very cheap, we don't need to keep much state around besides from a counter.

The downsides of this approach is when we come to _deallocate_ memory. We don't keep track around where individual allocations are made. However, this actually solves our _drop takes a long time_ problem, since we can just deallocate everything all at once.

Essentially if the impl of the bump allocator is:

```rust
const SIZE: usize = 32 * 1024 * 1024;
const LAYOUT: Layout = Layout::new::<[u8; SIZE]>();

struct Bump {
    // one big 32MiB block from the global allocator
    data: NonNull<u8>,
    // how much of it we've handed out
    used: Cell<usize>,
}

impl Bump {
    fn new() -> Self {
        let data = Global.allocate(LAYOUT).expect("out of memory").cast::<u8>();
        Bump { data, used: Cell::new(0) }
    }
}

impl Drop for Bump {
    fn drop(&mut self) {
        // hand the whole block back in one go
        unsafe { Global.deallocate(self.data, LAYOUT) }
    }
}

unsafe impl Allocator for &Bump {
    fn allocate(&self, layout: Layout) -> Result<NonNull<[u8]>, AllocError> {
        // where we are in memory
        let data = self.data.as_ptr();

        // how much we've used of `data` already
        let used = self.used.get();

        // given the requested alignment, how many bytes do we need past `data` + `used`
        let alignment_padding = data.wrapping_add(used).align_offset(layout.align());

        let start = used + alignment_padding;
        let end = start + layout.size();

        if end <= SIZE {
            // bump!
            self.used.set(end);
            let ptr = unsafe { NonNull::new_unchecked(data.add(start)) };
            return Ok(NonNull::slice_from_raw_parts(ptr, layout.size()));
        }

        // an actual bump allocator would grab another block here
        Err(AllocError)
    }

    unsafe fn deallocate(&self, _ptr: NonNull<u8>, _layout: Layout) {
        // deliberately a no-op!
    }
}
```

Then we can use it like so:

```rust
struct Node<'a> {
    value: u64,
    parent: Option<&'a Node<'a>>,
}

// our bump allocator
let bump = Bump::new();

// leak here gives us a ref tied to the arena, the memory still goes when we drop our allocator
let mut leaf: &Node = Box::leak(Box::new_in(Node { value: 0, parent: None }, &bump));

for i in 1..1_000_000 {
    leaf = Box::leak(Box::new_in(Node { value: i, parent: Some(leaf) }, &bump));
}

// this takes virtually no time at all!
drop(bump);
```

There are a few tradeoffs for doing things like this in a bump allocator, and not _every_ allocation is going to benefit. Hence why it's important to be able to adjust our allocation strategy locally rather than just globally. However, an astute reader might notice that, if we are bumping large swathes of memory, this is going to possibly cause our memory usage to increase considerably. Well, that brings us to our next section: memory accounting.

## Allocators can let us know how much memory they're using

The next challenge that can be solved by custom allocators is knowing _how much_ memory particular sections of code is being used. I.e, memory accounting. The key here is that, we can use existing allocators, and sprinkle in some extra bookkeeping to keep a pretty accurate ledger.

There are a number of use cases that accounting can shape the behaviour of code. Say you have a cache of objects, and you want to set a threshold above which, we start evicting entries. Or say you want to do an external sort, but you don't know how big your data structures are at the start, but you want to use up as _much_ memory as possible. Or maybe you are wanting to place a limit on a code sandbox, because it's sharing resources with other sandboxes.

What you can do is wrap an allocator, with a custom accounting layer, and use this to drive your decisions. I.e, here is a very trivial example:

```rust
struct AccountedAllocator<A: Allocator> {
    inner: A,
    allocated: AtomicUsize,
}

impl<A: Allocator> AccountedAllocator<A> {
    fn new(inner: A) -> Self {
        Self { inner, allocated: AtomicUsize::new(0) }
    }

    fn allocated(&self) -> usize {
        self.allocated.load(Ordering::Relaxed)
    }
}

unsafe impl<A: Allocator> Allocator for &AccountedAllocator<A> {
    fn allocate(&self, layout: Layout) -> Result<NonNull<[u8]>, AllocError> {
        // pass the allocation through, and count it if it succeeded
        let ptr = self.inner.allocate(layout)?;
        self.allocated.fetch_add(layout.size(), Ordering::Relaxed);
        Ok(ptr)
    }

    unsafe fn deallocate(&self, ptr: NonNull<u8>, layout: Layout) {
        unsafe { self.inner.deallocate(ptr, layout) };
        self.allocated.fetch_sub(layout.size(), Ordering::Relaxed);
    }
}
```

Then we can wrap the global allocator, and see how much our cache is using:

```rust
let accounted = AccountedAllocator::new(Global);

let mut cache: Vec<u8, _> = Vec::new_in(&accounted);

cache.extend_from_slice(&[0; 1024]);
// cache is using 1024 bytes
println!("cache is using {} bytes", accounted.allocated());

cache.extend_from_slice(&[0; 4096]);
// cache is using 5120 bytes
println!("cache is using {} bytes", accounted.allocated());

drop(cache);
// cache is using 0 bytes
println!("cache is using {} bytes", accounted.allocated());
```

### Shameless Plug: the Thresher crate

Now, a single atomic, while is great for an example, does have some perf overhead. So, one of the crates that I have created, while keeping in mind performance, is [`thresher`](https://github.com/cetra3/thresher): an accounting allocator with a callback on threshold. It uses a combined "local" task/thread and "global" counter to avoid atomic contention, feeding back in every 64KiB of allocation.

[`thresher`](https://github.com/cetra3/thresher) gets its name from the _cutest_ shark: the [Thresher Shark](https://en.wikipedia.org/wiki/Thresher_shark).

Now, what you can do with [`thresher`](https://github.com/cetra3/thresher) is, use it to trigger callbacks when allocations are hit:

```rust
let thresher = Thresher::new();
thresher.set_callback_threshold(100 * 1024 * 1024);
thresher.set_callback(|allocation| {
    println!("Threshold reached! Allocated: {} bytes", allocation);
});

// This is what we're adding to our vec each loop
let bytes = vec![0u8; 10 * 1024 * 1024];

// The vec allocates through the handle, so its buffer counts towards the
// threshold. `bytes` above uses the global allocator and is not counted
let mut vec = Vec::<u8, _>::new_in(thresher.handle());

for i in 0..10 {
    println!("Loop {i}, Vec Capacity: {}", vec.capacity());
    vec.extend(&bytes);
}
```

This prints out what you'd expect:

```
Loop 0, Vec Capacity: 0
Loop 1, Vec Capacity: 10485760
Loop 2, Vec Capacity: 20971520
Loop 3, Vec Capacity: 41943040
Loop 4, Vec Capacity: 41943040
Loop 5, Vec Capacity: 83886080
Loop 6, Vec Capacity: 83886080
Loop 7, Vec Capacity: 83886080
Loop 8, Vec Capacity: 83886080
Threshold reached! Allocated: 167772160 bytes
Loop 9, Vec Capacity: 167772160
```

One production use case, is that we're using [`thresher`](https://github.com/cetra3/thresher) on [logfire](https://pydantic.dev/logfire) to provide us heap profiles of our rust services when memory hits a configured threshold. I did a deep dive into heap profiles and why they are extremely useful in an older blog: [a tale of two lengths](/blog/a-tale-of-two-lengths/). In thresher you can find a [jemalloc example](https://github.com/cetra3/thresher/blob/main/examples/jemalloc.rs) where you can instrument this yourself!

OK so we can control memory when it is dropped, we can use memory accounting to know how much memory we're using, what's left? How about where it goes?

## Allocators can control where memory lives

This is probably the more esoteric of the 3 examples I wanted to give. There are some situations where you actually need to control where the memory lives, and sometimes by doing some allocation tricks you can get better performance out.

### Allocating on the stack

One example would be what [`smallvec`](https://github.com/smallvec/smallvec) and friends do. Instead of going straight to the heap to allocate things, they have a small amount that lives on the stack, and _only_ if allocations go above a certain size, then do they write to the heap.

While `smallvec` is more battle tested, here is some example code about how you could do this with the allocator API:

```rust
const SIZE: usize = 8192;

struct SmallStack {
    // store the first 8KiB on the stack
    data: UnsafeCell<[MaybeUninit<u8>; SIZE]>,
    // how much we've already allocated on our stack
    used: Cell<usize>,
}

impl SmallStack {
    fn new() -> Self {
        Self {
            data: UnsafeCell::new([MaybeUninit::uninit(); SIZE]),
            used: Cell::new(0),
        }
    }
}

unsafe impl Allocator for &SmallStack {
    fn allocate(&self, layout: Layout) -> Result<NonNull<[u8]>, AllocError> {
        // where we are in memory
        let data = self.data.get().cast::<u8>();

        // how much we've used of `data` already
        let used = self.used.get();

        // given the requested alignment, how many bytes do we need past `data` + `used`
        let alignment_padding = data.wrapping_add(used).align_offset(layout.align());

        let start = used + alignment_padding;
        let end = start + layout.size();

        // if we have enough on our local stack:
        if end <= SIZE {
            self.used.set(end);
            let ptr = unsafe { NonNull::new_unchecked(data.add(start)) };
            Ok(NonNull::slice_from_raw_parts(ptr, layout.size()))
        } else {
            // otherwise allocate on the standard heap
            Global.allocate(layout)
        }
    }

    unsafe fn deallocate(&self, ptr: NonNull<u8>, layout: Layout) {
        let data = self.data.get().addr();

        // if this isn't an address in our data
        if !(data..data + SIZE).contains(&ptr.as_ptr().addr()) {
            unsafe { Global.deallocate(ptr, layout) };
        }
    }
}
```

Then we can use our toy stack first allocator:

```rust
// needs to live longer than our vec
let stack = SmallStack::new();

let mut vec: Vec<u32, _> = Vec::new_in(&stack);

// fits in our stack data buffer
for i in 0..32 {
    vec.push(i);
}

// will eventually spill to the heap
for i in 32..10_000 {
    vec.push(i);
}
```

Et voila! We have smaller allocations on the stack and then push to the heap as needed. This can give some good performance boosts.

### Allocating with stronger alignment

Normally when you align memory allocations, this is based upon the thing that is being allocated for. I.e, is it a `u64`? align to 8 bytes or a `u32`? align to 4. If it's not correctly aligned, this can cause undefined behaviour and all sorts. You notice in both the bump and small stack example I had to do some shenanigans to find the next appropriately aligned address. However, there is _some_ scenarios where the alignment of memory is external to what is being allocated.

In linux, when you read anything from disk via standard syscalls, you are _usually_ reading from the kernel's page cache. I.e, the spare memory the kernel has around to make things transparently fast. Most of the time that is fine, but there are some times when you want to bypass it.

I.e, there is a [**long** standing bug in kubernetes](https://github.com/kubernetes/kubernetes/issues/43916) that can cause pod eviction if you are using the page cache. Or: if you are doing your own caching layer, sometimes it doesn't make sense to double up on memory devoted to caching files.

So, there is a bailout provided, in the guise of `O_DIRECT` that allows you to _directly_ access files from the file system. However: one of the requirements of using this flag is that you are reading or writing into memory that is aligned to the device's logical block size. This is _usually_ a multiple of 512, but can be less or more depending on the hardware.

The implementation of a block aligned allocator is pretty simple, here's one where we just hardset it to `4096`:

```rust
#[derive(Clone, Copy)]
struct BlockAligned;

// hard set here, but we could make this derived from the device itself!
const BLOCK_SIZE: usize = 4096;

unsafe impl Allocator for BlockAligned {
    fn allocate(&self, layout: Layout) -> Result<NonNull<[u8]>, AllocError> {
        let layout = layout.align_to(BLOCK_SIZE).map_err(|_| AllocError)?;
        Global.allocate(layout)
    }

    unsafe fn deallocate(&self, ptr: NonNull<u8>, layout: Layout) {
        let layout = layout.align_to(BLOCK_SIZE).expect("layout must come from BlockAligned");
        unsafe { Global.deallocate(ptr, layout) }
    }
}
```

With this, we can now write simple code to read directly using `O_DIRECT`:

```rust
let mut buf: Vec<u8, BlockAligned> = Vec::new_in(BlockAligned);
buf.resize(BLOCK_SIZE, 0);

let mut file = OpenOptions::new()
    .read(true)
    .custom_flags(libc::O_DIRECT)
    .open(path)?;

file.read_exact(&mut buf)?;
```

Here, the `read_exact` call _can_ fail on some devices if we're not using block size aligned allocations.

Like with the stack first allocation, there are libraries that allow you do this already, such as the [`aligned-vec`](https://github.com/sarah-quinones/aligned-vec) crate, with their own data types to work nicely.

But here's the rub: say you had an application where you wanted to blend and mix in different allocation strategies: some vecs you want to be block aligned, some you want allocated on the stack, but you want to be able to use shared methods for both. Do we _want_ to use `smallvec` and `aligned-vec` in order to deal with a data structure that is essentially the same thing, just _allocated_ differently?

std allocators allows you to be _generic_ over the collections, regardless of allocator used:

```rust
fn do_something<A: Allocator>(my_vec: &mut Vec<u32, A>) {}
```

And I think that's pretty powerful.

## Conclusion and wrapping up

It's going to take a bit of time for the ecosystem to catch up to custom allocators. A lot of popular crates have an MSRV policy that precludes them from using even new standard features, and some existing crates, which I would think are a shoe in for the new allocator API, as of writing do not yet have support.

The libs team isn't done yet. This is the MVP of allocators, and, like I mentiond in the [the last article](/blog/state-of-allocators-2026-part-2), this is the _beginning_ of custom allocator support. There is still work to go to support other collection types like maps etc.. so much to be done. I think that what we have is useful enough to start using it now, and also to start adapting and allowing library consumers the ability to provide their own allocators with their API.

I'm excited to see what people build, and what other use cases are unlocked with it!

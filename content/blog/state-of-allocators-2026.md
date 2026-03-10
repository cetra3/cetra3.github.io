+++

title = "The State of Allocators in 2026"
description = "A look at the unstable allocator trait and where we currently stand"
date = 2026-03-11

[taxonomies]
tags = ["rust"]
+++

The [Allocators RFC](https://rust-lang.github.io/rfcs/1398-kinds-of-allocators.html) is about to reach a **decade** milestone of it [being accepted](https://github.com/rust-lang/rfcs/pull/1398#issuecomment-207603823), but with no stabilisation.

With the [2025 State of Rust Survey](https://blog.rust-lang.org/2026/03/02/2025-State-Of-Rust-Survey-results/#challenges-and-wishes-about-rust) indicating **~12%** would be unblocked by stabilisation and another **~27%** would have improved code, it still seems to be something that a significant portion of rust users want.

I figured it was time to take a lay of the land and work out what the current state of allocators is, where the blockers are, and any outstanding but critical issues remain. I'm hoping this will spur some activity and breathe new life into one of the older RFCs

## The Current State

The allocator trait, [while implemented for nightly](https://github.com/rust-lang/rust/blob/b41f22de2a13a0babd28771e96feef4c309f54aa/library/core/src/alloc/mod.rs#L106-L369), is still unstable. It has been defined as a trait for a good 6 years or more, without many changes to the core methods of the trait.

Currently, the trait implementation contains two simple required methods:

```rust
pub unsafe trait Allocator {
    fn allocate(&self, layout: Layout) -> Result<NonNull<[u8]>, AllocError>;
    unsafe fn deallocate(&self, ptr: NonNull<u8>, layout: Layout);
    // ...other provided methods...
}
```

Implementing this trait, you can then use this with std container types such as `Box` or `Vec` to return values allocated by your custom allocator:

```rust
let mut vec: Vec<i32, MyAllocator> = Vec::new_in(MyAllocator);
```

The `Vec` container now has two generic types, `Vec<T, A>`, the type `T` it's a container for, and the `A` allocator itself (In fact, if you dig deep enough, you can see that `Vec<T>` is just a special cased version with a global allocator, i.e, `Vec<T, A = Global>`).

You need to keep around the allocator in the `Vec` because whenever you want to resize, truncate or drop the memory, you need to use the same allocator to do so, otherwise it's almost impossible not to hit Undefined Behaviour, if you are fiddling around with another allocator's memory.

In practice this field with a Zero Sized Type (ZST) like `Global` this does get inline compiled away, and vec is the standard `ptr`, `len`, `capacity`, but for other allocators that need state, this means increasing the size of `Vec` to include an allocator field.

The same with `Box<T>` as well, this is a `Box<T, A>` with `Box<T, A = Global>` by default. Although `Box` is much simpler in terms of memory allocations/chagnes, it's only really ever allocated when created, no resizes, truncations etc... but you still need to keep the allocator around to deallocate.

So, for such a simple trait definition, it seems like consensus should be easy right? Just two methods and there is already most of the grunt work done on nightly to support it.

However: even _this_ simple of a definition still has some outstanding questions unanswered.

## Current Blockers

As of writing this article, the rust allocator working group repo has [75 issues open](https://github.com/rust-lang/wg-allocators/issues), with the roadmap not touched in a long time, and probably due for _some_ cleanup.

Thom Chiovoloni [wrote a few years ago](https://shift.click/blog/allocator-trait-talk/) about the allocator trait and some of the blockers that exist, and even now a lot of the original referred issues have not been closed. There has been [many](https://old.reddit.com/r/rust/comments/1prgyc8/stackallocator_a_project_for_a_bright_future_with/) [discussions](https://old.reddit.com/r/rust/comments/1op6g64/whats_the_statusblocker_for_allocator_api_to_be/) on reddit and [other places](https://rust-lang.zulipchat.com/#narrow/channel/197181-t-libs.2Fwg-allocators/topic/State.20of.20Allocators/with/577423708) since.

Now in early 2026, it seems like consensus still hasn't been reached. While a number of those existing issues stand, here's my opinion on what some of the main ones are. The list isn't exhaustive, but I think covers some of the "bigger" style problems of the trait so far.

To be clear: I'm not strongly advocating the inclusion of any of these changes, my hope is to stimulate a bit of discussion.

### Zero Sized Allocations

Currently, the trait allows the `Layout` to be defined as zero sized, which back in 2020 was known to [cause issues with zero sized types](https://github.com/rust-lang/wg-allocators/issues/82). What happens is that the allocator contract, allows the same pointer for zero sized allocations, since we aren't actually "allocating" anything. Then this might mean that returning the same pointer for different things could be considered Undefined Behaviour (although I'm not sure it strictly is). Not only that: this breaks pointer equality as well.

This means you need to special case this in all of your allocator implementations, i.e,

```rust
impl Allocator for MyAllocator {
    fn allocate(&self, layout: Layout) -> Result<NonNull<[u8]>, AllocError> {
        if layout.size() == 0 {
            return Ok(NonNull::dangling());
        }
        // do actual allocations here
    }

    unsafe fn deallocate(&self, ptr: NonNull<u8>, layout: Layout) {
        if layout.size() == 0 {
            return;
        }
        // do actual deallocations here
    }
}
```

And so _every_ implementation of allocator will need code similar to this example, or at least consider zero sized allocations, or wastefully allocate space just to uphold the contract. So does it make sense to keep `Layout` as is?

One possible fix is to introduce a `NonZeroLayout` type which enforces that allocations must actually have a size. This foregoes some of the footguns around zero sized allocations, but it would introduce complexity for any trait consumers.

This is a small change to the trait interface:

```rust
/// Our new non zero layout (other constraints left out for brevity)
pub struct NonZeroLayout {
    size: NonZeroUsize,
    align: NonZeroUsize,
}

pub unsafe trait Allocator {
    fn allocate(&self, layout: NonZeroLayout) -> Result<NonNull<[u8]>, AllocError>;
    unsafe fn deallocate(&self, ptr: NonNull<u8>, layout: NonZeroLayout);
}
```

**Sidenote:** If we look at `malloc` in `C` how it behaves, it's allowed to [return a null pointer](https://en.cppreference.com/w/c/memory/malloc) (among other things). However, rust's `allocate` return is always `NonNull` and so I think that enforcing the `NonZeroLayout` balances out these argument types nicely.

### Context and Rust for Linux

In order to unblock things, and probably because it's a _different_ use case (panics are more severe in the kernel), the rust for linux team has pressed on ahead without requiring allocators to be stable. It's worth studying the shape they came up with, because I feel as if the trait _was_ stabilised as is, it still wouldn't be sufficient for their use case.

Their [allocator trait](https://rust.docs.kernel.org/kernel/alloc/trait.Allocator.html), as it stands is slightly similar to the existing trait, but introduces a few extra args:

```rust
pub unsafe trait Allocator {
    const MIN_ALIGN: usize;

    // Required method
    unsafe fn realloc(
        ptr: Option<NonNull<u8>>,
        layout: Layout,
        old_layout: Layout,
        flags: Flags,
        nid: NumaNode,
    ) -> Result<NonNull<[u8]>, AllocError>;

    // Provided methods
    fn alloc(
        layout: Layout,
        flags: Flags,
        nid: NumaNode,
    ) -> Result<NonNull<[u8]>, AllocError> { ... }

    unsafe fn free(ptr: NonNull<u8>, layout: Layout) { ... }
}
```

Essentially the `realloc` does the heavy lifting of `alloc` and `free` as well since they are provided, but it's otherwise similar to the existing trait.

The interesting takeaway here is the `Flags` struct and `NumaNode`: both imply that not all allocation requests are equal. In fact in the kernel there are different scenarios for [what should happen with memory allocations](https://rust.docs.kernel.org/kernel/alloc/flags/index.html).

This means that, there should be some way to provide _context_ for a given allocation. This is discussed in [wg #138](https://github.com/rust-lang/wg-allocators/issues/138) and a bit of a sketch of a trait is included:

```rust
pub unsafe trait Allocator {
    type Ctx: Copy;

    fn allocate(&self, ctx: Self::Ctx, layout: Layout) -> Result<NonNull<[u8]>, AllocError>;
    unsafe fn deallocate(&self, ptr: NonNull<u8>, layout: Layout);
}
```

While I think the ship has already sailed on rust kernel allocators, it does feel context is a strong consideration for inclusion in the base trait.

### Splitting the Traits

One of the goals of the allocator trait is to handle more "esoteric" style allocations. While we want to handle standard system style alloctors like `glibc`, `jemalloc` and `mimalloc`, we _also_ want to support other styles like garbage collected allocators and bump allocators.

Taking the bump allocator as an example, they deallocate all memory at once, rather than individually. So essentially a deallocation for an individual value is a no-op.

But as mentioned earlier, a `Box` keeps around two generic arguments, `Box<T, A>`. While this is fine for ZST allocators like `Global`, compiling down to a single pointer, if you keep around a field for the allocator, which is never actually going to be used to deallocate, then you are increasing the size of `Box` for no reason.

One solution discussed in the working group is to [split the traits into two](https://github.com/rust-lang/wg-allocators/issues/112): for allocation and deallocation.

This looks something like this:

```rust
pub unsafe trait Deallocator {
    unsafe fn deallocate(&self, ptr: NonNull<u8>, layout: NonZeroLayout);
}

pub unsafe trait Allocator: Deallocator {
    fn allocate(&self, layout: Layout) -> Result<NonNull<[u8]>, AllocError>;
}
```

And assuming that you have a way of converting allocators to other deallocators, then, using the example from zakarumych in the issue, you can do something like:

```rust
// used to keep the lifetime around
struct BumpDealloc<'a> {
  _marker: PhantomData<&'a Bump>,
}

unsafe impl<'a> Deallocator for BumpDealloc<'a> {
  fn deallocate(&self, _ptr: NonNull<u8>, _layout: Layout) {}
}

impl<'a> From<&'a Bump> for BumpDealloc<'a> {
  fn from(_bump: &'a Bump) -> Self {
    BumpDealloc { _marker: PhantomData }
  }
}

fn foo<'a>(bump: &'a Bump) {
  // currently includes `&'a Bump` in the `Box`, making it bigger than a pointer
  let box: Box<u32, &'a Bump> = Box::new_in(42, bump);
  // now it's pointer sized
  let box: Box<u32, Bumped<'a>> = box.into();
  assert_eq!(size_of_val(&box), size_of::<usize>());
}
```

There are suggestions that this could work without the split of the trait. For example:

```rust
struct BumpDealloc<'a>(PhantomData<&'a Bump>);

unsafe impl Allocator for BumpDealloc<'_> {
    fn allocate(&self, _: Layout) -> Result<NonNull<[u8]>, AllocError> {
        panic!("cannot allocate through BumpDealloc")
    }
    unsafe fn deallocate(&self, _: NonNull<u8>, _: Layout) {}
}
```

However, having this as a _runtime_ panic feels like it goes against the spirit of rust. So it's a trade off of keeping the trait simple, or allowing this sort of expressivity.

I have flipped back and forth on this one whether it's worthwhile to split the traits. The only strong case in the issue is the no-op deallocator, or an deallocator that takes up _less_ space than the allocator. So essentially you need an allocator always anyway, and the deallocator is a special pairing.

If there were _other_ use cases that would benefit, then maybe it is worth considering. I think that as long as people are not _prevented_ from doing what they want, then we should keep the trait simple.

### Associated Error Type

Allocations can fail, and the existing trait returns a `Result` with `AllocError` as the error type. This is a ZST error which doesn't include a whole lot of information about why the allocation failed, just that it did. It would be nice if an allocation fails, some more contextual information is included, so that the callee can recover from any issues.

This is discussed in [wg #23](https://github.com/rust-lang/wg-allocators/issues/23) and [wg #106](https://github.com/rust-lang/wg-allocators/issues/106) with some use cases around why it would be useful to provide custom error types.

With associated errors, the trait would look like:

```rust
pub unsafe trait Allocator {
    type Error: core::error::Error;

    fn allocate(&self, layout: Layout) -> Result<NonNull<[u8]>, Self::Error>;
    unsafe fn deallocate(&self, ptr: NonNull<u8>, layout: Layout);
    // ...other provided methods...
}
```

There is obviously alternatives to this. For instance, if context was a thing, you could provide an allocation number on allocations, so you could get back the error out of band. Or you could lookup a last error in thread*local etc.. but obviously this is prone to errors if another allocation happens in between. But all of these feel a little less \_nice* than using the existing `Result` type for what it's designed for.

However, one issue around associated types is around making it easily dyn compatible. If you had an associated error type, you would need to ensure you are using `dyn Allocator<Error = AllocError>` everywhere rather than `dyn Allocator`. I don't _think_ this is a big deal, and the std lib could provide some helper plumbing to make it seamless, but nothing that would prevent dyn compatible allocators.

### The Store API Alternative

There exists an alternative to the allocator trait, called the [Store API](https://github.com/rust-lang/rfcs/pull/3446) which has seen some momentum behind it. This would unlock a few more use cases above & beyond the existing trait, especially around putting things on the stack.

But (and this is just my personal hot take which is not worth much!) I think it's over-engineered and suffers from the [second system effect](https://en.wikipedia.org/wiki/Second-system_effect), when we haven't even got the first system out yet. It has a higher & more complex API surface area, introducing [5 traits](https://github.com/matthieu-m/rfcs/blob/store/text/3446-store.md#overview-1), and abstracting away _pointers_ to _handles_ feels like unnecessary indirection. I found it a bit more difficult to follow, trying to imagine porting a simple allocator to use it, compared to the existing trait.

I feel like there is a world where we can extend beyond the base trait to support some of the better bits of the Storage API, but it feels like an even bigger uphill battle to stabilise in lieu of the existing trait. i.e, let's take the best bits and stabilise them separately, if we can.

### A Stable Alternative

With the age of this feature, it's inevitable that there will be some experimentation of a bridge solution, while waiting for stabilisation. The [`allocator_api2`](https://github.com/zakarumych/allocator-api2) crate exists to fill this gap.

This crate mirrors the current trait definition in nightly, allowing use of the allocator trait, as it stands, on stable.

It is surprisingly a popular crate with over 250 million downloads on crates.io, and an optional dependency of crates like [`hashbrown`](https://github.com/rust-lang/hashbrown) and [`bumpalo`](https://github.com/fitzgen/bumpalo)

## The Cost of Monomorphisation

This isn't actually an existing issue, but more a discussion around some of the ramifications of the existing design. What I mean by the _cost_ of monomorphisation, apart from the code generation angle, is the developer experience. Having a new generic argument on all containers will cause a _lot_ of churn at call sites.

For example, right now, `Vec<T>` is generic against the value itself, and nothing else. I'm sure there are millions of functions out there that have a signature like:

```rust
fn do_something(input: Vec<String>) {
    // something with the vec
}
```

However, if we want out function to support custom allocated vecs, we now need to add trait bounds on the function signature:

```rust
fn do_something<A: Allocator>(input: Vec<String, A>) {
   // something with the vec
}
```

Not only that, mixing and matching vecs with different allocators also becomes a little painful, as `Vec<T, A> != Vec<T, B>`.

You can imagine this would be a nightmare to adjust everywhere. And so we need some guidance, or a solution for it. I did a small investigation around how other languages handled this, focusing on Zig and C++, since both support custom allocators.

Now, I am not an expert of either language, so there is a good chance I am wrong in some of this understanding. But the gist is that, it _feels_ like dynamic dispatch is not as bad as it seems here.

### Zig's Allocator

I wanted to look at how allocators worked in Zig, as the sentiment of using Zig is that it provides greater flexibility around memory allocations. So what does it do that rust doesn't, and how does it do it?

The [Allocator](https://github.com/ziglang/zig/blob/master/lib/std/mem/Allocator.zig) is a struct with two pointers: one for the state, and the other to a VTable of const fns:

```zig
// not really defined like this (but for illustrations only)
pub const Allocator = struct {
    ptr: *anyopaque,        // opaque pointer to allocator state
    vtable: *const VTable,  // pointer to constant function table
};
pub const VTable = struct {
    alloc:  *const fn (*anyopaque, usize, Alignment, usize) ?[*]u8,
    resize: *const fn (*anyopaque, []u8, Alignment, usize, usize) bool,
    remap:  *const fn (*anyopaque, []u8, Alignment, usize, usize) ?[*]u8,
    free:   *const fn (*anyopaque, []u8, Alignment, usize) void,
};
```

This is a type erased, dynamic dispatch style allocator. You would think that this would be slower than static dispatch, but in fact Nical [wrote a blog](https://nical.github.io/posts/rust-custom-allocators.html) that suggests that this isn't always the case: sometimes the code bloat from monomorphisation outweighs the performance benefits of static dispatch.

And besides, it does look like LLVM can, in most cases, [rewrite them as static calls anyway](https://pithlessly.github.io/allocgate.html), so you get to have your cake & eat it too.

How could we get something similar in rust? Well, with the trait as it exists now, we could just use `dyn Allocator` that essentially provides the same style dynamic dispatch, and use that as the `Allocator` bound in function calls:

```rust
fn do_something(input: Vec<T, Box<dyn Allocator>>) {
   // something with the vec
}
```

So with rust, we do have options to go down the dynamic dispatch path already, given the trait definition. Obviously object-safety, etc.. comes into it, but it's not completely out of the realm of possibility

### C++ Allocator story

C++ is an older language, and so you expect it to have some warts, moreso than rust.

C++ does have some of the same monomorphisation issues that rust will potentially have: different allocators means that the container type is different, but besides [templates](https://en.cppreference.com/w/cpp/language/templates.html) being a solution, C++17 introduced [polymorphic memory resources](https://www.modernescpp.com/index.php/polymorphic-allocators-in-c17/), or pmr to address this.

I.e, the following C++ code would error

```C++
auto vec1 = std::vector<int,allocator1>();
auto vec2 = std::vector<int,allocator2>();
auto vec = vec1;
vec = vec2;
```

pmr solves this with type erasure, in similar fashion to zig and `dyn` in rust (albeit probably not identical under the hood):

```C++
auto vec1 = std::pmr::vector<int>(&resource1);
auto vec2 = std::pmr::vector<int>(&resource2);
auto vec = vec1;
vec = vec2;
```

So C++ started off down the static route, and moved towards dynamic dispatch. Obviously you can still use the older container types, but it might be fitting that we pay a bit more attention to dynamic dispatch with allocators, since other languages are tending towards them.

If we had some LLVM trickery to be able to statically inline in the dynamic case (if it doesn't already exist in rust), as in Zig, we might not end up paying as much for it, or it _could_ be free.

## Three Steps Forward

So as you can see, almost 10 years on, there is still some outstanding decisions to be made. The inertia of getting this stabilised has definitely slowed down in recent years, but there is definitely still a need to think about this.

I like to frame it as boiling down to a few possible outcomes:

1. Wait for a better alternative to emerge or some language features
2. Stabilise the trait as-is: no further changes to the API and let's go for it
3. Work a little bit more on the trait and then stabilise

### Wait for a better alternative

This is essentially the current state of play and, unless there is something to help with the current level of inertia, in all likelihood we could still be here in another decade! I am not that pessimistic, however, and think that we can definitely solve it if there is some attention placed on it.

### Stabilise as-is

The trait, as it stands, is actually already useful and, in fact with the `allocator_api2` crate is already useable. And considering that the trait hasn't changed all that much recently, it might be an indication that it's ready for stabilisation. However, this could be because it's on nightly, meaning not many consumers of the API exist currently. So the risk of stabilising it means that there is a missed use case.

I am partially in favour of this direction. This would eliminate the rust for linux use case and other esoteric cases, but they have pressed on without it anyway.

### Work a little bit more on the trait

Coming back to the [75 open issues](https://github.com/rust-lang/wg-allocators/issues) and some of the main blockers mentioned, we could adjust the trait slightly so that it supports some extra needed features.

I was a little strategic in selecting the blockers to focus on, as they are _all_ composable together.

Omitting the trait split, but including: the context, non-zero layout and error associated type, you could envisage a trait looking like this:

```rust
pub unsafe trait Allocator {
    type Ctx: Copy;
    type Error: core::error::Error;

    fn allocate(
        &self,
        ctx: Self::Ctx,
        layout: NonZeroLayout
    ) -> Result<NonNull<[u8]>, Self::Error>;

    unsafe fn deallocate(&self, ptr: NonNull<u8>, layout: NonZeroLayout);
    // ...other provided methods...
}
```

I'm _not_ implying this should be the final shape, but I think something along these lines is within the realms of possibility.

Besides blockers mentioned above, there are some unsoundness issues like the [Box `noalias`](https://github.com/rust-lang/wg-allocators/issues/122) that might be resolved already, and a few other issues that might need some triage.

I think my approach would be:

- Divide issues into stabilisation blockers, design trade-offs, and miscellaneous/post stabilisation
- For the stabilisation blockers that don't _really_ require much change of the design, if at all, these should be worked through
- For the design trade-offs, a decision one way or another should be made, and incorporated in or not
- For the miscellaneous/post stabilisation issues, they can remain open if they are not critical.

I feel like working through the issues in the repo is the best path forward.

## Conclusions

As it stands, custom allocators are still not in a state to stabilise, and it would be good to stabilise them soon.

I haven't really touched on _why_ they are useful, since I feel that the benefits are pretty clear. My own interest in it comes from working on the [rust logfire team](https://pydantic.dev/logfire), [contributing to datafusion](https://datafusion.apache.org/), building, and [dealing with memory in production systems](/blog/a-tale-of-two-lengths/) that love to crash when they don't have memory. All of these things I am working on will benefit from custom allocators, and unblock me in some cases. I want to have the tools available to wrangle allocations where it counts.

I am hoping with this article that it at least spurs some discussion and breathes new life into custom allocators.

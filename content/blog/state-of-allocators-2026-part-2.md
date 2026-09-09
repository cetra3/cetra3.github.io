+++

title = "The State of Allocators in 2026 - 6 Months Later"
description = "Where we are up to 6 months later and exciting times ahead"
date = 2026-09-09

[taxonomies]
tags = ["rust"]
+++

From the time of [the last article](/blog/state-of-allocators-2026), to now, I'm pleased to report that there has been a resurgence of activity in getting custom allocators stabilised in rust and we are [closer than we've ever been](https://github.com/rust-lang/rust/pull/156882). I will preface this that while I have reported on it and pushed for it in person and online, I haven't been the main driving force. Nia has done an amazing job spearheading the effort, and so the majority of praise should go towards her and the rest of the libs team.

So, where do we stand currently with custom allocators, and what are the next steps?

## The Allocator Shape

As it stands today, the allocator shape, as per the [stabilise allocator trait PR](https://github.com/rust-lang/rust/pull/156882) and [subsequent stabilisation report](https://hackmd.io/nNHdKkp1TTK7jat0I-ABqA), does not look _very_ different from what the shape has been in nightly. The initial API surface, is rather limited, for reasons I will get into, but enough of a foundation to start building.

The main `Allocator` trait surface, is only two methods (with other methods provided):

```rust
unsafe trait Allocator {
    fn allocate(&self, layout: Layout) -> Result<NonNull<[u8]>, AllocError>;
    unsafe fn deallocate(&self, ptr: NonNull<u8>, layout: Layout);
}
```

The other surface layers, relate to using them with `Vec` and `Box`.

I.e, `Box::new_in`:

```rust
impl<T, A: Allocator> Box<T, A> {
    fn new_in(x: T, alloc: A) -> Box<T, A>;
}
```

And `Vec::new_in` and associated methods:

```rust
impl<T, A: Allocator> Vec<T, A> {
    // safe methods
    fn new_in(alloc: A) -> Vec<T, A>;
    fn with_capacity_in(capacity: usize, alloc: A) -> Self;
    fn allocator(&self) -> &A;

    // destructuring methods
    unsafe fn from_raw_parts_in(
        ptr: *mut T,
        length: usize,
        capacity: usize,
        alloc: A,
    ) -> Self;
    unsafe fn from_parts_in(
        ptr: NonNull<T>,
        length: usize,
        capacity: usize,
        alloc: A,
    ) -> Self;
    fn into_raw_parts_with_allocator(self) -> (*mut T, usize, usize, A);
    fn into_parts_with_allocator(self) -> (NonNull<T>, usize, usize, A);
}
```

### `dyn Allocator`

One of the outstanding questions during stabilisation is whether it was _safe_ to have `Allocator` become `dyn` compatible. With the reduced surface area, and because it would unlock a lot more use cases, the aim is to have `Allocator` dyn compatible straight off the bat. This is probably the biggest change in nightly since the last article.

```rust
let my_allocator: Arc<dyn Allocator> = <your allocator here>;

let my_vec = Vec::new_in(my_allocator);
```

I touched on this on the [last article](/blog/state-of-allocators-2026#zig-s-allocator) and having a dynamic dispatched allocator is not a big a deal for performance, in fact it's the default in Zig. Honestly I feel like this will become the de-facto way of using custom allocators when designing libraries, given the flexibility.

### Unwind and Allocators

One sharpening of the pencil is that unwinding from allocators is prohibited and the language around it has gotten stronger. There were a lot of unsoundness issues found by allowing it.

I.e, things like growing a vec, or resizing it, which can involve more than one allocation/deallocation, would mean that if you panicked _during_ these methods then the state of the vec itself can be inconsistent, i.e, ptr/len/cap are just wrong, which is a recipe for disaster, since the unwinder will call the vec destructor at that point, causing things like double free etc.. to be possible in safe code.

## What was left out

Because this change is so simple, but also so _fundamental_ to how rust works, there has been a _lot_ of work in finding out weird scenarios that would make certain usages _unsound_. As it stands there are no methods for other types of containers besides `Vec` and `Box`, and some methods on those types are excluded from custom allocators.

So you can think of this first stabilisation as an **MVP** rather than the end game for custom allocators: it's enough to get things going, but there is still a _lot_ of outstanding work until it's finished. To clarify, you will be able to use it _straight away_, but you might find some usages blocked on stabilisation in other areas.

I'll do my best to explain what was excluded, and the reasons why, but there are discussions in both the issues and zulip chat that dive much deeper into them.

### Allocators, Drop and Clone

When you `Drop` an allocator, what are you actually dropping here? Are you dropping the engine for allocating memory, or are you dropping the _memory_ itself? This is an interesting question, but actually does rely on what your implementation does and cares about. For some allocators, dropping on the destructor is something you want to do. For others, it's probably not the intention.

In past revisions, the following blanket impl was provided:

```rust
impl<T: ?Sized, A: Allocator + Clone> Clone for Arc<T, A>
```

That is, if we have an arc of something, which has a custom allocator, as long as the custom allocator was cloneable, we could clone the arc as well.

But what happens if the allocator we used would deallocate on drop?

I.e, take this psuedo code (and note the [original issue surfacing this](https://github.com/rust-lang/rust/issues/156920) has a more concrete version)

```rust
// your allocator that deallocates on drop, and *cloning* backs to the same store
let my_memory_arena = MemoryArena::new();

// allocate within the memory arena
let the_first_arc = Arc::new_in(123456, my_memory_arena);

// what are we cloning here
let the_second_arc = the_first_arc.clone();

// drop the first arc, causing the allocator to drop, but we have `the_second_arc` still!
drop(the_first_arc);

// drop the second arc, uh oh, double free!
drop(the_second_arc);
```

There is no unsafe code here (besides the unseen `MemoryArena` allocator impl). But, you might say, well, if you have `MemoryArena` or any other allocator you can _just_ write out in the unsafe contract that if we _clone_ it, then it is cloning the same memory as well, and so we can side-step the issue and we won't double free.

Which is all well and good, but `Clone` as a trait is _not_ unsafe, and so if someone downstream decides to wrap your type, with some `Box` shenanigans, as [per the original issue](https://github.com/rust-lang/rust/issues/156920), we can actually make unsound code without using `unsafe` at all:

```rust
pub trait NewTrait: Allocator {}
impl<A: Allocator> NewTrait for A {}

impl Clone for Box<dyn NewTrait> {
    fn clone(&self) -> Self {
        Box::new(System)
    }
}

fn main() {
    let the_first_arc = Arc::new_in(123456, Box::new(MemoryArena::new()) as Box<dyn NewTrait>);
    let the_second_arc = the_first_arc.clone();
    drop(the_first_arc);
    // uh oh, we're unsound without unsafe!
    drop(the_second_arc);
}
```

OK so that is the problem, but _why_ do we need clone in the first place? Well, beside it being extremely convenient in circumstances, some of the container types, I.e, `BTreeMap` need it to support custom allocators.

I.e, `split_off` needs to clone the allocator, so that the new split of the map can allocate again. But then say you go and delete the original map you've split from, dropping the original allocator:

```rust
let my_memory_arena = MemoryArena::new();

// let's make a map
let mut a = BTreeMap::new_in(my_memory_arena);
a.insert(1, "a");
a.insert(2, "b");
a.insert(3, "c");
a.insert(17, "d");
a.insert(41, "e");

// let's split it off
let b = a.split_off(&3);

// let's drop the original map, dropping the original allocator (but `b` still has a clone of it)
drop(a);

// uh oh, we've *lost* our underlying memory!
drop(b);
```

And so we _want_ to allow cloning an allocator, but with _extra_ requirements around cloning it. And so this leads us to the current design in that, we can't use the `Clone` trait, because its contract is _safe_ for things to implement, and too weak a guarantee for what we need.

Right now, the current (but still in flux) design is to introduce a new unsafe marker trait: `AllocatorClone` which can be used as a marker trait to provide the guarantees we need: i.e, we don't deallocate memory on drop, among other things. With this trait in place, we can add this to the method signatures, ensuring that only allocators that implement the marker trait can be used, pushing the guarantees into the `unsafe` portion, rather than allowing _potentially_ unsound code in safe rust.

### Pin and Memory Covariance

This one to me was a bit harder to get my head around when reading through it. And so, once again I'll do my best to illustrate with the [example from the issue](https://github.com/rust-lang/rust/issues/157089). But this is similar to the shape of why `clone` needs stronger guarantees.

#### Covariance

Firstly, what does _covariant_ mean? Because I had to work this out when reading the issue, and it's one of those things you build an instinct for when dealing with lifetimes that you don't really care about the underlying type theory terms (well, in my case anyway!).

Well, when it comes to lifetimes, say you have a lifetime `'a` defined somewhere in a method signature:

```rust
let static_str: &'static str = "hello world!";

// this works! because of covariance!
let shorter: &'a str = static_str;
```

But `'static != 'a`, however `'static` is a _subtype_ of `'a`, which feels backwards but makes sense. `'static` always lives longer than `'a` so anywhere `'a` is expected, `'static` can be used instead. And this substitution works because `&'a str` is _covariant in `'a`_.

#### Pin and its guarantees

Most people think of `Pin` as a type that is never _moved_ in memory. But I mean, something has to allocate it, and something has to deal with it when it's deallocated. But, there is another guarantee of pinned data: it _must_ be dropped before it is invalidated. I.e, if you have something that is in `Pin` we need to _ensure_ that drop is called on it before invalidating, so things can clean it up, etc...

And so, to tighten the pin guarantee (oversimplified):

- It must _not_ move in memory
- It must be dropped _before_ its memory is invalidated

OK, but why the second constraint? Why do we have to guarantee that drop is ran? Well, mainly so anything that is relying on it being pinned can know it's about to be destroyed/invalidated. I think the docs for pin say it nicer than I can paraphrase:

> There needs to be a way for a pinned value to notify any code that is relying on its pinned status that it is about to be destroyed. In this way, the dependent code can remove the pinned value's address from its data structures... with the knowledge that it can no longer rely on that value existing at the location it was pinned to.

But, and here is where the tension lies, `Drop::drop` does _not_ have to be called for every value. In fact, you can _leak_ values, or use `ManuallyDrop` to avoid calling `Drop::drop`.

But, _leaking_ a value actually still _upholds_ the guarantee! In, what I consider feels like an excercise in malicious compliance, you can leak a `Pin` value, because it's never _invalidated_, then drop never _needs_ to run! Because our guarantee is "don't invalidate before drop", we never invalidate, so we never have to drop either.

#### Let's do some boxing

Once again the [full real code is in the issue](https://github.com/rust-lang/rust/issues/157089), and this is a simplification.

The allocator in use here is a `Bump` allocator which has a `reset(&mut self)` method, that can be used to clear the allocator's memory.

So we can write things like:

```rust
let mut arena = Bump::new();

// allocate a value in the arena
let my_value = Box::new_in(Thing::new(), &arena);
```

However, notice we're feeding in `&arena` here to the box allocation.

This means that we can rely on lifetime rules to _prevent_ `reset()` from being called while the allocation lives:

```rust
// allocate a value in the arena
let my_value = Box::new_in(Thing::new(), &arena);

// forget the value.
std::mem::forget(my_value);

// reset the arena, after forgetting the value, invalidates the underlying memory
arena.reset();
```

And this is all fine, because this value _isn't_ pinned, and so we can invalidate memory without running the destructor. There are no guarantees here.. yet!

However, if we wanted to _pin_ this value, we need a way to do that safely, and ensure that we run the destructor before invalidating memory.

I.e, we could possibly write a _safe_ `Box::pin_in` method, that ensures, for our bump allocator example, we _can't_ call `reset()` on it. And one way to do that is to make the allocator `'static` so we can't get a `mut` reference to it.

And so, that _seemed_ to be the needed constraint, the allocator being `'static`, and for a long time in nightly that is what the signature read as:

```rust
pub fn pin_in(x: T, alloc: A) -> Pin<Self>
where
    A: 'static + Allocator, // we're all good, we're 'static!
{
    Self::into_pin(Self::new_in(x, alloc))
}
```

And so this wouldn't compile and guards safe code:

```rust
// our memory arena
let mut arena = Bump::new();

// nope does not compile, &arena is not 'static!
let pinned = Box::pin_in(Thing::new(), &arena);

// if it *did* compile we could do this, unsoundly
std::mem::forget(pinned);
arena.reset(); // boom! invalidate memory *before* drop ran, we've broken our pin
```

OK so then we can leak the bump arena to satisfy and compile things:

```rust
// our memory arena
let static_arena: &'static Bump = Box::leak(Box::new(Bump::new()));

// let's pin our allocation!
let pinned = Box::pin_in(Thing::new(), &static_arena);

// this is fine because we can't `reset()`: memory is leaked, but never invalidated
std::mem::forget(pinned);
```

Ok, but with _covariance_ we can convert a `'static` to a `'a` or another lifetime:

```rust
// our memory arena
let static_arena: &'static Bump = Box::leak(Box::new(Bump::new()));

// let's pin our allocation!
let pinned: Pin<Box<Thing, &'static Bump>> = Box::pin_in(Thing::new(), &static_arena);

// we'll convert this down to a different lifetime
let pinned_coerced: Pin<Box<Thing, &Bump>> = pinned;
```

Hmm, this still doesn't really seem like a bad thing does it? Really _nothing_ has changed, the bytes are still where they are, and really it is still a static allocator.

Well, for the _next_ part, we are back to our interaction with _Clone_ doing strange things. `Pin::clone()` is derived, and so it will defer to the inner type. In this case `Box`. And because `Box` is marked as a `#[fundamental]` type, we can actually provide our own implementation of it.

So, let's write an implementation, but we'll keep a reference to an allocator so we can do `Box::new_in` during clone.

And while we're there, we want our type to have `Pin` guarantees. I.e, `!Unpin`. Easiest way to do that is to add `PhantomPinned`:

I.e, `Thing` becomes (with a new impl):

```rust
struct Thing<'a> {
  alloc: &'a Bump,
  _pin: PhantomPinned,
}

impl<'a> Thing<'a> {
    pub fn new(alloc: &'a Bump) -> Self {
        Thing {
            alloc,
            _pin: PhantomPinned,
        }
    }
}
```

And to make things interesting we can have a impl drop that panics (which will never actually run):

```rust
impl<'a> Drop for Thing<'a> {
    fn drop(&mut self) {
        panic!("drop called!");
    }
}
```

And _then_ we can add a `Clone` for it:

```rust
impl<'a> Clone for Box<Thing<'a>, &'a Bump> {
    fn clone(&self) -> Self {
        Box::new_in(Thing {
            alloc: self.alloc,
            _pin: PhantomPinned,
        }, self.alloc)
    }
}
```

OK, and _now_ we're at the unsound bit. How can we combine this code we've built up, in a _safe_ way, but still violate the `Pin` invariants? Because if we can, we have written unsound code in safe rust. Which is not a good thing.

We can use a combination of clone, covariance/type coercion and some other shenanigans to make unsound code in safe rust.

We are going to use two allocators, a static bump allocator to satisfy the `Box::pin_in` requirements, and a short lived allocator, which we can coerce to using with covariance, _breaking_ pin guarantees.

```rust
// our static memory arena
let static_arena: &'static Bump = Box::leak(Box::new(Bump::new()));

// our short lived allocator
let mut short_arena = Bump::new();

// we have a short arena internally in `Thing`, but we have a static arena to satisfy `pin_in` requirements
let pinned: Pin<Box<Thing, &'static Bump>> = Box::pin_in(Thing::new(&short_arena), static_arena);

// ok we can coerce it now! No worries this is fine
let pinned_coerced: Pin<Box<Thing, &Bump>> = pinned;

// here is where it's gnarly, our *clone* will make a new value in our provided non-static allocator
let pinned_in_short: Pin<Box<Thing, &Bump>> = pinned_coerced.clone();

// forget our pinned_coerced value.  This fine for pin because it's allocated by a static allocator
std::mem::forget(pinned_coerced);
// forget our pinned in short value.  This is not great because it's allocated by a non static allocator. Uh oh...
std::mem::forget(pinned_in_short);

// since we've forgotten *both* values, there is no outstanding immutable &short_arena reference!
short_arena.reset(); // BOOM! We've violated pin, pinned_in_short has its invalidated memory without calling drop
```

At the `reset()` line here, we have managed to _invalidate_ the memory of `pinned_in_short`, without calling a destructor/drop.

The `drop` impl isn't run, because we've forgotten the value, but we have invalidated the memory with the `reset()` method.

And this means our `Pin` invariants are _not_ upheld and we have perfectly safe code doing unsound things.

The [original issue](https://github.com/rust-lang/rust/issues/157089), as already mentioned, has a much more fleshed out version with printlns etc.. showing this behaviour.

So what does _this_ mean? Where _does_ the design fall down? Is the allocator implementation _wrong_? Although this is a bit convoluted, we are combining a few innocuous looking things here to make things unsound. I don't feel like the bump allocator is at fault here, there are valid use cases for this shape. I also don't feel that the pin methods or cloning is at fault _either_, they are allowing something useful.

Really this comes down to the guarantees that the allocators provide and being able to ensure that they are used in a safe way. The current design thinking is that `'static` by itself on `pin_in` is not a strong enough guarantee, as evidenced by the raised issue. In fact because of covariance rules we can't rely on lifetimes here to save us. We need the guarantee on the allocator _itself_.

And so, a bit like the allocator clone problem, we can make an unsafe marker trait `StaticAllocator` in which, if an allocator is written, and impls, has some _extra_ invariants to uphold. This way we can split out bump allocators etc.. and use them where they are safe to use them, but if we want a custom allocator and `Pin` interaction then that allocator needs to be safe to use.

But for the MVP, `Box::pin_in` is not in the surface area, while this is fleshed out more.

### Unsoundness and API precedent

So, both of these issues are pretty gnarly, and, luckily were caught before stabilisation. There _is_ a possibility of a latent unsound issue coming out of these changes, albeit the reduced surface area mitigates this somewhat. But, this has happened before, and there has been _some_ precedent and guidance around what can be done about it. I.e, if some unsound pathway is discovered, what can be done about it, and what has happened before?

#### `Error::type_id`

In `1.34.0` the team stabilised the `Error::type_id` function. At the time this seemed like a pretty standard change. About a month later, the team _reverted_ this change as it was found to be unsound and released a [security advisory](https://blog.rust-lang.org/2019/05/13/Security-advisory/).

#### `std::env::set_var`

This one is probably one that some users would have had direct interaction with. The `set_var` can be unsound if set by multiple threads at once, causing race conditions and all sorts of undefined behaviour. On some not-so nice social media platforms this is used as ammunition to show that rust isn't a safe language.

The fix here, since it had pretty wide spread usage, was to adjust it to be `unsafe` after the `2024` edition.

#### Language semver

[RFC 1122](https://rust-lang.github.io/rfcs/1122-language-semver.html) does allow for breaking changes if they are unsound. So my bet is that if there are any unsoundness issues found after stabilisation, they will be quickly mitigated/reverted or tightened.

## What's next?

Well, the libs team members are in the process now of deliberating the change, and if it's all good, will soon enter into the final comment period. This is a 10 day "cooling off" period that allows anyone to raise concerns before it stabilises. Note that the 10 day period does not start until the majority of team members sign off, so this can take weeks after an FCP proposal.

Then, after that, and the 10 day window, it follows the standard cadence of releases. Nightly is merged straight away, 6 weeks later it goes to beta, and then 6 weeks after that it goes to stable.

I have started having a play around with it now, to see how far I can get. One of the things that I've already ran into is that allocation failures abort the process rather than panic. This made sense with a global allocator, but I'd like to be able to use custom allocators to _limit_ memory (i.e, in my [`thresher`](https://github.com/cetra3/thresher) crate that provides a global soft/hard memory limit). So an option to panic instead of abort on allocation failure would be one extra nicety.

So realistically, at a minimum this is still months off. But for someone, like myself, who has been waiting a long time for it, it's not that much longer to wait.

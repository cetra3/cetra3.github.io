+++

title = "Handling Breaking API Changes"
description = "Dealing with Breaking API"
date = 2020-02-03

[taxonomies]
tags = ["rust", "semver"]
+++

While upgrading dependencies (basically deleting `Cargo.lock`) in a big rust project I hit an issue.  A [new commit](https://github.com/serde-rs/json/commit/97f87f2587037dcd50b9504815ee1f1540b1c2b8) in `serde_json` caused has upstream failure in another library, [jmespath](https://github.com/mtdowling/jmespath.rs/issues/33), and possibly more crates. Because these structs and enums visibility has changed, this could be interpreted as a breaking change, depending on what you consider *public*.

While `jmespath` was doing the wrong thing here, using undocumented API, it got me thinking about what can library maintainers and library consumers do to ensure semver compatibility, and what tools are out there to assist both groups.

## What is Semantic Versioning

Semantic versioning (semver) is one standard used for version numbers which appears to have gained the most traction.   Since cargo and friends use semver to identify what versions to use, it is important to understand (excuse my pun) the semantics of what this means.

The full documentation of semver is on [https://semver.org/](https://semver.org/), but for brevity there are 3 numbers:

* Major: Any breaking API changes
* Minor: Additions of new features or functionality that are backwards compatible
* Patch: Bugfixes

There is some interesting differences for libraries `< 1.0.0` which I was not aware of for longer than I care to admit.  Namely either minor or patch numbers can be changes as you see fit.  I.e, `0.3.0` could be wildly different from `0.3.1`, including breaking changes.

However: Cargo is [more strict](https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html) than the semver documentation for `< 1.0.0` libraries:

> This compatibility convention is different from SemVer in the way it treats versions before 1.0.0. While SemVer says there is no compatibility before 1.0.0, Cargo considers `0.x.y` to be compatible with `0.x.z`, where `y ≥ z` and `x > 0`.


## Why we need it

We need some way of portraying changes to an API and, while it's not perfect, it appears to be a defacto standard which makes interoperability with other libraries and crates a lot easier.

It also gives cargo a programmatic way of finding updated versions, when it's used appropriately.

So it is one part advertising the intent of the version and another part ensuring that things will compile if you bump up certain numbers.

## API Evolution RFC

The Semver website is very general in its language and isn't specific to rust.  There is, however, a great RFC that has some excellent guidelines both for the language itself & for library maintainers: [RFC 1105: API Evolution](https://github.com/rust-lang/rfcs/blob/master/text/1105-api-evolution.md)

Here are a couple of excerpts:

> #### Major change: going from stable to nightly
> Changing a crate from working on stable Rust to *requiring* a nightly is
> considered a breaking change. That includes using `#[feature]` directly, or
> using a dependency that does so. Crate authors should consider using Cargo
> ["features"](http://doc.crates.io/manifest.html#the-[features]-section) for
> their crate to make such use opt-in.


> #### Major change: renaming/moving/removing any public items.
> 
> Although renaming an item might seem like a minor change, according to the
> general policy design this is not a permitted form of breakage: it's not
> possible to annotate code in advance to avoid the breakage, nor is it possible
> to prevent the breakage from affecting dependencies.

This RFC has some great advice, and I wish it was more publicly known, but seems relatively obscure.

## The Non-Exhaustive Attribute

Another RFC related to versioning is [Non-Exhaustive](https://github.com/rust-lang/rfcs/blob/master/text/2008-non-exhaustive.md), which is a handy way of planning breaking changes ahead of time.  It works with enums by forcing open ended `match` statements, which means consumers of your enum won't break if a new variant is introducted. It also works with structs and ensures users consuming libraries can't create them.

This was recently [stabilised in Rust 1.40](https://blog.rust-lang.org/2019/12/19/Rust-1.40.0.html) so it's a rather new feature of the language (on a side note: it's not easy to find when a particular RFC is stabilised, it would be great to have this reflected in the RFC somewhere).

As an example if you have an enum like this:

```rust
#[non_exhaustive]
pub enum Error {
    Message(String),
    Other,
}
```

You would need to `match` against the enum like this:

```rust
match error {
    Message(ref s) => ...,
    Other => ...,
    _ => ...,
}
```

Your library could then introduce a new variant, which wouldn't break the existing consumer:


```rust
#[non_exhaustive]
pub enum Error {
    Message(String),
    IOError,
    Other,
}
```

## Other Tools

There have been some tools to both manage breaking changes in the past, a lot of the things I found were unmaintained or not in active use.  There might also be some great language features that I haven't discovered yet that crate authors are already using.

### The Semver Trick

Documented here is the [semver trick](https://github.com/dtolnay/semver-trick), which touts to be able to assist in this with downstream dependencies.

There is mention of a `libcpocalypse`, which does not sound nice and may happen again if there is a [major version increase](https://github.com/rust-lang/libc/issues/547) to `libc`.  I tried to find some information about where this had happened before but searching for the term circles back to this project and doesn't give a great overview of the pains users had to go through before.

### Rust Semverver

[rust-semverver](https://github.com/rust-dev-tools/rust-semverver) is a Google Summer of Code Project which checks semver compliance with rust crates by using a novel approach:

> The approach taken is to compile both versions of the crate to rlibs and to link them as dependencies of a third, empty, dummy crate. Then, a custom compiler driver is run on the said dummy and all necessary analysis is performed in that context, where type information and other resources are available.

### Semantic-RS

An existing tool called [semantic-rs](https://github.com/semantic-rs/semantic-rs) touts to be able to do this for your crates automatically.  This doesn't appear to have any changes for over a year, but does have an active issue register, but no PRs have been accepted for a while.  It is highly opinionated as well, and may not be appropriate to bolt on to an existing library.  This looks like something you need to use from the beginning as well.

### Rust Breaking Changes

This also doesn't appear to be maintained, but was an automatic listing of breaking changes to rust: [https://killercup.github.io/bitrust/](https://killercup.github.io/bitrust/).   I do wonder though how many of those changes are internal, as I don't believe there have been many external breaking changes.

### Future Proofing

There is a chapter of the [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/about.html) which deals with how to future proof an API, but as with other tools I have found does not appear to be updated in [quite a while](https://github.com/rust-lang/api-guidelines/commits/master/src/future-proofing.md):

**Future Proofing**: [https://rust-lang.github.io/api-guidelines/future-proofing.html](https://rust-lang.github.io/api-guidelines/future-proofing.html)

## Conclusion

While there has been a [historic push to get crates to 1.0.0](https://blog.rust-lang.org/2018/03/12/roadmap.html#library-improvements), There does not appear to be much in the way of modern tooling in order to help with this and the focus of previous efforts see them moving elsewhere.

With more libraries reaching `1.0.0` I think it's important to introduce more safety nets for library authors and consumers.  As an idea, we can potentially see whether a crate has breaking change through lints etc.. so why don't we do that when we publish to crates.io?

There is a lot of room for improvement here, and as new users come on board and new companies look at adopting, it's important that the stability of semver is maintained.  I still think the way that cargo handles this is much better than other ecosystems.

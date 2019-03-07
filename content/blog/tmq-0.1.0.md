
+++
title = "TMQ 0.1.0 Release: ZeroMQ bindings for Tokio"
description = "Discussion around building TMQ"
date = 2019-02-07
+++

[TMQ](https://github.com/cetra3/tmq) is a rust library to use [ZeroMQ](http://zeromq.org/) within the [Tokio](https://tokio.rs/) ecosystem.  ZeroMQ is a distributed messaging queue written in C supporting a number of different messaging patterns.  While there are other options out there (including gRPC, etc..), I settled on ZeroMQ due to its cross-language support, great documentation and battle-tested nature.

[Version `0.1.0`](https://crates.io/crates/tmq/0.1.0) is an alpha release which implements `request`, `response`, `publish` and `subscribe` style sockets.


## Usage Example

This example uses `tmq` to subscribe to messages, and then prints them out via [pretty-env-logger](https://github.com/seanmonstar/pretty-env-logger):

```rust
extern crate failure;
extern crate futures;
#[macro_use]
extern crate log;
extern crate pretty_env_logger;
extern crate tmq;
extern crate tokio;

use futures::{Future, Stream};
use tmq::*;
use std::env;

fn main() {
    if let Err(_) = env::var("RUST_LOG") {
        env::set_var("RUST_LOG", "subscribe=DEBUG");
    }

    pretty_env_logger::init();

    let request = subscribe(&Context::new())
        .connect("tcp://127.0.0.1:7899")
        .expect("Couldn't connect")
        .subscribe("")
        .for_each(|val| {
            info!("Subscribe: {}", val.as_str().unwrap_or(""));
            Ok(())
        })
        .map_err(|e| {
            error!("Error Subscribing: {}", e);
        });

    tokio::run(request);
}
```

Other examples of usage can be found in the project itself: [https://github.com/cetra3/tmq/tree/master/examples](https://github.com/cetra3/tmq/tree/master/examples)

## Existing Rust Crates

To find out where `tmq` sits within the rust ecosystem, it makes sense to discuss this in terms of other ZeroMQ crates.

### [zmq](https://crates.io/crates/zmq) - Rust ZeroMQ Bindings

This library has bindings to the C API and provides a great way to use ZeroMQ within rust.  You still need a copy of ZeroMQ on your system and need it compiled in, but that difficulty is pretty much on par with using OpenSSL.  I have managed to use this library in OSX, CentOS and Ubuntu with not many issues.

The only downside of this library is the lack of bindings to tokio, and is one of the primary motivators for creating `tmq`.

### [zmq.rs](https://github.com/zeromq/zmq.rs) - A Native Implementation of ZeroMQ in Rust

This library was a Rewrite in Rust attempt at a full ZeroMQ reimplementation, which hasn't seen any changes [since 2015](https://github.com/zeromq/zmq.rs/commits/master) and can probably be considered abandoned.  Regardless: this API is still the old blocking style, which would still predicate a need for `tmq`

### [zmq-tokio](https://github.com/rotty/zmq-tokio) - Run ØMQ sockets using tokio reactors, futures, etc.

Not even published on crates.io but a great initial attempt at bridging the async gap.  This crate used the historic [tokio-core](https://github.com/tokio-rs/tokio-core) library which is a bit of a pain to adapt to the new [tokio](https://github.com/tokio-rs/tokio) crate.  Unfortunately it looks like it's since [been abandoned](https://github.com/rotty/zmq-tokio/pull/7).

### [tokio-zmq](https://crates.io/crates/tokio-zmq) - ZeroMQ Concepts with futures on Tokio's runtime

This library is probably the most polished version for `tokio` and one that would be compared mostly to `tmq`.

I would have chosen this library, but the one major roadblock is the [GPL](https://www.gnu.org/licenses/gpl-3.0.en.html) license.  The GPL within a rust project is viral.  You can't use this library without making the rest of your project GPL or GPL Compatible. If this doesn't concern you, then I would consider using this library.

### Comparison between tmq

In comparison to `tokio-zmq`, `tmq` has less boilerplate to acheive the same thing, but does use a couple of custom traits to speed things along.  This makes `tmq` a bit more opinionated, but less verbose.

Both styles have their merits, `tokio-zmq` does give you more control over things wheras `tmq` is, in my opinion, easier to write and reason about, but more restrictive in how you use it.

To do a direct comparison of both libraries we're using the `response` example.  This example is a simple echo response, which when it receives a message, it echos it back verbatim to the requester.

#### tokio-zmq

Here is the [excerpt](https://git.asonix.dog/asonix/async-zmq/src/branch/development/tokio-zmq/examples) from `tokio-zmq`:

```rust
let ctx = Arc::new(zmq::Context::new());
let req_fut = Req::builder(ctx).connect("tcp://localhost:5560").build();

let runner = req_fut.and_then(|req| {
    req.send(build_multipart(0)).and_then(|req| {
        let (sink, stream) = req.sink_stream(25).split();

        stream
            .zip(iter_ok(1..10_000))
            .map(|(multipart, i)| {
                for msg in multipart {
                    if let Some(msg) = msg.as_str() {
                        println!("Received: {}", msg);
                    }
                }
                build_multipart(i)
            })
            .forward(sink)
    })
});
```

On of the things you will notice is there is a lot of standard `futures` and `tokio` types used here: `sink`, `stream`, etc..  While this does make it more verbose, you are using constructs that you are familiar with in the futures style.

### tmq

Here's a [similar example](https://github.com/cetra3/tmq/blob/master/examples/response.rs) (although not using `multipart` messages) from `tmq`:

```rust
let responder = respond(&Context::new())
    .bind("tcp://127.0.0.1:7899")?
    .with(|msg: Message| {
        info!("Request: {}", msg.as_str().unwrap_or(""));
        Ok(msg)
    });
```

You'll notice that the library has a bit less boilerplate, but is more opinionated on how you structure a response.

Instead of a `sink/stream` approach, the responder is a stream but has a `with` method.  The `with` method takes anything that implements the `Responder` trait, of which there is a blanket implemenation for closures and functions that take a message and return a message.

## Future Changes

While currently usable, the `tmq` library is far from finished and requires some work to bring it out of an alpha state.  In no particular order here are the plans for the library:

#### Testing

Unit tests and integration tests need to be added in order to confirm and wire up connections.  As the library is rather lightweight now the need for integration tests hasn't really arisen, but this will become more complicated in the future as more socket types are added.

#### Benchmarking

Benchmarks need to be added to the library in order to show the performance of using this over standard `zmq`.  Running up some dummy benchmarks show that it is perfomant enough for my current use case, and provides less overhead than your standard REST API calls.

#### Windows Support

I *cheated* a little by using the [Evented](https://docs.rs/mio/0.6.16/mio/event/trait.Evented.html) trait for mio, which makes it dead easy to use in tokio via [PollEvented2](https://docs.rs/tokio/0.1.15/tokio/reactor/struct.PollEvented2.html).

Unfortunately the async story on windows is a bit different, and I haven't had a need to deploy on windows just yet.  But it is something that has been considered.

#### Documentation

The library is mostly undocumented besides from the examples.  Documentation should be pretty easy to do at this stage and won't take too long.

#### More Socket Types

Implementing more socket types, to make this feature complete with the standard `zmq` library.  There a plethora of different socket types for different use cases that give different guarantees.  The 4 standard ones implemented are enough for me to start using this library today, but could easily be expanded.

#### Multipart messages

Multipart support for messages.  Currently this is not supported, but should be mostly easy to implement

## Further thoughts

I'm currently using `tmq` within an `actix-web` [application](https://www.schoolbench.com.au/) to bridge some messages and audit logs between a polyglot backend (including Java & Python).  It has been quite solid so far, and I have plans to remove an existing ActiveMQ service to be replaced with `tmq` where appropriate.

Version `0.1.0` is the first real release, with previous releases having to vendor in the `zmq` library in order to publish.   While it is alpha, I don't plan to change what is there currently unless there is a compelling reason to do so.

Please give [tmq](https://crates.io/crates/tmq) a try and let me know your thoughts!

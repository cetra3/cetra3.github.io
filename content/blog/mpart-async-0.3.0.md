
+++

title = "Lessons learnt updating a library to std::future"
description = "Updating a multipart-async library to use futures"
date = 2020-01-22

[taxonomies]
tags = ["rust", "mpart-async"]
+++

With the new `std::future` way of doing things and [tokio](https://tokio.rs) slowly reaching maturation, it's time to look at updating the libraries out there that are using *the old ways*.  For one of my libraries, [tmq](https://crates.io/crates/tmq), a Tokio ZeroMQ library, there is some [awesome work already done](https://github.com/cetra3/tmq/pull/5) to get this updated.

But, I thought it pertinent to at least get my feet in the water to see how hard it would be, from a library maintainer perspective, to update to `std::future`.  For this effort, I chose my small library: [mpart-async](https://crates.io/crates/mpart-async).   You can see the changes I have made by comparing the [versions](https://github.com/cetra3/mpart-async/compare/0.2.1...0.3.0) here.  This blog is a small collection of notes & gotches I found when porting code across.

`mpart-async` is a *mostly* agnostic library for client side `multipart/form-data` requests.  There are existing libraries out there, like [multipart](https://crates.io/crates/multipart), but I found the API a little unwieldy for my taste (That: and async support is [still in alpha](https://github.com/abonander/multipart-async) according to the readme). I wanted something that worked, and was simple, but didn't offer an opinion on the web client. I use both the [actix client](https://github.com/actix/actix-web/tree/master/awc) &  [hyper](https://crates.io/crates/hyper) to make multipart requests depending on the project.

The challenge is that with multipart requests you mostly have fields & binary files.  For binary files, you can appreciate, should not be buffered entirely in memory, but streamed out as the bytes become available.  So `mpart-async` works with multiple internal streams of files and also provides a [convenience wrapper](https://github.com/cetra3/mpart-async/blob/981ba0437e19fa47f94a913cf9aaa4717fbe12bc/src/filestream.rs) (if using tokio) for sending files given a path.


## A New Example

All these changes to support `async fn` and is it actually easier to consume/use async libraries?

Given the [hyper](https://crates.io/crates/hyper) example, I would say yes.

Compare the [new example](https://github.com/cetra3/mpart-async/blob/981ba0437e19fa47f94a913cf9aaa4717fbe12bc/examples/hyper.rs):

```rust
#[tokio::main]
async fn main() -> Result<(), Error> {
    //Setup a mock server to accept connections.
    //....

    let client = Client::new();

    let mut mpart = MultipartRequest::default();

    mpart.add_field("foo", "bar");
    mpart.add_file("test", "Cargo.toml");

    let request = Request::post("http://localhost:3000")
        .header(
            CONTENT_TYPE,
            format!("multipart/form-data; boundary={}", mpart.get_boundary()),
        )
        .body(Body::wrap_stream(mpart))?;

    client.request(request).await?;

    Ok(())
}
```

With the [older one](https://github.com/cetra3/mpart-async/blob/c928f015fa31cd57533d4ba43551a9a96b61b0a2/examples/hyper.rs):

```rust
fn main() {
    // current_thread::Runtime can't be used because of the blocking file operations
    let mut rt = Runtime::new().expect("new rt");

    //Setup a mock server to accept connections.
    //....

    // Open `Cargo.toml` file and create a request
    let request = File::open("Cargo.toml")
        .map_err(|e| format!("{}", e))
        .and_then(|file| {
            // A Stream of BytesMut decoded from an AsyncRead
            let framed = FramedRead::new(file, BytesCodec::new());

            let mut mpart = MultipartRequest::default();
            mpart.add_field("foo", "bar");
            mpart.add_stream(
                "foofile",
                "Cargo.toml",
                "application/toml",
                framed.map(|b| b.freeze()),
            );
            Request::post("http://localhost:3000")
                .header(
                    CONTENT_TYPE,
                    format!("multipart/form-data; boundary={}", mpart.get_boundary()),
                ).body(Body::wrap_stream(mpart))
                .into_future()
                .map_err(|e| format!("{}", e))
        });

    // Send request
    let task = request.and_then(|request| {
        let client = hyper::Client::new();
        client
            .request(request)
            .map_err(|e| format!("{}", e))
            .and_then(|response| {
                response
                    .into_body()
                    .concat2()
                    .map_err(|e| format!("{}", e))
                    .and_then(|body| {
                        if let Ok(data) = str::from_utf8(&body) {
                            println!("Response: {}", data);
                        }
                        Ok(())
                    })
            })
    });

    rt.block_on(task).expect("request failed");
}
```

While there are still a few warts (such as having to manually add the boundary header), I find the code a lot more readable and easy to follow, rather than your standard combinator paths.


## No more `Error` Associated Type

The most drastic change to the `Stream` trait is that there is no longer an `Error` associated type.  The old `Stream`/`Future` traits had both `Item` and `Error`, as it assumed that streams were always going to be fallible.  The new traits do away with the `Error` associated type.  Instead, if you want your `Stream` to possibly be an error then you need to return a `Result` as your `Item` type.

For the `Stream` trait, the method you implement has changed to `poll_next` and uses the `std::task::Poll` enum as a return type.

The `Poll` enum did feel a little inside out when I started using it, but makes sense in terms of there being no `Error` type.  You don't return a `Result<Async<Option>, _>...` you instead return a `Poll<Option<Result<_,_>>>`.

Generally, this means if you wrote this:

```rust
return Ok(Async::Ready(Some(bytes)))
```

You instead will return:

```rust
return Poll::Ready(Some(Ok(bytes)))
```

## `StreamExt` traits and friends

As a consequence of no `Error` associated type, and an example of different ergonomics, if you are dealing with `Result` items, you may want to use the [TryStreamExt](https://docs.rs/futures/0.3.1/futures/stream/trait.TryStreamExt.html) trait instead of [StreamExt](https://docs.rs/futures/0.3.1/futures/stream/trait.StreamExt.html).  They are basically the same trait but one deals with `Result` a bit more nicely.  I'm not sure whether a simpler solution will be possible with higher kinded types in the future, but for now it is a tiny bit more complex than futures 0.1.

Interestingly, the [index page](https://docs.rs/futures/0.3.1/futures/stream/index.html) for stream docs doesn't currently mention that the `TryStreamExt` trait exists & I only found out about it by asking questions in the [discord chat](https://discord.gg/tokio).

But wait, there is also [StreamExt](https://docs.rs/tokio/0.2.10/tokio/stream/trait.StreamExt.html) from tokio  which is subtly different, but does as of `0.2.10` allow you to run `collect()` on `Result<Bytes, _>` item streams.

When would you use either `StreamExt`?  I would say that if you want to be more general, you should probably use the `futures`  implementation.  Bearing in mind you can't run [`try_collect()`](https://docs.rs/futures/0.3.1/futures/stream/trait.TryStreamExt.html#method.try_collect) on `Result<Bytes, _>` at the moment due to an [outstanding issue](https://github.com/tokio-rs/bytes/issues/324) with the bytes crate.  Why does this matter? Well, [hyper](https://hyper.rs/) passes around streams of `Result<Bytes, _>` when you are streaming a body in & out of a request.  Working with this particular stream signature is a bit clunky still, but I'm sure that this will be resolved in due course.

So we have:

* `StreamExt` from `futures` that works on `Stream` that aren't `Result`
* `TryStreamExt` from `futures` that works on `Stream` that are `Result` but doesn't work well with `Result<Bytes,_>` when trying to `try_collect()`
* `StreamExt` from `tokio` that works with `Result<Bytes,_>` and allows you to `collect()` to get the output but now means you have a dependency on `tokio`

This might end up confusing more than just me, and I do hope that it's simplified in the future.

Luckily in the core of the API I don't need to use them, only for tests & filestream, which means I'm quite happy to depend on the `tokio` implementation.

## Wrapping inner streams in `Pin`

I got tripped up on this for a while, and needed to again ask in chat for the answer here. If you have an inner stream and you want to call `poll_next` on it, you need to wrap it in a `Pin`, otherwise the `Stream` trait does not stick and you will get some gnarly error message with some weird suggestions:

The my original shot at the code was:

```rust
stream.stream.poll_next(cx)
```

But it needs to be:

```rust
Pin::new(&mut stream.stream).poll_next(cx)
```

The compiler error message is not too helpful here, and insists I restrict `S` by `Stream`, even though it is already restricted by `Stream`.

```
error[E0599]: no method named `poll_next` found for type `S` in the current scope
   --> src/lib.rs:282:41
    |
282 |                     match stream.stream.poll_next(cx) {
    |                                         ^^^^^^^^^ method not found in `S`
    |
    = help: items from traits can only be used if the type parameter is bounded by the trait
help: the following trait defines an item `poll_next`, perhaps you need to restrict type parameter `S` with it:
    |
254 | impl<E, S: futures::Stream + Stream> Stream for MultipartRequest<S>
    |         ^^^^^^^^^^^^^^^^^^^^
```


You need to make sure that your inner stream implements `Unpin` as well to go down this path:

```rust
impl<E, S> Stream for MultipartRequest<S>
where
    S: Stream<Item = Result<Bytes, E>> + Unpin,
```

## Storing the results from an `async fn` in a struct

Some of the methods in tokio are returns from an `async fn` , like the [File::open](https://docs.rs/tokio/0.2.10/tokio/fs/struct.File.html#method.open) method, which I use for `FileStream`.  I found the answer on [stack overflow](https://stackoverflow.com/questions/58354633/cannot-use-impl-future-to-store-async-function-in-a-vector) as to how to do this with `std::future`, since `async fn` returns an opaque type.

For the older version, the OpenFuture was a concrete type:

```rust
pub struct FileStream {
    inner: Option<FramedRead<File, BytesCodec>>,
    file: OpenFuture<PathBuf>,
}
```

The newer version is a return from an `async fn` and is opaque, so we wrap it using `Box::pin`:

```rust
pub struct FileStream {
    inner: Option<FramedRead<File, BytesCodec>>,
    file: Pin<Box<dyn Future<Output = Result<File, Error>> + Send + Sync>>,
}
```

You can then instantiate this with:

```rust
Box::pin(File::open(file.into()))
```


Then it's easy to call passing on the context from an existing `poll()`:

```rust
self.file.as_mut().poll(cx)
```

## Notifying a task

`FileStream` has two stages.  One is the future to open the file. The second is streaming out the bytes of the file.  When the future resolves to open the file, I want to notify the context that it should be polled again to start streaming.

The old way of doing this was `task::current().notify()`:

```rust
self.inner = Some(FramedRead::new(file, BytesCodec::new()));
task::current().notify();
```

The new way appears to be using  `cx.waker().wake_by_ref()` where `cx` is the context received from the poll:

```rust
self.inner = Some(FramedRead::new(file, BytesCodec::new()));
cx.waker().wake_by_ref();
```

I say appears because the test written works without `wake_by_ref()` being called.  Requires a bit more investigation here I think to know exactly what's going on.  My [simple test example](https://github.com/cetra3/mpart-async/blob/981ba0437e19fa47f94a913cf9aaa4717fbe12bc/src/filestream.rs#L56) works either way, strangely.


## Conclusion

It is not too onerous to convert to `std::future` for an existing library.  I would assume the leap from the old `tokio-core` would be harder, as the changes *feel* mostly cosmetic here.

The omission of the `Error` associated type to me actually makes things less ergonomic and things a little more fragmented (as evidenced by 3 `*StreamExt` traits..).  I was an advocate of this initially, but there probably needs to be a bit more work in making this nicer.

There are still a lot of libraries out there that will be required to be updated, a lot of old blogs that are no longer relevant, and a lot of exploration that needs to be done to see how the async ecosystem falls out.  But considering the friction of updating is quite small, I am quitely optimistic!
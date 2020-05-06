+++

title = "Lorikeet 0.11.0 - Upgrading to async"
description = "Upgrading lorikeet to async rust"
date = 2020-04-21

[taxonomies]
tags = ["rust", "lorikeet"]
+++

I have just spent some time doing an initial async version of [lorikeet](https://github.com/cetra3/lorikeet) now that the async/await syntax is stable and the ecosystem has caught up.  The major blocker was reqwest, as this is used extensively in the `http` test.

This async version is available now as version `0.11.0`.  You can also install the cli by running `cargo install lorikeet`.

## What is Lorikeet

**Lorikeet** is a command line tool and a rust library to run tests for smoke testing and integration testing.  *Lorikeet* currently supports bash commands and simple HTTP requests along with system information (RAM, Disk & CPU).  There is more information on the [github readme](https://github.com/cetra3/lorikeet) about how to write test files, including how to structure dependent tests and make assertions on the output.

As a simple example, here's a test plan to check to see whether reddit is up, and then tries to login if it is:

```yaml
check_reddit:
  http: https://www.reddit.com
  regex: the front page of the internet

login_to_reddit:
  http: 
    url: https://www.reddit.com/api/login/{{user}}
    form:
      user: {{user}}
      passwd: {{pass}}
      api_type: json
  jmespath: length(json.errors)
  matches: 0
  require:
    - check_reddit
```

You can run this as a simple cli command, ensuring that you have a `config.yml` looks something like:

```yaml
user: test
pass: test
```

Running it you will see the results:

```yaml
$ lorikeet -c config.yml test.yml
- name: check_reddit
  pass: true
  output: the front page of the internet
  duration: 1416.591ms

- name: login_to_reddit
  pass: true
  output: 0
  duration: 1089.0276ms
```

## Skipping futures 0.1

In retrospect, I am glad that I skipped out on rewriting lorikeet with the classic combinator style futures.  The migration from standard blocking code to async code is much easier than writing from combinators.  Having done the former with [mpart-async](../mpart-async-0-3-0/), it felt quite easy to sprinkle in a few `async` and `await` statements to get things all wired up.

As an example, [waiting for the request](https://github.com/cetra3/lorikeet/blob/0.10.0/src/step.rs#L391-L394) used to be:

```rust
let mut response = client
    .execute(request.build().map_err(|err| format!("{:?}", err))?)
    .map_err(|err| format!("Error connecting to url {}", err))?;
```

Now, with async, things are not much different:

```rust
let response = client
    .execute(request.build().map_err(|err| format!("{:?}", err))?)
    .await
    .map_err(|err| format!("Error connecting to url {}", err))?;
```


## Using tokio sync primitives

Tokio has a few primitives that are very close to their std counterparts.

Lorikeet now uses a [tokio mutex](https://docs.rs/tokio/0.2.18/tokio/sync/struct.Mutex.html) to record the outcome of each of the steps:

```rust
steps.lock().await[index] = Status::Completed(outcome);
```

Also using an [unbounded channel](https://docs.rs/tokio/0.2.18/tokio/sync/mpsc/fn.unbounded_channel.html) so each step can inform the main runner when they are finished:

```rust
let (tx, mut rx) = unbounded_channel();

// later on....

if let Some(finished_idx) = rx.recv().await {
  ...
}
```

Interesting to note that the `UnboundedSender` `send()` method is not async and does not block either.

## Multipart File Uploads

The new version of reqwest [does not support file uploads](https://github.com/seanmonstar/reqwest/issues/646) easily.  You don't want to buffer the entire file in memory.  Luckily I have had some experience on how to wire this up with [mpart-async](https://crates.io/crates/mpart-async), and can reuse some of the learnings there.

This does require the [tokio-util](https://crates.io/crates/tokio-util) crate, which bridges `AsyncRead`/`AsyncWrite` with `Sink`/`Stream`.

The relevent glue section [is as follows](https://github.com/cetra3/lorikeet/blob/ef8ba4a0f08e82d4cb33524b9e74cc50beec9078/src/step/http.rs#L157-L167):

```rust
let file_name = path_struct
    .file
    .file_name()
    .map(|val| val.to_string_lossy().to_string())
    .unwrap_or_default();

let file = File::open(&path_struct.file)
    .await
    .map_err(|err| format!("{:?}", err))?;

let reader = Body::wrap_stream(FramedRead::new(file, BytesCodec::new()));

form.part(key, Part::stream(reader).file_name(file_name)
```

## Detecting Blocking Code

There is no annotation or warning whenever you are using blocking code in an async context.  This is an issue especially when porting over older code which was previously blocking.  Chances are I have missed a few sections that are blocking and will need to revisit them accordingly.

Luckily, the fail case for blocking code is usually reduced throughput, down to the number of worker threads, so it is not cataclysmic if there are some blocking sections in your async code.

## Conclusion

I was happy with the ease at which it was to convert existing sections to async by sprinkling a few keywords here and there.  I am concerned that it is too easy to have blocking sections, so I hope there is some tooling/linting around that in the future.

Please give lorikeet a go and try out some of the newer features from [the readme](https://github.com/cetra3/lorikeet)!
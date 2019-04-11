+++

title = "Face Detection with Actix Web"
description = "Using MTCNN with Actix Web"
date = 2019-04-11

[taxonomies]
tags = ["rust"]
+++

Last article I wrote about how to [use tensorflow with rust](https://cetra3.github.io/blog/face-detection-with-tensorflow-rust/).  This time we're going to take what we've built on, and serve it as an HTTP API call.  As [Actix Web](https://github.com/actix/actix-web) is nearing its inevitable [1.0 release](https://github.com/actix/actix-web/issues/722), I thought it would be a great time to build something with it.

This article assumes you have some knowledge about [Futures](https://github.com/rust-lang-nursery/futures-rs) and how they work. I will do my best to explain in simpler terms, but understanding the futures ecosystem would be quite handy to help work through this article. For this, I suggest you start with [tokio](https://tokio.rs/).

*(Some people have suggested waiting for async/await and friends to land before diving into Futures.  I think you should get your hands dirty now: async programming will always be challenging and the changes being introduced only affect ergonomics, not fundamentals)*

Once again for the impatient, you can find the reference code here on the `actix-web` branch: 

[https://github.com/cetra3/mtcnn/tree/actix-web](https://github.com/cetra3/mtcnn/tree/actix-web)

## API Shape

The API shape here is rather simple.  We want to emulate what we've done on the command line: Submit a picture, have a picture returned.  To make things interesting, we'll provide a way to return the bounding boxes as a JSON array too.

For submitting binary data via HTTP, there are a few options off the top of my head:

* Simply submit the raw data as it stands
* Use `multipart/form-data`
* Encode it as a JSON submission

I think the easiest would be just the raw data so let's do that!  Multipart could possibly also work, but what about the case when you have to handle multiple images?  JSON Submission seems a bit wasteful, as you would inevitably have to convert binary data using base64 or similar.

So our API looks like this:

* Submit POST request as a raw file submission
* Run a session against mtcnn to extract faces
* Either return Bounding Boxes as JSON, or an Image Overlay as a JPEG like the command line example

## MTCNN as a struct

In our last blog we just simply used the `main` function to perform everything, but it's not going to fly with actix until we do a bit of a refactor.  We want to encapsulate the mtcnn behaviour as a struct, something that can be passed around and moved.  The end goal is to use it in the application state.

### Struct Shape

Let's make our struct include everything we need:

* The `Graph`
* The `Session`
* Some of the `Tensor` input params that don't change from request to request.


We'll start by creating a new file `mtcnn.rs` and adding in the struct definition.

```rust
use tensorflow::{Graph, Session, Tensor};

pub struct Mtcnn {
    graph: Graph,
    session: Session,
    min_size: Tensor<f32>,
    thresholds: Tensor<f32>,
    factor: Tensor<f32>
}
```

Ok, now we're just going to plop in the initiation stuff in a `new()` method.  As the creation of some of these values is not infallible, we'll return a `Result`:

```rust
pub fn new() -> Result<Self, Box<dyn Error>> {

    let model = include_bytes!("mtcnn.pb");

    let mut graph = Graph::new();
    graph.import_graph_def(&*model, &ImportGraphDefOptions::new())?;

    let session = Session::new(&SessionOptions::new(), &graph)?;

    let min_size = Tensor::new(&[]).with_values(&[40f32])?;
    let thresholds = Tensor::new(&[3]).with_values(&[0.6f32, 0.7f32, 0.7f32])?;
    let factor = Tensor::new(&[]).with_values(&[0.709f32])?;

    Ok(Self {
        graph,
        session,
        min_size,
        thresholds,
        factor
    })

}
```
### The Run Function

*(I'm going to race along here to get to the good parts, so if you get stuck or are unsure about what's going on, have a look at the [original article](https://cetra3.github.io/blog/face-detection-with-tensorflow-rust/) for an explanation of what's happening here.)*

We've added all the bits we need to run our session Let's create a method that does what we're asking the API to do: submit an image, return some bounding boxes:

```rust
pub fn run(&self, img: &DynamicImage) -> Result<Vec<BBoxes>, Status> {
    ...
}
```

Once again, we are returning a `Result` as the type, as there are some parts of the `run` that can fail.  We're going to use the `Status` type as that is normally the error type returned

Like our previous main function, we'll need to flatten the image input:

```rust
let input = {
    let mut flattened: Vec<f32> = Vec::new();

    for (_x, _y, rgb) in img.pixels() {
        flattened.push(rgb[2] as f32);
        flattened.push(rgb[1] as f32);
        flattened.push(rgb[0] as f32);
    }

    Tensor::new(&[img.height() as u64, img.width() as u64, 3])
        .with_values(&flattened)?
};
```

Then we'll feed in all the relevant inputs.  This is just the same as our previous `main` function, but we're going to just borrow values from `self` rather than create them for each run:

```rust
let mut args = SessionRunArgs::new();

args.add_feed(
    &self.graph.operation_by_name_required("min_size")?,
    0,
    &self.min_size,
);
args.add_feed(
    &self.graph.operation_by_name_required("thresholds")?,
    0,
    &self.thresholds,
);
args.add_feed(
    &self.graph.operation_by_name_required("factor")?,
    0,
    &self.factor,
);
args.add_feed(&self.graph.operation_by_name_required("input")?, 0, &input);
```

Next, let's grab the outputs we want:

```rust
let bbox = args.request_fetch(&self.graph.operation_by_name_required("box")?, 0);
let prob = args.request_fetch(&self.graph.operation_by_name_required("prob")?, 0);
```

### Running The Session

Now we have all our args set up, we can run the session:

```rust
&self.session.run(&mut args)?;
```

Uh oh.  We're getting a compiler error here... 

```rust
error[E0596]: cannot borrow `self.session` as mutable, as it is behind a `&` reference
  --> src/mtcnn.rs:68:10
   |
36 |     pub fn run(&self, img: &DynamicImage) -> Result<DynamicImage, Box<dyn Error>> {
   |                ----- help: consider changing this to be a mutable reference: `&mut self`
...
68 |         &self.session.run(&mut args)?;
   |          ^^^^^^^^^^^^ `self` is a `&` reference, so the data it refers to cannot be borrowed as mutable
```

Turns out that the `Session::run()` function takes `&mut self`.

What can we do to resolve this:

* Make our `run` function take `&mut self` so we can mutate the field
* Do some tricky interior mutability
* Submit an [issue on the tensorflow-rust crate](https://github.com/tensorflow/rust/issues/192) to see whether `Session` *really* needs to take `&mut self`.

We'll go with option 3!

Update your `Cargo.toml` to point to the git master, rather than the cargo version:

```
tensorflow = { git = "https://github.com/tensorflow/rust"}
```

### Getting the Bounding Boxes

This has not changed at all since our `main` method.  We grab the bounding boxes, put them in our handy dandy `BBox` struct:

```rust
//Our bounding box extents
let bbox_res: Tensor<f32> = args.fetch(bbox)?;
//Our facial probability
let prob_res: Tensor<f32> = args.fetch(prob)?;

//Let's store the results as a Vec<BBox>
let mut bboxes = Vec::new();

let mut i = 0;
let mut j = 0;

//While we have responses, iterate through
while i < bbox_res.len() {
    //Add in the 4 floats from the `bbox_res` array.
    //Notice the y1, x1, etc.. is ordered differently to our struct definition.
    bboxes.push(BBox {
        y1: bbox_res[i],
        x1: bbox_res[i + 1],
        y2: bbox_res[i + 2],
        x2: bbox_res[i + 3],
        prob: prob_res[j], // Add in the facial probability
    });

    //Step `i` ahead by 4.
    i += 4;
    //Step `i` ahead by 1.
    j += 1;
}

debug!("BBox Length: {}, BBoxes:{:#?}", bboxes.len(), bboxes);

Ok(bboxes)
```

And that's our `run` function finished.

### BBox Struct as JSON

We're gonna want to return a JSON representation of the BBox struct.  So let's add in `serde_derive`:

```rust
use serde_derive::Serialize;

#[derive(Copy, Clone, Debug, Serialize)]
pub struct BBox {
    pub x1: f32,
    pub y1: f32,
    pub x2: f32,
    pub y2: f32,
    pub prob: f32,
}
```

### Drawing the Output Image

We'll create a function that will take a list of bounding boxes and an input image, returning the output image:

```rust
pub fn overlay(img: &DynamicImage, bboxes: &Vec<BBox>) -> DynamicImage
```

This hasn't changed much either, but we are returning the image, rather than saving it to a file:

```rust
//Let's clone the input image
let mut output_image = img.clone();

//Iterate through all bounding boxes
for bbox in bboxes {
    //Create a `Rect` from the bounding box.
    let rect = Rect::at(bbox.x1 as i32, bbox.y1 as i32)
        .of_size((bbox.x2 - bbox.x1) as u32, (bbox.y2 - bbox.y1) as u32);

    //Draw a green line around the bounding box
    draw_hollow_rect_mut(&mut output_image, rect, LINE_COLOUR);
}

output_image
```

Ok, we're pretty much done with our `Mtcnn` struct and functions! Could we break this down even further? Yeah definitely! But for now, I think this is all we need.  We've encapsulated the behaviour and created a nice to use couple of functions.

## Our New Main Function

Ok, so we're no longer using it as a CLI, but as a self-hosted web app.  We're going to need to change the arguments our application takes since we no longer have input and output files.

I think the only argument we should be taking initially is the listen address, and even then we should use a sensible default. So let's use the help of [structopt](https://github.com/TeXitoi/structopt) to make this pretty minimal boilerplate:

```rust
#[derive(StructOpt)]
struct Opt {
    #[structopt(
        short = "l",
        long = "listen",
        help = "Listen Address",
        default_value = "127.0.0.1:8000"
    )]
    listen: String,
}
```

### Logging Framework

Actix Web uses the [log](https://github.com/rust-lang-nursery/log) crate to display errors and debug messages.

Let's move on from using `println!` and instead use log.  I like using [pretty_env_logger](https://github.com/seanmonstar/pretty-env-logger) as it prints different levels as a different colour, and we can have timestamps which is useful.

`pretty_env_logger` still uses environment vars.  Let's cheat and set our `RUST_LOG` var if none is provided then initiate our logger

```rust
//Set the `RUST_LOG` var if none is provided
if env::var("RUST_LOG").is_err() {
    env::set_var("RUST_LOG", "mtcnn=DEBUG,actix_web=DEBUG");
}

//Create a timestamped logger
pretty_env_logger::init_timed();
```

This sets up `DEBUG` level logs for our app and actix web, but allows us to change the log levels via environment variables still.

## Actix and State

We have some state we need to pass to actix to use: our `Mtcnn` struct and the run method.  There are a number of ways you can give State to actix, but probably the easiest way is the `App::data` method.  As we are now entering a multithreaded world, we're going to have to think about what things are `Send` and `Sync`.

Ok, so how do we share something between threads? Well, as a first step I would look to `std::sync` to see what we need.  Since we know that mtcnn's `run` function does not need to mutate it, only requiring a reference to immutable `self`, we can probably wrap it in an `Arc`.  If we had to mutate it, then it would probably require a Mutex as well, but we can avoid this if we use the `master` branch of `tensorflow-rust` as above.

So let's create our Arc:

```rust
let mtcnn = Arc::new(Mtcnn::new()?);
```

Now, to instantiate the server:

```rust
HttpServer::new(move || {
    App::new()
        //Add in our mtcnn struct, we clone the reference for each worker thread
        .data(mtcnn.clone())
        //Add in a logger to see the requests coming through
        .wrap(middleware::Logger::default())
        // Add in some routes here
        .service(
            ...
        )
})
.bind(&opt.listen)? // Use the listener from the command arguments
.run()
```

Ok, let's step through what we're doing:

* We first build a `HttpServer`
* This takes a closure which should return an `App`.  This `App` is instantiated for each thread the http server is running
* We add our `Arc<Mtcnn>` using the `data` method, and clone it for each thread listener.
* We add a logger middleware.
* We set up some routes with the `service` function
* Then we bind to a listen address and run

## Handling a Request

Actix Web is an Async framework and uses tokio.  Our function is Synchronous and takes some time to complete.

In other words, our request is blocking.  Can we mix and match sync and async? Absolutely, but it is a little more cumbersome as you'll see.


### Function Signature and Extractors

Actix 1.0 makes heavy use of Extractors, which is a way of providing functions of quite different shapes.  You basically specify what you want your web function to receive, and actix will wire it up for you.  Take care though:  This does mean that some things won't be discovered wrong until runtime.  A perfect example I had when I used the wrong type signature for the `web::Data` argument.

So what do we need to *extract* from our request? The bytes of the request body, and and also our `mtcnn`:

```rust
fn handle_request(
    stream: web::Payload,
    mtcnn: web::Data<Arc<Mtcnn>>,
) -> impl Future<Item = HttpResponse, Error = ActixError> {

    ...

}
```

We will use this type signature for `mtcnn` a fair bit, so let's make a type alias for it:

```rust
type WebMtcnn = web::Data<Arc<Mtcnn>>;
```

## Getting the Image from a Payload

Ok, so we need a way of retrieving the image from a payload and returning a Future.  The `web::Payload` struct implements [Stream](https://docs.rs/futures/0.1.26/futures/stream/trait.Stream.html) with `Item` set to [Bytes](https://docs.rs/bytes/0.4.12/bytes/struct.Bytes.html).

Well, we don't really care about the individual bytes we get from a stream, we want the whole lot to decode the image!  So let's convert the `Stream` into a `Future`, and merge all the individual bytes we'll be getting into one big bucket of bytes.  Sounds complicated, but luckily Stream has a method: [concat2](https://docs.rs/futures/0.1.26/futures/stream/trait.Stream.html#method.concat2). 

This is a pretty powerful combinator which allows us to join the results of individual Stream polls into one if the item implements [Extend](https://doc.rust-lang.org/nightly/core/iter/trait.Extend.html) (and some other traits) which `Bytes` happens to.


So this looks like:

```rust
stream.concat2().and_then(....)
```


### Image Decoding and `web::block`

Ok, second thing we need to sort out, if we're going to be decoding an image, that is probably going to block the thread until it's done.  How long? Well if it's a huge image, it might take milliseconds!  So we want to ensure that we're not blocking the http workers while this is happening.  Luckily, actix web has a way of taking a blocking portion of code, and wrapping that as a future.

Enter `web::block`.  We'll do our decoding in a cpu pool, bridging sync and async together:

```rust
stream.concat2().and_then(move |bytes| {
    web::block(move || {
        image::load_from_memory(&bytes)
    })
})
```

Ok, that is pretty succint: we take a stream, convert it into a future and a bundle of bytes, then use `web::block` to decode the bytes into an image in a background thread and return the result.  the `load_from_memory` function already returns a Result, which means we can just use that as the return type.

### Balancing the Error Type

So, our Item is converted to `Bytes` to `DynamicImage`, but we still haven't dealt with the error types yet and it won't compile.  What should our error type be?  Let's use `actix_web::Error` as `ActixError`:

```rust
use actix_web::{Error as ActixError}

fn get_image(stream: web::Payload) -> impl Future<Item = DynamicImage, Error = ActixError> {
    stream.concat2().and_then(move |bytes| {
        web::block(move || {
            image::load_from_memory(&bytes)
        })
    })
}
```

Ok, that is giving us few really gnarly errors when we try and compile it:

```
error[E0271]: type mismatch resolving `<impl futures::future::Future as futures::future::IntoFuture>::Error == actix_http::error::PayloadError`
  --> src/main.rs:67:22
   |
67 |     stream.concat2().and_then(move |bytes| {
   |                      ^^^^^^^^ expected enum `actix_threadpool::BlockingError`, found enum `actix_http::error::PayloadError`
   |
   = note: expected type `actix_threadpool::BlockingError<image::image::ImageError>`
              found type `actix_http::error::PayloadError`
```

There's a couple more that aren't listed.

When you are combining streams, mapping them as futures, and otherwise trying to get some output from these combinators you are actually dealing with both the `Item` type, and the `Error` type.

The ergonomics of this are not as advanced as the `Result` type, where we can use the `?` operator to automatically adjust to the right error (if a conversion is available). When both `ops::Try` and `async/await` syntax stabilises, this might be a different story, but for now, we need to deal with error types.

What we have instead is the magic™️ `from_err()` method.  This is basically the same as the `?` operator but for futures.  We have two futures we're dealing with: our bundle of bytes from the stream, and the image from the blocking closure. We have 3 error types we're dealing with: the Payload error, the Image load from memory error, and the blocking error.

Let's start by putting `from_err()` on both of the futures:

```rust
fn get_image(stream: web::Payload)
  -> impl Future<Item = DynamicImage, Error = ActixError> {
    stream.concat2().from_err().and_then(move |bytes| {
        web::block(move || {
            image::load_from_memory(&bytes)
        }).from_err()
    })
}
```

That worked! There is enough conversions for our error types to get to where we need to get it.  The `ActixError` type has a few blanket conversions from common error types enough to be able to make this magic happen.

## Getting the bounding boxes from the image

At the core of it, we need to run the following:

```rust
mtcnn.run(&img)
```

But we want this to run in a threadpool too:

```rust
web::block(|| mtcnn.run(&img))
```

Let's work through the function signature we'll need.  At a minimum we're gonna need the image, and the mtcnn struct.  Then we want to return a `Vec` of BBoxes.  We should try and keep our error types the same, so we'll use the ActixError type.

The function signature looks like this:

```rust
fn get_bboxes(img: DynamicImage, mtcnn: WebMtcnn) 
  -> impl Future<Item = Vec<BBox>, Error = ActixError> 
```

We know that we'll need a `from_err()` on the `web::block` to convert the error type, and we'll have to use a `move` to give the image to the closure:

```rust
fn get_bboxes(img: DynamicImage, mtcnn: WebMtcnn) -> impl Future<Item = Vec<BBox>, Error = ActixError> {
    web::block(move || mtcnn.run(&img)).from_err()
}
```

But does this compile? Not yet:

```
error[E0277]: `*mut tensorflow_sys::TF_Status` cannot be sent between threads safely
  --> src/main.rs:75:5
   |
75 |     web::block(move || mtcnn.run(&img)).from_err()
   |     ^^^^^^^^^^ `*mut tensorflow_sys::TF_Status` cannot be sent between threads safely
   |
   = help: within `tensorflow::Status`, the trait `std::marker::Send` is not implemented for `*mut tensorflow_sys::TF_Status`
   = note: required because it appears within the type `tensorflow::Status`
   = note: required by `actix_web::web::block`
```


`tensorflow::Status`, which is the error type, can't be sent between threads.

Let's just shortcut here, and convert the error to a `String`:

```rust
fn get_bboxes(img: DynamicImage, mtcnn: WebMtcnn) -> impl Future<Item = Vec<BBox>, Error = ActixError> {
    web::block(move || mtcnn.run(&img).map_err(|e| e.to_string())).from_err()
}
```

This allows us to move the `Result` across thread boundaries, as `String` does implement `Send`

## Returning JSON BBoxes

Ok, so we have 2 functions, one to get the image from the request, and one to get the bounding boxes.  We're gonna want to return json `HttpResponse`:

```rust
fn return_bboxes(
    stream: web::Payload,
    mtcnn: WebMtcnn,
) -> impl Future<Item = HttpResponse, Error = ActixError> {
    // Get the image from the input stream
    get_image(stream) 
        // Get the bounding boxes from the image
        .and_then(move |img| get_bboxes(img, mtcnn)) 
        // Map the bounding boxes to a json HttpResponse
        .map(|bboxes| HttpResponse::Ok().json(bboxes))
}
```

Cool! Let's put this in our App definition:

```rust
HttpServer::new(move || {
    App::new()
        .data(mtcnn.clone()) 
        .wrap(middleware::Logger::default()) 
        // our new API service
        .service(web::resource("/api/v1/bboxes").to_async(return_bboxes))
})
.bind(&opt.listen)?
.run()
```


And let's run it, using `curl` to submit a query to get some results back

```
$ curl --data-binary @rustfest.jpg  http://localhost:8000/api/v1/bboxes

[{"x1":471.4591,"y1":287.59888,"x2":495.3053,"y2":317.25327,"prob":0.9999908}....
```

Awesome! Using [jmespath](http://jmespath.org/) to see we get our 120 faces back:

```
$ curl -s --data-binary @rustfest.jpg  http://localhost:8000/api/v1/bboxes | jp "length(@)"
120
```

## Returning an Overlay Image

The other API call we want is to return an image with the bounding boxes overlayed.  This is not much of a stretch, but the act of drawing boxes on an image is definitely a blocking action, so we'll need to do the same thing and send it to a thread pool.

Let's wrap our overlay function, converting it into a Future:


```rust
fn get_overlay(img: DynamicImage, bboxes: Vec<BBox>)
   -> impl Future<Item = Vec<u8>, Error = ActixError> {
    web::block(move || {
        let output_img = overlay(&img, &bboxes);
        
        ...

    }).from_err()
}
```

We're going to want to return a `Vec` of `u8` bytes so we can use this in the return body. So we'll need to allocate some buffer and instruct `image` to write out a JPEG from the image:

```rust
let mut buffer = vec![];

output_img.write_to(&mut buffer, JPEG)?; // write out our buffer

Ok(buffer)
```

Ok, so let's put our little function together and see if it compiles:

```rust
fn get_overlay(img: DynamicImage, bboxes: Vec<BBox>)
  -> impl Future<Item = Vec<u8>, Error = ActixError> {
    web::block(move || {
        let output_img = overlay(&img, &bboxes);

        let mut buffer = Vec::new();

        output_img.write_to(&mut buffer, JPEG)?; // write out our buffer

        Ok(buffer)
    }).from_err()
}
```

Not quite yet: we're missing a type annotation:

```
error[E0282]: type annotations needed
  --> src/main.rs:82:5
   |
82 |     web::block(move || {
   |     ^^^^^^^^^^ cannot infer type for `E`
```

Why is there an issue with the type? Well, it relates to this line here:

```
Ok(buffer) // What's the `Error` type here?
```

At the moment, the only error type is from the `write_to` method which is `ImageError`.  But this line here doesn't have an error type, and could be anything.

There are 3 ways I can immediately think to handle this:

**Way Number 1**: Declare the error type in `web::block`

```rust
web::block::<_,_,ImageError>
```

This looks more like a turbosubmarine than a turbofish! But it compiles!

**Way Number 2**: Declare the Result type with `as` 

```rust
Ok(buffer) as Result<_, ImageError>
```

**Way Number 3**: Use `map` to return the buffer on success:

```rust
output_img.write_to(&mut buffer, JPEG).map(|_| buffer)
```

I think for readability, #2 is probably easiest.  The `web::block` function takes 3 type arguments which can be confusing on first read of the code.  #3 is good too but I think it looks a bit strange.

Our final method looks like:

```rust
fn get_overlay(img: DynamicImage, bboxes: Vec<BBox>)
   -> impl Future<Item = Vec<u8>, Error = ActixError> {
    web::block(move || {
        let output_img = overlay(&img, &bboxes);

        let mut buffer = Vec::new();

        output_img.write_to(&mut buffer, JPEG)?;

        // Type annotations required for the `web::block`
        Ok(buffer) as Result<_, ImageError> 
    }).from_err()
}
```

### The API call

Ok, we have our little futures that we need to return bounding boxes and image overlays.  Let's stitch this together and return a `HttpResponse`:


```rust
fn return_overlay(
    stream: web::Payload,
    mtcnn: WebMtcnn,
) -> impl Future<Item = HttpResponse, Error = ActixError> {
    //... magic happens here
}
```

Ok, first step is to get the image from the stream:

```rust
get_image(stream)
```

And Then once the future has resolved, we want to get the bounding boxes:

```rust
get_image(stream).and_then(move |img| {
    get_bboxes(img, mtcnn)
})
```

### Moving Images Around

Now we want to get the image overlay.  We have an issue though! we give the `get_bboxes` future our image, and it returns a Vec of bboxes, consuming the image.  There are a couple of options here.  We could `clone()` the image when we give it to bboxes, but that is duplicating memory.  We could wait for `Pin` and `async`/`await` to be finished and probably deal with it in an easier way then.

Or we could adjust our `get_bboxes` method to return a tuple of both the image and bounding boxes:

```rust
fn get_bboxes(
    img: DynamicImage,
    mtcnn: WebMtcnn,
) -> impl Future<Item = (DynamicImage, Vec<BBox>), Error = ActixError> {
    web::block(move || {
        mtcnn
            .run(&img)
            .map_err(|e| e.to_string())
            //Return both the image and the bounding boxes
            .map(|bboxes| (img, bboxes))
    })
    .from_err()
}
```

Making sure to update our `return_bboxes` function too:

```rust
fn return_bboxes(
    stream: web::Payload,
    mtcnn: WebMtcnn,
) -> impl Future<Item = HttpResponse, Error = ActixError> {
    get_image(stream)
        .and_then(move |img| get_bboxes(img, mtcnn))
        .map(|(_img, bboxes)| HttpResponse::Ok().json(bboxes))
}
```

### Getting the Overlay

It would be great if rust could desugar a tuple into command arguments.  Unfortunately not for us, so we will need to create a small closure:

```rust
//Create our image overlay
.and_then(|(img, bbox)| get_overlay(img, bbox))
.map(|buffer| {
// Return a `HttpResponse` here
})
```

### Generating the Response

Our `HttpResponse` needs to wrap the buffer into a Http Request with the buffer as the body:

```rust
HttpResponse::with_body(StatusCode::OK, buffer.into())
```

Is that it? Well no, we have to set the content type header to be a jpeg:

```rust
let mut response = HttpResponse::with_body(StatusCode::OK, buffer.into());

response
    .headers_mut()
    .insert(CONTENT_TYPE, HeaderValue::from_static("image/jpeg"));
```

Ok now we can return the result:

```rust
fn return_overlay(
    stream: web::Payload,
    mtcnn: WebMtcnn,
) -> impl Future<Item = HttpResponse, Error = ActixError> {
    get_image(stream)
        .and_then(move |img| {
            get_bboxes(img, mtcnn)
        })
        .and_then(|(img, bbox) | get_overlay(img, bbox))
        .map(|buffer| {
            let mut response = HttpResponse::with_body(StatusCode::OK, buffer.into());
            response
                .headers_mut()
                .insert(CONTENT_TYPE, HeaderValue::from_static("image/jpeg"));
            response
        })
}
```

And add that to our `App` builder:

```rust
HttpServer::new(move || {
    App::new()
        .data(mtcnn.clone()) //Add in our data handler
        //Add in a logger to see the requets coming through
        .wrap(middleware::Logger::default()) 
        //JSON bounding boxes
        .service(web::resource("/api/v1/bboxes").to_async(return_bboxes))
        //Image overlay
        .service(web::resource("/api/v1/overlay").to_async(return_overlay))
}
```

Great! Let's run it:

```
$ curl --data-binary @rustfest.jpg  http://localhost:8000/api/v1/bboxes > output.jpg
```

And we have our original overlay!

[![](/photos/rustfest_faces.jpg)](/photos/rustfest_faces.jpg)


## Conclusions

We stepped through converting a CLI app into a HTTP service, dipping our toes into the brave new async world.

As you can see, [actix web](https://github.com/actix/actix-web/) is a very versatile web framework.  My interest in it was borne out of having all the features I need to build up web apps: multipart, thread pools, great efficiency.

While it is hard to bridge the sync and async gap, it's not impossible.  It would be great if there were some more ergonomic ways to do so, as I think a lot of developers struggle with this: I have seen a lot of questions around integrating with diesel and friends.

If you are looking for more actix web examples, the evergrowing examples repo is your best bet:

[https://github.com/actix/examples](https://github.com/actix/examples)

I look forward to seeing what the community builds in the future!
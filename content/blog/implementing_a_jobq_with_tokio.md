+++

title = "Implementing a Job queue with Tokio-Serde"
description = "An async background job queue written with tokio-serde"
date = 2020-05-13

[taxonomies]
tags = ["rust", "async"]
+++

In the [last blog](../implementing-a-jobq/) of this series, I implemented job queue with [tmq](https://github.com/cetra3/tmq).  I noted back then that **tmq** is great if you need to interact with other languages, but may be a little overkill if you are just using rust.  I wondered what it'd take to build the job queue with a smaller library footprint, using something like [tokio-serde](https://github.com/carllerche/tokio-serde) instead of [tmq](https://github.com/cetra3/tmq).  It was successful, and this blog will step through some of the changes needed.

To follow along, these changes are on the `tokio_serde` branch of the [jobq repo](https://github.com/cetra3/jobq/tree/tokio_serde).

If you are getting lost make sure you [review the last blog](../implementing-a-jobq/) to see some of the logic & design tradeoffs.

## What is Tokio-Serde

**Tokio-Serde** is a little glue library that allows you to serialise/deserialise objects on the wire. With the help of [tokio-util's codec feature](https://docs.rs/tokio-util/0.3.1/tokio_util/codec/index.html), you can use *Tokio-Serde* to encode/decode bytes into discrete messages to be passed on the wire.

From the *Tokio-Serde* [documentation](https://docs.rs/tokio-serde/) using tokio-serde is 3 layers deep:

* `tokio::net::TcpStream`: The raw Stream/Sink where we can get/send bytes
* `tokio_util::codec::Framed`: The ability to chunk bytes into discrete frames
* `tokio_serde::Framed`: The ability to take these frames & then serialize/deserialize them


### The TCP Stream

The TCP stream is your standard tcp connection.  In tokio, you can either `bind` or `connect` to a TCP socket.

With `connect`, you will get a straight tcp stream, if successful:

```rust
let tcp_stream = TcpStream::connect("127.0.0.1:17653").await?;
```

With `bind` you will get a listener, which will produce tcp streams when new connections come in:

```rust
let mut listener = TcpListener::bind("127.0.0.1:17653").await?;

while let Some(tcp_stream) = listener.try_next().await? {
    //do something with the tcp stream
}
```

### The Frame

Ok so we have bytes coming across the wire now from our standard TCP stream.  As bytes come in, the tcp stream will be woken up so work can commence on it.  But TCP streams do not have a beginning or end, they are just a stream of bytes. We need something to *break* up this stream of bytes into discrete objects.  This is where `tokio_util::codec::Framed` comes in: it chunks down bytes on the wire into frames.

The `tokio_util` crate comes with the [Encoder](https://docs.rs/tokio-util/0.3.1/tokio_util/codec/trait.Encoder.html)/[Decoder](https://docs.rs/tokio-util/0.3.1/tokio_util/codec/trait.Decoder.html) traits which allow you to implement this.  `tokio-util` also comes with a simple, [Length Delimited Codec](https://docs.rs/tokio-util/0.3.1/tokio_util/codec/length_delimited/index.html) which, on the wire looks like this:

```
+----------+--------------------------------+
| length   |          payload               |
+----------+--------------------------------+
```

When decoding, this will read the length header, allocate some memory and wait until the payload is fully delivered.  If there are not enough bytes on the wire to do this, then it will be continuously polled until it is ready, keeping around a buffer. The semantics of `Decoder` mean that if you return no bytes, then it's an indication there needs to be more to decode a frame successfully.

When encoding, the whole thing is ready to go, so tack on the length, your payload and let tokio handle the rest.

### The Serde Frame

We now have some raw bytes that have been bundled into frames which can be encoded/decoded from the raw bytes into something more meaningful.  Rather than implement this by hand, we can rely on `tokio-serde` with one of the existing codecs, such as JSON, CBOR, Bincode or Messagepack.

Our existing job queue uses CBOR so we'll use that here as well.  We can use the same messages [we have previously used](https://github.com/cetra3/jobq/blob/master/src/lib.rs) but we will condense down the Client/Worker to just [`ClientMessage`](https://github.com/cetra3/jobq/blob/518c6c025a9e10477819fcc87acf8747383e9ba3/src/lib.rs#L21-L25):

```rust
#[derive(Serialize, Deserialize, Debug)]
pub enum ServerMessage {
    Hello(String), // Hello with client name
    Request(JobRequest),
    Completed(Job),
    Failed(Job, String),
}

#[derive(Serialize, Deserialize, Debug)]
pub enum ClientMessage {
    Hello(String),
    Order(Job),
    Acknowledged(Job),
}
```

Putting these 3 layers together, we get a `Stream`/`Sink` we can send values to:

```rust
let tcp_stream = TcpStream::connect(addr).await?;

let length_delimited = tokio_util::codec::Framed::new(tcp_stream, LengthDelimitedCodec::new());

let connection = tokio_serde::Framed::new(length_delimited, Cbor::default());
```

With that we have our transport that we want: length delimited cbor frames.  On the wire this is not too different to ZeroMQ, but would probably shave a few bytes here and there.  Now we need to implement the Server/Client components.

## Implementing a Server

We want to accept connections from clients/workers and then route out messages to the appropriate connection.  We will leave the server logic itself *mostly* untouched, and instead replace the `tmq::router` with a new [`Router`](https://github.com/cetra3/jobq/blob/tokio_serde/src/router.rs)  struct which uses `tokio-serde`

### The Router

The [`Router`](https://github.com/cetra3/jobq/blob/tokio_serde/src/router.rs) struct handles incoming client connections and allows our server logic to send messages based upon their name.

We have multiple *clients/workers* that will be connecting to our jobq server so we will need to keep around some state to handle those connections.  How will we do that? Well, a simple `HashMap<String, Connection>` is probably a good start.  When a connection comes in, we want to add it to our map of known clients.

If a connection drops off (i.e, returns `Poll::Ready(None)` from the `Stream`) then we can clean it from our known connections. If a new client with the same name comes in, we can just drop off the old client connection (alternatively, we could reject the new one).  These semantics are obviously different from the [ZeroMQ Router socket](http://api.zeromq.org/4-3:zmq-socket#toc25), and not nearly as robust, but are good enough for a PoC.

The router can be implemented as a `Stream` which, when polled will check:

* Any pending tcp connections
* Any pending clients that are in the middle of handshaking
* Any pending messages from existing clients

This can be seen as a loop, whereby each time we hit `poll_next` we will hit all 3 areas in case we have anything waiting for us.  If there is nothing in one area, then we move onto the next one, and so on.  We can do an `if let Poll::Ready()` pattern here to skip things that aren't ready.

#### Pending Clients

How do we know the client name when connecting?  Well we can implement a list of pending clients.  When they send a `ServerMessage::Hello`  message, we can then put them in our known clients list.  This is a pretty lightweight handshake, but accomplishes our goals.

First, we convert the tcp stream from listener into a list of pending clients:

```rust
let self_mut = &mut self.as_mut();

if let Poll::Ready(val) = Pin::new(&mut self_mut.listener).poll_next(cx) {
    match val {
        Some(Ok(tcp_stream)) => {
            let length_delimited =
                CodecFramed::new(tcp_stream, LengthDelimitedCodec::new());

            let framed = Framed::new(length_delimited, Cbor::default());

            self_mut.pending_clients.push(framed);
        }
        Some(Err(err)) => {
            error!("Error checking for new requests:{:?}", err);
        }
        None => {
            return Poll::Ready(None);
        }
    }
}
```

Now, with that checked off, we can then consult our pending clients.  Note here I am swapping out the pending clients `Vec` with an empty `Vec` to avoid doubly borrowing as mutable.  There may be an easier/more efficient way of doing this and would be interested in knowing what that may look like.

We will repopulate the list as we go through, so we won't lose any pending clients, but one or two may drop off if they are not doing the handshake correctly or they have already disconnected.  We'll also add the `Hello` message to our messages to send as a `Stream` response.  That's our `buffer` that we will be using.

```rust
let mut new_pending = Vec::new();
mem::swap(&mut self_mut.pending_clients, &mut new_pending);

for mut pending in new_pending.into_iter() {
    if let Poll::Ready(val) = Pin::new(&mut pending).poll_next(cx) {
        match val {
            Some(Ok(ServerMessage::Hello(name))) => {
                debug!("New Client connection from `{}`", name);
                self_mut.buffer.push(ServerMessage::Hello(name.clone()));
                self_mut.clients.insert(name, pending);
            }
            Some(Ok(msg)) => {
                warn!("Received unknown message during handshake:{:?}", msg);
            }
            Some(Err(err)) => {
                error!("Error checking for new requests:{:?}", err);
            }
            None => (),
        }
    } else {
        self_mut.pending_clients.push(pending);
    }
}
```
#### Existing Clients

We'll do the same *trick* for the client `HashMap`: swap it out for an empty one as we're processing.

We'll put all received messages into an internal buffer, which we will use later to send the actual response from the stream.

```rust
let mut new_clients = HashMap::new();
mem::swap(&mut self_mut.clients, &mut new_clients);

for (name, mut client) in new_clients.into_iter() {
    match Pin::new(&mut client).poll_next(cx) {
        Poll::Ready(Some(Ok(val))) => {
            trace!("Received message from `{}`: {:?}", name, val);
            self_mut.buffer.push(val);
            self_mut.clients.insert(name, client);
        }
        Poll::Ready(None) => {
            //Finished
            debug!("Client `{}` disconnecting", name);
        }
        Poll::Ready(Some(Err(err))) => {
            //Error
            error!("Error from `{}`: {} Removing connection.", name, err);
        }
        Poll::Pending => {
            self_mut.clients.insert(name, client);
        }
    }
}
```

#### Returning Messages


We've buffered up the returned messages into a `Vec` and we can return one.  But if there are more after popping we can let the waker know we may have more stuff to process:

```rust
if let Some(val) = self_mut.buffer.pop() {
    if self_mut.buffer.len() > 0 {
        cx.waker().wake_by_ref();
    }
    return Poll::Ready(Some(Ok(val)));
}

return Poll::Pending;
```

### Sending Messages

Sending messages requires that you know the name of the client, but it is otherwise a simple wrapper:

```rust
pub async fn send_message(&mut self, client: &str, msg: ClientMessage) -> Result<(), Error> {
    if let Some(connection) = self.clients.get_mut(client) {
        connection.send(msg).await?;
    } else {
        return Err(anyhow!("Client `{}` not connected!", client));
    }

    Ok(())
}
```

We can even expose a way to check if any clients are connected of that name, and simply not bother sending a message in this case:

```rust
pub fn is_connected(&self, client: &str) -> bool {
    self.clients.contains_key(client)
}
```

### Changes to the Server

The `Server` struct itself does not really change all that much.  Instead of using a `tmq::router` we'll use our new `Router`.  Although strictly not equivalent, for the purpose of the job queue, this is an OK analogue:

```rust
let mut router = Router::new(&self.job_address).await?;

while let Some(server_msg) = router.next().await {
    // existing server logic here
}
```

## Implementing Clients

Not much has changed here as well.  We could get away with not using our own struct definition here at all and just set up a connection, but would be great if we had something we could plop in where the old `tmq::dealer` was used.

### Dealer

We implement a `Dealer` struct which simply wraps a `tokio_serde::Framed`:

```rust
type ClientFramed = Framed<
    CodecFramed<TcpStream, LengthDelimitedCodec>,
    ClientMessage,
    ServerMessage,
    Cbor<ClientMessage, ServerMessage>,
>;

pub struct Dealer {
    connection: ClientFramed,
}

impl Dealer {
    pub async fn new<A: ToSocketAddrs>(addr: A) -> Result<Self, Error> {
        let tcp_stream = TcpStream::connect(addr).await?;

        let length_delimited = CodecFramed::new(tcp_stream, LengthDelimitedCodec::new());

        let connection = Framed::new(length_delimited, Cbor::default());

        Ok(Self { connection })
    }
}
```

With the dealer, both receiving and sending marry up with `Stream` and `Sink` quite well, so we can implement that quite easily:

```rust
impl Stream for Dealer {
    type Item = Result<ClientMessage, Error>;

    fn poll_next(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Option<Self::Item>> {
        let self_mut = &mut self.as_mut();

        match Pin::new(&mut self_mut.connection).poll_next(cx) {
            Poll::Ready(Some(val)) => Poll::Ready(Some(val.map_err(|err| err.into()))),
            Poll::Ready(None) => Poll::Ready(None),
            Poll::Pending => Poll::Pending,
        }
    }
}
```


### Worker Changes

The worker doesn't change much, we [call split](https://github.com/cetra3/jobq/blob/518c6c025a9e10477819fcc87acf8747383e9ba3/src/worker.rs#L19) on the `Dealer` struct to get our two pipes, but otherwise treat it as per normal:

```rust
let (mut send_skt, recv) = Dealer::new(&job_address).await?.split();
```

## Running the Changes

Running the branch we should see a lot of the same chatter we saw from the [previous version](../implementing-a-jobq/):

```
 2020-05-13 11:13:02 DEBUG jobq::server > New: Job { id: 5500, username: "test_client", name: "test", uuid: 95182793-7a08-4329-a9f1-4be0c1f1d041, params: Null, priority: Normal, status: Queued }
 2020-05-13 11:13:02 WARN  jobq::server > Job failed: 5496, Reason: Simulating failure
 2020-05-13 11:13:02 WARN  jobq::server > Job failed: 5484, Reason: Simulating failure
 2020-05-13 11:13:02 WARN  jobq::server > Job failed: 5472, Reason: Simulating failure
 2020-05-13 11:13:03 WARN  jobq::server > Job failed: 5460, Reason: Simulating failure
```

## Conclusion

Implementing a more barebones messaging system native in rust took not much time at all.  While this solution is not nearly as robust as using ZeroMQ, it does provide a more building blocks approach to messaging, reducing external dependencies.  I found the hardest portion was gluing everything together between the 3 main crates and getting the types right.  *Tokio-Serde* could use a bit more fleshing out and does seem quite embryonic still, but it is enough to get your hands dirty.

The initial design of the jobq made it easy to slot in another transport layer as well.  All the structs & types were all ready and rolling to go, we just needed a couple of extra libraries to fill in the pieces.
+++

title = "Implementing a Job queue with Tokio, PostgreSQL & ZeroMQ"
description = "An async background job queue written in rust"
date = 2020-04-09

[taxonomies]
tags = ["rust", "tmq"]
+++

One of the challenges I have had with on premise solutions is the lack of reliable environments and constrained resources.   Not only are you limited in your ability to control things, you need to ensure that things continue on in the case of failure.

I was tasked with rewriting the job processing pipeline for my company's product, [SchoolBench](https://www.schoolbench.com.au/), to ensure a greater level of robustness in the case of service or system failure.

This article steps through an approach to this using async rust & the help of tokio ZeroMQ library: [tmq](https://github.com/cetra3/tmq) along with [tokio-postgres](https://crates.io/crates/tokio-postgres).

As some excellent work has been put in bringing tmq to async, I thought it pertinent to look at reapproaching the pipeline using async code.  I have published an [initial prototype](https://github.com/cetra3/jobq) of what the end result may look like, and will be stepping through the approach here, and some of the tradeoffs I made

## Why TMQ/ZeroMQ?

For those not familiar with [ZeroMQ](https://zeromq.org/), it is a messaging library with a lot of features, from standard publish/subscribe to more elaborate arrangements.  [tmq](https://github.com/cetra3/tmq) is a [tokio](https://www.tokio.rs) compatible binding that allows you to bridge ZeroMQ with the async world of rust.

You would use ZeroMQ if you need to interact with other languages easily and don't want it to be too opinionated on what message formats are being sent.  If you are purely using rust then this may be not be a great fit, but it does still provide you some great interprocess & internetwork capabilities that you won't get from just tokio.

We use ZeroMQ in SchoolBench as it is a polyglot application and has a lot of components written in python & java as well as rust.  It is lightweight enough not to cause overheads & complex enough for the use cases I have thrown at it so far.

## Prior Work

There are some existing rust based job queues that do similar things and may be more suited to your own personal tastes:

* [https://github.com/kureuil/batch-rs](https://github.com/kureuil/batch-rs)
* [https://github.com/badboy/oppgave](https://github.com/badboy/oppgave)
* [https://github.com/rusty-celery/rusty-celery](https://github.com/rusty-celery/rusty-celery)
* Probably lots more I haven't found

Most of them use some sort of message broker, such as [RabbitMQ](https://www.rabbitmq.com/) or [Redis](https://redis.io/).

## What is a Job?

It helps to explain what a job is.  In the context of SchoolBench a job is a piece of work that may take some time to do (think seconds), is quite intensive, and may fail. Once an asset is saved into SchoolBench, lots of processes are kicked off in the background to *fill in the blanks* and provide information about the asset.

Some examples of jobs are:

* Running [Facial Recognition](../face-detection-with-tensorflow-rust/) on images using tensorflow
* Generating Thumbnails and Watermarked renditions of images
* Calculating whether a [photo is a duplicate](https://github.com/cetra3/dhash) of another photo and marking the set of duplicates

A lot of these are quite intensive & can sometimes fail in weird and wonderful ways:  An image uploaded may be corrupted, so renditions may not run;  There may be an issue running the neural net;  The VM may be experience ballooning and fail to allocate memory, etc...

So we need a way of describing a discrete piece of work which may happen some time in the future and may happen by a completely isolated process.

### Resource Contention

We have very limited resources in an on premise environment. After providing a minimum recommendation it is important to tune for that worst case.  This is different in a cloud environment where you can spin up guests in a work stealing fashion if you throw more money at your infrastructure. on premise is usually more fixed in terms of server and hardware allocations, and so we need to plan for that.

This is one area where the job queue deviates from a lot of existing systems out there: we assume we're just running on the one set of CPUs and so there is a fixed number of active jobs at any given time.

We also want a simple way to prioritise tasks as they are submitted, so that some jobs finish first, with higher value jobs such as thumbnail generation happening first. The job queue uses a simple 3-tier priority system to do this: High, Medium & Low.

This simple system could mean if there are not enough resources to finish jobs then there is a chance that lower tier jobs may never get around to completing.  In practice the sort of work loads we have seen this is not the case, but it is a possibility. In which case more processors would be needed to keep up with the load or another style of priority to be implemented (such as fair queueing).

### Persistence

Jobs need to survive crashes and server restarts in a robust fashion.  Existing job queues utilise an existing persistence layer/message broker such as [Redis](https://redis.io/) or [RabbitMQ](https://www.rabbitmq.com/).  I have chosen PostgreSQL as the persistence layer as it is performant enough, and is already in use for metadata storage.  However, you could easily swap out another type of persistence with a bit of refactoring.

## The Overall Process

The high level proces is as follows:

* A User Action triggers some sort of logic which requests a job.  This could be an automatic rule somewhere or could be the a user specifically requesting a job to run
* A server listens for requests and puts them into a queue
* If there are jobs that can be processed then they are submitted to the individual worker who is responsible for finishing the job
* When a job completes or fails, then it is marked as so with a duration on how long things take
* Everytime a new event is triggered, such as a job in a queue or a job completed/failed, the server checks to see if it can mark jobs as processing
* All job status updates & information is persisted to a db, so that things can pick up again if there is a failure & reports can be ran using standard SQL tools

### Client, Server, Worker

We split the roles of the job queue into 3 different types:

* **Clients** that submit requests to the server
* The **Server** which does the main job queue loop
* The **Workers** that receive jobs and do work, returning whether the work was completed *(not the work outcome itself, just the status; we assume that the worker will update something out of band like another db or filesystem)*

One possible thing to note is that there is only ever one server, but there can be as many workers and clients as allowed.  The workers themselves can also be clients and submit requests if they want to.

### Communication

The server, worker and clients all communicate via a ZeroMQ `DEALER/ROUTER` socket.

Clients and Workers use the `DEALER` style socket to retrieve & send messages, but also identify themselves, and the server uses `ROUTER`. The identity is included as the first ZeroMQ multipart frame when it is received and so can be used to route messages back to the appropriate place.

The messages are serialised on the wire as [CBOR](https://cbor.io/) using [serde_cbor](https://crates.io/crates/serde_cbor).  CBOR was chosen because it allows for the message type to be flexible & there are plenty of implementations in other languages.  JSON could be used as well, as arguably the support for JSON is much higher, but would mean larger message sizes. 

With serde it's pretty easy to adjust what serialisation is used, so some experimentation could be worthwhile.

## The Job Structs

The [`Job` struct](https://github.com/cetra3/jobq/blob/a09ebcaff164c2153cdceba0031dbfb7caa1ea3b/src/lib.rs#L42-L50) is defined as follows (with the `Status` & `Priority` enums listed too):

```rust
pub struct Job {
    pub id: i64, // ID of the Job so it can be tracked
    pub username: String, // Username of who submitted the job
    pub name: String, // Name of the job/worker, `rendition`, etc..
    pub uuid: Uuid, // Unique ID of the asset
    pub params: Value,// Any parameters of the job in question
    pub priority: Priority, // The priority of the job
    pub status: Status,// The status of the job
}
```

The job has a current status which starts with `Queued`, goes to `Processing` when it is active and then marked as either `Completed` or `Failed`:

```rust
pub enum Status {
    Queued,
    Processing,
    Completed,
    Failed,
}
```

The priority is a simple 3-tier system of `High`, `Normal` & `Low`:

```rust
pub enum Priority {
    High,
    Normal,
    Low,
}
```

### Message types

Messages are split up based upon who the destination is with a shared `Hello` message that can be serialised as any enum:

```rust
#[derive(Serialize, Deserialize, Debug)]
pub enum ServerMessage {
    Hello,
    Request(JobRequest),
    Completed(Job),
    Failed(Job, String),
}

#[derive(Serialize, Deserialize, Debug)]
pub enum ClientMessage {
    Hello,
    Acknowledged(Job),
}

#[derive(Serialize, Deserialize, Debug)]
pub enum WorkerMessage {
    Hello,
    Order(Job),
}
```

The  `JobRequest` is very similar to a `Job` but does not have an allocated id or status yet:

```rust
pub struct JobRequest {
    pub name: String,
    pub username: String,
    pub uuid: Uuid,
    pub params: Value,
    pub priority: Priority,
}
```

### The Params Value

One thing to note is the `params` is of the type `serde_json::Value`.  This is really to allow the greatest flexibility into what parameters are sent with each job & also to have it persist to PostgreSQL as a `jsonb` column.

Alternatively, a tighter `enum` could be used for different job names if you knew ahead of time the jobs that will be requested.

### Serialisation helpers

To reduce some of the repetitiveness, there is a serialisation helper which simply serialises to `CBOR` for any struct that implements `Serialize`:

```rust
pub trait ToMpart {
    fn to_mpart(&self) -> Result<Multipart, Error>;

    fn to_msg(&self) -> Result<Message, Error>;
}

impl<T: serde::ser::Serialize> ToMpart for T {
    fn to_mpart(&self) -> Result<Multipart, Error> {
        let bytes = serde_cbor::to_vec(&self)?;

        Ok(Multipart::from(vec![&bytes]))
    }

    fn to_msg(&self) -> Result<Message, Error> {
        let bytes = serde_cbor::to_vec(&self)?;

        Ok(Message::from(&bytes))
    }
}
```

## Client Requests

Clients can send requests by constructing job requests and sending them to the server.  They just need to construct the `Request` enum and send it on the wire:

```rust
let (mut send, mut recv) = dealer(&Context::new())
    .set_identity(b"test_client")
    .connect(&config.job_address)?
    .split::<Multipart>();

let job = JobRequest {
    name: "test".into(),
    username: "test_client".into(),
    params: Value::Null,
    uuid: Uuid::new_v4(),
    priority: PriorityLow,
};

send.send(ServerMessage::Request(job).to_mpart()?).await?;
```

## Worker Processing

For workers doing the work, they should do something & then return whether it's failed or completed.

To make this easier, a `Worker` trait is implemented.  Since some work will be async & some not, we want our trait to be async-capable.  At the writing of this article you can't have `async` in trait definitions, but there is the [async_trait](https://crates.io/crates/async-trait) crate that allows you to decorate a trait definition and do what we're after.

For errors we're going to use the [anyhow](https://crates.io/crates/anyhow) crate, but a more generic error approach could be used with a bit of a refactor.

```rust
#[async_trait]
pub trait Worker: Sized {
    const JOB_NAME: &'static str;

    async fn process(&self, job: Job) -> Result<(), Error>
}
```

The implementation of this trait means that you can do lots within the `process` body and simplifies it.

An original version had the `process` method accept `&mut self`, but that caused some contention when trying to process jobs concurrently.  Instead you'll need some interior mutability if you need to provide any mutable references to `&self`.

As an [example](https://github.com/cetra3/jobq/blob/2028ea12388bd077056d984016ecf46c88e6626c/src/worker.rs#L82-L94), the test worker will simply wait for 100 milliseconds and then fail every 12th job based upon the id:

```rust
impl Worker for TestWorker {
    const JOB_NAME: &'static str = "test";

    async fn process(&self, job: Job) -> Result<(), Error> {
        delay_for(Duration::from_millis(100)).await;
        if job.id % 12 == 0 {
            return Err(anyhow!("Simulating failure"));
        }

        Ok(())
    }
}
```

### The Work Method

There is another method, `work`, [on the trait](https://github.com/cetra3/jobq/blob/2028ea12388bd077056d984016ecf46c88e6626c/src/worker.rs#L16) which sets up & hides some of the complexity away of getting a worker to listen for requests coming through.   This is plumbing to make it easy to run a `Worker`.

```rust
async fn work(&self, job_address: &str) -> Result<(), Error> {
    ...
} 
```

You can `await` this method for anything that implements `Worker` to have it process incoming requests:

```rust
use jobq::worker::{Worker, TestWorker};

tokio::spawn(async move {
    if let Err(err) = TestWorker.work(&worker_config.job_address).await {
        error!("{}", err);
    }
});
```

Under the hood it uses the job address to communicate with a Server and glues up some requests

Firstly, [it creates a dealer socket](https://github.com/cetra3/jobq/blob/2028ea12388bd077056d984016ecf46c88e6626c/src/worker.rs#L20-L23), setting the identity accordingly:

```rust
let (mut send_skt, recv) = dealer(&Context::new())
    .set_identity(job_name.as_bytes())
    .connect(&job_address)?
    .split::<Multipart>();
```

As ZeroMQ sockets are `Send` but not `Sync`, we spin up a background task connected by an `unbounded` queue which allows the sender to be cloned.  This allows both heartbeats to be sent every 10 seconds from a background task & also send results of processing:

```rust
let (send, mut recv_skt) = unbounded::<ServerMessage>();

tokio::spawn(async move {
    while let Some(jobq_message) = recv_skt.next().await {
        if let Ok(msg) = jobq_message.to_mpart() {
            if let Err(err) = send_skt.send(msg).await {
                error!("Error sending message:{}", err);
            }
        }
    }
});
```

The [10 second heartbeat](https://github.com/cetra3/jobq/blob/2028ea12388bd077056d984016ecf46c88e6626c/src/worker.rs#L39-L46) is similiarly set up as a separate task:

```rust
tokio::spawn(async move {
    loop {
        if let Err(err) = ping_sender.send(ServerMessage::Hello).await {
            error!("Error:{}", err);
        };
        delay_for(Duration::from_millis(10000)).await;
    }
});
```

There is then a [big combinator statement](https://github.com/cetra3/jobq/blob/2028ea12388bd077056d984016ecf46c88e6626c/src/worker.rs#L48-L75) which in effect listens for job requests & then runs them in parallel:

```rust
recv.filter_map(|val| {
    match val
        .map_err(Error::from)
        .and_then(|msg| serde_cbor::from_slice(&msg[0]).map_err(Error::from))
    {
        Ok(WorkerMessage::Order(job)) => return ready(Some(job)),
        Ok(WorkerMessage::Hello) => {
            debug!("Pong: {}", job_type);
        }
        Err(err) => {
            error!("Error decoding message:{}", err);
        }
    }

    return ready(None);
})
.map(|job| (self.process(job.clone()), send.clone(), job))
.for_each_concurrent(None, |(status, mut send, job)| async move {
    let server_message = match status.await {
        Ok(()) => ServerMessage::Completed(job),
        Err(err) => ServerMessage::Failed(job, err.to_string()),
    };

    if let Err(err) = send.send(server_message).await {
        error!("Error sending server message: {}", err);
    }
})
.await;
```

There is a bit of cloning here that could probably be avoided, but the overhead of cloning a job is not that big.

The [`for_each_concurrent`](https://docs.rs/futures/0.3.4/futures/stream/trait.StreamExt.html#method.for_each_concurrent) allows the worker to run concurrently, and is one of the reasons the trait takes `&self` rather than `&mut self`.

All of this is essentially generic on the `Worker` trait, meaning this plumbing is handled for you when you call `work(&address)`.

### Blocking Tasks

Some tasks may be blocking, which is a no-no in the async world.  If one of the async worker threads is blocked then it can't listen for more events & will slow things down.  Luckily tokio does provide the [`spawn_blocking`](https://docs.rs/tokio/0.2.18/tokio/task/fn.spawn_blocking.html) helper for executing blocking work on a dedicated threadpool:

```rust
let result =
   tokio::task::spawn_blocking(move || do_blocking_work(job));

result.await??;
```

The double `.await??` is because the `JoinHandle` returned from `spawn_blocking` itself could fail and the internal result from the closure could fail.

To control how many threads are used for blocking operations, you need to manually [build your tokio run time](https://docs.rs/tokio/0.2.18/tokio/runtime/struct.Builder.html), rather than use the `tokio::main` macro.

The the number of blocking threads is `max_threads - core_threads`.  For instance, if you wanted only 2 dedicated blocking threads:

```rust
let mut rt = tokio::runtime::Builder::new()
    .threaded_scheduler()
    .enable_all()
    .core_threads(4)
    .max_threads(6)
    .build()?
```

## The Server Loop

The [server loop](https://github.com/cetra3/jobq/blob/master/src/server.rs) is the main loop responsible for coordinating tasks & otherwise tasks are finished in the order they are supposed to.  The server here responds to messages from the ZeroMQ socket & acts according to the message received.

### Setup Phase

The [setup phase](https://github.com/cetra3/jobq/blob/2028ea12388bd077056d984016ecf46c88e6626c/src/server.rs#L26-L36) involves setting up a db handle, then querying the database for any outstanding processing jobs.  This is to ensure that they don't become stuck as part of a server restart, but can mean that there may be duplicate jobs submitted.  If your jobs provide the same outcome based upon the job at hand this isn't a big deal, so the workers should do work that is idempotent if at all possible.

Once that is done, the server keeps an active count of jobs & increments/decrements them when jobs finish/fail or start.

### Main Loop

The [main loop](https://github.com/cetra3/jobq/blob/2028ea12388bd077056d984016ecf46c88e6626c/src/server.rs#L38-L119) simply waits until a message is received, and acts accordingly:

```rust
while let Some(msg) = recv.try_next().await? {

    // This is the `ROUTER` socket identity
    let client_name = &msg[0];
    // This is the message
    let server_msg = serde_cbor::from_slice::<ServerMessage>(&msg[1]);
    
    match server_msg {
        ...
    }

}
```

#### Hello Message

If a `Hello` message is received then the server will send any outstanding processing jobs it knows about:

```rust
//Drain out existing processing jobs
let (jobs, outstanding): (Vec<Job>, Vec<Job>) =
    processing.into_iter().partition(|job| job.name == name);

processing = outstanding;

for job in jobs {
    send_job(&handle, job, &mut send).await?;
}
```

The `processing` vec is only ever populated at [server start](https://github.com/cetra3/jobq/blob/2028ea12388bd077056d984016ecf46c88e6626c/src/server.rs#L34), and so should mostly be empty unless the server was restarted with jobs that were active.

After checking all this it responds back with a `Hello`

#### Job Request

If a `Request` message comes in, the [server adds this to the db](https://github.com/cetra3/jobq/blob/2028ea12388bd077056d984016ecf46c88e6626c/src/server.rs#L70) & then sends an `Acknowledged` message [back to the client](https://github.com/cetra3/jobq/blob/2028ea12388bd077056d984016ecf46c88e6626c/src/server.rs#L84-L91) with the populated details.  This doesn't yet start the job however, merely it sets the job in a `Queued` state.

#### Completed

If a `Completed` message comes in then the [server marks it as completed](https://github.com/cetra3/jobq/blob/2028ea12388bd077056d984016ecf46c88e6626c/src/server.rs#L93-L97) decrementing the active job count.

#### Failed

If a `Failed` message comes in the [server will mark the job as failed](https://github.com/cetra3/jobq/blob/2028ea12388bd077056d984016ecf46c88e6626c/src/server.rs#L98-L102) and submit the reason with it, decrementing the active job count

#### Error Deserialising

There is a possibility that a connected client could send an invalid message, so we just log with a `warn` level in this case, skipping the message:

```rust
match server_msg {
    ....
    Err(err) => {
        warn!("Could not deserialize message:{}", err);
    }
}
```

### Submit Tasks

After the main loop matching against the message types, the server checks to see if there are any active slots & if so, queries the database for any queued jobs:

```rust
// If we have less active tasks lets check the queued stuff
if active < self.num {
    let jobs = handle
        .get_queued_jobs(self.num as i64 - active as i64)
        .await?;

    for job in jobs {
        send_job(&handle, job, &mut send).await?;
        active = active + 1;
    }
}
```

After this processing, the server then waits again for the next message.

## Database Persistence

One of the requirements is ensuring that jobs stay around and are persisted somewhere.   The prototype uses [PostgreSQL](https://www.postgresql.org/) to do this.

As this is async, we'll use [tokio-postgres](https://crates.io/crates/tokio-postgres) to perform updates.  tokio postgres has some [nice docs](https://docs.rs/tokio-postgres/0.5.3/tokio_postgres/) to get us started, so we will use their example to build a [database handler](https://github.com/cetra3/jobq/blob/a09ebcaff164c2153cdceba0031dbfb7caa1ea3b/src/db.rs):

```rust
use tokio_postgres::{NoTls, Error};

#[tokio::main] // By default, tokio_postgres uses the tokio crate as its runtime.
async fn main() -> Result<(), Error> {
    // Connect to the database.
    let (client, connection) =
        tokio_postgres::connect("host=localhost user=postgres", NoTls).await?;

    // The connection object performs the actual communication with the database,
    // so spawn it off to run on its own.
    tokio::spawn(async move {
        if let Err(e) = connection.await {
            eprintln!("connection error: {}", e);
        }
    });

    // Now we can execute a simple statement that just returns its parameter.
    let rows = client
        .query("SELECT $1::TEXT", &[&"hello world"])
        .await?;

    // And then check that we got back the same string we sent over.
    let value: &str = rows[0].get(0);
    assert_eq!(value, "hello world");

    Ok(())
}
```


### Enums and Postgres Types

You can use the [`postgres_types`](https://crates.io/crates/postgres-types) crate to save simple enums to a table.

So the `Priority` struct can be persisted to the db like so:

```sql
CREATE TYPE "Priority" as enum ('High', 'Normal', 'Low');
```

### A Simple Migration

Hopefully our schema never changes.. Right?

Well, in the prototype it won't. So we can simply embed an [SQL script](https://github.com/cetra3/jobq/blob/master/src/setup.sql) to run when the a db handler [starts up](https://github.com/cetra3/jobq/blob/a09ebcaff164c2153cdceba0031dbfb7caa1ea3b/src/db.rs#L13-L27):


```rust
pub struct DbHandle {
    client: Arc<Client>,
}

impl DbHandle {
    pub(crate) async fn new(url: &str) -> Result<Self, Error> {
        let (client, connection) = tokio_postgres::connect(&url, NoTls).await?;

        tokio::spawn(async move {
            if let Err(e) = connection.await {
                error!("connection error: {}", e);
            }
        });

        client.batch_execute(include_str!("setup.sql")).await?;

        Ok(DbHandle {
            client: Arc::new(client),
        })
    }
    ...
}
```

This `setup.sql` script should be designed to run more than once, but skip the parts that it's already done.  There are a couple tricks to do this, but obviously doesn't work if you are adding/removing columns from a table etc..

For the enum types, you can just check the `pg_type` to make sure that they exist & if not add them:

```sql
IF NOT EXISTS (SELECT 1 FROM pg_type WHERE typname = 'Status') THEN
    CREATE TYPE "Status" as enum ('Queued', 'Processing', 'Completed', 'Failed');
END IF
```

With the table & indexes you can do a `CREATE <blah> IF NOT EXISTS`:

```sql
CREATE INDEX IF NOT EXISTS status_idx ON jobq (
    status
);
```

This won't error out if the index/table already exists, meaning you're safe to run the `setup.sql` multiple times!


### Getting jobs from the DB

Querying for jobs is quite simple, you just select the columns you want and do a `get()` or `try_get()` on them:

```rust
impl DbHandle {
    ....
    pub(crate) async fn get_queued_jobs(&self, num: i64) -> Result<Vec<Job>, Error> {
        let query = "select 
                        id,
                        name,
                        username,
                        uuid,
                        params,
                        priority,
                        status
                     from jobq
                     where 
                        status = 'Queued'
                     order by
                     priority asc, time asc
                     limit $1";

        let mut jobs = Vec::new();

        for row in result {
            let id = row.try_get(0)?;
            let name = row.try_get(1)?;
            let username = row.try_get(2)?;
            let uuid = row.try_get(3)?;
            let params = row.try_get(4)?;
            let priority = row.try_get(5)?;
            let status = row.try_get(6)?;

            jobs.push({
                Job {
                    id,
                    username,
                    name,
                    uuid,
                    params,
                    priority,
                    status,
                }
            });
        }

        Ok(jobs)
    }
    ...
```

There are a number of other methods as [complete_job](https://github.com/cetra3/jobq/blob/2028ea12388bd077056d984016ecf46c88e6626c/src/db.rs#L29), [fail_job](https://github.com/cetra3/jobq/blob/2028ea12388bd077056d984016ecf46c88e6626c/src/db.rs#L37) that handle the SQL stuff & make it simple for the Server to use.

Keeping things persisted in the database means we can use standard SQL tools to introspect the job queue and see where it's at.

### Recording the job duration

The duration of the job is recorded when the [job is finished/failed](https://github.com/cetra3/jobq/blob/2028ea12388bd077056d984016ecf46c88e6626c/src/db.rs#L29-L35):

```rust
pub(crate) async fn complete_job(&self, id: i64) -> Result<(), Error> {
    let query = "update jobq set status = 'Completed', duration = extract(epoch from now() - \"time\") where id = $1";

    self.client.query(query, &[&id]).await?;

    Ok(())
}
```

This could've been done inside rust as well using `Instant` but would require a bit more state to be handled in the Server.  I've elected here to use a PostgreSQL solution instead to lean on the db.

## Running the example

The example starts the test worker, a server and a client to submit 500 simple jobs:

```rust
for i in 0..500 {
    let priority = if i % 2 == 0 {
        Priority::High
    } else {
        Priority::Normal
    };

    let job = JobRequest {
        name: "test".into(),
        username: "test_client".into(),
        params: Value::Null,
        uuid: Uuid::new_v4(),
        priority,
    };

    send.send(ServerMessage::Request(job).to_mpart()?).await?;
}
```

Running with `cargo run` you will see the output after some time, with the simulated failures:

```log
2020-04-14 16:04:33 DEBUG jobq         > Message:Acknowledged(Job { id: 2999, username: "test_client", name: "test", uuid: fe6e21fc-8064-4c9a-965d-b1d124f2416d, params: Null, priority: High, status: Queued })
2020-04-14 16:04:33 DEBUG jobq::server > New: Job { id: 3000, username: "test_client", name: "test", uuid: d20b4e9b-477b-45ca-9a6a-1f6423378779, params: Null, priority: Normal, status: Queued }
2020-04-14 16:04:33 DEBUG jobq         > Message:Acknowledged(Job { id: 3000, username: "test_client", name: "test", uuid: d20b4e9b-477b-45ca-9a6a-1f6423378779, params: Null, priority: Normal, status: Queued })
2020-04-14 16:04:39 WARN  jobq::server > Job failed: 3000, Reason: Simulating failure
2020-04-14 16:04:39 WARN  jobq::server > Job failed: 2988, Reason: Simulating failure
2020-04-14 16:04:39 WARN  jobq::server > Job failed: 2976, Reason: Simulating failure
```

You can also run some simple stats directly against the db to see how long things have taken (hopefully ~100ms):

```sql
select avg(duration) from jobq;

    avg     
------------
 0.10634147
(1 row)
```

## Conclusions

I hope for anyone new that this article will help dip their feet into the waters of rust and async to see what can be built.

Taking a few libraries and gluing them together we can get a pretty decent, albeit rough, outline of a background job queue.  The new `async/await` ecosystem while embryonic already has the tools there to build some cool stuff.

There will be a separate article around the release of `tmq` version `0.2.0` and some of the changes that have been made, but thought it might be great to provide a good example of working code before getting aroudn to that.
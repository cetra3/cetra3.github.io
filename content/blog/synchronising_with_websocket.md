+++

title = "Synchronizing state with Websockets and JSON Patch"
description = "A simple and extendable, almost realtime, state-sharing method for frontend and backend"
date = 2023-01-29

[taxonomies]
tags = ["rust"]
+++

At [formlogic](https://www.formlogic.com/), we are using rust for a number of components within our tech stack, including: internal and external web applications, coordinating and logging machining workflows, parsing an industrial domain specific language ([G-Code](https://en.wikipedia.org/wiki/G-code)), and more.

One of the challenges we faced early on was we had state that needed to be synchronized between rust applications, browsers and the database itself. We wanted to make sure that everyone could subscribe to state changes and everyone could have the same view. We also wanted the ability to roll back state, but preserve history i.e, have an append only immutable event log, while also being able to extend the structure of our state if we choose to in the future.

This blog will step through one possible solution for this challenge using a combination of [WebSockets](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API) for transport and [JSON Patch](http://jsonpatch.com/) for describing changes to an arbitrary structure.

To provide an example of this, we'll be building up everyone's favourite **Todo Application**: a list of todos with a mark to say whether they're completed or not. However, our version will ensure that all connected clients will, in close to real time, have updates shown.

For those wanting to skip ahead and see the finished product, you can [view the source here](https://github.com/cetra3/websocket_jsonpatch/).

<video width="100%" autoplay muted loop src="/videos/websocket_jsonpatch.mp4" ></video>

## What is JSON Patch

[JSON Patch](http://jsonpatch.com/) is an RFC standard for describing patch updates to an arbitrary JSON structure.

For instance, if you had a JSON object that looked like this:

```json
{
  "name": "Hello"
}
```

And a patch that looks like this:

```json
[
  {
    "op": "replace",
    "path": "/name",
    "value": "Hello World!"
  }
]
```

Then applying this patch, your object will now look like this:

```json
{
  "name": "Hello World!"
}
```

Another way to think about it, is that JSON Patch can encode transitions of state, and can be considered a lightweight form of [event sourcing](https://martinfowler.com/eaaDev/EventSourcing.html).

There are a number of libraries available in all languages, including [rust](https://github.com/idubrov/json-patch) and for [browsers](https://github.com/Starcounter-Jack/JSON-Patch), and as the format itself is just JSON, it makes it easy to save the patches themselves in `jsonb` column on your database.

JSON Patch does not, by itself, enforce any schema for the JSON document. There are some operations that can cause an error if applied incorrectly, such as out of bounds replacements in arrays, but nothing that enforces that the end result is what you are expecting. For that purpose we're going to leverage [`serde`](https://github.com/serde-rs/serde) for its great type safety, ensuring our state is in the shape we want.

## WebSockets and Updates

With JSON Patch we have a stream of changes to a structure we can act upon, but how do we transport it? Well, since we want browsers to be a part of this as well, a great candidate is [WebSockets](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API). WebSockets are a great way to stream things into a frontend without having the overhead of having a HTTP Connection per message. This channel can stay open for as long as the browser is open. We can receive also messages in near real time: only network latency and a bit of decoding act as a bottleneck.

Luckily WebSocket support is not only a feature of backend frameworks, but also has clients suited to connecting as well. We'll only be investigating browsers in this blog for brevity, but one could easily expand it to use something like [`tokio-tungstenite`](https://github.com/snapview/tokio-tungstenite) to consume our messages in rust.

## Rust Backend

For brevity's sake we will choose `axum` as the backend, and their [websockets example](https://github.com/tokio-rs/axum/tree/main/examples/websockets) as the base. For this example to build by itself, there are a few dependent crates:

```toml
axum = { version = "0.6.4", features = ["ws"]}
tokio = { version = "1", features = ["full"] }
```

The `ws` feature adds in websocket support to axum, otherwise all we'll use the defaults

For debug logs, we'll use [`tracing`](https://github.com/tokio-rs/tracing) with `tracing-subscriber` to get things logged nicely to stderr:

```toml
tracing = "0.1.37"
tracing-subscriber = "0.3.16"
```

And for our example we're going to use `serde` and `serde_json` for serialization, with `json-patch` as mentioned above. We'll also include the `futures` crate, as we'll see that comes in use later when we want to split the rx/tx of a websocket channel:

```toml
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
futures = "0.3"
json-patch = "0.3.0"
```

### The Main Method

With the main method we want to instantiate some logging, set up our routes for our web app & then serve on a port. While the initial example includes some static file hosting, we will skip those routes for brevity and only include the websocket route.

Our main function becomes quite concise:

```rust
async fn main() {

    // Set a sensible default for logging to ensure we see something
    if std::env::var_os("RUST_LOG").is_none() {
        std::env::set_var("RUST_LOG", "websocket_jsonpatch=debug")
    }

    // Initialise the `fmt` subscriber which will print logs to stderr
    tracing_subscriber::fmt::init();

    // Add in our application with the websocket handler
    let app = Router::new()
        .route("/ws", get(ws_handler))
        // Add in a shared websocket state struct `WsState`
        .layer(Extension(Arc::new(WsState::new())));

    // Listen on port 3333 for connections
    let addr: SocketAddr = "127.0.0.1:3333".parse().unwrap();

    debug!("Listening for requests on {}", addr);

    // Start up the server
    axum::Server::bind(&addr)
        .serve(app.into_make_service())
        .await
        .unwrap();
}
```

## The Todo Struct

Let's start shaping our todo struct:

- We want to name a todo list itself, such as `Shopping List` or `Writing a blog about Rust`. This also gives us an easy thing to update in the structure.
- We want a list of entries, which have a name and a boolean as to whether they are marked `Completed` or not.
- We want some convenience derives such as for serialization with serde and sensible defaults.

With that in mind, we have the main `Todo` struct:

```rust
#[derive(Serialize, Deserialize, Default, Debug, Clone)]
pub struct Todo {
    // The main name of the todo list
    name: String,
    // The list of todos
    todos: BTreeMap<u32, TodoRow>,
}
```

And each Row has the following shape:

```rust
#[derive(Serialize, Deserialize, Default, Debug, Clone)]
pub struct TodoRow {
    // The name of the specific todo
    name: String,
    // Whether this todo is completed or not
    completed: bool,
}
```

#### A Note on using a `BTreeMap` instead of a `Vec`

An initial version of this blog included todos as a `Vec<TodoRow>`, but that was found to be quite noisy when clearing out completed todos. Refactoring this to use a `BTreeMap` instead took not long at all, and is a testament to this design being malleable. Readers are invited to adjust this shape back to a `Vec` or provide other fields/values to see how to make such a change.

### Making Changes

If you have used the [redux pattern](https://redux.js.org/) before, you would be familiar with use actions to mutate state. We'll build something similar here, to make it easier to describe the intent of the change and to have all the logic in one module. These actions are what are going to be sent from the browser or other clients.

When actions are received, the todo state is changed, and we generate a diff between the old and new JSON, which is then sent over the wire back to the frontend in the form of JSON Patches.

These changes are executed in serial by means of a held mutex, rather than allow concurrent updates to the state. We'll also only have one "todo" list, but could imagine an easy extension to have multiple different lists, each with their own changes. For concurrency without making changes serial, a [CRDT](https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type) like [`automerge`](https://github.com/automerge/automerge-rs) could be a suitable candidate. In practice, we've found that, when given a low concurrent user count per JSON object, the simplicity and auditability of JSON patches is a great trade-off.

### Actions

We're going to have a few actions that are available to use (all of which should be self-descriptive):

```rust
#[derive(Deserialize)]
#[serde(tag = "type")]
pub enum TodoAction {
    Add { row: TodoRow },
    ChangeName { name: String },
    Update { index: u32, row: TodoRow },
    Remove { index: u32 },
    RemoveCompleted,
}
```

We are using a tag to make it easier to define these in TypeScript later on. Serde will use a field named `type` to store the enum variant.

With those actions, we can then use them to apply to our `Todo` struct and update the state:

```rust
impl Todo {
    pub fn apply(&mut self, action: TodoAction) {
        match action {
            TodoAction::Add { row } => {
                // Find the next available index
                let index = self.todos.keys().max().copied().unwrap_or_default() + 1;

                // Insert this into our map
                self.todos.insert(index, row);
            }
            TodoAction::Update { row, index } => {
                self.todos.insert(index, row);
            }
            // Change the name of the todo list
            TodoAction::ChangeName { name } => self.name = name,
            TodoAction::Remove { index } => {
                self.todos.remove(&index);
            }
            // Filter and remove all completed todo rows
            TodoAction::RemoveCompleted => self.todos.retain(|_, val| !val.completed),
        }
    }
}
```

## WebSockets

We are going to use websockets as our primary means of communication between the front and backend, providing a bi-directional channel with which we can send messages.

### Shared WsState

In the `main()` function, we added a reference counted `WsState` struct via `AddExtensionLayer`. This represents our shared state between all websocket sessions.

Inside this struct, we are going to keep two things:

- The `Todo` struct as per our modelling above inside a tokio `Mutex`
- A list of websocket [sinks](https://docs.rs/futures/0.3.17/futures/sink/trait.Sink.html) that we can send updates to. So when one session makes a change, it is reflected on all other sessions.

This struct looks like so:

```rust
pub struct WsState {
    // Our main state, behind a tokio Mutex
    todo: Mutex<Todo>,
    // A list of sessions we will send changes to
    txs: Mutex<Vec<SplitSink<WebSocket, Message>>>,
}
```

#### Adding Sessions

When we add a session, we want to send a full state to get it caught up, and then add it to our list:

```rust
async fn add_session(&self, tx: SplitSink<WebSocket, Message>) {
    // Get a mutable lock of our transactions
    let mut txs = self.txs.lock().await;

    // Send the initial state update
    if let Err(err) = tx
        .send(Message::Text(
            // This method will not fail in "normal" operations so an `expect()` is OK here
            serde_json::to_string(&ServerMessage::Full {
                todo: &*self.todo.lock().await,
            })
            .expect("Serialize Error"),
        ))
        .await
    {
        warn!("Could not send initial state update: {}", err);
        return;
    }
    // Add session to our list of sessions
    txs.push(tx);
}
```

### Applying State Changes

On `WsState` we'll create a method called `apply` which takes a `TodoAction` and applies this to the state, then broadcasts changes to all connected sessions. We'll break this method down as there is a lot happening here.

The signature of this method looks like this:

```rust
async fn apply(&self, action: TodoAction) -> Result<(), Error> {

    ...
}
```

The first thing we need to do is grab a mutable reference to our existing state:

```rust
let mut state = self.todo.lock().await;
```

Then we want to get a copy of the state as it stands currently, but as a `serde_json::Value` which we can use to diff:

```rust
let existing_json = serde_json::to_value(&*state)?;
```

After we have our `existing_json`, we can use the `Todo::apply()` method to mutate the value:

```rust
// Apply the action to our todo list.  This mutates it in place
state.apply(action);
```

With that applied, we can grab the new state as a `Value` as well:

```rust
// Serialize out the new JSON for diffing
let new_json = serde_json::to_value(&*state)?;
```

Then, we can use the [`diff`](https://docs.rs/json-patch/0.2.6/json_patch/fn.diff.html) method from `json_patch` to automatically generate the patches needed to go from the existing to the new state:

```rust
// Get the changes using the `diff` method from `json_patch`
let ops = diff(&existing_json, &new_json).0;
```

With this, you'll notice we are quite agnostic to what the shape is of the state, and how the state changes. All we need to be able to do is serialize it before and after, and we can completely change the structure internally. A further excercise would be to generalize this method to accept anything that implements `Serialize`

Ok, we have a list of patches, let's print them out so we can inspect them:

```rust
debug!("New Patches:{:?}", ops);
```

#### Broadcasting Our Changes

Generating changes to state is one half of this method, the other half is to broadcast this out to every session.

We will first check whether there were any changes worth broadcasting, and then will serialize it to a string

```rust
// If there are no changes, don't bother broadcasting
if !ops.is_empty() {

    // Serialize it to a json string here
    let message = serde_json::to_string(&ServerMessage::Patch { ops })?;

    ...
}
```

The next thing we want to do is grab all of our sessions and lock them for the time being:

```rust
let mut txs = self.txs.lock().await;
```

Now, some session between now and the last broadcast could have gone away.

What we are going to do is take all the sessions out of the existing vec, replacing it with an empty one using `std::mem:take()`, and then try sending to each session. If there is no issue sending, we add it back to the list.

If there is an issue sending, we will log the error but not add it back, to be dropped/cleaned up when the object is out of scope:

```rust
// We take all the txs to iterate, and replace with an empty `Vec`
for mut tx in mem::take(&mut *txs) {
    // If there is an issue sending a message we will warn about it
    if let Err(err) = tx.send(Message::Text(message.clone())).await {
        warn!("Client disconnected: {}", err);
    // If there is no issue sending, then we add it back to our `Vec`
    } else {
        txs.push(tx)
    }
}
```

And then we're done! To recap, when applying changes we first generate a list of patches to apply. We then send that list to all the connected sessions, cleaning up any erroneous sessions in the process.

### Axum Websocket Interface

Axum provides a `WebSocketUpgrade` struct, with which you can feed it a function (or closure) to start an async task when the websocket upgrade happens. We use this to feed in our `WsState` value (behind an `Arc`):

```rust
async fn ws_handler(
    ws: WebSocketUpgrade,
    Extension(state): Extension<Arc<WsState>>,
) -> impl IntoResponse {
    debug!("New Websocket Connection");
    ws.on_upgrade(|socket| handle_socket(socket, state))
}
```

### Server Messages

To keep things simple we will have an enum of possible messages the server/backend can send to clients. There are going to be only two types: a list of patches to apply or a full todo struct. When clients first connect, we need to get them caught up with the current state of the world, so we send them the full state.

```rust
#[derive(Serialize, Debug)]
#[serde(tag = "type")]
enum ServerMessage<'a> {
    Patch { ops: Vec<PatchOperation> },
    Full { todo: &'a Todo },
}
```

As per the `TodoAction` we'll use a `type` tag for ease of integration with TypeScript. Also notice that we will be able to send multiple patches in one message. This is for in the case of a change causing a cascade of patches, such as when removing completed items in the todo list.

This `ServerMessage` enum will be serialized on the wire as JSON. Websockets support standard text messages, as well as binary types. A further exercise would be to use a different encoding, such as [`CBOR`](https://cbor.io/), but does make things a bit less human readable

### Handle Socket

Our `handle_socket` method is responsible for initiating new session & feeding that session into the `WsState` struct.

```rust
pub async fn handle_socket(socket: WebSocket, state: Arc<WsState>) {
    // Magic happens here
}
```

Firstly, we'll split up the websocket into a tx/rx pair. We'll have the `handle_socket` hold onto the `rx` part, and move the `tx` into the `WsState` value. If you have a struct that implements `Stream` and `Sink` you can use futures and `StreamExt` to split this into two separate objects, with separate ownership of both.

```rust
let (tx, mut rx) = socket.split();
```

We'll then add this session to our shared state:

```rust
state.add_session(tx).await;
```

This method will fire off a `ServerMessage::Full` with the current state, returning early if this fails

Then, we simply loop any messages received from the client & apply them to our shared state. The type of messages we receive from the client will be our `TodoAction` enum.

To loop while there are messages, we can use the following `let` binding:

```rust
while let Some(Ok(msg)) = rx.next().await {
    ...
}
```

This loop will terminate if there are no more messages to receive (i.e, the stream is finished and returning `None`), and if there are any errors receiving messages.

Inside the loop, we decode the message as JSON, and throw a warning if it's a message we're not expecting. We then apply that message to our shared state. As above, the shared state will in turn broadcast out JSON patches to update the state:

```rust
// Loop until there are no messages or an error
while let Some(Ok(msg)) = rx.next().await {
    if let Message::Text(text) = msg {
        // Decode our message and warn if it's something we don't know about
        if let Ok(action) = serde_json::from_str::<TodoAction>(&text) {
            // Apply the state, which will broadcast out changes as a JSON patch
            if let Err(err) = state.apply(action).await {
                warn!("Error applying state:{}", err);
            }
        } else {
            warn!("Unknown action received:{}", text);
        }
    }
}
```

Ok! So we have the shape of our data, some actions to mutate the state and some scaffolding to wire up the websockets. What's next is to build the frontend.

## React Frontend

We're going to build a companion browser application to use our backend. This will be built in react, but could be built using any framework, or even vanilla javascript. [Vite](https://vitejs.dev/) will be the build tool of choice, given it has a fast development feedback loop and supports React/TypeScript. For styling, we'll use [spectre.css](https://picturepan2.github.io/spectre/) to make it look a little better than the standard style and provide some responsivness. You can create a new vite project by running:

```
yarn create vite front --template react-ts
```

We'll want to run the rust backend in another terminal window with:

```
cargo run
```

And then we want to proxy all requests to `/ws` to the backend which we can do by changing the `vite.config.ts`:

```js
export default defineConfig({
  plugins: [react()],
  server: {
    proxy: {
      "/ws": {
        target: "http://127.0.0.1:3333",
        ws: true,
      },
    },
  },
});
```

With that in place, we can `yarn dev` to get up a developer window.

### Todo Structure

We want to have the todo structure imitate what's in the backend. For brevity here we'll just write them by hand, but there does exist tools to have these structures generated automatically (see the blog: [Publishing Rust types to a TypeScript frontend](/blog/sharing-types-with-the-frontend/)).

Starting off with the `Todo` struct itself:

```typescript
export interface Todo {
  name: string;
  todos: { [index: number]: TodoRow };
}

export interface TodoRow {
  id: number;
  name: string;
  completed: boolean;
}
```

Then, we deviate a little bit with the actions. In the backend we have used a tagged enum with the `type` property to define what action is in use. In typescript, we will use a [**Discriminated Union**](https://www.typescriptlang.org/docs/handbook/2/narrowing.html#discriminated-unions) pattern to define these actions in a compatible way:

```typescript
export interface Add {
  type: "Add";
  row: TodoRow;
}

export interface ChangeName {
  type: "ChangeName";
  name: string;
}

export interface Update {
  type: "Update";
  row: TodoRow;
  index: number;
}

export interface Remove {
  type: "Remove";
  index: number;
}

export interface RemoveCompleted {
  type: "RemoveCompleted";
}

export type TodoAction = Add | ChangeName | Update | Remove | RemoveCompleted;
```

### Websocket Component

For the Websocket component, we'll cheat a little bit and use some global variables to hold around both the websocket connection itself & the shared state. Using react hooks we'll integrate with components to allow state changes to come through.

Firstly, we define our server messages as per the backend. We'll be using the same **Discriminated Union** pattern.

```typescript
interface Patch {
  type: "Patch";
  // `Operation` from `fast-json-patch` module
  ops: Operation[];
}

interface Full {
  type: "Full";
  todo: Todo;
}

type ServerMessage = Patch | Full;
```

Then, let's set up our global variables:

```typescript
let websocket: WebSocket | undefined;
let todo: Todo | undefined;
```

We use `undefined` to indicate those values aren't initialised. Both the websocket & the todo itself are initialised at different times.

Then, we'll create a `setupWebsocket` function which will handle the connection & and plumbing needed to send and receive messages. We'll want to have a callback function to accept a new `todo` object which we'll use to integrate into react hooks later.

So with the the signature looks like the following:

```typescript
const setupWebsocket = (onTodoUpdate: (todo: Todo) => void) => {
   ...
}
```

Then we want to create a new `WebSocket`, but we need to pass it a valid uri. Websockets use `ws://` instead of `http://` and `wss://` instead of `https://` for their URLs, so we'll split apart the current `window.location` and use components of it to build our uri:

```typescript
const loc = window.location;
const uri = `${loc.protocol === "https:" ? "wss:" : "ws:"}//${loc.host}/ws`;
console.log(`Connecting websocket: ${uri}`);

const connection = new WebSocket(uri);
```

Ok, with our connection created we will need to listen to different events, and execute accordingly.

Firstly, when the connection is opened, we'll log that its connected and then set our global `websocket` variable equal to this connection:

```typescript
connection.onopen = () => {
  console.log("Websocket Connected");
  websocket = connection;
};
```

Now, there is a case that a websocket connection drops while you have the page open. We want to recover from this somewhat gracefully, retrying the connection until it's successful.

One way of doing this is to re-execute the `setupWebsocket` method after a bit of time has passed, and that's what we'll do here, retry the setup after half a second has passed. If the websocket has closed gracefully, then we don't bother retrying:

```typescript
// If we receive a close event the backend has gone away, we try reconnecting in a bit of time
connection.onclose = (reason) => {
  websocket = undefined;

  // https://developer.mozilla.org/en-US/docs/Web/API/CloseEvent
  if (reason.code !== 1000 && reason.code !== 1001) {
    console.error("Websocket connection closed", reason);

    setTimeout(() => {
      setupWebsocket(onTodoUpdate);
    }, 500);
  }
};
```

Then, if there is an error with the websocket we can log what that error is, then close it

```typescript
connection.onerror = (error) => {
  console.error("Error with websocket", error);
  connection.close();
};
```

Ok with the websocket life cycle out of the way, now comes to the meat of our event loop. What we're going to do is listen for server events, and if either type is received, act accordingly:

- For the `Full` server message, the `todo` object will be replaced outright.
- For the `Patch`, we will use the `fast-json-patch` module to mutate `todo` in place, and then execute our callback:

Whatever the type of message, the full `todo` object is provided to the callback, with the patch merging logic happening here, and shielded from our callback update.

```typescript
connection.onmessage = (message) => {
  // Parse the message via JSON, asserting what we'll get from the backend
  const msg = JSON.parse(message.data) as ServerMessage;

  switch (msg.type) {
    case "Patch": {
      // Mutate the todo state in place, and then send an update
      if (todo !== undefined) {
        let { newDocument: newTodo } = applyPatch(todo, msg.ops, false, false);

        onTodoUpdate(newTodo);
        todo = newTodo;
      }
      break;
    }
    case "Full": {
      // Send on the full todo state
      onTodoUpdate(msg.todo);
      todo = msg.todo;
      break;
    }
  }
};
```

Now, we're almost done with the websocket, we need to integrate it a tiny bit nicer with react and react hooks.

Firstly, using the `useEffect` hook, we'll expose the `todo` state to a component which will rerender on object updates. You can provide a function starting with the keyword `use` to have it part of of the react render lifecycle.

We'll bind a call back to update a variable `todo` handled with `useState` when there are changes, and then return that variable state from the function to use in another component.

```typescript
export const useWebsocket = () => {
  // Keep our local state of the todo app to trigger a render on change
  let [todo, updateTodo] = useState<Todo>();

  useEffect(() => {
    // Update our app state when changes are received
    setupWebsocket((msg) => {
      updateTodo(msg);
    });
    // If the destructor runs, clean up the websocket
    return () => {
      if (websocket) {
        websocket.close(1000);
      }
    };
    // The empty `[]` dependency list makes this `useEffect` callback execute only once on construction
  }, []);

  return todo;
};
```

Another way of looking at it is this function is purely glue code to make it easy to integrate with react.

Lastly, we'll cheat again with our global variables and have a `sendAction` method which will, if the websocket is up and running, send a `TodoAction` to the backend:

```typescript
export const sendAction = (action: TodoAction): void => {
  if (websocket) {
    websocket.send(JSON.stringify(action));
  }
};
```

### React Components

We now have our general shape of our todo app, and have our websocket connection abstracted away nicely it's time to build up components.

We'll start with the main `App` component. This will be our root component, and will be responsible for building other components based upon the state. The `App` component is a perfect candidate to use our `useWebsocket` hook we have defined.

We'll add in a small "loading" indicator if the `todo` state is undefined, otherwise we'll pass on our `todo` state to a component:

```js
function App() {
  const todo = useWebsocket();

  return (
    <div className="container grid-lg">
      <div className="columns">
        <div className="column col-lg-12 todo-title">
          <h1>Todo App Example</h1>
        </div>
      </div>
      {todo === undefined && <div className="loading loading-lg"></div>}
      {todo && <TodoComponent todo={todo} />}
    </div>
  );
}
```

#### Todo Component

Our `TodoComponent` is responsible for some of the more "global" actions of our state, such as adding a new todo, changing the todo list name and clearing out completed todos.

Let's start with a standard functional component that takes a `todo` property as input:

```js
function TodoComponent({ todo }: { todo: Todo }) {
  return <></>;
}
```

Now, let's build in the `todo.name` property. If we want users to be able to edit it, it should live in an input field:

```js
<input value={todo.name} />
```

This will ensure the value is reflected, and since our `todo` object is "hooked" in to react, it will update when it changes.

The next step is to use an `onChange` callback to the input to send actions to the backed on change. We want to fire the `ChangeName` action, and so we can use the `sendAction` function from the websocket module to trigger this change.

```js
<input
  value={todo.name}
  onChange={(ev) => {
    sendAction({
      type: "ChangeName",
      name: ev.currentTarget.value,
    });
  }}
/>
```

With that in place, we have the bare minimum implemented to see state changes. Now would be a great time, if you have been following along, to fire up your IDE and see what happens when you type in that input field (make sure the backend is running!).

Now, for each `TodoRow` we'll create a `TodoRowComponent`, which will be responsible for the state of that given row:

```js
{
  Object.entries(todo.todos).map(([index, row]) => (
    <TodoRowComponent key={index} row={row} index={+index} />
  ));
}
```

And underneath our the list, we will have two buttons.

One that only displays if there are todos that are completed:

```js
Object.entries(todo.todos).some(([, val]) => val.completed) && (
  <button
    className="btn"
    onClick={() => {
      sendAction({
        type: "RemoveCompleted",
      });
    }}
  >
    Remove Completed
  </button>
);
```

And one that adds an extra item to the list:

```js
<button
  onClick={() => {
    const entries = Object.entries(todo.todos);
    sendAction({
      type: "Add",
      row: {
        name: "",
        completed: false,
      },
    });
  }}
>
  Add Todo
</button>
```

#### TodoRowComponent

The TodoRowComponent will take a `TodoRow` and the `index` of that row, and display actions for the items in our todo list:

```js
function TodoRowComponent({ row, index }: { row: TodoRow; index: number }) {
    ...
}
```

We can add in a check box to mark things completed or not:

```js
<input
  type="checkbox"
  checked={row.completed}
  onChange={() => {
    sendAction({
      type: "Update",
      row: {
        ...row,
        completed: !row.completed,
        index,
      },
    });
  }}
/>
```

And an input for the name of the specific todo list:

```js
<input
  value={row.name}
  type="text"
  onChange={(ev) =>
    sendAction({
      type: "Update",
      row: {
        ...row,
        name: ev.currentTarget.value,
        index,
      },
    })
  }
/>
```

And a button to remove that row:

```js
<button
  onClick={() =>
    sendAction({
      type: "Remove",
      index,
    })
  }
/>
```

## Running and Debugging

Ok, we now have a frontend wired up to a backend, with a websocket used for state updates. Let's change a few values around and see what is going across the "wire".

The easiest way to do this, is via your browser's network tab, looking for the request to `ws://localhost:3000/ws`. In chrome, the tab you're looking for is `Messages`, and in firefox it's `Response`.

You can also use a tool like [`websocat`](https://github.com/vi/websocat) to view the messages in a terminal.

Running this up, you'll see the initial message come through when subscribed:

```json
{ "type": "Full", "todo": { "name": "", "todos": [] } }
```

Then, making changes you will see the updates feed in. For instance if you click the `Add Todo` button, you'll see an outbound action:

```json
{ "type": "Add", "row": { "name": "", "completed": false } }
```

Followed by a patch update from the server:

```json
{
  "type": "Patch",
  "ops": [
    {
      "op": "add",
      "path": "/todos/0",
      "value": { "completed": false, "name": "" }
    }
  ]
}
```

Go ahead and try out other updates to see what patches are generated and get a feel for the JSON patch format.

## Conclusions

Starting from the backend and then building out the frontend, we've used Websockets and JSON Patch to provide a platform of shared state updates.

Using existing standards and combining them together makes for a powerful and solid foundation. With rust, we get some great concurrency primitives that we can rely on to ensure our logic is correct.

We've demonstrated here that, we can provide a powerful framework, abstracting away our transport and state update logic, with room to grow.

The code for this article is available in a [source repository here](https://github.com/cetra3/websocket_jsonpatch/), so please feel free to tweak.

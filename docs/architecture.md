# FletUI architecture

## Status

This document defines the proposed FletUI `0.2` architecture. The design uses:

- an `Application` interface for the model, update, and view lifecycle;
- owned `Event` values for terminal and application messages; and
- consuming `Command` callables for deferred work.

The public contract is intentionally small. Applications own their state,
FletUI owns the terminal and event loop, and commands communicate with the
application only by emitting events.

## Public contract

```flecha
export interface Application {
    Command initialCommand(&const self) default;

    (Command, error) update(
        &self,
        Event event
    ) virtual;

    (View, error) view(
        &const self
    ) virtual;
};

export interface CommandContext {
    error emit(&self, Event event) virtual;

    error emitAfter(
        &self,
        duration::Duration delay,
        Event event
    ) virtual;

    void requestQuit(&self) virtual;
};

export once func Command(
    &CommandContext context
) => (error);

export error run(&Application application);
```

A zero callable represents the absence of a command:

```flecha
Command Application::initialCommand(&const self) {
    return none;
}
```

This contract requires Flecha to accept `none` as the zero value of a callable.
The `0.4` callable ABI already has an all-null representation, but the current
semantic checker does not yet admit `none` as a callable initializer or return
value. FletUI treats that as a required compiler capability. Invoking an empty
callable must have defined checked-trap behavior rather than performing a null
function-pointer call. A live command is move-only and invoking it consumes it.

## Application example

```flecha
structure Counter << fletui::Application {
    i32 count;
};

(fletui::Command, error) Counter::update(
    &self,
    fletui::Event event
) override {
    key, isKey := event.key();
    if !isKey {
        return (none, none);
    }

    if key.matches("q") {
        return (fletui::quit(), none);
    }

    if key.matches("space") {
        self->count++;
    }

    return (none, none);
}

(fletui::View, error) Counter::view(&const self) override {
    content, formatError := render_counter(self->count);
    if formatError != none {
        return (none, formatError);
    }

    return (
        fletui::View::from(content)
            .altScreen(true)
            .cursorHidden(true),
        none
    );
}

i32 main() {
    Counter application = none;

    runError := fletui::run(&application);
    if runError != none {
        return 1;
    }

    return 0;
}
```

## Commands

A command receives a runtime-owned `CommandContext`. It may produce zero, one,
or several events, schedule an event for later delivery, or request that the
program quit.

```flecha
fletui::Command load_data(string url) {
    return once [own url](&fletui::CommandContext context) => {
        response, requestError := fetch(url.view());

        if requestError != none {
            return context->emit(
                fletui::Event::message(
                    Message.LoadFailed.cast<u32>,
                    requestError.string()
                )
            );
        }

        return context->emit(
            fletui::Event::message(
                Message.Loaded.cast<u32>,
                response
            )
        );
    };
}
```

This shape is preferable to `Command() -> Event` because the context supports
multiple results, timers, and quit requests without changing the callable
signature.

The `Application::update` result has no `<-self` borrow contract. A conforming
implementation therefore cannot return a command that retains a borrow of the
application. This establishes the following invariants:

- application mutation happens only in `update`;
- rendering observes the application through `&const self`;
- commands cannot mutate the application behind the event loop; and
- a future executor may run commands asynchronously without changing the
  application contract.

### Command execution in 0.2

Public async execution is not yet available in Flecha. The initial FletUI
runtime therefore executes custom commands after rendering, on the event-loop
thread. Such commands must be short or use nonblocking operations.

Timers must not implement delays with blocking sleep. `emitAfter` registers a
deadline with the runtime reactor and returns immediately. The context-based
contract permits a future executor to run commands concurrently without
redesigning applications.

## Events

Flecha does not have payload enums. `Event` is therefore an opaque tagged
structure with constructors and checked accessors.

```flecha
export enum EventKind: u8 {
    default None,
    Key,
    Resize,
    Paste,
    Focus,
    Blur,
    Message,
    CommandError,
};

export structure Event {
    private EventKind kind;
    private KeyEvent keyValue;
    private Size resizeValue;
    private string textValue;
    private u32 messageTag;
    private error errorValue;
};
```

Applications inspect events through methods rather than unrelated payload
fields:

```flecha
EventKind Event::kind(&const self);

(KeyEvent, bool) Event::key(&const self);
(Size, bool) Event::resize(&const self);
(stringv, bool) Event::paste(&const self) <-self;
(MessageEvent, bool) Event::message(&const self) <-self;
(error, bool) Event::commandError(&const self);
```

Construction uses explicit functions:

```flecha
Event Event::message(u32 tag, string payload);
Event Event::keyEvent(KeyEvent key);
Event Event::resizeEvent(Size size);
Event Event::pasteEvent(string text);
```

An event may own strings and is consequently move-only. The runtime event queue
owns each event until `Application::update` consumes it.

### Application messages

Flecha `0.4` interfaces are non-generic and reference-only. FletUI therefore
cannot safely reproduce Go's arbitrary owning `Msg any` without implementing
an unsafe type-erased box.

The `0.2` design uses an explicit application-message envelope:

```flecha
export structure MessageEvent {
    u32 tag;
    string payload;
};
```

Applications define their own tag enum:

```flecha
enum Message: u32 {
    default None,
    Loaded,
    LoadFailed,
    Tick,
};
```

This is less strongly typed than an associated message type, but remains safe,
explicit, and simple. FletUI will not implement a home-grown unsafe `Any`
container. If fully typed command results become a requirement, the runner must
become generic or Flecha must gain a suitable associated-type or owning-
interface facility.

## Runtime lifecycle

The event loop follows this sequence:

1. Open the terminal and install deterministic cleanup.
2. Query the initial terminal size.
3. Obtain `Application::initialCommand`.
4. Deliver the initial resize event.
5. Render the first view.
6. Execute the queued initial command.
7. Wait for terminal input, timers, resize notifications, or queued events.
8. Pass the next event to `Application::update`.
9. Render the resulting view.
10. Execute the command returned by `update`.
11. Enqueue emitted events instead of recursively calling `update`.
12. On quit or failure, restore all terminal state before returning.

The runtime provides these guarantees:

- `update` is never concurrent with another `update`;
- `view` is never concurrent with `update`;
- events are delivered in FIFO order;
- commands execute only after the state produced by their update has been
  rendered;
- scheduled timers do not block the event loop;
- emitted events are queued, preventing recursive update stacks; and
- terminal state is restored on every controlled exit path.

Terminal, application-update, view, and unrecoverable command errors terminate
`run` after restoration. Recoverable domain failures should be represented as
application messages. A later implementation may deliver command failures as
`CommandError` events when doing so does not hide a failure to enqueue or
allocate the event.

## Declarative views

`View` describes both rendered content and the desired terminal state.

```flecha
export structure View {
    private string contentValue;
    bool altScreen;
    bool cursorHidden;
    bool bracketedPaste;
    bool reportFocus;
    MouseMode mouseMode;
    Cursor cursor;
    string title;
};
```

The runtime compares the new view with the previous terminal configuration and
applies only the required transitions. Application code does not directly emit
terminal-mode escape sequences.

The first renderer should favor correctness over optimization:

1. Move the cursor to the frame origin.
2. Render the complete content.
3. Clear rows left over from the previous frame.
4. Restore the cursor requested by the view.
5. Flush one complete output buffer.

Frame diffing may be added later after measurement demonstrates a need.

## Internal architecture

```text
src/
├── application.fl       Application interface and run()
├── command.fl           Command and CommandContext
├── event.fl             Event values and accessors
├── key.fl               Key parsing and matching
├── view.fl              Declarative terminal view
├── terminal.fl          Advanced low-level terminal API
└── internal/
    ├── runtime.fl       Native declarations
    ├── abi_posix.fl     POSIX terminal backend
    ├── decoder.fl       Stateful input decoder
    ├── reactor.fl       Polling, timers, and resize wakeups
    ├── scheduler.fl     Command and event queues
    └── renderer.fl      View reconciliation and output
```

### Terminal layer

The terminal layer owns raw-mode acquisition and restoration, terminal size,
complete output writes, and native input reads. It remains available as an
advanced low-level API, but ordinary applications should use `run`.

### Decoder

The decoder is a persistent stream parser rather than a function over one
complete byte slice. It must support:

- input split across arbitrary read boundaries;
- multiple events in one read;
- UTF-8 validation and Unicode scalar decoding;
- Escape versus Alt-key timeout handling;
- CSI and SS3 key sequences;
- bracketed paste; and
- explicit incomplete and invalid-input states.

### Reactor

The reactor blocks until input, a resize notification, a timer deadline, or a
runtime wakeup becomes ready. An idle application must not busy-poll.

Resize delivery should use `SIGWINCH` through a signal-safe wakeup mechanism,
such as a self-pipe, rather than running application logic in a signal handler.

### Scheduler

The scheduler owns pending events, timers, and commands. Events are FIFO.
Resize events may eventually be coalesced to the newest size, but the initial
implementation should prefer literal ordering and predictable behavior.

### Renderer

The renderer owns terminal escape sequences and view reconciliation. Layout
and styling code must not assume byte length equals terminal-cell width.
Unicode grapheme, combining-mark, wide-character, emoji, and ANSI-aware width
measurement must precede higher-level layout primitives.

## Testing architecture

The runtime must not require a real terminal in unit tests. Internal boundaries
must permit injection of:

- input byte streams;
- captured output;
- fixed and changing terminal sizes;
- a fake monotonic clock;
- synthetic resize notifications; and
- deterministic command completions.

Testing is divided into three layers:

1. Decoder unit and property tests, including arbitrary input chunking.
2. Headless application tests that send events and inspect rendered frames.
3. PTY integration tests for raw mode, restoration, signals, resizing, and ANSI
   output.

A future generic test harness may own a concrete application:

```flecha
structure Harness<T: Application> {
    T application;
};
```

It should expose operations such as `send(Event)`, `frame()`, and access to the
concrete application state without requiring a terminal.

## Implementation sequence

1. Replace the stateless decoder with buffered `feed` and `nextEvent`
   operations.
2. Add blocking polling and resize notification.
3. Implement opaque events and key-matching helpers.
4. Implement `View` and correct full-frame rendering.
5. Add `Application`, `CommandContext`, and `run`.
6. Add `emit`, `emitAfter`, and `quit` command helpers.
7. Build the headless test harness and PTY integration suite.
8. Port Arrowfall as the first complete acceptance application.

The first successful milestone is an application with no raw terminal calls,
manual ANSI clearing, polling loop, or restoration code. Its update and view
logic must be testable without a TTY, and an idle run must consume effectively
no CPU.

# FletUI

FletUI is an early-stage terminal user-interface library for
[Flecha](https://flecha-lang.org/). It begins with deterministic terminal
ownership and typed input events, and is intended to grow into a complete TUI
toolkit with rendering, layouts, styling, and widgets.

The current `0.1.0` development version provides the terminal foundation. It
can open a POSIX terminal in raw mode, restore it automatically, query its
dimensions, write complete byte sequences, and perform nonblocking reads. A
typed event model and ANSI keyboard decoder are implemented and tested
internally, but they are not yet connected through a public `readEvent()` API.

## Status

FletUI is under active development. The API, event representation, and package
layout may change before the first stable release.

Implemented today:

- deterministic raw-terminal acquisition and restoration;
- terminal size queries;
- complete writes with interrupted and partial-write handling;
- poll-style, nonblocking byte reads;
- typed key, modifier, resize, and terminal-size values; and
- internal decoding for ASCII control keys, CSI and SS3 navigation keys,
  alternate Home/End forms, and ANSI Shift/Alt/Control modifiers.

Still ahead:

- a buffered public `Terminal::readEvent()` API;
- UTF-8 decoding and fragmented-input buffering;
- resize-event delivery;
- rendering, styles, layouts, and widgets; and
- a non-POSIX terminal backend.

## Platform support

The current backend targets POSIX terminals through `termios`, `ioctl`, and
Unix file descriptors. Opening a terminal requires both standard input and
standard output to be attached to a TTY. Windows support has not been
implemented.

Raw reads are nonblocking: a successful read of zero bytes means that no input
is currently available, not that the terminal has reached end-of-file.

## Install

FletUI is not yet published in the Flecha package registry. Add this checkout
as a local dependency in your project's `longbow.toml`:

```toml
[dependencies]
fletui = { path = "../fletui" }
```

The dependency alias controls the library import path:

```flecha
@imports
    :: lib::fletui::fletui;
```

## Quick start

`Terminal::open()` switches the process terminal to raw mode. The returned
`Terminal` owns that state and restores the original settings automatically
when it leaves scope.

```flecha
module example;

@imports
    :: lib::fletui::fletui;

i32 main() {
    terminal, openError := fletui::Terminal::open();
    if openError != none {
        stderrln("FletUI requires an interactive terminal.");
        return 1;
    }

    writeError := terminal.writeAll("FletUI terminal ready.\r\n");
    if writeError != none {
        return 1;
    }

    size, sizeError := terminal.size();
    if sizeError != none or size.columns == 0 or size.rows == 0 {
        return 1;
    }

    [64]u8 input = none;
    _, readError := terminal.read(input[:]);
    if readError != none {
        return 1;
    }

    return 0;
}
```

The example performs one nonblocking read and exits immediately. Real
applications will poll or wait around `read()` until the public event reader is
available.

## Terminal API

The public terminal surface currently consists of:

- `Terminal::open()` — acquire the process terminal and enable raw mode;
- `Terminal::isOpen()` — report whether a terminal handle is active;
- `Terminal::size()` — return its column and row count;
- `Terminal::writeAll()` — write an entire `stringv`; and
- `Terminal::read()` — read the currently available bytes into a caller-owned
  slice.

`Terminal` is a deterministic owner. Its teardown is idempotent, restores the
saved terminal settings, and releases the native handle. A zero `Terminal` is
closed and its operations return `FletuiError.Closed`.

## Event model

`Event` has three kinds: `None`, `Key`, and `Resize`. A key event carries a
`KeyCode`, a Unicode scalar slot, and independent Shift, Alt, and Control
flags. The current key-code set includes:

- characters, Enter, Escape, Backspace, and Tab;
- Up, Down, Left, and Right;
- Home, End, Page Up, and Page Down; and
- Insert and Delete.

The internal decoder recognizes common CSI and SS3 terminal encodings,
including modified navigation sequences such as Shift+Up, Alt+Right, and
Control+Left. Malformed, incomplete, unknown, and oversized sequences are
rejected without producing an event.

This decoder is intentionally internal for now. Raw terminal reads can split
an escape or UTF-8 sequence across calls, so exposing decoded events safely
requires a persistent input buffer and an explicit incomplete-input state.

## Errors

Terminal operations return explicit errors. `FletuiError` distinguishes:

- a process without an attached terminal;
- failures while opening, sizing, writing, or reading;
- unsupported platform behavior; and
- operations attempted on a closed terminal.

Allocation failures use Flecha's standard `AllocationError.OutOfMemory` value.
Callers should handle returned errors rather than assuming that standard input
and output are interactive terminals.

## Project layout

```text
src/
├── event.fl              Typed event and key values
├── terminal.fl           Safe terminal owner and public byte I/O
└── internal/
    ├── abi.fl            POSIX native implementation
    ├── decoder.fl        ANSI keyboard decoding
    ├── errors.fl         FletUI error values
    ├── runtime.fl        Native function declarations
    └── utils.fl          Native-status translation
```

## Development

Use the Flecha and Longbow toolchain available on `PATH`:

```sh
longbow check
longbow test --verbose
longbow build
git diff --check
```

The unit suite covers event defaults and payloads, plain ASCII and control
keys, navigation variants, modifier combinations, and malformed escape
sequences. Tests do not require an interactive terminal.

## Roadmap

- [x] Deterministic POSIX raw-terminal ownership
- [x] Terminal size, complete writes, and nonblocking byte reads
- [x] Typed event representation
- [x] Tested ANSI keyboard decoder
- [ ] Buffered `Terminal::readEvent()`
- [ ] UTF-8 characters and fragmented escape sequences
- [ ] Resize events
- [ ] Retained or immediate rendering surface
- [ ] Styling, layout primitives, and reusable widgets
- [ ] Additional terminal backends

FletUI favors a small, explicit core: platform-specific unsafe code remains
behind the terminal boundary, while event decoding and future rendering logic
stay deterministic and testable without a live TTY.

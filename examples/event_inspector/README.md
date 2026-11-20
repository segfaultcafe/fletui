# FletUI event inspector

This small interactive program exercises the usable FletUI surface today:

- deterministic raw-terminal acquisition and restoration;
- complete frame writes through `Terminal::writeAll`;
- terminal-size queries and resize events;
- buffered key decoding, including navigation keys, modifiers, and UTF-8; and
- typed `Event`, `KeyEvent`, and `Size` values.

Run it from this directory in an interactive POSIX terminal:

```sh
longbow run
```

Press keys or resize the terminal to inspect the events FletUI delivers. Press
`q`, Escape, or Ctrl+C to exit. The example owns its loop directly because the
higher-level `Application` runtime is not implemented yet.

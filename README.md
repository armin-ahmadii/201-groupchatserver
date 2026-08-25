# Group Chat Server with Fuzzing Clients

A multi-threaded TCP group chat server written in C, plus a client that acts as a
custom fuzzer — it generates random messages, streams them to the server, and logs
everything it receives back.

Built for CMPT 201 (Systems Programming) at SFU.

The server accepts an arbitrary number of concurrent clients, tags every incoming
message with the sender's address, and relays it to all connected clients — including
the sender — in a globally consistent order. Shutdown is coordinated through a
simplified two-phase commit so that no client is left hanging.

## Features

- **Concurrent clients** — one POSIX thread per connection, tested up to 100 clients
  sending 100 messages each
- **Total message ordering** — every client sees every message in the same order,
  guaranteed by a single-writer broadcast thread draining a shared FIFO queue
- **Length-agnostic framing** — messages are read a byte at a time up to the `'\n'`
  delimiter, so binary address fields containing `0x0A` don't break parsing
- **Graceful shutdown** — two-phase commit protocol between server and clients
- **Custom fuzzer client** — random payloads from `getentropy(2)`, hex-encoded, with
  a dedicated receiver thread so sending and receiving happen concurrently

## Building

Requires CMake 3.20+, a C compiler, and pthreads.

```bash
cmake -S . -B build
cmake --build build
```

This produces two executables, `server` and `client`.

## Running

Start the server, giving it a port and the number of clients it should expect:

```bash
./build/server 8000 3
```

Then start clients, pointing each at the server's IP and port, with the number of
messages to send and a path for its log file:

```bash
./build/client 127.0.0.1 8000 100 client0.log
```

You can also poke at the server manually:

```bash
telnet localhost 8000
```

## Wire Protocol

Every message is at most 1024 bytes and terminated by `'\n'`. The first byte is a
`uint8_t` type tag.

**Client → server**

| Offset | Size | Field |
| --- | --- | --- |
| 0 | 1 | type (`0` = chat, `1` = done sending) |
| 1 | n | payload |
| 1+n | 1 | `'\n'` |

**Server → client** (type `0` only)

| Offset | Size | Field |
| --- | --- | --- |
| 0 | 1 | type (`0`) |
| 1 | 4 | sender IP, `uint32_t`, network byte order |
| 5 | 2 | sender port, `uint16_t`, network byte order |
| 7 | n | payload |
| 7+n | 1 | `'\n'` |

A type `1` message is just the tag followed by `'\n'` in both directions.

Because the IP and port are raw integers rather than strings, any of those six bytes
can legitimately be `0x0A`. Both sides therefore parse by field width first and only
scan for `'\n'` once they reach the free-form payload.

## Architecture

### Server

```
accept loop ──┬── client thread 0 ─┐
              ├── client thread 1 ─┼──► message queue ──► broadcaster ──► all clients
              └── client thread n ─┘      (mutex + condvar)
```

Each client thread reads one framed message at a time, rewrites type `0` messages to
include the sender's address, and appends the result to a shared linked-list queue
under a mutex, signalling a condition variable.

A single broadcaster thread waits on that condition variable, pops one message, and
writes it to every connected socket. Funnelling all outbound traffic through one
thread is what makes total ordering fall out for free — the order in which messages
land in the queue is the order every client sees them.

### Client

The main thread generates `<# of messages>` payloads: 10 random bytes from
`getentropy()`, converted to a 20-character uppercase hex string, wrapped in a type
`0` frame. After the last one it sends a type `1` frame and blocks on the receiver
thread.

The receiver thread runs for the life of the process, decoding incoming frames and
writing each to stdout and the log file as:

```
192.168.1.12   9000      9391DE3E275ADB19637D
```

using the format string `"%-15s%-10u%s\n"`. It exits when it sees a type `1` message
from the server.

### Shutdown

1. Each client finishes sending and emits a type `1` message.
2. The server counts type `1` messages. Once every expected client has checked in, it
   broadcasts a type `1` message to everyone, half-closes each socket, and exits.
3. Each client sees the type `1` message, flushes and closes its log, and exits.

## Project Structure

```
├── CMakeLists.txt
├── src/
│   ├── server.c      # accept loop, per-client threads, broadcaster
│   └── client.c      # fuzzer main thread + receiver thread
└── bin/
    ├── server-tester.{amd64,arm64}
    └── client-tester.{amd64,arm64}
```

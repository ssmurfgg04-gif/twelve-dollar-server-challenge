# c-libco - coroutine per connection, one process per core

Same HTTP, JWT, JSON and SQLite layers as the `c-iouring` submission
(`http.c`, `json.c`, `jwt.c`, `util.c` are identical files), on a different
engine: [Tencent libco](https://github.com/Tencent/libco), the coroutine
library that has run WeChat's backend since 2013.

## The shape

- `LIBCO_PROCS` processes (default 2), each with its own `SO_REUSEPORT`
  listener, its own SQLite connection, and one single-threaded libco
  event loop
- one coroutine per connection, taken from an idle pool, with shared
  stacks (128 stacks x 128KB), so physical stack memory is O(stacks)
  instead of O(connections)
- per-connection code is straight-line: `co_poll(POLLIN)` -> `read` ->
  route -> `write` (poll `POLLOUT` on EAGAIN). There is no connection
  state machine, no buffer rings, no backpressure bookkeeping
- the database layer needs no locks by construction: coroutines are
  cooperative and never yield inside a transaction, so one connection per
  process is serialized without a mutex; a transaction always commits
  before its responses are sent (the same ordering guarantee the io_uring
  version's group commit provides)
- processes share the database through WAL (`busy_timeout` 1000ms bounds
  the worst-case write-lock wait; a passive checkpoint runs every 5s from
  the stats coroutine)

## libco notes (things worth knowing before copying this)

- the vendored libco widens `g_rpchook_socket_fd` from 102400 to 1048576
  entries. Stock libco's syscall-hook fd map stops at 102400 fds per
  process; past that, hooks treat sockets as unregistered and
  blocking-style reads would park the whole thread. 8MB of BSS, no
  behavior change below 102400
- accepted fds are marked `O_NONBLOCK` through libco's `fcntl` hook, so
  `read()`/`write()` stay single raw syscalls and EAGAIN is waited out
  with `co_poll()`. This is the same syscall count as the "blocking
  style" hooks without their 1s default timeout cliff
- libco is Apache 2.0 (`src/libco/LICENSE.txt`)

## Conformance and benchmarks

42/42 on the challenge conformance suite (same handler code as
`c-iouring`). The bench history, methodology and the runner-ceiling
analysis live in the submission discussions; the interesting comparison
is that this engine holds the same load as the io_uring engine with a
fraction of the connection-layer code.

## Files

```
build.sh     SQLite amalgamation (pinned, sha-checked) + libco + server
install.sh   apt deps (build-essential, curl, unzip)
start.sh     runs bin/server with SPEC.md env vars
src/
  server.cpp   the connection layer: coroutines, slots, respond(), fork
  db.c         SQLite: statements, handlers, inline transactions
  http.c       request parsing + routing (identical to c-iouring)
  json.c       JSON scanner/escaper (identical)
  jwt.c        HS256 verification (identical)
  util.c       buffers and time (identical)
  libco/       vendored Tencent libco (Apache 2.0)
```

Environment: `HOST`, `PORT`, `SQLITE_PATH`, `JWT_SECRET` (per SPEC.md),
plus `LIBCO_PROCS`, `LIBCO_STACKS`, `LIBCO_MAXCONN`.

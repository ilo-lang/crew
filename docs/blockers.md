# Blockers

Things crew needs from ilo. Most of the original blockers have shipped; this
file now tracks what's left plus a couple of gaps the server rewrite surfaced.

## Shipped (no longer blocking)

### ILO-46 / ILO-379 / ILO-448 — HTTP server + streaming

**Status: shipped.** `ilo httpd <handler.ilo> --port N` serves a `handler`
function over HTTP/1.1, with chunked transfer-encoding for `L t` response
bodies (ILO-379). Client-side lazy stream consumers (`get-stream`,
`pst-stream`) shipped under ILO-448. ilo docs:
[`docs/streaming.md`](https://github.com/ilo-lang/ilo/blob/main/docs/streaming.md).

The server side of crew now compiles and runs against this directly - no Bun
shim. `src/server.ilo` is the live handler; run it with
`ilo httpd src/server.ilo --port 7777`.

### spawn — background loops (ILO-477)

**Status: shipped.** `spawn fn args > _` fires a fire-and-forget background
OS thread. This unblocks the daemon-style concurrency crew-agent needs (HTTP
server + SSE consumer + queue drainer in one process). The server side does
not use it; only crew-agent will.

## Open gaps surfaced by the server rewrite

### `ilo httpd` does not resolve `use` imports

`ilo httpd` lexes/parses/verifies only the single handler file. Unlike
`ilo run` and `ilo check`, it never calls the import-resolution pass, so a
handler that does `use "store_jsonl.ilo"` fails at startup with
`undefined function`. Consequence: `src/server.ilo` has to be self-contained,
so the type + jsonl-store logic is inlined there (it also lives, verified, in
the modular `types.ilo` / `store_jsonl.ilo` for `ilo check` and the
`ilo run`-based tooling). Keep the two in sync until httpd learns imports.
Worth filing upstream.

### Streaming responses are buffered, not handler-driven

`ilo httpd` accepts an `L t` body and frames it as chunked transfer-encoding,
but it materialises the whole list before writing the first byte (the List arm
in `handle_http_connection` does `items.iter().map(to_string).collect()`). So
a handler cannot hold the connection open and emit feed lines as they are
appended. `GET /events/stream` therefore returns a one-shot SSE *snapshot* of
today's feed, correctly chunk-framed, then closes - it is not a live tail.

A true file-tailing SSE stream needs httpd to accept a lazy `L t` (or a pull
callback) for the response body. Until then, crew-agent should poll
`/events?since=<ts>` rather than rely on a held-open `/events/stream`.

### crew-agent process model is unresolved

`ilo httpd` owns the process (it runs the accept loop and blocks). crew-agent
needs an MCP HTTP server *plus* a background SSE consumer / queue drainer.
With `spawn` we can launch background loops, but `ilo httpd`'s own loop is the
foreground - so either crew-agent runs httpd and spawns the loops as threads
(unverified that spawn co-exists cleanly with the httpd accept loop), or it
runs as a second process alongside httpd. This is the open question that
defers `src/agent.ilo` / `src/cache.ilo` to a separate task.

## TCP / RESP client (no tracking issue yet)

**What's missing:** ilo has HTTP client builtins but no raw TCP socket. Blocks
the Redis storage adapter (`src/store_redis.ilo` errors loudly until then).

**Crew's impact:** small for v1 - the JSONL adapter is fine. If contention or
pubsub demands push us toward Redis, we'll either add a TCP client to ilo or
shell out to `redis-cli` via `run`.

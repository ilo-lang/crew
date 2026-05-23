# Blockers

Things crew needs from ilo that don't exist yet.

## ILO-46 — HTTP streaming + `ilo httpd` server subcommand

**Status:** roadmap placeholder upstream. Tracked at <https://linear.app/ilo-lang/issue/ILO-46>. ilo docs: [`docs/streaming.md`](https://github.com/ilo-lang/ilo/blob/main/docs/streaming.md).

**What crew needs that ILO-46 ships:**

- `ilo httpd <handler.ilo> --port N` with `fn handle (req:Request) > Response`
  - `crew-server` and `crew-agent` are both HTTP servers. Without this, ilo can't serve HTTP at all.
- `get-stream url > L t` (lazy SSE iterator, `@chunk stream {...}`)
  - `crew-agent` subscribes to `crew-server`'s `/events/stream` to keep the local mirror fresh.
- `get-stream-h`, `pst-stream`, `pst-stream-h` for custom headers + streaming POST.
  - We need bearer-token auth on the SSE subscription.

**Crew's relationship to ILO-46:** crew is the load-bearing dogfood case. Every day ILO-46 isn't merged is a day crew runs through a shim instead of through ilo end-to-end. Use crew as motivation to prioritise the ticket.

**Workaround until ILO-46 lands:** [`shim/`](../shim/) (TBD) — a thin Bun HTTP server that hands each request to an ilo handler via stdin/stdout. Deleted the day ILO-46 merges.

## Concurrency primitives (no tracking issue yet)

**What's missing:** ilo has no `spawn`, no green threads, no async runtime. `par-map` exists for parallel fan-out over a fixed list but not for long-lived background loops.

**Why crew needs this:** `crew-agent` runs three loops concurrently:
1. MCP HTTP server (foreground)
2. SSE consumer (background, long-lived)
3. Write-behind queue drainer (background, ticker)

**Current workaround:** structure the daemon as a single-threaded event loop where the HTTP server polls the queue between requests and the SSE connection is interleaved. Workable but ugly. Worth filing as a follow-up to ILO-46.

## TCP / RESP client (no tracking issue yet)

**What's missing:** ilo has HTTP client builtins but no raw TCP socket. Blocks the Redis storage adapter.

**Crew's impact:** small for v1 — the JSONL adapter is fine. If/when contention or pubsub demands push us toward Redis, we'll either add a TCP client to ilo or shell out to `redis-cli` via `run`.

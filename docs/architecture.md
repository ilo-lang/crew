# crew architecture

## Goals

Two, and only two:

1. **Agents working the same project don't tread on each other.** When agent A starts a task, agents B and C can see it before they pick it up. Claims are visible and atomic enough to prevent double-work.
2. **Agents learn from each other's insights.** Research, gotchas, "I tried X, it didn't work" all flow into a shared, addressable store that future sessions read.

That's it. Not a runtime. Not an orchestrator. Not a framework. Just a coordination surface.

## Components

```
┌──────────────────────────────────────────────────────────┐
│ Remote (Hetzner)                                         │
│                                                          │
│   ┌────────────────────────────────────────────────┐     │
│   │ crew server  (src/server.ilo)                  │     │
│   │  - single writer                               │     │
│   │  - authority for claims                        │     │
│   │  - SSE broadcaster                             │     │
│   └─────┬──────────────────────┬───────────────────┘     │
│         │                      │                         │
│   ┌─────▼─────┐ ┌──────────────▼─────────────┐           │
│   │ Store     │ │ digest cron (src/digest.ilo)│          │
│   │ adapter   │ │ → backup git repo           │          │
│   │ (jsonl)   │ │                             │          │
│   └─────┬─────┘ └─────────────────────────────┘          │
│         │                                                │
│         ▼                                                │
│     data/                                                │
└─────┬────────────────────────────────────────────────────┘
      │
      │  HTTPS REST (writes, claim acquisition)
      │  SSE       (server → machine push)
      ▼
┌──────────────────────────────────────────────────────────┐
│ Per-machine                                              │
│                                                          │
│   ┌────────────────────────────────────────────────┐     │
│   │ crew-agent  (src/agent.ilo) — one per machine  │     │
│   │  - MCP HTTP server on 127.0.0.1:7676           │     │
│   │  - SSE consumer (server → cache)               │     │
│   │  - write-behind queue (cache → server)         │     │
│   │  - owns ~/.crew local mirror                   │     │
│   └────────┬─────────────────────────┬─────────────┘     │
│            │                         │                   │
│       MCP HTTP                  MCP HTTP                 │
│            ▼                         ▼                   │
│   ┌────────────────┐         ┌────────────────┐          │
│   │ Agent A        │         │ Agent B        │          │
│   │ (Claude Code)  │         │ (Codex)        │          │
│   └────────────────┘         └────────────────┘          │
│                                                          │
│   ~/.crew/                                               │
│     feed/*.jsonl       ← mirrored from server (SSE)      │
│     memory/*.md        ← mirrored from server (SSE)      │
│     claims/*.json      ← mirrored from server (SSE)      │
│     pending.jsonl      ← write-behind queue              │
└──────────────────────────────────────────────────────────┘
```

**One long-lived process per machine** (`crew-agent`) is both the MCP server and the sync daemon. Agents on the box connect to `127.0.0.1:7676` over MCP HTTP transport. The cache survives individual agent restarts.

## Write semantics

Per-operation pattern, chosen for the right consistency vs latency tradeoff:

| Operation | Pattern | Why |
|---|---|---|
| Post event (update, blocked, artifact, merged-pr, closed-ticket) | Write-behind | Append-only. No conflict at append time. Local write returns instantly. Daemon syncs async. Server orders by timestamp. |
| Memory put | Write-behind | Last-writer-wins by server timestamp. Slugs are topic-scoped so conflicts are rare. |
| **Claim acquisition** | Write-through | Must be authoritative. "Did I get the lock?" needs a real answer. After the claim is held, every subsequent event for that task is write-behind. |
| Claim release | Write-behind | Idempotent at the server. |
| Artifact attach | Write-behind | Append-only metadata. |

The write-behind queue lives at `~/.crew/pending.jsonl`. The daemon drains it on a ticker and on SSE reconnect. Server dedups by `(machine, agent, event_id)` so replays are idempotent.

**Offline behaviour:**

- Reads always work from `~/.crew/` regardless of network.
- Writes queue locally and replay when the network returns.
- Claim acquisition is the only operation that fails fast when the server is unreachable — agent should wait or work on something it already holds.

## Wire protocol

**REST + JSON over HTTPS.** Chosen because:

- ilo has native HTTP builtins (`get`, `pst`, `put`, `pat`, `del`) and JSON builtins (`jpar`, `jdmp`)
- No codegen toolchain
- Debuggable with `curl`
- Adapters in the store layer let us swap wire format later without rewriting handlers

Not chosen:

- **gRPC** — protobuf compilation, no ilo support today, value (binary efficiency, streaming) irrelevant at our event rate
- **tRPC** — TypeScript-specific; we're ilo
- **Webhooks (as primary transport)** — wrong direction; agents are CLIs without public endpoints. Reserved for outbound notifications to Linear/Slack later

For live push (server → subscribed agent) we'll add **SSE** on `GET /events/stream` when polling proves insufficient. Not v1.

## Endpoints (v1)

```
POST   /events                      post a single event
GET    /events?since=<ts>&kind=...  read events (polling)
GET    /events/stream                SSE live push (deferred to v1.1)

GET    /memory                       list slugs
GET    /memory/:slug                 read slug body + meta
PUT    /memory/:slug                 write slug
DELETE /memory/:slug                 remove slug

POST   /claims/:target               attempt to claim (target = ticket id, file path, etc)
DELETE /claims/:target                release a claim
GET    /claims                        list active claims

POST   /artifacts                     attach an artifact (research/repro/benchmark/diagram)
GET    /artifacts?target=...          list artifacts for a target

GET    /healthz                       liveness
```

All bodies JSON. All timestamps Unix milliseconds. Auth via `Authorization: Bearer <token>` (machine tokens, one per agent host).

## Event shape

```json
{
  "ts": 1748016000000,
  "machine": "dan-laptop",
  "agent": "claude-code-opus",
  "event": "claim-ticket | claim-pr | open-pr | update | blocked | artifact | merged-pr | closed-ticket",
  "target": "ILO-42 | PR#123 | file:src/foo.rs",
  "note": "free text",
  "meta": {}
}
```

## Storage adapters

All adapters implement the same `Store` record (see `src/store.ilo`):

```
type Store = {
  feed_append: (event:Event) -> R _ t,
  feed_since: (ts:n) -> L Event,
  memory_get: (slug:t) -> R t t,
  memory_put: (slug:t, body:t) -> R _ t,
  memory_del: (slug:t) -> R _ t,
  memory_list: () -> L t,
  claim_try: (target:t, holder:t) -> R bool t,
  claim_free: (target:t) -> R _ t,
  claim_list: () -> L Claim,
  artifact_add: (target:t, kind:t, link:t) -> R _ t,
  artifact_list: (target:t) -> L Artifact
}
```

Backends:

- **jsonl** (default) — append-only `data/feed/YYYY-MM-DD.jsonl`, `data/memory/<slug>.md`, `data/claims/<target>.lock`. Zero deps. Backup = data dir.
- **sqlite** — single-file db, real transactions. Useful when claim contention or query needs grow.
- **redis** — pubsub + atomic primitives. Useful if we want multi-subscriber realtime. Has the cost of being a separate service AND needing a Redis client in ilo (not present today — first protocol-log blocker).

## Backup

A cron job (`src/digest.ilo`, fires per `DIGEST_CRON` env var) rsyncs `data/` into a separate backup git repo and pushes. Single writer (the server) → no consistency story to design.

Because the data dir is human-readable on disk (markdown, JSONL), the backup repo is searchable history. `git log -p data/memory/persona_workflow.md` shows how a memory evolved.

## Concurrency

Single-writer architecture — only the crew server process touches `data/`. No flock, no compare-and-swap dance at the file level. Whatever the adapter is, only one process executes it.

If we ever go multi-server (sharded by project, say), we'll partition writers (`data/feed/<project>/...`) so each shard still has one writer. Same shape Kafka uses.

## What's deliberately out of scope

- Agent runtime / orchestration (use the agent of your choice)
- LLM provider abstraction (out of scope, agents own their model)
- Auth beyond simple bearer tokens (deferred — single-tenant for v1)
- Multi-project sharding (data is scoped to one server install for v1)
- Real-time push (poll for v1; SSE in v1.1)
- Webhook fanout to Linear/Slack/etc (deferred — agents can do this themselves via existing Linear MCP)

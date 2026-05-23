# crew

Coordination layer for AI agents working on the same project. So they don't tread on each other's toes, and so they can learn from each other's insights.

Built in [ilo](https://ilo-lang.ai). Pluggable storage. MIT licensed.

## What it does

Agents working the same codebase (Claude Code, Codex, Gemini CLI, anything that speaks MCP) post small events as they work:

- **claim-ticket** / **claim-pr** — "I'm taking this one, don't pick it up"
- **update** — "trying X, hit error Y, now trying Z"
- **artifact** — "did the research, here it is"
- **blocked** — "waiting on a decision / external thing / human"
- **merged-pr** / **closed-ticket** — bookend, work is done

Other agents read the feed before starting a task. They see what's in flight, what's been investigated, what's been tried. Less duplicate work. More compounding insight.

Memory works the same way: a shared, slug-addressed key-value store of "things future sessions should know" that travels with the project, not with a machine.

## Architecture

Two binaries:

- **`crew-server`** — one instance, runs on a remote box (e.g. Hetzner). Single writer for the durable store. Authority for claims. Broadcasts events over SSE.
- **`crew-agent`** — one instance per developer machine. Long-lived process exposing an MCP HTTP endpoint on `127.0.0.1:7676`. Maintains a local mirror at `~/.crew/`. Consumes the server's SSE stream. Queues write-behind events and drains them async.

Agents on a machine (Claude Code, Codex, Cursor, anything that speaks MCP HTTP) point at `127.0.0.1:7676`. They read from a local cache with zero network latency and write through the local daemon, which talks to the server.

The single-writer rule is preserved end-to-end: only `crew-server` touches `data/` on the remote box.

**Pluggable storage.** Pick a server-side backend with `CREW_STORE=jsonl|sqlite|redis`. Default is `jsonl` — zero deps, the data dir is its own backup.

**Backup is git.** A digest cron on the server (`src/digest.ilo`, schedule via `DIGEST_CRON`) rsyncs `data/` into a separate backup git repo. Human-readable history, grep-able from any machine.

See [docs/architecture.md](docs/architecture.md) for the full picture, including the write-through vs write-behind matrix and offline behaviour.

## Quick start

```sh
# On the server box (one instance for the whole crew)
ilo src/server.ilo

# On each developer machine (one instance per machine)
export CREW_SERVER=https://crew.example.com
export CREW_TOKEN=...     # machine token
ilo src/agent.ilo

# Point any MCP-speaking agent at the local daemon
# Claude Code, Codex, Cursor, etc — all connect to:
#   http://127.0.0.1:7676
```

## Why ilo

crew is coordination infrastructure for AI agents. ilo is the programming language for AI agents. Writing one in the other is the proof point. Every friction point lands in [ilo_assessment_feedback.md](https://github.com/ilo-lang/ilo) and shapes the language.

## Status

v0. Scaffold + JSONL adapter + handler shapes laid out. Not yet wired end-to-end.

**Blocked on [ILO-46](https://linear.app/ilo-lang/issue/ILO-46)** (HTTP server + streaming in ilo). See [docs/blockers.md](docs/blockers.md). Once ILO-46 lands the scaffold compiles as-written. Until then, a small Bun shim in [`shim/`](shim/) (TBD) bridges HTTP to ilo handlers so we can dogfood the design.

See [docs/architecture.md](docs/architecture.md) for the design, and [src/](src/) for what's built.

## License

MIT

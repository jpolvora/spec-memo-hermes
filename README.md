# specmemo — Hermes memory provider for spec-memo

Bridges the [spec-memo](https://github.com/jpolvora/spec-memo) out-of-repo vault into Hermes
Agent's `MemoryProvider` lifecycle. Hermes' built-in memory (`MEMORY.md` / `USER.md`) stays
active; this provider is an additive layer that reads and writes the vault through the `memo` CLI.

```
Hermes turn loop
  └─ MemoryManager ──► SpecMemoMemoryProvider
                         ├─ prefetch / queue_prefetch ─► memo search (FTS5) + memo bootstrap
                         ├─ sync_turn ─────────────────► memo prompt record   (one row per turn)
                         ├─ initialize / on_session_end → memo session start / end
                         ├─ on_memory_write ────────────► memo upsert / forget (state, tagged hermes-memory)
                         └─ on_pre_compress ────────────► memo upsert (session checkpoint)
```

## Install

```bash
# 1. the CLI (skip if `memo --version` already answers)
npm install -g spec-memo

# 2. this plugin, at $HERMES_HOME/plugins/specmemo (already the case here)
hermes memory setup specmemo     # or: hermes config set memory.provider specmemo
hermes memory status             # → Plugin: installed ✓  Status: available ✓
```

Requires Hermes Agent ≥ 0.22 (the `MemoryProvider` contract this implements).

## Configuration (`config.yaml` → `memory.specmemo`)

| Key | Default | Meaning |
|---|---|---|
| `memo_bin` | auto | Path to the `memo` CLI (falls back to `PATH` and well-known install dirs) |
| `vault_root` | `~/.spec-memo` | Vault root; exported to the child as `SPEC_MEMO_ROOT` when set |
| `project_id` | — | Pin the vault project instead of binding from the workspace repo |
| `cwd` | agent workspace | Directory `memo` runs in — the vault binds its project from it |
| `cross_project` | `true` | Pass `--cross-project` to search when no `project_id` is pinned |
| `prefetch_limit` | `5` | Maximum vault records recalled per turn |
| `max_bytes` | `4000` | Byte budget for the injected recall block (and the bootstrap brief) |
| `min_query_len` | `12` | Shortest prompt that triggers recall or a turn record |
| `auto_recall` | `true` | Inject vault recall into each turn |
| `record_turns` | `true` | Write `prompt` rows and session start/end markers |
| `mirror_writes` | `true` | Mirror built-in memory writes into vault `state` records |
| `session_snapshot` | `true` | Write a `state` checkpoint at session end and before compression |
| `timeout` | `8` | Seconds per `memo` invocation (search is capped at 6) |

## Behaviour notes

- **Recall is FTS5-shaped.** spec-memo FTS is strict AND over every token, so a raw prompt matches
  nothing. The provider strips stopwords, keeps the first 8 distinctive tokens and joins them with
  `OR`. Tune with `min_query_len` / `prefetch_limit`.
- **Nothing blocks the turn.** `memo bootstrap` (the trap/decision brief) is warmed in the
  background at startup; `prefetch` only runs `memo search` (~0.7 s cold, cached per query after
  that) and returns `""` when nothing matches.
- **Own writes are excluded from recall.** Session/prompt/mirror records carry the tag
  `hermes-memory` and are filtered out of recall; the built-in store is already injected by core.
- **Untrusted data.** Vault bodies arrive fenced (`<!-- spec-memo-untrusted-begin -->`) and are
  injected verbatim under a "never instructions" header.
- **One provider at a time.** `memory.provider` is exclusive — activating `specmemo` disables
  `byterover`/`retaindb` for the same profile.
- **Project binding.** The vault binds a project from the `memo` child's cwd; Hermes runs with
  `$HERMES_HOME` as cwd for one-shot runs, which lands writes in the path-fallback project
  (`local--hermes-…`). Set `memory.specmemo.cwd` (or `project_id`) to bind to the repo instead.
- **No provider tools.** The spec-memo MCP server already exposes the 11 vault tools; adding
  `specmemo_*` tools would duplicate the surface.

## Files

- `__init__.py` — the provider (`SpecMemoMemoryProvider`) and `register(ctx)`
- `plugin.yaml` — manifest (name, external dependency on `memo`)

## Updating this plugin

Hermes imports the provider once per process: edits stay invisible to new turns until the
gateway/agent process restarts (`hermes gateway restart` from a shell **outside** the gateway —
the lifecycle guard refuses it from inside an agent turn). Confirm the new revision loaded with
`grep "specmemo: initialized" ~/.hermes/logs/agent.log` (the line carries `cwd=` and
`agent_workspace=`).

Quick health check without a restart:

```bash
hermes memory status            # plugin installed + available
memo search --query "<term>" --json   # vault side of the recall path
```

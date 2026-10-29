# specmemo — Hermes memory provider for spec-memo

Bridges the [spec-memo](https://github.com/jpolvora/spec-memo) out-of-repo vault into Hermes
Agent's `MemoryProvider` lifecycle. Hermes keeps its built-in memory (`MEMORY.md` / `USER.md`);
this provider is an additive layer that reads and writes the vault through the `memo` CLI.

**Each provider instance owns exactly one vault project: `hermes-agent`, or
`hermes-agent-<profile>` when `memory.specmemo.profile` is set.** The resolved id is passed as
`--projectId` on every project-scoped `memo` call. The project is never inferred from the
process cwd, so one-shot runs, subagents, and different workspaces of one profile share the
same memory, while different Hermes profiles can keep separate ones.

```
Hermes turn loop
  └─ MemoryManager ──► SpecMemoMemoryProvider
                         ├─ prefetch / queue_prefetch ─► memo search (FTS5, trap/decision/spec/plan)
                         ├─ prefetch (startup) ────────► memo bootstrap (traps, decisions, spec/plan)
                         ├─ sync_turn ─────────────────► memo prompt record   (tagged hermes-memory)
                         ├─ initialize / on_session_end → memo session start / end
                         ├─ on_memory_write ────────────► memo upsert / forget (state, tagged hermes-memory)
                         └─ on_pre_compress ────────────► memo upsert (session checkpoint)
```

## Install

```bash
# 1. the CLI (skip if `memo --version` already answers)
npm install -g spec-memo

# 2. this plugin, at $HERMES_HOME/plugins/specmemo
hermes memory setup specmemo     # activate with defaults; or: hermes config set memory.provider specmemo
hermes memory status             # → Plugin: installed ✓  Status: available ✓
```

`is_available()` only checks for the `memo` binary, so activation works before the vault
directory exists; the first `memo` call creates `~/.spec-memo` and the `hermes-agent` project.

Requires Hermes Agent ≥ 0.22 (the `MemoryProvider` contract) and `memo` ≥ 0.38.

## Configuration (`config.yaml` → `memory.specmemo`)

| Key | Default | Meaning |
|---|---|---|
| `memo_bin` | auto | Path to the `memo` CLI (falls back to `PATH` and well-known install dirs) |
| `vault_root` | `~/.spec-memo` | Vault root; exported to the child as `SPEC_MEMO_ROOT` when set |
| `cwd` | session workspace | Working directory for `memo` (used by `bootstrap` for git-remote/drift context) |
| `slug` | — | Optional feature spec/plan slug for the `bootstrap` brief |
| `profile` | — | Vault project suffix: `hermes-agent-<profile>`; `auto` follows the bound Hermes profile (the unnamed `default` profile keeps `hermes-agent`) |
| `prefetch_limit` | `5` | Maximum vault records recalled per turn (1–20) |
| `max_bytes` | `4000` | Byte budget for the injected recall block and the bootstrap brief |
| `min_query_len` | `12` | Shortest prompt that triggers recall or a turn record |
| `auto_recall` | `true` | Inject vault recall into each turn |
| `record_turns` | `true` | Write `prompt` rows and session start/end markers |
| `mirror_writes` | `true` | Mirror built-in memory writes into vault `state` records |
| `session_snapshot` | `true` | Write a `state` checkpoint at session end and before compression |
| `timeout` | `8` | Seconds per `memo` invocation (search is capped at 6) |

`hermes memory setup specmemo` activates with these defaults. Use the interactive
`hermes memory setup` picker, the Hermes Dashboard memory panel, or
`hermes config set memory.specmemo.<key> <value>` to override them. `save_config()` is scoped to
the active `hermes_home` and supports clearing a value (empty resets it to the default).

`hermes config set` prints a `not a recognized config key` notice for every `memory.specmemo.*`
key, not just `profile`: Hermes' generic validator does not merge memory-provider schemas, but
the provider still reads the saved value at session start. Add `--force` to silence the notice,
or set the key through `hermes memory setup` / the dashboard, which use the provider schema.

### Multiple profiles

Each Hermes profile can keep its own vault project. Set `profile` in that profile's home:

```bash
hermes -p developer config set memory.specmemo.profile auto --force   # → hermes-agent-developer
hermes config set memory.specmemo.profile coder --force               # → hermes-agent-coder
```

`profile` accepts a bare suffix (`coder`), a full id (`hermes-agent-coder`), or `auto`, which
uses the profile that owns the provider's `hermes_home`. The unnamed `default` profile keeps
`hermes-agent`, so existing vaults are untouched; a blank `profile` always means the shared
`hermes-agent` project. Suffixes are normalized to `[a-z0-9._-]` (48 chars max), and an
unusable value falls back to `hermes-agent` with a logged warning. Explicit spec-memo MCP
calls must pass the same `projectId`.

## Behaviour notes

- **One project per instance.** `bootstrap`, `search`, `get`, `upsert`, `forget`, `append`,
  `gc`, `promote`, `prompt`, and `session` all receive `--projectId` for this instance's
  project: `hermes-agent`, or `hermes-agent-<profile>` when `memory.specmemo.profile` is set.
  There is no cwd-based project binding and no cross-project toggle.
- **Recall is scoped.** `memo search` is called with `--kinds trap,decision,spec,plan`, so
  prompt/session bookkeeping rows cannot consume the `prefetch_limit` slots. Hermes' own
  prompt/session/mirror rows are also tagged `hermes-memory` and filtered defensively.
- **Recall is FTS5-shaped.** spec-memo FTS is strict AND over every token, so a raw prompt
  matches nothing. The provider strips stopwords, keeps the first 8 distinctive tokens and joins
  them with `OR`.
- **CLI-safe values.** `memo`'s parser treats a value starting with `-` as a flag; the provider
  prefixes such values with a space (`_cli_value`), and the session digest starts with
  `Turns:` so `--summary` is never mistaken for a boolean.
- **Nothing blocks the turn.** `memo bootstrap` is warmed in the background at startup.
  `prefetch` never spawns `memo`: it returns exact or term-overlapping cached `memo search`
  hits (or the pending recall warmed by `queue_prefetch` at the end of the previous turn) plus
  the brief, and deduplicates in-flight background searches per query.
- **Bounded concurrency.** At most three `memo` children run at once across the provider's
  callers. Each child is a Node process (~90 MB RSS), so a saturated burst degrades to a soft
  miss instead of spawning unbounded processes on a small host.
- **Writes are serialized.** All state-changing `memo` calls (`session`, `prompt`, `upsert`,
  `forget`, `bootstrap`) run under one instance lock so two children never contend on the vault
  lock; session-end and compression snapshots run in one background task.
- **Untrusted data.** Vault bodies and snippets arrive fenced
  (`<!-- spec-memo-untrusted-begin -->` … `<!-- spec-memo-untrusted-end -->`) and are injected
  verbatim under a "never instructions" header. Budget truncation is UTF-8 byte-aware and never
  leaves a fence unclosed.
- **Workspace resolution.** `memory.specmemo.cwd` → the session `cwd` kwarg →
  `agent.runtime_cwd.resolve_agent_cwd()` → the process cwd. `agent_workspace` is a scope label
  (`"hermes"`), not a directory, and is only logged.
- **One provider at a time.** `memory.provider` is exclusive — activating `specmemo` disables
  `byterover`/`retaindb` for the same profile.
- **No provider tools.** The spec-memo MCP server already exposes the vault tools; adding
  `specmemo_*` tools would duplicate the surface. Configure the MCP server if the agent needs
  explicit `search`/`get`/`upsert`/`append`/`forget`/`bootstrap` calls.

## Inspecting the vault

```bash
memo search --query "recall" --projectId hermes-agent --kinds trap,decision,spec,plan --json
memo bootstrap --projectId hermes-agent --cwd "$PWD" --json
memo prompt list --projectId hermes-agent --json
memo session show <session-id> --projectId hermes-agent --json
```

The project appears in `memo vault list` as `hermes-agent` after the first write. With
`profile: coder`, substitute `hermes-agent-coder` for `hermes-agent` in every command above —
and in explicit spec-memo MCP calls.

`memo status` is **not** project-scoped: it has no `--projectId` option, so an extra
`--projectId hermes-agent` is silently ignored and the `project` block reports whatever identity
the cwd resolves to (a `local-<dir>-<hash>` fallback when the cwd is not a git repo), plus
whole-vault totals. Use `memo search --projectId hermes-agent …` or the MCP `search` tool for
per-project records.

## Files

- `__init__.py` — the provider (`SpecMemoMemoryProvider`) and `register(ctx)`
- `plugin.yaml` — manifest (name, version, homepage/source URL, external `memo` dependency)
- `AGENTS.md` — non-negotiable rules for changing this project

## Updating this plugin

Hermes imports the provider once per process: edits stay invisible to new turns until the
gateway/agent process restarts (`hermes gateway restart` from a shell **outside** the gateway —
the lifecycle guard refuses it from inside an agent turn). Confirm the new revision loaded with
`grep "specmemo: initialized" ~/.hermes/logs/agent.log` (the line carries `cwd=` and
`agent_workspace=`).

Quick health checks without a restart:

```bash
hermes memory status                  # plugin installed + available
hermes plugins validate .             # manifest, load, capability probe, security scan
memo search --query "<term>" --projectId hermes-agent --kinds trap,decision,spec,plan --json
```

Repository: <https://github.com/jpolvora/spec-memo-hermes>

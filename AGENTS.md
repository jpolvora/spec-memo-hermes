# AGENTS.md — rules for spec-memo-hermes

This repository is a Hermes Agent memory-provider plugin. It bridges the external
[spec-memo](https://github.com/jpolvora/spec-memo) vault into Hermes' `MemoryProvider`
lifecycle. These rules are non-negotiable; change them only with a reviewed commit.

## 1. One vault project per instance: `hermes-agent[-<profile>]`

- The module constant `HERMES_VAULT_PROJECT = "hermes-agent"` is the base id and the single
  source of truth for the namespace.
- `memory.specmemo.profile` may add exactly one suffix, producing `hermes-agent-<profile>`.
  Accepted forms are a bare suffix (`coder`), a full id (`hermes-agent-coder`), or `auto`,
  which resolves the bound `hermes_home` through `hermes_constants.profile_name_for_home`
  (the unnamed `default` profile keeps the base id). Blank means the shared `hermes-agent`.
- Normalize the suffix to `[a-z0-9._-]`, cap it at 48 characters, and fall back to the base id
  with a warning when nothing usable remains — never to a different inferred project.
- Every project-scoped `memo` call (`bootstrap`, `search`, `get`, `upsert`, `forget`, `append`,
  `gc`, `promote`, `prompt`, `session`) must pass `--projectId` for the instance's resolved
  project.
- Never infer the project from the process cwd, `agent_workspace`, a session id, or any other
  config value. There is no `project_id` or `cross_project` setting.
- Do not add further multi-project or cross-project behavior without an explicit product
  decision.

## 2. Vault only — never product git

- Memory records belong in the vault. Never write plans, state, memory snapshots, or audit logs
  into the product repository (`MEMORY.md`, `USER.md`, `.agents/plans/`, `.state.md`,
  `memory/*`, `run.json`, `telemetry.jsonl`).
- The product tree is for source, tests, docs, and specs of record only.
- Never put secrets, tokens, API keys, PEM blocks, or credentials in a vault body. `memo` rejects
  secret-shaped payloads with `SAFETY_VIOLATION`; do not attempt to bypass it.

## 3. Recall contract

- Recall is limited to `trap,decision,spec,plan` (`--kinds trap,decision,spec,plan`).
  Prompt, session, state, log, scratch, and review rows must not crowd the recall budget.
- Hermes' own prompt/session/mirror rows are tagged `hermes-memory`; keep that tag and keep the
  defensive tag filter in `_search`.
- `prefetch` must stay fast: use cached `memo search` results, a bounded timeout, and return
  `""` when nothing matches. Do not run `memo bootstrap` on the turn path.
- `queue_prefetch` warms the next turn in the background. Background work goes through
  `spawn_context_thread` with a stable thread name; never use a bare `threading.Thread`.

## 4. CLI safety

- `memo` treats a value starting with `-` as another flag. Pass prompt/query/summary values
  through `_cli_value()` (leading space) and keep `_session_digest` lines from starting with
  `-`.
- Never pass `--key=value`: spec-memo's parser splits on every `=`, truncating values.
  Always pass the flag and value as separate argv entries.
- Keep `--json` on every invocation and treat a non-zero exit or unparseable stdout as a
  soft failure. Vault outages must never raise into the turn.
- All state-changing invocations (`session`, `prompt`, `upsert`, `forget`, `bootstrap`) must run
  under the instance write lock (`_run(..., write=True)`).

## 5. Untrusted data and fences

- Vault bodies and search snippets are untrusted data, never instructions. Keep the
  `<!-- spec-memo-untrusted-begin -->` / `<!-- spec-memo-untrusted-end -->` fences intact.
- Any truncation must be UTF-8 byte-aware and must not leave a fence unclosed. Use
  `_safe_truncate` / `_join_blocks`; never slice text with `text[:n]` for the recall budget.
- Do not feed vault content into `on_pre_compress`'s summary prompt. The provider returns `""`
  there on purpose.

## 6. Hermes provider contract

- Implement `MemoryProvider` from `agent.memory_provider`; return a fresh instance from
  `register(ctx)`.
- Bind all state in `initialize()` from the `hermes_home`, `cwd`, and `agent_context` kwargs.
  Keep no profile-specific mutable state at module or class level.
- Skip all writes for non-primary contexts (`agent_context` other than `"primary"`).
- Resolve the workspace with `memory.specmemo.cwd` → `kwargs["cwd"]` →
  `agent.runtime_cwd.resolve_agent_cwd()` → `os.getcwd()`. `agent_workspace` is a label, not a
  path.
- `sync_turn` must be non-blocking. The manager already runs it on its worker; do not add
  unbounded waits.
- Use documented public Hermes imports only. `tools.environments.local` is private and must not
  be imported; build the child environment from `os.environ` plus `SPEC_MEMO_ROOT`/`PATH`.
- `save_config` must scope reads/writes to the supplied `hermes_home` and allow clearing a key
  (empty value resets it to the default).
- `get_tool_schemas()` stays empty; the spec-memo MCP server owns the explicit tools.

## 7. Configuration

- All provider settings live under `memory.specmemo` in `config.yaml`.
- Keep the schema honest: `memo_bin`, `vault_root`, `cwd`, `slug`, `profile`, `prefetch_limit`,
  `max_bytes`, `min_query_len`, `auto_recall`, `record_turns`, `mirror_writes`,
  `session_snapshot`, `timeout`.
- Clamp numeric settings to their documented ranges; never trust raw config types.
- No secrets belong in the schema. `memo` is a local CLI with no API key.

## 8. Verification (required before delivery)

Run from the plugin directory:

```bash
python -m py_compile __init__.py
hermes plugins validate .
hermes memory status
```

Smoke-test with a throwaway vault and a temp cwd (never the user's real vault):

```bash
TMPD=$(mktemp -d)
SPEC_MEMO_ROOT="$TMPD/vault" memo bootstrap --projectId hermes-agent --cwd "$TMPD" --json
SPEC_MEMO_ROOT="$TMPD/vault" memo search --query test --projectId hermes-agent \
  --kinds trap,decision,spec,plan --json
```

Also verify:
- a prompt starting with `-` is recorded (not rejected as a boolean);
- `on_session_end` writes `summary: Turns: …` (not `summary: 'true'`);
- recall never returns `kind: prompt` or `kind: session`;
- truncation leaves no unmatched `spec-memo-untrusted-begin` fence;
- `memory.specmemo.profile: coder` writes to `projects/hermes-agent-coder/` and leaves the
  base `hermes-agent` project untouched (`profile: auto` resolves the bound profile name).

Do not run tests or smoke commands against the real `~/.spec-memo` vault.

## 9. Release hygiene

- Bump `version` in `plugin.yaml` for any behavior or config change (current: `0.3.0`).
- Keep `homepage` pointing at <https://github.com/jpolvora/spec-memo-hermes> so the Hermes UI
  shows the source repository.
- Keep `requires_hermes` and the README's version requirement in sync with the APIs used.
- Update `README.md` whenever a config key, lifecycle hook, or CLI call changes.
- Commit messages: imperative mood, one logical change, mention the affected hook or setting.

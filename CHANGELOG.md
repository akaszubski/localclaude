# Changelog

All notable changes to **`akaszubski/localclaude`**, the main repo for the Apple-Silicon Claude Code stack. Sister repos (`akaszubski/vllm-mlx` fork, `akaszubski/searxng-mcp`) have their own commit histories.

Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/). Versioning: dated, since this is an orchestration repo not a library.

## [unreleased] — 2026-04-27

This repo absorbed the previous `akaszubski/local-claude-code-mlx` umbrella (now archived) and is the single user-facing entry point for the stack.

### Added (from the absorbed umbrella)

- **`README.md`** — full overview, install, daily flow, perf knobs, model storage, multi-Mac options, commands reference, profiles table, allowlist presets.
- **`ARCHITECTURE.md`** — process layout, port allocations, lifecycle of `localclaude start`, request data flow, fork-patches inventory, state-on-disk inventory, known constraints / upstream issues, security posture, failure-mode recovery.
- **`AGENTS.md`** — instructions for AI agents installing or modifying this stack. Names the predictable agent traps explicitly (PyPI install, Linux/Intel, skip-container, improvise-deps, `brew install claude` GUI trap, `~/.cache/huggingface/hub/` use). Single-Mac vs multi-Mac default guidance.
- **`install.sh`** at the root — Mac-only, idempotent, 8-phase one-shot install:
  1. Pre-flight (Apple Silicon + macOS + RAM)
  2. System deps via Homebrew (git, python3.10+, claude-code cask, OrbStack cask)
  3. Sister repos cloned as siblings of this repo (`vllm-mlx` fork, `searxng-mcp`)
  4. Python deps (`pip install -e ./vllm-mlx` with PEP 668 fallback; `searxng-mcp/.venv` for MCP runtime)
  5. OrbStack engine + `localclaude-searxng` container
  6. Register `searxng` MCP server with Claude Code (`claude mcp add`)
  7. Append PATH export to shell rc (with prompt)
  8. Run `localclaude doctor` to verify

  Flags: `--yes` `--no-mcp` `--no-path` `--no-container` `--dry-run`.
- **`.env.example`** — every relevant env var documented in one place.
- **`.github/workflows/install-test.yml`** — CI runs `install.sh` on fresh `macos-14` + `macos-15` Apple Silicon runners on every install.sh push, on PRs, and via manual dispatch. Catches PEP 668, locale/UTF-8, missing-dep regressions on fresh Macs.

### Changed

- **`localclaude` script** in this repo:
  - Auto-starts the OrbStack engine and `localclaude-searxng` container if either is down (idempotent, never destructive — prints recovery recipe for stale-mount errors).
  - SSD KV-cache tier was briefly default-on (2026-04-26) but flipped back to default-off the next day after [`akaszubski/vllm-mlx#1`](https://github.com/akaszubski/vllm-mlx/issues/1) was found: the writer thread crashed on bf16 KV (every production model), and the resulting MLX stream-binding fault terminated the whole server. Fix shipped in [`akaszubski/vllm-mlx#2`](https://github.com/akaszubski/vllm-mlx/pull/2); the default will flip back once that's merged into the fork's pinned version. Re-enable now with `LOCALCLAUDE_SSD_CACHE_DIR=$HOME/.localclaude/ssd-cache`.
  - Default-on prompt optimizer + tool stubs (via the vllm-mlx fork patches).
  - `coder-480` profile uses `LOCALCLAUDE_CODER_480_REMOTE` env var (no hardcoded SSH endpoints). Bails with a clear recovery menu on Macs with <256 GB RAM and no remote configured.
  - New `LOCALCLAUDE_NO_REMOTE=1` escape hatch — force-local-only behaviour.
  - Reads `~/Models/` first, falls back to `~/.cache/huggingface/hub/` for legacy installs.
- **Layout** changed from "umbrella with three sister directories" to "this repo with two siblings":
  - Old: `~/Dev/local-claude-code-mlx/{localclaude,vllm-mlx,searxng-mcp}`
  - New: `~/Dev/{localclaude,vllm-mlx,searxng-mcp}`
  - The localclaude script's `_resolve_workspace_dir` already used `dirname($SCRIPT_DIR)`, so no script change was needed — sister auto-resolution still works.

### Removed

- **`bench/`** directory — was producing single-trial anecdotal data that we explicitly downgraded in the docs. Net maintenance reduction. The vllm-mlx upstream benchmark suite is more rigorous; users should use that instead. Past observations remain in commit history of the archived umbrella repo.

### vllm-mlx fork patches (the actual reason this stack is fast)

The install source is the **[`akaszubski/vllm-mlx`](https://github.com/akaszubski/vllm-mlx) fork**, branched from upstream [`waybarrios/vllm-mlx`](https://github.com/waybarrios/vllm-mlx). `pip install vllm-mlx` from PyPI is **wrong** — only the fork has the patches. The fork rebases on upstream regularly so we don't lose upstream fixes.

| Patch | Commit | Flag(s) | What it does |
|---|---|---|---|
| Anthropic /v1/messages prompt optimizer | `818f3fcb` | `--optimize-prompts` | Master switch for the optimizer. Off by default upstream; localclaude turns it on. |
| Tool allowlist | `818f3fcb` | `--optimize-tool-allowlist <csv>` | Drops tool definitions whose names aren't on the list. The default `code` allowlist sends 33 tools (vs 274+ otherwise). |
| Tool description stubs | `818f3fcb` | `--optimize-stub-tools` | Replaces verbose tool descriptions and JSON schemas with short stubs. Combined with the allowlist: ~98% prefill-token reduction (~195K chars → ~3.5K). |
| Auto-disable thinking on tool calls | `b680dc20` | (automatic) | Forces `enable_thinking=false` for any request carrying `tools`. Lets reasoning models work as agents instead of emitting `<think>` blocks. |
| Stubs for 11 more Claude Code 2.x native tools | `ae25fb83` | (automatic) | Hand-tuned short stubs for `EnterWorktree`, `CronCreate`, `TaskCreate`, etc. |

Documented in detail at [vllm-mlx/docs/guides/optimizer.md](https://github.com/akaszubski/vllm-mlx/blob/main/docs/guides/optimizer.md) (the fork).

### Decisions (with reasoning)

#### SSD KV cache: **default-off** (briefly default-on, 2026-04-26 → 2026-04-27)

Intent was default-on for the cold-restart win: after `localclaude stop` (or a reboot), the next `start` previously recomputed the prefix cache for the system prompt + tool definitions — ~20-30 s of dead time before the first useful token. With the SSD tier on, the same pages reload from disk in <1 s.

Reality after one day in production: the SSD spill writer crashed on `bfloat16` KV layers (the dtype every production model uses). `safetensors.numpy` cannot serialise bf16, and the resulting `RuntimeError` in the daemon writer thread tripped a secondary MLX `Stream(gpu, N) not in current thread` violation that terminated the whole server via uncaught C++ exception. See [`akaszubski/vllm-mlx#1`](https://github.com/akaszubski/vllm-mlx/issues/1) for the full root cause and traceback.

Fix shipped in [`akaszubski/vllm-mlx#2`](https://github.com/akaszubski/vllm-mlx/pull/2) — switches both serializers to MLX-native `mx.save_safetensors`/`mx.load`, and pre-evaluates layers on the producer thread before queue handoff to fix the stream binding. Validated end-to-end (55/55 tests, lossless bf16 round-trip, live server clean across 10+ requests + 1 real spill).

Default will flip back to on once the fix is merged into the fork's main and the localclaude pinned version moves past it. Until then, opt back in explicitly with `LOCALCLAUDE_SSD_CACHE_DIR=$HOME/.localclaude/ssd-cache`.

#### KV-cache 8-bit quantization: **opt-in only**

Real bug — `_QuantizedCacheWrapper` doesn't implement `state`, so `mlx_lm.save_prompt_cache` fails on shutdown. Combined with the SSD-cache default, **6/7 cache entries silently fail to persist** on shutdown, defeating the SSD tier's value across restarts.

Tracked upstream: [`waybarrios/vllm-mlx#443`](https://github.com/waybarrios/vllm-mlx/issues/443). Will revisit defaults once that lands.

#### Warm-prompts: **opt-in only**

Architectural blocker — the seed file is project-specific. There's no obvious "default seed" that's right for every project. Future: per-profile seed-capture flow.

#### MTP (speculative decoding via `--enable-mtp`): **opt-in only, profile-gated**

Only fires on profiles whose model has MTP heads (Qwen3-Next family, Qwen3.5/3.6). On `coder` (default profile, Qwen3MoE base) vllm-mlx logs `[MTP] MTP validation failed — --enable-mtp will be ignored`. So default-on is a no-op there.

### Issues filed during the build-out

| Repo | # | Summary |
|---|---|---|
| akaszubski/autonomous-dev | [#977](https://github.com/akaszubski/autonomous-dev/issues/977) | scaffold-doctor — detect partial autonomous-dev installs |
| akaszubski/autonomous-dev | [#978](https://github.com/akaszubski/autonomous-dev/issues/978) | fixture sanitizer — block CLAUDE.md / personal paths in fixtures |
| akaszubski/autonomous-dev | [#979](https://github.com/akaszubski/autonomous-dev/issues/979) | audit-context — token-cost breakdown for captured Claude Code requests |
| waybarrios/vllm-mlx | [#443](https://github.com/waybarrios/vllm-mlx/issues/443) | `--kv-cache-quantization` breaks prefix-cache persistence |
| akaszubski/vllm-mlx | [#1](https://github.com/akaszubski/vllm-mlx/issues/1) → [#2](https://github.com/akaszubski/vllm-mlx/pull/2) | SSD cache writer crashes on bfloat16 KV (filed + fixed same day) |

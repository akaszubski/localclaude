# localclaude

Single-command lifecycle manager for **Claude Code CLI against a local
[vllm-mlx](https://github.com/akaszubski/vllm-mlx) server** on Apple Silicon.

One vllm-mlx server, profile-based model selection, detached lifecycle, and
visible Claude Code connect commands so you stay in control. Auto-starts the
sister [`searxng-mcp`](../searxng-mcp/) container so web research works out of
the box.

## Why

Running Claude Code against a local LLM means juggling four things every time:

1. Start a `vllm-mlx serve …` with the right model + parser flags
2. Wait for the model to load
3. Make sure the SearXNG container (web-search backend) is up
4. Launch `claude` with the right `ANTHROPIC_BASE_URL`, `ANTHROPIC_API_KEY`, and `ANTHROPIC_MODEL` env vars

`localclaude` does (1)–(3) automatically, prevents two servers running at
once, and either prints the exact `claude` command for (4) or launches it
directly with `localclaude cc`.

## Install

This script lives inside the [`local-claude-code-mlx`](..) umbrella repo. From
the umbrella root:

```bash
echo "export PATH=$(pwd)/localclaude:\$PATH" >> ~/.zshrc
source ~/.zshrc
```

### Prereqs

- [`vllm-mlx`](https://github.com/akaszubski/vllm-mlx) on `PATH` (`pip install vllm-mlx`, or `pip install -e ../vllm-mlx` for the local fork)
- [`claude`](https://docs.claude.com/claude-code) CLI 2.x
- [OrbStack](https://orbstack.dev) or Docker Desktop — runs the `localclaude-searxng` container that powers `mcp__searxng__*` tools
- [`searxng-mcp`](../searxng-mcp/) sister directory present (auto-resolved from script location)
- Apple Silicon Mac

`localclaude doctor` checks all of the above and prints fix recipes for whatever's missing.

## Usage

```
localclaude start <profile> [-allowlist code|web|minimal|all] [-bind <host>]
                               Kill any existing server, start the profile,
                               wait for "Application startup complete",
                               then print the claude command to run.
                               Auto-starts OrbStack engine + SearXNG container
                               if either is down.
localclaude stop               Kill the running server.
localclaude restart            Restart the last-active profile (inherits flags).
localclaude cc                 Launch claude attached to the active profile
                               (skips the env-var copy/paste).
localclaude status             Show running model + connect command + recent stats.
localclaude list               List cached MLX models on this Mac.
localclaude doctor             Health-check the full stack: vllm-mlx, claude CLI,
                               docker daemon, SearXNG container, MCP registration.
localclaude test               End-to-end smoke: real query + decoder-collapse
                               detection + SearXNG round-trip.
localclaude -h                 Help.
```

### Profiles

| Key | Model | Tool parser | Notes |
|---|---|---|---|
| `coder` | `mlx-community/Qwen3-Coder-30B-A3B-Instruct-4bit` | `qwen3_coder` | MoE A3B coder, ~120 tok/s on M4 Max |
| `coder-next` | `lmstudio-community/Qwen3-Coder-Next-MLX-8bit` | `qwen3_coder` | 8-bit, top SWE-bench Pass@5 |
| `coder-480` | `mlx-community/Qwen3-Coder-480B-A35B-Instruct-4bit` | `qwen3_coder` | ~250 GB on disk; auto-routes via SSH to a remote host that has the weights when run on a smaller Mac |
| `instruct` | `mlx-community/Qwen3-30B-A3B-Instruct-2507-4bit` | `qwen` | General MoE baseline. ⚠ See [vllm-mlx#431](https://github.com/waybarrios/vllm-mlx/issues/431) — `qwen` parser drops whitespace-only deltas mid-stream; markdown layout collapses (cosmetic). |
| `qwen36` | `mlx-community/Qwen3.6-35B-A3B-4bit` | `qwen` | Newest, beats Gemma 4 (download on demand). Same `qwen`-parser caveat as `instruct`. |
| `gemma4` | `mlx-community/gemma-4-31b-it-4bit` | `gemma4` | ⚠ **Currently broken** — see [vllm-mlx#380](https://github.com/waybarrios/vllm-mlx/issues/380): nonsense output under continuous batching, stray `&lt;channel&#124;&gt;` tokens. Use `qwen36` instead until upstream fix lands. |

Pass arbitrary models via `localclaude start -model <hf-id>`.

### Allowlist presets

`-allowlist <preset>` controls the **vllm-mlx prompt optimizer**'s tool
allowlist. The optimizer (fork patches `818f3fcb` + `ae25fb83` + `b680dc20`,
flags `--optimize-prompts --optimize-stub-tools --optimize-tool-allowlist`)
drops tool definitions whose names aren't on the list and replaces verbose
descriptions with short stubs. This is the single biggest perf win for local
Claude — without it, prefill of an 80K-token Claude Code request takes ~50s
on M4 Max instead of ~3-5s.

| Preset | Tools sent | Token reduction | Notes |
|---|---|---|---|
| `minimal` | 6 (Read, Edit, Bash, Grep, Glob, Write) | ~99.5% | File ops + shell only. Fastest prefill. |
| `code` (default) | 33 Claude Code natives + `mcp__searxng__*` | ~98% | Full agentic toolkit + local web research. WebSearch / WebFetch dropped (no-ops vs local server) so the model uses MCP instead. Other MCP servers (Gmail, Calendar, Home Assistant, etc.) excluded so prefill stays fast. |
| `web` | (alias for `code`) | | Kept for backwards compat. |
| `all` / `off` / `none` | Full 274+ tool catalog | 0% | Slowest first prefill (~50s for 80k tokens). Every registered MCP tool included. Use when you specifically need a tool the `code` allowlist filters out. |

The optimizer is documented in detail at
[vllm-mlx/docs/guides/optimizer.md](https://github.com/akaszubski/vllm-mlx/blob/main/docs/guides/optimizer.md).

### Daily flow

```bash
# In whatever project you want claude to work in:
cd ~/Dev/myproject

# Spin up server (one terminal):
localclaude start coder

# Either let `cc` wire up env vars and launch claude in another terminal:
localclaude cc

# …or copy the env-var command that `start` printed:
ANTHROPIC_BASE_URL=http://localhost:8000 ANTHROPIC_API_KEY=not-needed \
  ANTHROPIC_MODEL=mlx-community/Qwen3-Coder-30B-A3B-Instruct-4bit claude

# Do work, exit claude when done.
# Server stays alive (prefix cache stays warm) until:
localclaude stop
```

### Why two terminals?

The server is long-running and chatty (logs, throughput, optimizer/gate fires).
Keeping it visible in its own terminal lets you watch what's happening — token
counts, KV cache hits, errors — without it being mixed up with the Claude Code
TUI. `localclaude status` summarizes the relevant lines if you don't want to
tail logs.

## Performance knobs

### Default-on: SSD KV-cache tier

Since 2026-04-26, `localclaude start` enables `--ssd-cache-dir
~/.localclaude/ssd-cache --ssd-cache-max-gb 20` for every profile. This
persists the prefix cache to disk so cold restarts reuse the system-prompt /
tool-definition prefill instead of recomputing it.

Override via env vars:

| Var | Default | Notes |
|---|---|---|
| `LOCALCLAUDE_SSD_CACHE_DIR` | `~/.localclaude/ssd-cache` | Set to `off` to disable; set to a different path to relocate. |
| `LOCALCLAUDE_SSD_CACHE_MAX_GB` | `20` | GB cap on the on-disk cache. |

```bash
LOCALCLAUDE_SSD_CACHE_DIR=off localclaude start coder       # disable
LOCALCLAUDE_SSD_CACHE_MAX_GB=50 localclaude start coder     # bigger cap
```

### Opt-in knobs

`vllm-mlx` ships two more cache optimizations that aren't on by default.
Enable via `LOCALCLAUDE_EXTRA_VLLM_ARGS`, which the script forwards verbatim
to `vllm-mlx serve`:

```bash
# 8-bit KV quantization (halves KV memory)
# WARNING: as of 2026-04-26 this BREAKS prefix-cache persistence on shutdown
# (vllm-mlx _QuantizedCacheWrapper missing .state). Stacks fine in-memory but
# defeats the SSD-cache default. Use only if memory pressure is the bottleneck.
LOCALCLAUDE_EXTRA_VLLM_ARGS="--kv-cache-quantization --kv-cache-quantization-bits 8" \
  localclaude start coder

# Pre-warm the prefix cache from a captured Claude Code system prompt
LOCALCLAUDE_EXTRA_VLLM_ARGS="--warm-prompts ~/Dev/local-claude-code-mlx/bench/cases/seed.warm.json" \
  localclaude start coder
```

Use the umbrella's [`bench/`](../bench/) harness to A/B-test these
configurations before flipping defaults — see `bench/README.md`.

## Path overrides

The script auto-resolves sister components from its own location
(`<workspace>/vllm-mlx`, `<workspace>/searxng-mcp`). Override individually
if you've moved them:

| Var | Default |
|---|---|
| `LOCALCLAUDE_WORKSPACE_DIR` | parent of the script |
| `LOCALCLAUDE_VLLM_MLX_DIR` | `$WORKSPACE/vllm-mlx` |
| `LOCALCLAUDE_SEARXNG_MCP_DIR` | `$WORKSPACE/searxng-mcp` |

## State

| Path | What it is |
|---|---|
| `~/.localclaude/.active` | Last-used profile config (read by `restart` / `cc` / `status`) |
| `~/.localclaude/logs/<profile>.log` | Per-profile server logs |
| `~/.localclaude/ssd-cache/` | (if enabled) persistent KV cache pages |

The script itself has no state — moving it is fine. State path is hardcoded
to `~/.localclaude/` for portability.

## Pairs with

- [`vllm-mlx`](../vllm-mlx/) — fork with prompt optimizer + thinking-gate patches
- [`searxng-mcp`](../searxng-mcp/) — local web-search MCP server (auto-managed by `localclaude start`)
- [Claude Code](https://docs.claude.com/claude-code) v2.x
- Apple Silicon Macs with enough RAM for the chosen model

## Single-server invariant

`start` always kills anything on port 8000 first. You can't accidentally have
two vllm-mlx servers running. Saves ~17–250 GB of duplicated weights in RAM.

## SearXNG auto-start behavior

When `start` runs with the `code` (default) or `minimal` allowlist, it:

1. Checks `docker info` — if the daemon is down and `orb` is installed, runs `orb start`.
2. Checks the `localclaude-searxng` container state — if missing/exited/dead, runs `docker compose up -d` from `searxng-mcp/`.
3. Surfaces all actions visibly; never runs destructive operations like `docker rm`. If the container is stuck on a stale-mount error, prints the exact `docker rm -f localclaude-searxng && docker compose up -d` recipe instead of doing it for you.

Skipped on `-allowlist all` — that mode assumes you're managing MCP yourself.

## Remote profiles

Profiles can name a remote SSH host that has the weights cached. When you
`localclaude start coder-480` on a small Mac, it transparently SSHes to the
remote host, runs the server there bound to loopback, and forwards port 8000.
From `claude`'s perspective nothing changes. See `coder-480` definition in the
script for the syntax.

# localclaude

Single-command lifecycle manager for **Claude Code CLI against a local
[vllm-mlx](https://github.com/akaszubski/vllm-mlx) server** on Apple Silicon.

One vllm-mlx server, profile-based model selection, detached lifecycle, and
visible Claude Code connect commands so you stay in control.

## Why

Running Claude Code against a local LLM means juggling three things every time:

1. Start a `vllm-mlx serve …` with the right model + parser flags
2. Wait for the model to load
3. Launch `claude` with the right `ANTHROPIC_BASE_URL`, `ANTHROPIC_API_KEY`, and `ANTHROPIC_MODEL` env vars

`localclaude` does (1) and (2) automatically, prevents two servers running at
once, and prints the exact `claude` command to copy into another terminal for (3).

## Install

```bash
git clone git@github.com:akaszubski/localclaude.git ~/Dev/localclaude
echo 'export PATH=$HOME/Dev/localclaude:$PATH' >> ~/.zshrc
source ~/.zshrc
```

Requires:
- `vllm-mlx` on `PATH` (`pip install vllm-mlx` or the [fork with optimizer + thinking-gate](https://github.com/akaszubski/vllm-mlx))
- `claude` (Anthropic Claude Code CLI 2.x)
- macOS / Apple Silicon

## Usage

```
localclaude start <profile>    Kill any existing server, start the profile,
                               wait for "Application startup complete",
                               then print the claude command to run.
localclaude stop               Kill the running server.
localclaude restart            Restart the last-active profile.
localclaude status             Show running model + connect command + recent stats.
localclaude list               List cached MLX models on this Mac.
localclaude -h                 Help.
```

### Profiles

| Key | Model | Notes |
|---|---|---|
| `coder` | `mlx-community/Qwen3-Coder-30B-A3B-Instruct-4bit` | MoE A3B coder, ~120 tok/s on M4 Max |
| `coder-next` | `lmstudio-community/Qwen3-Coder-Next-MLX-8bit` | 8-bit, top SWE-bench Pass@5 |
| `opus` | `mlx-community/Qwen3.5-27B-Claude-4.6-Opus-Distilled-MLX-4bit` | Claude-style dense 27B |
| `instruct` | `mlx-community/Qwen3-30B-A3B-Instruct-2507-4bit` | General MoE baseline |
| `qwen36` | `mlx-community/Qwen3.6-35B-A3B-4bit` | Newest, beats Gemma 4 (download on demand) |
| `gemma4` | `mlx-community/gemma-4-31b-it-4bit` | Google dense 31B |

### Daily flow

```bash
# In whatever project directory you want claude to work in:
cd ~/Dev/myproject

# Spin up server (one terminal):
localclaude start coder

# (script prints the claude command — copy/paste into another terminal)
ANTHROPIC_BASE_URL=http://localhost:8000 ANTHROPIC_API_KEY=not-needed ANTHROPIC_MODEL=mlx-community/Qwen3-Coder-30B-A3B-Instruct-4bit claude

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

## State

| Path | What it is |
|---|---|
| `~/.localclaude/.active` | Last-used profile (read by `restart` / `status`) |
| `~/.localclaude/logs/<profile>.log` | Per-profile server logs |

The script itself has no state — moving it is fine. State path is hardcoded
to `~/.localclaude/` for portability.

## Pairs with

- [`vllm-mlx`](https://github.com/akaszubski/vllm-mlx) (fork with prompt optimizer + thinking-gate)
- [Claude Code](https://docs.claude.com/claude-code) v2.x
- Apple Silicon Macs with enough RAM for the chosen model

## Single-server invariant

`start` always kills anything on port 8000 first. You can't accidentally have
two vllm-mlx servers running. Saves ~17–240GB of duplicated weights in RAM.

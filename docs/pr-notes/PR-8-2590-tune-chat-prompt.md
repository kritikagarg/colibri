# PR #8 — Backport: render `coli tune`'s prompts with each checkpoint's own template

| | |
|---|---|
| **Branch** | `port/2590-tune-chat-prompt` → `main` (this fork) |
| **Upstream issue** | <https://github.com/JustVugg/colibri/issues/2590> |
| **Kind** | Backport of an upstream `dev` commit |
| **Risk** | Low–medium — affects `coli tune` only, not serving |

## What this is

A **cherry-pick** of one upstream commit, authorship preserved.

| Commit | Author | Subject |
|---|---|---|
| `290aea95` | JustVugg | coli tune: the qwen36 family's prompts rendered by the gateway, each checkpoint with its own template |

## Why backport rather than rewrite

The fix already exists on upstream `dev`, but this fork's `main` (`0a5504a1`)
is 970 commits behind `dev` and does not have it.

## The bug

`coli tune` failed on Qwen3-Coder-30B with:

```
baseline: engine reported no decode speed
```

Reported on Windows + CUDA (RTX 4050) and independently reproduced on Ubuntu
CPU-only, so it is not backend-specific. Normal serving worked fine; only tune
failed.

Cause: tune rendered every qwen36-family checkpoint's prompts with **one**
template, which ends in an open `<think>` block. Qwen3-Coder has no thinking
mode, so after that `<think>` it ended its turn on the first token — zero decode
tokens, no baseline speed, tune stops.

The fix renders the family's prompts through the gateway
(`openai_server.tune_chat_prompt`), each checkpoint with its own template and
the default reasoning a chat request would get — so the request tune times is
the request `coli serve` is actually sent. Qwen3-Coder now tunes to the end, and
Qwen3.6 is timed without thinking, as it is served unless asked.

## Conflict resolved

One file conflicted: **`CHANGELOG.md`**, because `dev`'s changelog has since
gained an unrelated 2.2.0 section that this `main` does not have.

Resolution: kept this `main`'s changelog, and placed the commit's **own entry
verbatim** under `## [Unreleased]` → `### Fixed`, matching the heading style
already used in that file (`### Fixed` is the most common heading there). The
code and test changes are untouched from upstream.

## Files changed

| File | Change |
|---|---|
| `c/coli` | tune calls the gateway renderer |
| `c/openai_server.py` | `tune_chat_prompt` |
| `c/family_registry.py` | per-checkpoint template selection |
| `c/tests/test_tune_chat_prompt.py` | **new** |
| `c/tests/test_family_registry.py` | updated expectations |
| `CHANGELOG.md` | entry under `[Unreleased]` → `### Fixed` (conflict-resolved, see above) |

Total +135 / −16 across 6 files.

## How to run it

### Requirements

Python **3.10+**. On this development host the system `python3` is 3.9 and too
old for the repo, so use `python3.13`.

### Verify

```bash
cd c
python3.13 -m pytest tests/test_tune_chat_prompt.py \
                     tests/test_family_registry.py \
                     tests/test_openai_server.py
```

`test_openai_server.py` is included because this commit touches
`openai_server.py`; it is the regression guard for the gateway.

### Observed result

```
======================= 536 passed, 1 skipped in 35.58s ========================
```

### Verify against a real checkpoint

```bash
coli stop
coli tune --model ~/colibri-models/qwen3-coder-30b
```

This is the exact command that failed in the issue. It should now establish a
baseline decode speed and run the sweep to completion instead of stopping at
`baseline: engine reported no decode speed`. For contrast, a Qwen3.6 checkpoint
should be timed *without* thinking:

```bash
coli tune --model ~/colibri-models/qwen36-35b-a3b
```

## What was NOT verified

- **`coli tune` was never run.** It needs an engine binary and model weights;
  there is no `make`/`gcc`/`clang`/`nvcc` on the development host and no GPU or
  checkpoint, so neither the Qwen3-Coder failure nor the fix was reproduced
  end to end.
- The ported tests cover the prompt-rendering logic in Python only — they do
  not measure decode speed.

## Reverting

```bash
git revert 04410fbb
```

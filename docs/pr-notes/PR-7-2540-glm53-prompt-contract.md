# PR #7 — Backport: select the full GLM-5.3 prompt contract from checkpoint metadata

| | |
|---|---|
| **Branch** | `port/2540-glm53-prompt-contract` → `main` (this fork) |
| **Upstream issue** | <https://github.com/JustVugg/colibri/issues/2540> |
| **Kind** | Backport of an upstream `dev` commit |
| **Risk** | Medium — changes the rendered prompt for GLM checkpoints |

## What this is

A **verbatim cherry-pick** of one upstream commit, authorship preserved.

| Commit | Author | Subject |
|---|---|---|
| `4d848a9c` | Rudy Mizrahi Celekli | fix(api): select the full GLM-5.3 prompt contract from checkpoint metadata |

Applied cleanly onto `main` with no conflicts.

## Why backport rather than rewrite

The fix already exists on upstream `dev`, but this fork's `main` (`0a5504a1`)
is 970 commits behind `dev` and does not have it.

## The bug

GLM-5.3 reports `model_type` `glm_moe_dsa`, so `ARCH == "glm"`, and
`render_chat_for_arch` sent it to `render_chat` — the **GLM-5.2** renderer. A
client that sends neither `reasoning_effort` nor `enable_thinking` (Open WebUI
does neither) therefore got the 5.2 prompt: no effort line, and a *closed*
`<think></think>`. For a single user message:

```
[gMASK]<sop><|user|>hi<|assistant|><think></think>
```

The fix classifies the checkpoint from its own metadata and routes a full
GLM-5.3 to `render_chat_glm53` with `full_model=True`, so it gets the effort
line and the open `<think>` its template expects.

The issue carries an independent confirmation that the renders reproduce
byte-for-byte on both `main` and `dev`, and a second independent validation of
this commit on macOS arm64 with all comparisons byte-exact.

## Files changed

| File | Change |
|---|---|
| `c/openai_server.py` | metadata-driven flavour selection (`detect_chat_flavor`) and dispatch |
| `c/tools/convert_fp8_to_int4.py` | carries the template metadata through conversion |
| `docs/api.md` | documents the prompt contract selection |
| `c/tests/test_glm53_full_template.py` | **new** |
| `c/tests/test_fp8_chat_template_metadata.py` | **new** |
| `c/tests/glm53_chat_template_harness.py` | updated |

Total +217 / −10 across 6 files. Documentation is included in the upstream
commit, so none was added on top.

## How to run it

### Requirements

Python **3.10+**. On this development host the system `python3` is 3.9 and too
old for the repo, so use `python3.13`.

### Verify

```bash
cd c
python3.13 -m pytest tests/test_glm53_full_template.py \
                     tests/test_fp8_chat_template_metadata.py \
                     tests/test_openai_server.py \
                     tests/test_chat_history_reuse.py
```

The last two are the regression guard: this commit changes renderer dispatch, so
the broad gateway suites matter as much as the two new files.

### Observed result

```
======================= 478 passed, 3 skipped in 32.44s ========================
```

### Verify against a real checkpoint

```bash
# Serve a GLM-5.3 int4 checkpoint and send a request with no reasoning flags,
# which is the exact shape that reproduced the bug.
coli serve --model <glm-5.3-int4>

curl -s http://127.0.0.1:8000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"glm-5.3","messages":[{"role":"user","content":"hi"}],"max_tokens":16}'
```

The rendered prompt should now carry the effort line and an **open** `<think>`.
The project's byte-identity harnesses are the stricter check:

```bash
cd c
python3.13 tests/glm53_chat_template_harness.py --model-dir <checkpoint>
```

Note these harnesses are argparse scripts that need the vendor
`chat_template.jinja` from the checkpoint; they are not part of the pytest run
above and will skip without it.

## What was NOT verified

- **No GLM-5.3 checkpoint was available**, so the renderer was exercised only
  through the ported tests' fixtures — not against a real model directory, and
  not against a live generation.
- **No C compiler** (`make`, `gcc`, `clang`, `nvcc` all absent) and the host is
  aarch64 Linux, so nothing was built and no engine was run.
- No GPU. The byte-identity harnesses above were not run.

## Reverting

```bash
git revert ce19c1cf
```

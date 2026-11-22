# PR #6 — Backport: bound the OpenMP team by the container's CPU quota

| | |
|---|---|
| **Branch** | `port/2914-cgroup-cpu-quota` → `main` (this fork) |
| **Upstream issue** | <https://github.com/JustVugg/colibri/issues/2914> |
| **Kind** | Backport of upstream `dev` commits |
| **Risk** | Medium — changes default thread-team size inside containers |

## What this is

A **verbatim cherry-pick** of two upstream commits, with their original
authorship preserved. No reimplementation.

| Commit | Author | Subject |
|---|---|---|
| `6f35377e` | JustVugg | coli: the launcher's default OpenMP team bounded by the container's CPU quota and cpuset |
| `4a4276c8` | Léon Avic Simmons | resource_plan, autotune: the plan's team, the tuning sweep and a saved profile within the CPU quota and the affinity mask |

Both applied cleanly onto `main` with no conflicts.

## Why backport rather than rewrite

The fix already exists on upstream `dev`, but this fork's `main` (`0a5504a1`)
is 970 commits behind `dev` and does not have it. Cherry-picking a known-good
fix is lower risk than writing a new one, and keeps credit with the original
authors.

## The bug

On a many-core Linux host inside a container limited by a **CFS quota** rather
than a cpuset, the launcher sized the OpenMP team from the host's physical
cores instead of the CPUs the container may actually use.

The reporter's pod: `nproc` says 96, the topology shows 48 physical cores, but
`/sys/fs/cgroup/cpu.max` is `765000 100000` — i.e. 7.65 CPUs. The engine started
with a 48-thread team:

```
[OMP] qwen36: 48 physical-core threads instead of 96 logical CPUs; ...
```

Measured effect: decode 2–4 tok/s by default, against 134 tok/s with
`OMP_NUM_THREADS=7`. The oversized team also made the trunk placement probe
non-deterministic — it kept the trunk on the CPU in three starts of four.

## Scope limit — read this

This PR is the **launcher/planner half only**.

The C half of upstream's series (`e2a3fe6e`, touching `c/omp_tune.h`,
`c/deepseek_v4.c`, `c/qwen38_cuda_chain.h`, `c/vk_dmoe.h`) does **not**
cherry-pick cleanly onto this `main` — it conflicts in three headers — and
could not be compiled or tested here. It is deliberately excluded, so the
engine-side `compat_cgroup_cpu_limit` path is **not** part of this change. The
trunk-probe fix (`95bcb4f1`, `c/qwen36.c`) is likewise excluded.

Practical consequence: the launcher will now pass a quota-aware
`OMP_NUM_THREADS`, but an engine started directly (not through `coli`) still
sizes its own teams from host cores. Completing the series needs someone with
a C toolchain.

## Files changed

| File | Change |
|---|---|
| `c/resource_plan.py` | quota/cpuset-aware core count |
| `c/autotune.py` | the tuning sweep and a saved profile stay within the quota |
| `c/tests/test_cgroup_cpu.py` | **new** — cgroup v1/v2 parsing and bounding |
| `c/tests/test_resource_plan.py`, `c/tests/test_autotune.py`, `c/tests/test_cli_output.py` | updated expectations |

Total +493 / −14 across 6 files.

## How to run it

### Requirements

Python **3.10+**. On this development host the system `python3` is 3.9 and too
old for the repo, so use `python3.13`.

### Verify

```bash
cd c
python3.13 -m pytest tests/test_cgroup_cpu.py tests/test_resource_plan.py \
                     tests/test_autotune.py tests/test_cli_output.py
```

### Observed result

```
============================= 212 passed in 4.84s ==============================
```

### Verify in a real container

```bash
# Reproduce the reporter's shape: a quota, not a cpuset.
docker run --rm --cpus=8 -v "$PWD:/src" -w /src python:3.13 \
  sh -c 'cat /sys/fs/cgroup/cpu.max; python3 c/resource_plan.py --json | head'

# On the host, for comparison:
nproc; cat /sys/fs/cgroup/cpu.max 2>/dev/null || echo "no v2 quota here"
```

The planned team should track the quota (8), not `nproc`. An explicit
`OMP_NUM_THREADS` still overrides everything, as does `COLI_NO_OMP_TUNE`.

## What was NOT verified

- **No container run.** `docker` is not available on the development host, so
  the commands above are untested here; the cgroup files are exercised only
  through the ported tests' fixtures.
- **No C compiler** (`make`, `gcc`, `clang`, `nvcc` all absent) and the host is
  aarch64 Linux, so nothing was built and no engine was run.
- **No GPU and no model weights**, so the reporter's tok/s numbers were not
  reproduced. The ported tests cover the Python planner logic only.

## Reverting

```bash
git revert e1f8fd55 b64f6943
```

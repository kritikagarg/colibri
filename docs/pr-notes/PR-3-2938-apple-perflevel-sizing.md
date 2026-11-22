# PR #3 — Size the Apple OpenMP team by perflevel name, not position

| | |
|---|---|
| **Branch** | `fix/2938-apple-perflevel-core-count` → `main` (this fork) |
| **Upstream issue** | <https://github.com/JustVugg/colibri/issues/2938> |
| **Kind** | Bug fix, macOS / Apple Silicon |
| **Risk** | Medium — changes default thread-team size on Apple hardware |

## What this changes

The OpenMP team on Apple Silicon is now sized by reading each performance
tier's **name** (`hw.perflevelN.name`) and counting every tier that is not the
efficiency tier, instead of assuming the *last* tier is always the efficiency
cores.

## Root cause

The fix for #2194 counted `hw.perflevel0 … hw.perflevel(N-2)` — "every tier but
the last". On an M5 Pro that assumption is false:

```
hw.nperflevels: 2
hw.perflevel0.name: Super         hw.perflevel0.logicalcpu: 6
hw.perflevel1.name: Performance   hw.perflevel1.logicalcpu: 12
```

Neither tier is an efficiency tier, so both sizing paths dropped the 12
Performance cores and picked **6 threads out of 18 cores**. The reporter
measured the cost against `OMP_NUM_THREADS=18`.

Two independent implementations carried the same rule, and both are fixed:

| Path | What it drives |
|---|---|
| `c/resource_plan.py` → `physical_cpu_count()` | the launcher's `OMP_NUM_THREADS` for the engine |
| `c/omp_tune.h` → `coli_physical_cores()` | the engine's own team sizing |

## Behaviour table

| Topology | Tiers | Before | After |
|---|---|---|---|
| M5 Pro | Super 6 + Performance 12 | 6 | **18** |
| M6-style (#2194) | Super 2 + Performance 4 + Efficiency 6 | 6 | 6 |
| M1–M4 | Performance P + Efficiency E | P | P |
| Single tier | `hw.nperflevels` = 1 | all | all |
| Intel Mac | no `hw.perflevel*` | `hw.physicalcpu` | `hw.physicalcpu` |
| Non-Apple | — | unchanged | unchanged |

The name match is case-insensitive. If any name cannot be read, or if every
tier looks like an efficiency tier, both paths **fall back to the previous
all-but-the-last rule**, so no platform regresses when the sysctl is missing.

## Files changed

| File | Change |
|---|---|
| `c/omp_tune.h` | name-aware tier selection + a `__APPLE__`-guarded `coli_perflevel_is_efficiency()` helper (hand-rolled case-insensitive match: no `<ctype.h>`, no non-standard `strcasestr`); `char nm[64]` zeroed, `sysctlbyname` given `sizeof nm - 1` so a NUL always remains; tier count clamped to 16 to bound `keep[16]` |
| `c/resource_plan.py` | new `sysctl_str()` helper (`sysctl_int()` now wraps it) plus a `perf_tiers(levels)` helper implementing the identical rule and fallbacks |
| `c/tests/test_resource_plan.py` | 4 new tests on the existing fake-`sysctl` pattern |

Total +131 / −14.

## How to run it

### Requirements

Python **3.10+**. On this development host the system `python3` is 3.9 and too
old for the repo (it fails on `dataclass(slots=True)`), so use `python3.13`.

### Verify

```bash
cd c
python3.13 -m pytest tests/test_resource_plan.py
```

To confirm the new M5 Pro regression test is load-bearing, revert just the
Python half and watch it fail:

```bash
cd c
git stash push -m pr3-check -- resource_plan.py
python3.13 -m pytest tests/test_resource_plan.py -k m5 ; # expect AssertionError: 6 != 18
git stash list --format='%H %gs' | head -1   # find your entry
git stash apply <sha-from-above>             # restore (apply, not pop)
```

### Observed result

```
$ cd c && python3.13 -m pytest tests/test_resource_plan.py
============================= 131 passed in 1.76s ==============================
```

Baseline on `main` before the change was 127 passed, so the four new tests are
additive and nothing regressed. With the Python half reverted, the M5 Pro test
fails with `AssertionError: 6 != 18`.

### On real Apple hardware

```bash
# What the fix reads:
sysctl hw.nperflevels hw.perflevel0.name hw.perflevel0.logicalcpu \
       hw.perflevel1.name hw.perflevel1.logicalcpu

# What the launcher now passes to the engine:
python3 c/coli start --background --no-browser
#   then check OMP_NUM_THREADS in the engine process environment
```

`OMP_NUM_THREADS` set by hand still overrides everything, as does
`COLI_NO_OMP_TUNE`.

## What was NOT verified

- **No macOS host and no Apple hardware.** Every Apple topology is exercised
  only through a faked `sysctl` in the Python tests.
- **`c/omp_tune.h` is uncompiled.** There is no `make`/`gcc`/`clang`/`nvcc` on
  the development host, it is aarch64 Linux, and the changed branch sits inside
  `#if defined(__APPLE__)`. It was reviewed line by line against the Python
  implementation; the name-buffer handling in particular deserves a second
  reviewer.
- **No throughput re-measurement.** The tok/s figures in the issue are the
  reporter's, not reproduced here.
- `c/tests/test_omp_tune.c` has no Apple-branch coverage and could not be built.

## Related, deliberately not changed

`coli tune`'s sweep only tries teams at or below the planner's count, so it can
never surface this class of mis-sizing — it tested `omp-3` and kept the
baseline. Worth a follow-up; no code touched here.

## Reverting

```bash
git revert e2d12bef
```

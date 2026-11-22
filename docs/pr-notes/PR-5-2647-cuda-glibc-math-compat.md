# PR #5 — Make the CUDA/glibc math compatibility shim opt-out-able

| | |
|---|---|
| **Branch** | `feat/2647-cuda-glibc-math-compat-optout` → `main` (this fork) |
| **Upstream issue** | <https://github.com/JustVugg/colibri/issues/2647> |
| **Kind** | Build configurability, Linux CUDA only |
| **Risk** | Low by default — the default build is byte-identical |

## What this changes

Adds a documented switch, **`CUDA_GLIBC_MATH_COMPAT`**, that controls exactly
one flag: the forced include of `glibc_c23_math_compat.h` in `NVCCFLAGS`.

| Setting | Effect |
|---|---|
| unset (default) or `=1` | shim on — `-Xcompiler=-include,$(CURDIR)/glibc_c23_math_compat.h` is appended, exactly as before |
| `=0` | that one flag is dropped; every other compiler, architecture and warning flag is retained |

Following the `LTO` and `NO_TC` idiom already in these Makefiles, only the exact
value `1` keeps the shim — a typo reads as the opt-out, which the check target
below reports.

## Why

`c/Makefile` appended the forced include unconditionally. A distributor whose
toolchain already reconciles the CUDA/glibc declarations (CUDA 13.x headers do)
had to patch the Makefile or restate the whole `NVCCFLAGS` value to drop this
one compatibility layer. The shim itself — the #966 fix for the glibc ≥ 2.41
C++17 `noexcept` clash — stays on by default, because the clash is a hard
compile error on a current glibc with CUDA ≤ 12.9.

Out of scope, per the issue: no automatic compiler probe, no kernel, model
format or numerical change, no change to the dependency-free CPU build.
Windows and macOS are unaffected (the shim is Linux-only — MSVC never sees
glibc headers).

## GPU-free configuration check

The issue asked that the default and the opt-out recipe be distinguishable
"without needing a GPU". New phony target:

```bash
make -C c cuda-glibc-math-compat-check
```

It runs only `echo` and `test`: it prints the selected mode, the expected and
actual include state, and the final `NVCCFLAGS`, and exits non-zero if the two
disagree. The two derived variables (`CUDA_GLIBC_MATH_COMPAT_WANT` / `_GOT`) are
recursively expanded (`=`, not `:=`) on purpose, so the later `DEEPGEMM` and
`NCCL` `override NVCCFLAGS +=` appends are reflected in what it reports.

## Files changed

| File | Change |
|---|---|
| `c/Makefile` | `CUDA_GLIBC_MATH_COMPAT ?= 1` + comment (after `NVCC_STD ?= c++17`, ~line 325); `ifeq` guard nested inside the existing `ifneq ($(LINUX),)` (~line 355); check target (~line 1249) |
| `docs/BUILD-cuda-glibc241.md` | new section "Opting out of the bundled shim (`CUDA_GLIBC_MATH_COMPAT`)", with a two-row table showing both resulting flag lines |
| `docs/cuda.md` | one paragraph in the requirements section linking to the above |
| `CHANGELOG.md` | one bullet under `[Unreleased]` → `### Build, tests and CI` |

Total +76 / −0.

`docs/BUILD-cuda-glibc241.md` was chosen as the home because it is the document
about this exact glibc 2.41 / CUDA clash. `GPU_BACKENDS.md` has no CUDA
build-options table (its only variables table covers Windows HIP SDK
selection), so no row was added there.

## How to run it

### Requirements

GNU `make`. To actually compile the CUDA path you additionally need `nvcc`, a
CUDA toolkit and a glibc ≥ 2.41 host.

### Verify the switch without a GPU

```bash
# Default: shim on
make -C c cuda-glibc-math-compat-check

# Opt out: the one include disappears, nothing else changes
make -C c cuda-glibc-math-compat-check CUDA_GLIBC_MATH_COMPAT=0

# A typo reads as off, and the check says so
make -C c cuda-glibc-math-compat-check CUDA_GLIBC_MATH_COMPAT=yes
```

### Confirm the default is unchanged

```bash
# Compare the flag line against the parent commit; expect no difference.
make -C c -n glm CUDA=1 2>/dev/null | grep -o '\-Xcompiler=-include[^ ]*'
git stash push -m pr5 && make -C c -n glm CUDA=1 2>/dev/null \
  | grep -o '\-Xcompiler=-include[^ ]*'
git stash list --format='%H %gs' | head -1   # find your entry
git stash apply <sha-from-above>             # restore (apply, not pop)
```

### Full build, for a reviewer with a toolchain

```bash
make -C c glm CUDA=1                                 # default, shim on
make -C c glm CUDA=1 CUDA_GLIBC_MATH_COMPAT=0        # opt-out
```

## What was NOT verified

- **Nothing was built or executed.** The development host has no `make`, no
  `gcc`/`clang` and no `nvcc`, so `make`, `make -n`, the CUDA compile and the
  new check target were **never run**, and no GPU was involved. Every command
  in the section above is untested.
- Compensating inspection that *was* done: recipe lines confirmed to use real
  TABs with `cat -A`; whole-file `ifeq`/`ifneq`/`endif` depth verified to return
  to 0 and never go negative, before and after, with the three trailing
  `endif`s pairing to the new `ifeq`, `ifneq ($(LINUX),)` and
  `ifneq ($(IS_WIN),)` in that order; and a throwaway Python re-implementation
  of just that conditional region reproduced the flag lines for
  Linux/Windows × default/`=1`/`=0`, confirming the default is byte-identical
  to `HEAD` and `=0` removes exactly one flag. macOS takes the same non-Windows
  branch with `LINUX` empty, so it is unaffected in both modes.
- **A reviewer with a toolchain should run the check target in both modes and
  compile `make -C c glm CUDA=1` on a glibc ≥ 2.41 host before trusting this.**

## Upstream note

Upstream `dev` already carries a switch of the same name
(`CUDA_GLIBC_MATH_COMPAT ?= 1`, `c/Makefile:376`), arrived at independently. It
is not on this fork's `main`. Upstream's version has no equivalent of the
`cuda-glibc-math-compat-check` target. If you sync with `dev`, expect a
conflict here and prefer upstream's variable definition.

## Reverting

```bash
git revert 80422ca8
```

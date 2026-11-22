# PR #2 — Rename the `idot.h` local `grp2`, which `<dlgs.h>` defines

| | |
|---|---|
| **Branch** | `fix/2611-idot-windows-i-macro` → `main` (this fork) |
| **Upstream issue** | <https://github.com/JustVugg/colibri/issues/2611> |
| **Kind** | Build fix, Windows / MSYS2 UCRT64, AVX-512 VNNI only |
| **Risk** | Low — a local variable rename, no logic change |

## What this changes

Renames the local pointer `grp2` to `grpb` inside `i8r16_group2()` in
`c/idot.h`. Two lines, one function.

## Root cause

On Windows the build fails with:

```
idot.h:1292:39: error: expected identifier or '(' before numeric constant
```

A full `<windows.h>` includes `<dlgs.h>`, which defines the dialog control IDs
`grp1`–`grp4`, `stc1`–`stc32` and friends as **lowercase object-like macros**
(`grp2` is `0x0431`). `c/qwen36.c` includes `serve_poll.h` (which pulls a full
`<windows.h>`, with no `WIN32_LEAN_AND_MEAN`) *before* `idot.h`, so by the time
the compiler reaches the declaration, `grp2` has been replaced by an integer
literal — hence "expected identifier … before numeric constant".

### It is not the `I` parameter

The issue title and the reporter's first patch blame the parameter `I`. That is
wrong, and renaming `I` alone does not fix the build:

- GCC reports **column 39** of line 1292. Column 39 is the `g` of `grp2`;
  `I` sits at column 59.
- `int I` is already a parameter at `c/idot.h:74` (`qrow_i8`) in *unguarded*
  code that every build compiles, plus at lines 87, 152, 250, 294 and ~20 more.
  A macro named `I` would break line 74 first, ~1200 lines earlier.
- `<windows.h>` never pulls in `<complex.h>`, so `I` is not defined at all in
  that include chain.

The upstream maintainer confirmed the same diagnosis in the issue thread.

## Scope

Only `i8r16_group2()` is touched. The function is compiled solely under
`#if defined(__AVX512VNNI__) && defined(__AVX512BW__)`, so on Windows this
affects Zen 4/5 and Intel Ice Lake, Tiger Lake, Rocket Lake and Xeon Scalable
from Cascade Lake on. Alder Lake and later client CPUs (no AVX-512) and the
default `ARCH=x86-64-v3` build never compiled it.

Deliberately **not** changed:

- The `I` parameter and its four uses — left byte-identical.
- Other functions declaring a parameter named `I` (lines 74, 87, 152, 250, 294,
  610, 1219, 1311, …) — not a live bug.
- `hyper` in `c/qwen38_core.h` / `c/qwen38_chain.h`, and `near`/`far` in some
  test files: these *are* lowercase `<windows.h>` macro names used as
  identifiers, but in translation units that do not currently read a full
  `<windows.h>`. Latent, not live. Worth a follow-up source test.

Two alternatives were considered and not implemented, because both need a real
Windows build to validate: `#undef`ing the offending `<dlgs.h>` names after the
Windows headers, or setting `WIN32_LEAN_AND_MEAN` in `serve_poll.h` (which
`c/compat.h` already does at line 93).

## Files changed

`c/idot.h` — 1 file, +2 / −2. Both changed lines (1292, 1295) are inside
`i8r16_group2()`.

## How to run it

### Requirements

To *build* this you need the affected platform: Windows 11, MSYS2 **UCRT64**
shell, `mingw-w64-ucrt-x86_64-gcc`, and an AVX-512 VNNI CPU (or
`ARCH` set so the VNNI path compiles).

### Verify on Windows (the real test)

```bash
# In the MSYS2 UCRT64 shell:
cd c
make qwen36.exe ARCH=native     # must now compile past idot.h:1292
```

Before this change the same command stops with the
`expected identifier or '(' before numeric constant` error above.

### Verify without a toolchain

```bash
# No `grp2` identifier survives anywhere in the tree.
# (The only hit is inside the new explanatory comment, which the
#  preprocessor strips before macro expansion.)
grep -rn '\bgrp2\b' c/

# The new name is declared and used, both inside the one function.
grep -n 'grpb' c/idot.h

# The diff is confined to that function.
git diff main -- c/idot.h
```

### Observed result

```
$ grep -rn '\bgrp2\b' c/
c/idot.h:1292:  (comment only — "not grp2: <dlgs.h> ... #defines it (#2611)")
```

Structural checks after the edit, against `main`: whole-file brace count
162/162 unchanged, line count 1380 unchanged, parens inside the function
balance 41/41. A scan of every `.c/.h/.cpp/.cu/.cuh/.m/.mm` under `c/`,
`c/tests`, `c/tools` and `c/train` (comments and string literals blanked)
against the full lowercase `<dlgs.h>` macro set returns **0 hits** after the fix.

## What was NOT verified

- **Nothing was compiled or executed.** The development host has no `make`,
  `gcc`, `clang` or `nvcc`, and is aarch64 Linux while this is x86-64 AVX-512
  code. There is also no AVX-512 hardware here to run the kernel.
- The reporter validated this exact rename on MSYS2 UCRT64 with GCC 16.2.0.
- The Windows SDK's `dlgs.h` (as opposed to mingw-w64's) was not inspected.

## Reverting

```bash
git revert d610dff6
```

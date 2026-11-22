# PR #4 — `start-here.sh`: find Homebrew and versioned Python interpreters

| | |
|---|---|
| **Branch** | `fix/2812-start-here-python-detect` → `main` (this fork) |
| **Upstream issue** | <https://github.com/JustVugg/colibri/issues/2812> |
| **Kind** | Bug fix, setup / macOS |
| **Risk** | Low — detection only; the already-working path is unchanged |

## What this changes

`./start-here.sh` now looks for a usable Python under **versioned interpreter
names** and in **common install prefixes**, not just the bare `python3` and
`python` on `PATH`.

## Root cause

The reporter has Homebrew Python 3.12 and 3.15 installed, `python3 --version`
reports 3.15.2, and `./start-here.sh` still prints:

```
colibri needs Python 3.10 or newer, and it was not found.
```

The detection loop probed only two names:

```sh
for candidate in python3 python; do
```

On macOS that fails in two ways. A Homebrew Python is often reachable only as
`python3.15` / `python3.12` — the unversioned `/opt/homebrew/bin/python3`
symlink belongs to whichever `python@3.x` formula claimed it. And the `PATH` a
script inherits frequently resolves `python3` to Apple's `/usr/bin/python3`
stub, which is older than 3.10 or exits non-zero with
`xcrun: error: invalid active developer path`. The loop then tries `python`,
which does not exist on modern macOS, and gives up.

Shell aliases cannot help here: a script does not see interactive aliases,
which is why the reporter's aliases made no difference.

### The version comparison was already correct

It is tempting to blame a lexicographic `"3.15" < "3.10"` comparison. That bug
is **not** present. The original line 25 reads:

```sh
"$candidate" -c 'import sys; sys.exit(0 if sys.version_info >= (3, 10) else 1)'
```

`sys.version_info` is a numeric tuple, so 3.15 compares correctly. Verified
with a shim that forces `sys.version_info = (3, 15, 2)`: the *original* check
accepts 3.15 (exit 0) and rejects 3.9 (exit 1). The comparison is left
untouched — this PR only changes *which interpreters get probed*.

## How the search works now

A small POSIX-sh `usable_python()` helper (a `case` guard requires `-x` for
path-form candidates and uses `command -v` for bare names), then a nested loop:

| Prefixes, in order | Names, in order |
|---|---|
| `''` (plain `PATH`), `/opt/homebrew/bin/`, `/usr/local/bin/`, `/opt/local/bin/` | `python3`, `python`, `python3.19` … `python3.10` |

`PATH`'s `python3` and `python` are still tried **first**, so an already-working
setup resolves to exactly the same interpreter as before. The missing-Python
message, the per-platform install hint and the exit status 1 are unchanged.
The shebang stays `#!/bin/sh` and no new external commands are introduced.

Out of scope on purpose: pyenv shims, and keg-only Cellar/`libexec` paths.

## Files changed

`start-here.sh` — 1 file, +21 / −5.

## How to run it

### Requirements

A POSIX shell. Nothing is built.

### Verify syntax

```bash
bash -n start-here.sh
sh   -n start-here.sh
```

### Verify behaviour without macOS

`start-here.sh` goes on to run the real setup, so do not run it unguarded.
Point it at a controlled `PATH` instead. This host happens to have a 3.9
`python3` and a real `/usr/bin/python3.13`, which reproduces the bug exactly:

```bash
mkdir -p /tmp/pr4 && cd /tmp/pr4
git show main:start-here.sh > old.sh
cp /path/to/worktree/start-here.sh new.sh
chmod +x old.sh new.sh

env -i PATH=/usr/bin sh ./old.sh   # before
env -i PATH=/usr/bin sh ./new.sh   # after
```

### Observed result

```
$ env -i PATH=/usr/bin sh ./old.sh
colibri needs Python 3.10 or newer, and it was not found.

$ env -i PATH=/usr/bin sh ./new.sh
python3.13: can't open file '/tmp/pr4/coli': [Errno 2] No such file or directory
```

The second message is the point: detection succeeded and `exec`'d the launcher
under Python 3.13.18. It then fails only because the scratch directory has no
`coli` launcher, which is expected for this isolated test.

Also checked, with shim interpreters that fake `sys.version_info` and a stubbed
`uname`:

| Scenario | Before | After |
|---|---|---|
| Apple-stub `python3` that errors, + brew `python3.15`/`3.12` on `PATH` | error, exit 1 | runs `python3.15`, exit 0 |
| Versioned names only, no unversioned `python3` | error | runs `python3.15` |
| Python only in an off-`PATH` install prefix | error | runs `python3.15` |
| Already working (`python3` = 3.15 on `PATH`) | runs `python3` | same interpreter |
| Only `python3` = 3.9 | error, exit 1 | error, exit 1 |
| No Python at all | error, exit 1 | identical text and status |
| Non-executable file / dangling symlink named `python3` in a prefix | — | skipped |

The Darwin branch of the error message is byte-identical before and after.

## What was NOT verified

- **No macOS host and no real Homebrew Python.** Apple's real stub behaviour,
  the real `/opt/homebrew` layout, and macOS `/bin/sh` (bash 3.2 in sh mode)
  were modelled with shims; the reporter's scenario was not reproduced on real
  hardware.
- The literal prefix paths are correct **by inspection only** — `/opt` is not
  writable on the development host, so the mechanism was exercised with
  substituted directories.
- Nothing past detection was exercised: there is no `make`/`gcc`/`clang` here,
  so setup itself never ran.

## Upstream note

Upstream has since fixed this on its `dev` branch (PR #2815, +85 / −6), taking
the same approach and additionally adding a `COLIBRI_PYTHON_DIRS` override and
a committed regression test (`tests/test_start_here_python_detection.sh`). That
work is **not** on this fork's `main`. If you later sync with `dev`, prefer
upstream's version and drop this one.

## Reverting

```bash
git revert b3df2ff0
```

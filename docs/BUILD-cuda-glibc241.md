# Building the CUDA backend on glibc ≥ 2.41 (Debian trixie and newer)

**Author:** Peter Groom · 2026-07-18

Symptom: `make glm CUDA=1` fails with four errors like

    mathcalls.h(79): error: exception specification is incompatible with that of
    previous function "cospi" (declared at ... crt/math_functions.h)

Cause: glibc 2.41 added C23 `sinpi/cospi/sinpif/cospif` with exception specs that clash
with CUDA ≤ 12.9's `math_functions.h` declarations. **CUDA 13.x headers fix it.**

Recipe that needs neither root-owned toolkits nor distro driver packages (safe next to a
`.run`-installed or dkms driver — important in LXC setups where userland must match the
host kernel module version):

```bash
# micromamba (single static binary) + NVIDIA's conda channel = complete nvcc
curl -Ls https://micro.mamba.pm/api/micromamba/linux-64/latest | tar -xj -C /opt bin/micromamba
/opt/bin/micromamba create -y -p /opt/cuda13 -c nvidia -c conda-forge cuda-nvcc=13 cuda-cudart-dev=13
ln -s /opt/cuda13/lib /opt/cuda13/lib64      # Makefile expects lib64; conda ships lib

make glm CUDA=1 CUDA_ARCH=sm_86 CUDA_HOME=/opt/cuda13   # your arch here
```

The binary links `libcudart.so.13` dynamically — ship it next to the binary and run with
`LD_LIBRARY_PATH`, or drop it in `/usr/local/lib`. Driver ≥ the runtime's minimum is the
only host requirement (any recent driver runs CUDA 13 binaries).

## Opting out of the bundled shim (`CUDA_GLIBC_MATH_COMPAT`)

Since [#966](https://github.com/JustVugg/colibri/issues/966) the Linux CUDA build force-includes
`c/glibc_c23_math_compat.h` into nvcc's host pass (`-Xcompiler=-include,...`), which strips
`__THROW` from those declarations and makes the clash above go away without a newer toolkit.

| `c/Makefile` variable | what `NVCCFLAGS` gets |
|---|---|
| `CUDA_GLIBC_MATH_COMPAT=1` — the default | `-O3 -std=$(NVCC_STD) -ftz=false $(CUDA_GENCODE) -Xcompiler=-Wall,-Wextra -Xcompiler=-include,$(CURDIR)/glibc_c23_math_compat.h` |
| `CUDA_GLIBC_MATH_COMPAT=0` | the same line **without** the trailing `-Xcompiler=-include,...`; `-O3`, `-std`, `-ftz=false`, `$(CUDA_GENCODE)` and the `-Wall,-Wextra` host warnings are untouched |

Set it to `0` when the toolchain already reconciles the two declarations — CUDA 13.x headers do —
or when a distribution patches the compatibility in elsewhere and wants one fewer forced include.
It is a build-interface switch only: no kernel, model-format or numerical change, and it does not
touch the CPU build, Windows or macOS (the shim has always been Linux-only, since MSVC never sees
glibc headers). Prefer it over overriding `NVCCFLAGS` wholesale, which would also discard the
architecture and warning flags ([#2647](https://github.com/JustVugg/colibri/issues/2647)). Only
the exact value `1` keeps the include, as with `LTO`, so a typo reads as the opt-out — which the
check below prints.

Which recipe a build will use can be checked without nvcc, a CUDA toolkit or a GPU:

```bash
make -C c cuda-glibc-math-compat-check                            # default: include on
make -C c cuda-glibc-math-compat-check CUDA_GLIBC_MATH_COMPAT=0   # opt-out: include off
```

It prints the selected mode and the resulting `NVCCFLAGS`, and exits non-zero if the two
disagree — which also catches a hand-written `NVCCFLAGS=...` on the command line, since a
command-line value overrides the Makefile's `+=` and would silently drop the include.

Dead ends, for the record: Debian's `nvidia-cuda-toolkit` hard-depends on Debian's driver
userland (version-clashes a `.run`/dkms driver — and its `nvidia-installer-cleanup`
postinst will try to delete your hand-installed userland); pip's `nvidia-cuda-nvcc-cu12`
wheel ships only `ptxas`, not the compiler frontend.

---
title: "Oracle and gate"
weight: 40
description: "gfortran 15.2.0 as the reference, the foracle commands, and the gate of bib section 4.2 with its first figures and its verdict, A first."
---

<!-- From the README of go-embedded-fortran/fortran at v0.16.0, sections "The oracle" and "The gate (bib §4.2)". Change the README first, then here. -->

## The oracle

The reference is **gfortran 15.2.0** run natively (GCC `releases/gcc-15.2.0`):
it is the compiler LFortran's and llvm-test-suite's reference outputs are made
with. It is GPL and is only ever run as a program, never linked or ported (see
`NOTICE`). "Correct" means the same exit status and the same stdout and stderr
bytes, except where the output is processor-dependent (bib section 8).

```sh
go install github.com/go-embedded-fortran/fortran/cmd/foracle@latest
export FORACLE_HOME=$HOME/foracle-home
foracle toolchain          # installs gfortran 15.2.0 into $FORACLE_HOME, no root
foracle version
foracle run prog.f90       # compile with the oracle, run, relay stdout/stderr/status
```

gfortran is obtained as the conda-forge package `gfortran==15.2.0`, installed by
a pinned **micromamba** (release `2.10.0-0`, sha256 checked) into
`$FORACLE_HOME/env`. `foracle` checks `gfortran --version` and refuses anything
but 15.2.0. macOS arm64 and Linux amd64 are supported. flang 23.1.4 is optional
and Linux only; that path is written but not exercised by CI yet.

Programs are compiled with `-O0 -w -fno-backtrace`. Runs are bounded: a
wall-clock timeout (10 s), a cap on stdout and stderr (1 MiB), a fixed
environment and an address-space limit (1 GiB). The memory limit is enforced on
Linux; macOS accepts `RLIMIT_AS` but does not enforce it.

| command | what it does |
|---|---|
| `foracle golden` | regenerates `testdata/programs.golden` from the 14 programs in `testdata/programs/` |
| `foracle corpus --dir D` | compiles and runs every Fortran file below D; one line per file |
| `foracle bench` | the reference column of the gate: the four kernels with gfortran `-O2`, 7 rounds, median and spread; refuses above 1-minute load 8 |
| `foracle gate [--smoke] [--wait-load D]` | the whole gate: the contenders on the kernels, checksum bounds, the decision rule; refuses above 1-minute load 8, or with `--wait-load` waits and reruns a round the load spoiled |
| `foracle parse-rate --dir D` | parses every Fortran file below D and compares with `gfortran -fsyntax-only` |
| `foracle parse-bench --dir D` | parse throughput against `gfortran -fsyntax-only`; refused above load 8 |
| `foracle toolchain --home H [--flang]` | installs the pinned toolchain |
| `foracle version` | prints the pinned versions and whether the toolchain was found |

The 14 golden programs (`testdata/programs/*.f90`) are original: list-directed
and formatted output, integer and real kinds, arrays and sections, column-major
order, modules, derived types, recursion, strings and internal I/O, `stop`,
`error stop` and stderr. CI regenerates the golden file on linux/amd64 and
darwin/arm64 and requires no difference.

No corpus is committed (the gcc testsuite is GPL; FCVS has its own terms). CI
fetches them by commit: gcc `gfortran.dg` and `gfortran.fortran-torture` (gcc
15.2.0, `5115c7e447fc`), llvm-test-suite FCVS (`19b349c5aa39`) and LFortran
`integration_tests` (`4f2cd049c87f`).

## The gate (bib section 4.2)

`foracle gate` decides between (A) a pure-Go evaluator and (B) compiling to
wasm32. The decision rule is fixed in bib section 4.2 and implemented once, in
`internal/gate/rule.go`, with its thresholds (5, 2, 1.2) pinned by a test.

| contender | what it is |
|---|---|
| **A** | package `eval` (prototype, the seed of M3-A): the typed tree compiled once into closures over typed slots; `real(8)` arrays of rank 1 and 2, column-major, bounds-checked; whole-array expressions and `matmul` handed to go-ndarray v0.40.3 |
| **B-near**, **B-upper** | the C transliterations in `testdata/gate/c/` compiled by clang 23.1.4 to `wasm32-wasip1` at `-O0` and `-O2`, run on wazero v1.12.0 (the version go-embedded-c pins) |
| **gfortran** | 15.2.0 `-O2`, native: the reference |
| **direct** | go-ndarray called straight from Go, no evaluator, no copy: the baseline of A's array kernels |

Every contender times its kernel with a clock around the kernel itself, one core
(`GOMAXPROCS=1`), in interleaved rounds (7 rounds, median and spread). **A
checksum out of its bound is a failure, not a time.** The bound is computed with
`math/big` and a contender is within `1.01 * m * 2^-53 * exact`, `m` being the
number of rounding steps on the longest chain of the kernel (nonnegative data,
so the bound holds for any summation order, FMA or vectorisation).

Only the desktop home is implemented: the playground (headless Chromium) needs
a browser lane that does not exist yet
([#5](https://github.com/go-embedded-fortran/fortran/issues/5)). The rule needs
both homes to pick A, so with the desktop alone it can only answer B or "A
first".

### Figures

2026-11-14, v0.10.0, `foracle gate --wait-load 10h`. Desktop darwin/arm64, one
core, 7 interleaved rounds; every kept round is bounded by 1-minute load samples
between 6.2 and 7.9; 3 rounds were discarded for load after 32 minutes of
waiting. Seconds, median and spread (max-min):

| kernel | A | B-near | B-upper | gfortran -O2 | direct go-ndarray |
|---|---|---|---|---|---|
| K1 | 0.5050 +0.0141 | 1.5028 +0.0473 | 0.7010 +0.0741 | 0.0606 +0.0009 | - |
| K2 | 3.4121 +0.2056 | 3.3757 +0.4956 | 1.6949 +0.0554 | 0.0366 +0.0019 | - |
| K3L | 2.0977 +0.3126 | 2.2330 +0.2965 | 1.0762 +0.0276 | 0.0194 +0.0002 | - |
| K3M | 0.0066 +0.0012 | 0.0067 +0.0027 | 0.0071 +0.0015 | 0.0058 +0.0003 | 0.0062 +0.0009 |
| K4 | 0.0712 +0.0081 | 0.2940 +0.0242 | 0.2728 +0.0676 | 0.0350 +0.0113 | 0.0898 +0.0194 |

Time over gfortran on the loop kernels: K1 A 8.33, B-near 24.79, B-upper 11.56;
K2 93.31, 92.31, 46.35; K3L 108.06, 115.03, 55.44. Inputs to the rule:
g(rA) = 43.79, g(rB0) = 64.09, **g(rA)/g(rB0) = 0.683**; K4 A / direct = 0.793,
K3-matmul A / direct = 1.065.

**Verdict: A first** (between; re-gate after go-embedded-c M3). The desktop
ratio is at most 2 and both array kernels are within 1.2 of direct go-ndarray,
but the playground home is not measured (#5), so the rule cannot pick A
outright. The gap to gfortran on scalar loops (8x to 108x) is an open defect:
[#17](https://github.com/go-embedded-fortran/fortran/issues/17).

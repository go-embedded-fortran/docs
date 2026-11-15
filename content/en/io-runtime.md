---
title: "I/O runtime"
weight: 30
description: "Package rt: formatted and list-directed input and output ported from flang-rt, made to print what gfortran 15.2.0 prints, with its differential."
---

<!-- From the README of go-embedded-fortran/fortran at v0.16.0, section "The I/O runtime (M4, part 1)". Change the README first, then here. -->

Package `rt` is the Fortran runtime's formatted and list-directed input and
output (bib section 5 row 1), in pure Go, ported from LLVM flang-rt
(llvmorg-23.1.4, Apache-2.0 WITH LLVM-exception, see `NOTICE` and the headers of
the ported files) and made to print what **gfortran 15.2.0** prints, because
that is what a notebook's `print *` shows. `eval`'s `print` and `write` run on it
(units `*`, 6 and 0; a format is a character literal or `*`).

```go
tab := rt.NewTable(stdout, stderr, stdin, fsys) // units 6, 0, 5; fsys is read-only
u := tab.Get(6)
err := u.Write(rt.Spec{Format: "(F8.3,1X,ES12.4)", HasFormat: true},
    []rt.Val{rt.Real8(3.14159), rt.Real8(1e-5)}) // nil, or *rt.Error{Stat, Msg}
r := rt.NewInternalRead("  12 3.5 abc")            // read from a character variable
err = r.Read(rt.Spec{}, items)                     // list-directed; items updated in place
```

| | |
|---|---|
| edit descriptors | `I B O Z F E D EN ES G L A`; `EX` is implemented (flang-rt's) but off, because gfortran 15.2 refuses it (`rt.EnableEX`) |
| control | `X T TL TR / : kP S SP SS BN BZ RN RU RD RZ RC RP DC DP`, quoted strings and `nH`, repeat counts, nested groups, `*( )`, format reversion |
| list-directed | output as gfortran; input with `r*c`, null values, `/`, quoted strings, complex pairs |
| units | in-memory table (5, 6, 0), read-only files on an `fs.FS`, in-memory scratch files with rewind/backspace, internal files (scalar and array), `IOSTAT` values as libgfortran's, non-advancing output |
| bounds | format length, nesting depth, repeat count, field width, digits, items per statement, record length and file size are refusals, asserted under a memory limit; `FuzzFormat` and `FuzzInput` (corpus committed) |

Float to decimal is correctly rounded: `TestDecimalAgainstBig` compares the
digits for the classic hard cases and 3000 random doubles, at 11 digit counts,
with `math/big`; `TestRoundingModesBracket` checks RU, RD, RZ and RC against the
exact expansion.

## Differential against gfortran 15.2.0

`rt/diff_test.go` and `rt/diffin_test.go` generate programs (formats times
values, including ±0, denormals, infinities, NaN, field overflow, huge widths,
errors with `iostat`), compile and run them with the pinned gfortran, and
compare the text byte for byte with rt's. Last run, 71 135 cases:
**71 096 identical, 39 differing in documented classes, 0 unexplained**.

| area | cases | identical | kept differences |
|---|---|---|---|
| REAL(8) and REAL(4), every real edit descriptor, widths, digits, kP, S, rounding modes, DC | 63 674 | 63 648 | 26: `EN` with many digits (REAL(4) with d ≥ 6 at a carry boundary, d ≥ 16): gfortran rounds to d+1 digits and pads with zeros |
| INTEGER, LOGICAL, CHARACTER, B/O/Z, mismatches | 4 063 | 4 063 | |
| control edit descriptors, reversion, T/TL/TR, format errors | 86 | 86 | |
| list-directed output (REAL, COMPLEX, mixed lists, 40-item lines) | 632 | 632 | |
| formatted and list-directed input | 2 576 | 2 563 | 13: BZ with an exponent letter or a lone sign (6), `;` without `DC` (1), a cut-short complex (6) |
| repeat counts `r*c` | 104 | 104 | |

The differences gfortran has from the standard (and from flang-rt) that `rt`
copies are listed in the package documentation (`go doc ./rt`): `F0.d` prints
`.50`, `E0.d` an E+000 exponent, `G8.1` of 0.95 is `1.`, `I5.8` is refused, `A`
and `L` accept integers, a repeat count cannot cross kinds on input, and so on.
Of the 14 golden programs, `eval` runs all 14 and they are byte-identical to
their goldens (stdout, stderr, exit status); `TestGoldenPrograms` counts the
runs and the refusals.

## Performance

A statement on a warm unit allocates nothing (0 allocations for `F`, `ES` and
list-directed writes, asserted by `TestAllocations`; formats are compiled once
and cached). Throughput against gfortran's own I/O is
`RT_PERF=1 go test ./rt -run TestPerfVsGfortran -v` (interleaved rounds, single
core, gfortran -O2); it refuses to measure at a one-minute load of 8 or more.
**Not measured: load** (the machine was at 38 to 51 when this was written); no
figure is claimed.

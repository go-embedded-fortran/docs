---
title: "Front end"
weight: 20
description: "The parser (syntax), its parse rate on three corpora, the semantic analysis (sema) with its differential against gfortran, and the cell splitter."
---

<!-- From the README of go-embedded-fortran/fortran at v0.17.0, sections "The parser", "Semantic analysis" and "The cell splitter". Change the README first, then here. -->

## The parser

`github.com/go-embedded-fortran/fortran/syntax` is a pure-Go (`CGO_ENABLED=0`)
lexer and recursive-descent parser producing a typed AST with byte offsets that
resolve to `file:line:col`. The package documentation cites the clauses of F2018
(J3/18-007r1) that were read for it: 6.3.1 to 6.3.3 (source form, free and
fixed), 6.4 (`INCLUDE`), and 10.1.1 (operator precedence). The AST design
follows LFortran's `AST.asdl` (BSD-3, a design reference); its GLR grammar is
not ported.

```go
f, err := syntax.Parse("prog.f90", src, &syntax.Options{FS: includeFS})
// err is a syntax.ErrorList: "prog.f90:12:7: expected ')', found ','"
for _, u := range f.Units { ... }
fmt.Print(syntax.Print(f))   // canonical free-form source; parses back to the same AST
```

- **Source form** by extension (`.f .for .f77 .ftn` fixed, the rest free, as
  gfortran) or `Options.Form`.
- **`INCLUDE`** resolves through `Options.FS` only: with a nil FS every include
  is refused, the host file system is never read by default, absolute and `..`
  paths are refused, and an include of a file already being included is refused.
- **Names and keywords** are case-insensitive and normalised to lower case.
  There are no reserved words (`if = 1` is an assignment).
- **Errors** are `file:line:col: message`; the parser recovers at the next
  statement and never panics. Bounds assert a refusal: source bytes (16 MiB,
  also summed over includes), nesting depth (200), include depth (8),
  continuation lines of a statement (255, the standard's limit), tokens of a
  statement (262 144) and reported errors (50).
- **Tests**: a printer round trip on every file of `testdata/programs` and
  `testdata/gate` (19 files, all parse), golden AST dumps, three fuzz targets
  (`FuzzParseFree`, `FuzzParseFixed`, `FuzzLex`) whose corpus is committed, and
  bomb tests that assert a refusal under a soft memory limit.

### Parse rate on the corpora

`foracle parse-rate --dir D` parses every Fortran file below D and runs
`gfortran -fsyntax-only` on the same file. Numbers taken on the pinned
revisions with gfortran 15.2.0:

| corpus | files | we parse | gfortran accepts | both accept | fail-only-ours | fail-gfortran-only | fail-both |
|---|---|---|---|---|---|---|---|
| FCVS (`fcvs21_f95`) | 180 | 180 (100 %) | 179 | 179 | 0 | 1 | 0 |
| gcc `gfortran.fortran-torture` | 314 | 313 (99.7 %) | 313 | 312 | 1 | 1 | 0 |
| gcc `gfortran.dg` | 8 306 | 7 496 (90.2 %) | 6 096 | 5 973 | 123 | 1 523 | 687 |

Of the files gfortran accepts, the parser accepts 100 % of FCVS, 99.7 % of the
torture suite and 98.0 % of `gfortran.dg`. The 123 `gfortran.dg` files that
gfortran accepts and we refuse are, by first error: derived-type parameters
(30, excluded from v1), DEC `parameter x = v` without parentheses and other
extensions (28), preprocessor remnants of `.F90` files (19), `contains`
followed by `module procedure` (17, submodules are v1-excluded), coarray
statements (7), files with more than 255 continuation lines (5, where gfortran
is lenient and we apply the standard's limit on purpose), Hollerith outside
`FORMAT`, UTF-16/32 encodings, and a few single files.

**Not measured yet:** the throughput against `gfortran -fsyntax-only`. The
development machine's 1-minute load stayed between 10 and 100, above the
fleet's limit of 8, so `parse-bench` refused to run and no throughput figure is
published rather than a loaded one. Allocation counts are exact under load:
parsing the 19 testdata files is 4 470 allocations (380 KB) per pass.

## Semantic analysis

`github.com/go-embedded-fortran/fortran/sema` turns a parsed file into the
**typed tree**: the syntax tree annotated by `sema.Info` (the design of
`go/types`). `Info.Types` maps every expression to its type, kind, rank and
constant value, `Info.Uses` maps a name to its symbol, `Info.Scopes` maps a
program unit to its scope. Both back ends of bib section 4.2 walk the one tree.
LFortran's ASR is the design reference (BSD-3); no code was ported.

```go
f, _ := syntax.Parse("a.f90", src, nil)
info, errs := sema.Analyze(f, &sema.Options{ImplicitNone: true})
// errs: "a.f90:3:5: Symbol 'x' at (1) has no IMPLICIT type"
```

- **Part 1**: scopes and symbols, implicit typing and `implicit none`
  (`Options.ImplicitNone` is the notebook default), kinds validated against
  gfortran's kinds, folding of named constants, typing of expressions with the
  promotion rules of Table 10.2, derived types, use association,
  `iso_fortran_env` and `iso_c_binding` constants, statement functions and
  statement checks.
- **Part 2, calls**: generic resolution (the typed tree names the specific),
  ambiguity (C1511, C1512, C1514), argument association (count, keywords,
  type, kind, rank, `pointer`/`allocatable`, definable `intent(out)` actuals),
  explicit-interface requirements, function versus subroutine, defined
  operations and assignment.
- **Part 3, shapes**: `TV.Shape` is the extent of each dimension, a number when
  it is a compile-time constant and -1 when it is not; conformance of operands,
  array assignments, initialisations, elemental arguments and `where`; the
  shape rules of `reshape`, `matmul`, `transpose`, `dot_product`, `spread` and
  the reductions. An unknown extent is never the reason for a refusal.

The analyser never panics (a panic is recovered into an error and the fuzzer
treats it as a failure); its bounds (`Limits`) are refusals, asserted by bomb
tests.

### Differential against gfortran 15.2.0

`foracle sema-rate --dir D` parses and analyses every file and compares with
`gfortran -fsyntax-only`, which also makes the semantic checks. The verdicts of
gfortran are committed (`sema/testdata/gf-*.tsv`), so that `sema/corpus_test.go`
asserts the counts without the compiler.

| corpus | files | both accept | refused only by us | accepted only by us | both refuse |
|---|---|---|---|---|---|
| FCVS | 180 | 179 | 0 | 1 | 0 |
| gcc `gfortran.fortran-torture` | 314 | 313 | 1 (parser) | 0 | 0 |
| gcc `gfortran.dg` | 8 306 | 5 753 | 291 (125 parser, 166 analyser) | 1 202 | 1 060 |
| `gfortran.dg`, `dg-do run` files | 3 151 | 2 675 | 155 | 95 | 226 |

The 1 202 files accepted only by us are the `gfortran.dg` tests written to be
refused for reasons not checked yet (type-bound procedure rules, input/output
tag checks, polymorphism, `-std` conformance). The refused-only-by-us files are
defects or gaps of ours. `BenchmarkAnalyze` (the 12 programs of
`testdata/programs`, one core) is 3 733 allocations and 550 KB per pass, exact;
its time was **not measured: load**.

## The cell splitter and the session module

`sema.Session` analyses the cells of a notebook in order, with the shape of
go-notebook's `analysis.Sequential` (`Begin`, `AnalyzeNext(state, index, src)`).
`syntax.Options.Cell` is the grammar of a cell: statements outside any program
unit are main-program material that needs no `END`. A cell is then split:

- top-level declarations and top-level subroutines and functions go into the
  session module `__cell<N>`;
- top-level executable statements are wrapped in the subroutine
  `__cell<N>_run`, a procedure of that module;
- every earlier session module and every module an earlier cell defined is an
  implicit `use` of `__cell<N>`;
- redefinition shadows; implicit typing is off unless
  `SessionOptions.ImplicitTyping`;
- a cell with errors returns the state it was given, so a broken cell leaves
  nothing behind.

`sema.GoType` is the values table of bib section 7 (`integer(k)` to
`int8..int64`, `real` to `float32|float64`, `complex(4|8)` to
`complex64|complex128`, `logical` to `bool`, `character` to `string`, arrays to
`*ndarray.Array`, derived types opaque; `integer(16)`, `real(10)` and
`real(16)` have none). The adapter that fills go-notebook's `analysis.Cell`
belongs to the language registration in go-notebook (`analysis/parsers`).

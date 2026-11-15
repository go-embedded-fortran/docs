---
title: "Status"
weight: 10
description: "The milestones M0 to M6 as the fortran README states them, what the evaluator runs, and what it still refuses."
---

<!-- From the README of go-embedded-fortran/fortran at v0.16.0, sections "Status" and "Not yet". Change the README first, then here. -->

## Milestones

| milestone | contents | state |
|---|---|---|
| M0 oracle harness | `internal/oracle`, `cmd/foracle`, pinned gfortran 15.2.0, golden file, corpus runner, reference column of the gate | **done** |
| M1 front end, part 1 | lexer (free and fixed form, continuation, `include`), parser for the gate's kernels; **the §4.2 gate**, report published | in progress (syntax done) |
| M2 front end, part 2 | semantics to a typed tree, the v1 subset, cell splitter and session module | in progress: package `sema` (scopes, symbols, implicit typing, kinds, constants, expression typing, `use`, derived types), the cell splitter, session module and go-notebook-facing analysis, generic resolution, argument association, explicit-interface requirements and `intent` at calls, shape conformance, and `eval` consumes the typed tree; the playground side of the gate and the remaining semantics are open |
| M3 back end | as the gate decides: A = closure compiler over the typed tree, B = lowering to go-embedded-c's IR | planned |
| M4 runtime | I/O (ported from flang-rt), intrinsics on go-ndarray and go-fft | in progress: package `rt` (formatted and list-directed input and output, units, internal files), and `eval`'s print and write run on it; the elemental intrinsics and the rest of the runtime are open |
| M5 notebook bridge | values table, `file:line` errors, cells on desktop and playground | planned |
| M6 conformance | FCVS, gfortran.dg run tests, reference LAPACK built by our front end; performance table | planned |

## The evaluator, part by part

There is no complete evaluator yet: package `eval` grows from the gate's
prototype into the v1 language in parts (M3-A, bib section 10 row M3).

| part | release | adds |
|---|---|---|
| 1 | v0.11.0 | the kinds (integer(1\|2\|4\|8), logical, real(4), real(8)), relational and logical operators, `if` constructs, `do while`, `exit` and `cycle`, `select case` on integers and logicals, `stop` and `error stop`, output of constants of every intrinsic type, implied-do output lists, `write` to `error_unit` |
| 2 | v0.12.0 | internal subroutines and functions, external ones of the same file, scalar dummy arguments, `result(r)`, `recursive` (bounded by `MaxCallDepth`, 2^15), `return`, the bit functions |
| 3 | v0.13.0 | modules of the file (variables, named constants, module procedures, `use`, `only`, renames, `private`/`public`, generic interfaces) and character scalars |
| 4 | v0.14.0 | arrays of the general kind (integer, logical, real(4\|8), rank 1 to 7, any lower bounds, column-major, bounds-checked), allocatable arrays, sections with strides, array constructors, whole-array and elemental expressions, `where` (no `elsewhere`), assumed-shape dummies, and the intrinsics size, shape, lbound, ubound, sum, product, maxval, minval, maxloc, minloc, any, all, count, reshape, transpose, matmul, dot_product, pack, spread, cshift and eoshift |
| 5 | v0.15.0 | substrings, character variables whose length is an expression of the dummy arguments, `character(len=:), allocatable`, merge, scan, verify, and internal files (WRITE to and READ from a character variable, with `iostat=`) |
| 6 | v0.16.0 | derived types: scalar objects and rank-1 fixed-size arrays of them, scalar integer, real, logical and character components, array components (fixed shape or allocatable), components of derived type, default initialisation, structure constructors, component references, deep-copy intrinsic assignment, type-bound procedures, `move_alloc`; with it `eval` runs all 14 golden programs |

Resource bounds are refusals: an array of more than `MaxArrayElems` (2^28)
elements, a character result of more than `MaxStringLen` (2^26) characters and an
output list of more than `MaxArrayElems` values are refused at run time before
anything is allocated.

## Still refused by the evaluator

Type extension, `select type` and dynamic dispatch, pointer components,
derived-type named constants and allocatable derived-type variables, arrays of
derived type of rank above 1 or with sections, character arrays, `read` from a
unit, complex variables, vector subscripts, `elsewhere`, a `dim=` or `mask=`
argument of a reduction, optional arguments, an initialised local variable of a
procedure, host- and module-associated arrays, internal files and `read` (the README's
list, as written; part 5 above added internal files, and the README does not
reconcile the two).

## Not yet

- Semantics not done, of bib section 1.1: shapes of pointer assignment, of
  `forall` and of type-bound results, type-bound generics and `pass`,
  the `!GCC$ ATTRIBUTES NO_ARG_CHECK` directive, `select type` dispatch,
  derived-type I/O, submodules, `entry` result typing, `common`/`equivalence`
  storage rules, `data` value conversion, the go-notebook adapter and the
  "value from Go into a cell" direction of bib section 7.
- Not parsed yet: submodule procedure bodies, derived-type parameters (refused
  by design), coarray and team statements, `do concurrent` locality
  specifiers, Hollerith constants outside `FORMAT`, Cray pointers and DEC
  extensions, and `#` preprocessor directives (such lines are skipped, never
  interpreted).
- The flang path of the oracle is not exercised in CI, and the Linux memory
  limit is tested only in CI.

---
title: "Licences"
weight: 50
description: "BSD 3-Clause; what is ported with its notice, and what is only ever run."
---

<!-- From the README and NOTICE statements of go-embedded-fortran/fortran at v0.17.0. Change the README first, then here. -->

The `fortran` repository is under the BSD 3-Clause licence (see its `LICENSE`).
Its `NOTICE` lists what is only run.

| Component | How it is used |
|---|---|
| gfortran 15.2.0 (GPL) | only ever run as a program, the oracle; never linked or ported |
| the gcc `gfortran.dg` and `gfortran.fortran-torture` testsuites (GPL) | fetched by CI by commit; none is committed |
| llvm-test-suite FCVS | fetched by CI; none is committed; it has its own terms |
| LLVM flang-rt (Apache-2.0 WITH LLVM-exception) | ported to Go in package `rt`, with its notice and in the headers of the ported files |
| LFortran (BSD-3) | a design reference for the AST and the typed tree; no code was ported, so `NOTICE` is unchanged |
| the Fortran standard, F2018 (J3/18-007r1) | the clauses read for each package are cited in its documentation |

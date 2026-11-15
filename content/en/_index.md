---
title: "go-embedded-fortran documentation"
linkTitle: "Home"
type: docs
cascade:
  type: docs
description: "Fortran in pure Go, in development: a front end, an evaluator checked against gfortran 15.2.0, an I/O runtime ported from flang-rt, an oracle harness and a measured gate."
---

<!-- Every statement and figure on this site is from the README of go-embedded-fortran/fortran at v0.17.0 (github.com/go-embedded-fortran/fortran). Change the README first, then here. -->

**go-embedded-fortran is in development.** Its repository,
[fortran](https://github.com/go-embedded-fortran/fortran) (module
`github.com/go-embedded-fortran/fortran`, latest release v0.17.0), is meant to
become a Fortran implementation written in Go (`CGO_ENABLED=0`), embedded in Go
programs and run in the browser through WebAssembly. It backs the
` ```fortran ` cells of [go-notebook](https://github.com/go-notebook) since
v0.17.0, which run in one session (`eval.Session`; their variables do not cross
to other languages yet, and the playground's core does not link the runtime
yet), as [go-embedded-c](https://github.com/go-embedded-c/c) backs its C cells.

The plan is the bibliography of the go-scikit roadmap,
[`bib/fortran.md`](https://github.com/go-scikit/roadmap/blob/main/bib/fortran.md).
Every stage is checked against a reference, never against expectations written
by hand.

## What exists at v0.17.0

- **A front end in Go.** `syntax`, a lexer and parser (free and fixed form),
  and `sema`, a semantic analyser producing a typed tree.
  See [Front end]({{< relref "front-end" >}}).
- **An evaluator.** `eval` runs all 14 golden programs, byte-identical to their
  goldens (stdout, stderr and exit status), which gfortran 15.2.0 produced.
- **An I/O runtime.** `rt`, ported from flang-rt: 71 096 of 71 135 differential
  cases against gfortran 15.2.0 are identical and the 39 others are in
  documented classes. See [I/O runtime]({{< relref "io-runtime" >}}).
- **An oracle harness and a gate.** `foracle` runs gfortran 15.2.0 as a
  program, and `foracle gate` decides the back end from measurements; its first
  figures give the verdict **A first**. See [Oracle and gate]({{< relref "oracle-and-gate" >}}).

## Known limits

- The evaluator's scalar loops are 8x to 108x behind gfortran -O2: open defect
  [#17](https://github.com/go-embedded-fortran/fortran/issues/17).
- The gate's playground home (a headless browser lane) is not implemented, so
  the rule cannot pick A outright:
  [#5](https://github.com/go-embedded-fortran/fortran/issues/5).
- Some performance figures were not measured because the machine's load was
  above the bound of 8; none is published rather than a loaded one.
- The evaluator still refuses part of the language; the list is on
  [Status]({{< relref "status" >}}).

## Pages

| Page | What it holds |
| --- | --- |
| [Status]({{< relref "status" >}}) | the milestones M0 to M6 and what is not done yet |
| [Front end]({{< relref "front-end" >}}) | the parser, its parse rate on three corpora, the semantic analysis and the cell splitter |
| [I/O runtime]({{< relref "io-runtime" >}}) | `rt`: edit descriptors, units, bounds and the differential against gfortran |
| [Oracle and gate]({{< relref "oracle-and-gate" >}}) | gfortran 15.2.0 as the reference, `foracle`, and the gate of bib section 4.2 with its figures |
| [Licences]({{< relref "licences" >}}) | what is ported, what is only run |

## Repositories

| Repo | What it is |
| --- | --- |
| [fortran](https://github.com/go-embedded-fortran/fortran) | the implementation (source) |
| [docs](https://github.com/go-embedded-fortran/docs) | this site |
| [go-embedded-fortran.github.io](https://github.com/go-embedded-fortran/go-embedded-fortran.github.io) | the landing page |
| [brand](https://github.com/go-embedded-fortran/brand) | the logo, icons and social images |

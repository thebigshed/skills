---
name: gnatmetric
description: This skill should be used when the user wants to "compute code metrics", "run gnatmetric", "measure cyclomatic complexity", "measure McCabe complexity", "check essential complexity", "measure coupling", "count lines of code", or is otherwise working with GNATmetric to measure Ada source code size, complexity, or coupling.
license: Apache-2.0
metadata:
  version: "0.1.0"
---

# GNATmetric Code Metrics Skill

`gnatmetric` is AdaCore's static metrics tool: it computes McCabe cyclomatic
complexity, essential complexity, size (line/declaration/statement counts),
and coupling metrics for Ada source. It does not check correctness or prove
anything (that's `gnatprove`); it measures structure — the same role tools
like McCabe IQ/Battlemap historically played for other languages.

## Locating gnatmetric

Unless the user has already told you how to invoke gnatmetric, determine the
right invocation in this order:

1. **Check for an `alire.toml` in the project root.** If present, use
   `alr exec -- gnatmetric ...` (or `PATH="$HOME/.alire/bin:$PATH" alr exec --
   gnatmetric ...` if it was installed standalone via `alr install`, since
   `$HOME/.alire/bin` is not automatically on `PATH` even under `alr exec` —
   same caveat as the `gnattest` skill).
2. **Check if `gnatmetric` is already on `PATH`.** Run `which gnatmetric`.
3. **If not found anywhere, it needs to be built from source.** `gnatmetric`
   is not part of a plain GNAT/GCC install, and AdaCore's old GNAT Community
   edition that used to bundle it prebuilt was discontinued in 2022. Today
   it comes from AdaCore's `libadalang_tools` crate on Alire, built from
   source (no prebuilt binary release exists for it, so `alr install` won't
   work even on Alire versions new enough to have that subcommand). See
   [building-gnatmetric.md](references/building-gnatmetric.md) for the build
   recipe and a set of environment gotchas worth checking before re-deriving
   a fix from scratch (an older Alire not wiring its own downloaded GNAT
   toolchain into `PATH`, a project's Alire dependency failing to resolve,
   `alr` prompting to install a system package mid-run, and running against
   a project with no `.gpr` file at all).
4. **Ask the user.** GNAT Pro installations may need environment setup
   beyond a binary path. Don't guess.

## Quick Start

### Per-file text metrics for a project

```bash
gnatmetric -P <project.gpr> -U
```

`-U` (with `-P` and no explicit source arguments) processes the closure of
the whole project, not just its immediate units — without it, `gnatmetric`
silently only measures the project's immediate units, which under-reports
everything and makes coupling metrics wrong. Default behavior with bare
`-P` (no `-U`) is immediate-units-only; always add `-U` unless deliberately
scoping to one unit's closure with `-U <main_unit>`.

Each source file gets its own `<file>.metrix` text report, placed in the
project's object directory (or the project's source directory if it has no
object directory) unless redirected with `--output-dir=<dir>`.

### Whole-program coupling metrics

```bash
gnatmetric -P <project.gpr> -U --coupling-all
```

Coupling metrics are inherently whole-program: `gnatmetric` only counts
dependencies among the units it was actually given, so a partial file list
produces wrong (silently incomplete) numbers. Always use `-P ... -U` for
coupling, never a hand-picked file list.

### XML output (for scripted/aggregated analysis)

```bash
gnatmetric -P <project.gpr> -U --generate-xml-output --xml-file-name=metrics.xml
```

Without a project file, XML defaults to `metrix.xml` in the current
directory. `--no-text-output` suppresses the per-file `.metrix` files and
forces XML on. Use `--generate-xml-schema` to also emit a matching `.xsd`.

### One metric category only

By default `gnatmetric` reports everything. Passing any positive metric
switch (e.g. `--complexity-cyclomatic`) switches to reporting *only* what
was explicitly requested:

```bash
gnatmetric -P <project.gpr> -U --complexity-cyclomatic --complexity-essential
```

### Global/summary numbers across all files

Metrics that are summed over the whole set of sources (total LOC, average
complexity, etc.) go to stdout by default; capture them with
`--global-file-name=<file>` instead of grepping every per-file `.metrix`.

### No project file at all

`gnatmetric` doesn't require a `.gpr` — bare source filenames are valid
arguments on their own:

```bash
gnatmetric date_utils.ads date_utils.adb days_since_1900.adb
```

`-U` only means something with `-P`; whole-program coupling is then scoped
to exactly the files listed on the command line.

## Core principles / gotchas

- **`-U` is not optional for whole-project analysis.** This is the single
  most common mistake: running `gnatmetric -P proj.gpr` alone measures only
  the project's immediate units, not its full closure, and silently produces
  incomplete coupling numbers. Always pair `-P` with `-U` unless
  deliberately scoping to `-U <main_unit>`'s closure.
- **Complexity excludes exception handlers, nested units, and
  assertions/contracts.** Cyclomatic and essential complexity are computed
  per outermost unit and *skip* the code inside exception handlers, inside
  any nested program unit (those get their own separate metrics as local
  units), and inside preconditions/postconditions/predicates/invariants.
  Don't expect a subprogram's reported complexity to include its nested
  helper subprograms' branching — check each nested unit's own report too.
- **Cyclomatic complexity is reported as three numbers**: "statement
  complexity" (control statements only), "expression complexity"
  (short-circuit `and then`/`or else` forms only), and "cyclomatic
  complexity" (their sum, the number to compare against a threshold).
- **One compilation unit per file model.** `gnatmetric` follows GNAT's
  one-unit-per-file convention; a subprogram declaration, generic
  instantiation, or renaming only gets its own metrics if it's the
  *outermost* entity in its file — nested ones don't.
- **Static loops count by default.** A loop with a statically-known
  iteration count still adds to cyclomatic complexity unless
  `--no-static-loop` is passed. Don't assume "it's just a fixed-size loop"
  explains away a high complexity number without checking this switch's
  effect.
- **When a project's whole dependency closure comes down via `-U`**,
  separate the project's own source files from any vendored dependency's
  units before reporting metrics back to the user — a dependency's internal
  coupling/complexity numbers aren't the user's code, and can dwarf the
  actual project in the raw output.
- **This tool measures, it does not verify.** A high complexity or coupling
  number is a signal to look closer (and a good candidate for the `gnattest`
  skill's stubbing/isolation approach, or splitting the unit), not itself a
  defect. Don't refactor purely to move a number without the user asking
  for that judgment call.
- **When comparing against a McCabe-style threshold**: there's no universal
  cutoff `gnatmetric` enforces — the classic McCabe guidance (complexity
  >10 warrants closer review, >20 is high risk) is a convention from the
  original McCabe methodology, not a rule `gnatmetric` itself applies. State
  it as a convention if citing it, not as something the tool asserts.

## Reference files

| Topic | File | When to read |
|-------|------|--------------|
| Full CLI switch reference | [command-reference.md](references/command-reference.md) | Looking up any gnatmetric flag, project-file `Metrics` package syntax, or output-file placement rules |
| Metric definitions & interpretation | [metrics-reference.md](references/metrics-reference.md) | Explaining what a specific reported number means, or deciding which metrics to enable for a given question |
| Building/locating gnatmetric | [building-gnatmetric.md](references/building-gnatmetric.md) | `gnatmetric` isn't found on `PATH` and needs to be built via Alire, or an `alr`/project-dependency error looks like one of the documented gotchas (toolchain not on `PATH`, an Alire dependency failing to resolve, a mid-build sudo prompt, a project with no `.gpr`) |

# gnatmetric metric definitions

## Complexity metrics

### McCabe cyclomatic complexity

Reported as three values per body:

- **Statement complexity** — complexity contributed by control statements
  (`if`, `case`, loops, exits, selects, etc.), excluding short-circuit
  forms.
- **Expression complexity** — complexity contributed only by short-circuit
  control forms (`and then`, `or else`), and, for Ada 2012, conditional
  and quantified expressions.
- **Cyclomatic complexity** — the total, statement + expression. This is
  the number to compare against a threshold if using one.

Exclusions (apply to both cyclomatic and essential complexity): code inside
exception handlers, code inside any nested program unit (nested units get
their own separate report instead), and code inside
preconditions/postconditions/predicates/invariants.

By default, a statically-bounded loop still adds to cyclomatic complexity;
`--no-static-loop` turns that off.

**Interpreting the number**: `gnatmetric` doesn't enforce a threshold. The
classic McCabe convention (not something the tool asserts) is: complexity
≤10 is generally fine, >10 warrants a look, >20 is high-risk and a good
refactor/testing-priority candidate. State this as convention, not tool
output, when relaying it to a user.

### Essential complexity

McCabe cyclomatic complexity computed on a version of the control flow
reduced by removing all *purely structural* Ada control statements. A
compound statement stops being "purely structural" (and so stays counted)
if it contains a `raise` or `return` as a subcomponent, a `goto` that jumps
outside it, or is a selective `accept` with a `terminate` alternative.
`exit` statements count as non-structural (goto-like) by default —
`--no-treat-exit-as-goto` changes that.

Essential complexity close to 1 means the code's control flow is built
almost entirely from single-entry/single-exit structured constructs
(sequence, if/then/else, structured loops) with minimal unstructured
jumping — a proxy for how mechanically decomposable the logic is,
independent of how *branchy* it is (that's what cyclomatic complexity
measures instead).

### Loop nesting

Maximum static nesting depth of loop constructs in a body.

### Extra exit points

Counts return statements and unhandled raises beyond the minimum a
subprogram needs (a function always needs at least one `return`, so that
minimum is subtracted). High values indicate many distinct places control
can leave the subprogram — relevant to how hard the subprogram is to
reason about or to instrument for coverage.

## Line metrics

Straightforward counts (total lines, code lines, comment lines, lines with
trailing end-of-line comments, comment percentage, blank lines) plus:

- **Average code lines per body** — averaged across subprogram, task, and
  entry bodies, and package-body statement sequences, for the whole set of
  sources processed (a whole-source-set metric, like
  `--complexity-average`).
- **SPARK line count** — lines written in SPARK, useful when a codebase
  mixes plain Ada and SPARK and you want to track SPARK adoption over
  time (pairs well with the `gnatprove` skill).

## Syntax element metrics

- **Declaration / statement counts** — raw counts; their sum is sometimes
  called LSLOC (logical source lines of code) as opposed to physical line
  counts.
- **Nesting levels** — max static nesting of inner program units, and max
  nesting of composite syntactic constructs (if/loop/block/etc. nested
  inside each other), reported separately from loop nesting specifically.
- **Public/all subprogram and type counts** — public types further break
  down into abstract, root-tagged, private (incl. private extensions),
  task, and protected sub-counts.
- **Parameter counts** — per-subprogram parameter counts (in/out/in-out
  breakdown where applicable); not reported for generic/formal
  subprograms.

A subprogram declaration, generic instantiation, or renaming only gets its
own syntax/complexity metrics if it is the *outermost* entity in its
source file (GNAT's one-compilation-unit-per-file model) — nested
occurrences don't get separately reported unless local-unit metrics are
enabled (the default; `--no-local-metrics` turns them off).

## Contract metrics

Counts of public subprograms that have contracts at all, that have
postconditions specifically, that have "complete" contracts, and the
McCabe complexity of public subprograms specifically (as opposed to all
subprograms) — useful for gauging how much of a public API is formally
specified, a natural companion question when using the `gnatprove` skill.

## Coupling metrics

Two independent dimensions:

- **Kind**: tagged/class coupling (relationships via tagged/interface
  types), hierarchy/category coupling (relationships across an Ada
  hierarchy of packages defining tagged/interface types), unit coupling
  (plain `with` dependencies between units), control coupling (calls
  between subprograms).
- **Direction**: fan-out ("efferent" — how many things this entity depends
  on) and fan-in ("afferent" — how many things depend on this entity).
  Control fan-in is only reported for units that define subprograms.

High fan-out suggests a unit doing too much / depending on too much
(harder to test in isolation — see the `gnattest` skill's stubbing
guidance). High fan-in suggests a unit that's a de facto interface/hub —
changes to it have wide blast radius, and it's a good candidate for
thorough test coverage.

**Coupling is only meaningful whole-program.** `gnatmetric` counts
dependencies only among the units it was actually given — always run with
`-P <project.gpr> -U` (the full closure) for coupling numbers, never a
hand-picked subset, or the numbers will be silently incomplete rather than
erroring out.

If no unit in the analyzed set defines a tagged or interface type, the
tagged/hierarchy coupling values simply don't appear in the output — that's
expected, not a tool failure.

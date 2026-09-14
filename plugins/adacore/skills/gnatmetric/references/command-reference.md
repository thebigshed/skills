# gnatmetric command-line reference

Source: AdaCore GNAT Metrics User's Guide. Quotes are paraphrased from the
official docs; treat this as a working reference and check
`gnatmetric --help` or the live AdaCore docs if a switch's exact behavior
matters for a decision.

## Invocation

```
gnatmetric [switches] {filename}
```

- `filename` arguments are Ada source files (wildcards allowed). If none
  are given, at least one `--files=<file>` switch is required.
- `gnatmetric` needs no language-version switch — it handles Ada 83 onward
  automatically.
- Project-aware via the standard `-P`/`-X`/`-U` mechanism shared with other
  GNAT tools.

| Switch | Meaning |
|---|---|
| `-P file` | Project file describing the sources to process. An aggregate project is only allowed if it aggregates exactly one non-aggregate project. |
| `-X name=value` | Set external variable `name=value` for the project. No effect without `-P`. |
| `-U` | With `-P` and no explicit source arguments (directly or via `--files`): process the full closure of the project's units. Without `-U`, only the project's *immediate* units are processed. No effect if sources are given explicitly. |
| `-U main_unit` | Process the closure rooted at `main_unit` only. |
| `--RTS=rts-path` | Select runtime library location (same meaning as `gnatmake`). |
| `--subdirs=dir` | Put tool output files in this subdir of the project's object dir (or project dir if none). No effect without a project, or with `--no-objects-dir`. |
| `--files=file` | Read the list of source files to process from `file` (one per line, blank lines ignored). Repeatable. Legacy short form: `-files filename`. |
| `--ignore=filename` | Skip the sources listed in `filename`. |
| `--verbose` / `-v` | Verbose: print version + trace of sources processed. |
| `--quiet` / `-q` | Quiet mode. |
| `--version` | Print version and exit, ignoring all else. |
| `--help` | Print usage and exit, ignoring all else. |

## Output control

| Switch | Meaning |
|---|---|
| `--generate-xml-output` / `-x` | Generate XML output. |
| `--generate-xml-schema` / `-xs` | Generate XML output plus a `.xsd` schema file (same basename, `.xml`→`.xsd`). |
| `--no-text-output` / `-nt` | Suppress per-file text output; implies `-x`. |
| `--output-dir=dir` / `-d dir` | Directory for per-file text metric files. |
| `--output-suffix=suf` / `-o suf` | Use `suf` instead of `.metrix` as the per-file output suffix. |
| `--global-file-name=file` / `-og file` | Write whole-source-set (summed/average) metrics to `file` instead of stdout. |
| `--xml-file-name=file` / `-ox file` | Write XML output to `file` (implies `--generate-xml-output`). Default `metrix.xml` in cwd if unset and no project given. |
| `--short-file-names` / `-sfn` | Use short (no-directory) source file names in output; default is absolute path. |
| `--wide-character-encoding=e` / `-We` | `e=8` for UTF-8, `e=b` (default) for Brackets encoding. |
| `--no-local-metrics` / `-nolocal` | Skip metrics for eligible local (nested) program units. |

**Per-file text output placement**: same directory as the source if no
project file; otherwise the project's object directory (or source
directory if the project defines none); `--subdirs=` and `--output-dir=`
override this.

**Project file `Metrics` package** (sets default switches, overridable on
the command line):

```ada
package Metrics is
   for Default_Switches ("Ada") use
     ("--generate-xml-output",
      "--xml-file-name", XML_File_Name,
      "--lines-all");
end Metrics;
```

## Line metrics (`--lines-*`)

| Switch | Reports |
|---|---|
| `--lines-all` | All line metrics below |
| `--lines` | Total line count |
| `--lines-code` | Code line count |
| `--lines-comment` | Comment line count |
| `--lines-eol-comment` | Code lines with end-of-line comments |
| `--lines-ratio` | Comment percentage |
| `--lines-blank` | Blank line count |
| `--lines-average` | Average code lines per subprogram/task/entry body and per package-body statement sequence |
| `--lines-spark` | Lines written in SPARK |

Each has a `--no-lines-*` counterpart to suppress it.

## Syntax element metrics (`--syntax-all` etc.)

| Switch | Reports |
|---|---|
| `--syntax-all` | All syntax metrics below |
| `--declarations` | Total declaration count |
| `--statements` | Total statement count |
| `--public-subprograms` | Public subprogram count |
| `--all-subprograms` | All-subprogram count |
| `--public-types` | Public type count (with abstract/root-tagged/private/task/protected breakdowns) |
| `--all-types` | All-type count |
| `--unit-nesting` | Max static nesting level of inner program units |
| `--construct-nesting` | Max nesting level of composite syntactic constructs |
| `--param-number` | Subprogram parameter counts |

Each has a `--no-*` counterpart.

## Contract metrics

| Switch | Reports |
|---|---|
| `--contract-all` | All contract metrics below |
| `--contract` | Public subprograms with contracts |
| `--post` | Public subprograms with postconditions |
| `--contract-complete` | Public subprograms with complete contracts |
| `--contract-cyclomatic` | McCabe complexity of public subprograms |

Each has a `--no-*` counterpart.

## Complexity metrics

| Switch | Reports |
|---|---|
| `--complexity-all` | All complexity metrics below |
| `--complexity-cyclomatic` | McCabe cyclomatic complexity (statement + expression + total) |
| `--complexity-essential` | Essential complexity |
| `--loop-nesting` | Max loop nesting level |
| `--complexity-average` | Average cyclomatic complexity across all bodies (whole-source-set only) |
| `--extra-exit-points` | Extra subprogram exit points (returns/unhandled raises beyond the minimum) |
| `--no-treat-exit-as-goto` / `-ne` | Don't treat `exit` as non-structural (goto-like) when computing essential complexity |
| `--no-static-loop` | Don't count statically-bounded loops toward cyclomatic complexity |

Each of the report switches has a `--no-*` counterpart.

## Coupling metrics

| Switch | Reports |
|---|---|
| `--coupling-all` | All coupling metrics below |
| `--tagged-coupling-out` / `--tagged-coupling-in` | Tagged (class) fan-out / fan-in |
| `--hierarchy-coupling-out` / `--hierarchy-coupling-in` | Hierarchy (category) fan-out / fan-in |
| `--unit-coupling-out` / `--unit-coupling-in` | Unit fan-out / fan-in |
| `--control-coupling-out` / `--control-coupling-in` | Control fan-out / fan-in (fan-in only for units defining subprograms) |

No documented `--no-*` counterparts for individual coupling switches (unlike the other categories).

**Coupling requires the whole program**: invoke with `-P proj.gpr -U` (or an
explicit complete file list) — dependencies are only counted among units
actually passed to the tool, so a partial set silently under-reports.

## Legacy short forms

| Short | Long |
|---|---|
| `-x` | `--generate-xml-output` |
| `-xs` | `--generate-xml-schema` |
| `-nt` | `--no-text-output` |
| `-d` | `--output-dir` |
| `-o` | `--output-suffix` |
| `-og` | `--global-file-name` |
| `-ox` | `--xml-file-name` |
| `-sfn` | `--short-file-names` |
| `-We` | `--wide-character-encoding` |
| `-nolocal` | `--no-local-metrics` |
| `-ne` | `--no-treat-exit-as-goto` |
| `-files` | `--files` |
| `-v` | `--verbose` |
| `-q` | `--quiet` |

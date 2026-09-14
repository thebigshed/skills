# Building and locating gnatmetric

`gnatmetric` is not part of a plain GNAT/GCC install (e.g. a distro's
`gnat` package is typically just the FSF/GCC Ada compiler, with no
AdaCore-specific tools). AdaCore's old GNAT Community edition used to
bundle it prebuilt, but that edition was discontinued in 2022. Today it
comes from AdaCore's own `libadalang_tools` crate on Alire
(GPL-3.0-or-later, tagged `gnatmetric`/`gnatpp`/`gnatstub`), built from
source — there is no prebuilt binary release for it, so `alr install`
(even on Alire versions new enough to have that subcommand, which only
installs prebuilt binary releases) won't work; it must be fetched and
built.

## Building it

```bash
alr get --build libadalang_tools
```

This drops source + build under a `libadalang_tools_<version>_<hash>/`
directory in the current directory, and produces `bin/gnatmetric` (plus
`bin/gnatpp`, `bin/gnatstub`, `bin/gnattest`) inside it. There is no
installer step — treat that path as the tool's location, add it to `PATH`,
or invoke it by full path.

## Gotcha: an older Alire may not wire up its own downloaded toolchain

Some Alire versions offer, on first use, a "toolchain selection assistant"
that can download and select an Alire-managed `gnat_native` compiler —
useful when the system's own `gnat` is a different major version than what
a given crate release was built/tested against, and fails to compile it
with genuine compiler errors (not just warnings).

A known failure mode on at least one older packaged Alire release
(`1.2.1`): even after a `gnat_native` version is selected and confirmed
downloaded (via `alr toolchain` and the Alire config file), `alr build` /
`alr exec` / `alr printenv` do **not** actually put that toolchain's
`bin/` directory on `PATH`. `which gnat` under `alr exec` still resolves
to the system compiler, and `gprconfig` then fails with `can't find a
native toolchain for language 'ada'`, or the build fails compiling against
the wrong compiler version.

**Workaround**: find where Alire actually deployed the toolchain compiler
(typically under Alire's dependency cache directory, look for a
`gnat_native_<version>_<hash>` folder) and prepend its `bin/` to `PATH` by
hand before calling `alr build`/`alr exec`:

```bash
find ~/.config/alire/cache/dependencies -maxdepth 1 -iname "gnat_native_*"

GNAT_BIN=<path-found-above>/bin
PATH="$GNAT_BIN:$PATH" alr -n build
```

Verify first with `PATH="$GNAT_BIN:$PATH" alr exec -- which gnat` — it
should print the path inside Alire's cache, not the system compiler,
before trusting a build to use the right one. If a newer Alire is
available, check whether this bug still reproduces before assuming the
workaround is needed.

## Gotcha: one specific dependency crate failing to resolve is not general Alire breakage

If a project's Alire dependency fails to resolve (e.g. `alr exec` reports
a crate as `missing` and the subsequent `gnatmetric`/build run fails with
`imported project file "<crate>" not found`), treat it as a per-crate
resolution gap — check that specific crate/version — rather than assuming
`alr exec` is broken as a rule. Other dependencies in the same environment
may resolve and fetch without any issue.

**Workaround for the failing crate specifically**, when `gnatmetric` is
the goal (not a real build/link): if a system package already provides the
dependency's `.gpr` (e.g. a distro package providing a GUI toolkit binding
that a project also declares as an Alire dependency), skip `alr exec`
entirely for that project and point `GPR_PROJECT_PATH` at wherever that
system package installs its project files:

```bash
GNATMETRIC=<path-to-built>/bin/gnatmetric
GPR_PROJECT_PATH="<system-gpr-directory>" "$GNATMETRIC" -P <project>.gpr -U
```

This is valid for metrics purposes even if the system package's version
doesn't exactly match what the project's `alire.toml` pins — `gnatmetric`
only parses the project graph, it doesn't check API compatibility the way
a real build/link would. If a metrics run genuinely needs the *exact*
pinned dependency version's sources, this shortcut isn't valid — fall back
to actually resolving the Alire dependency instead.

## Gotcha: `alr exec`/`alr build` can prompt to install a system package mid-run

Some Alire dependency solutions pull in a `*_system` crate (a package
manager-installed system library, commonly seen with GUI-toolkit bindings
that need native development headers). When that happens, `alr` may
prompt to install it via the platform's package manager, defaulting to
"yes" even non-interactively, and then fail if it can't authenticate:

```
The system package '<name>' is about to be installed.
This action might require admin privileges and impact your system installation.
Do you want Alire to install this system package?
Using default: Yes
sudo: A terminal is required to authenticate
ERROR: Deployment of system package from platform software manager: <name> to ... failed
```

Do not feed it credentials or otherwise push this through non-interactively
just to compute metrics — installing system packages is a real, persistent
change to the machine, not something a metrics run should trigger
silently. Use the same per-crate `GPR_PROJECT_PATH` fallback described
above instead, if a matching system package for the dependency already
exists; otherwise stop and ask the user before letting `alr` install
anything system-wide.

## Gotcha: a project with no `.gpr` file at all

Not every Ada codebase has a project file — sometimes it's just loose
`.ads`/`.adb` sources. `gnatmetric` doesn't require one; bare source
filenames are valid arguments on their own:

```bash
gnatmetric date_utils.ads date_utils.adb days_since_1900.adb test_date_utils.adb
```

No `-U` (that flag only means something with `-P`), no `GPR_PROJECT_PATH`.
Whole-program coupling metrics still work across the files given — they're
just scoped to exactly the files listed on the command line, same caveat
as always about an incomplete set under-reporting coupling.

## Summary recipe

```bash
# One-time build (in a scratch directory, not inside the target project):
alr get --build libadalang_tools   # may need the toolchain-PATH workaround above

GNATMETRIC=<wherever>/libadalang_tools_*/bin/gnatmetric
```

Per-project run — try `alr exec` first, since most Alire dependencies
resolve without any special handling:

```bash
cd <ada-project-with-.gpr>
PATH="<dir-of-gnatmetric>:$PATH" alr -n exec -- gnatmetric -P <project>.gpr -U
```

Only fall back to a system-`.gpr` `GPR_PROJECT_PATH` shortcut (skipping
`alr exec` entirely) for a project whose specific dependency fails to
resolve, and only when a matching system package actually provides the
needed `.gpr` — don't reach for it pre-emptively:

```bash
GPR_PROJECT_PATH="<system-gpr-directory>" "$GNATMETRIC" -P <project>.gpr -U
```

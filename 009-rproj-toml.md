
# Tidyup 9: `rproj.toml`, a modern project & package manifest for R

**Champion**: Gábor Csárdi<br> **Co-champion**: TBD<br> **Status**:
Draft

## Abstract

R has one native way to declare a project’s requirements: the
`DESCRIPTION` (DCF) file. This tidyup proposes a new manifest format for
this purpose, `rproj.toml`, designed as the single authoritative source
of truth for both R packages and projects. It also specifies how tools
read it and how they generate a valid `DESCRIPTION` from it as a build
artifact. Any tool can implement it. `rig` is one implementation and
serves as the example in this document. `rproj.toml`’s main goals are:

1.  Suitable for packages and projects.
2.  Everything `DESCRIPTION` can express.
3.  A flexible version-constraint dialect.
4.  Optional dependencies on two axes: published extras and local dev
    groups.
5.  Cargo-style workspaces.
6.  Declared binaries/scripts.
7.  A `[tool.*]` namespace for other tools’ configuration.

## Motivation

`DESCRIPTION` was designed around the needs of CRAN package submission.
Modern R project support needs capabilities that are hard to support in
`DESCRIPTION`.

1.  **Flexible version constraints.** Only simple `>=`/`<=`/`==`
    comparisons against a single package are possible. There is no
    caret/tilde shorthand.

2.  **Dependency classes that fit project needs better.**
    `Imports`/`Depends`/`Suggests`/`Enhances` were designed for CRAN
    policy checks, not for expressing “this is a dev-only tool” or “this
    is a dependency for an optional feature a downstream user can opt
    into.”

3.  **Package source/repository declaration.** You cannot say “install
    this dependency from this specific repository” or “this dependency
    comes from this git branch” without using non-standard fields.

4.  **Workspaces.** Multi-package repositories (monorepos) where sibling
    packages can reference each other and share dependency versions.

5.  **Runnable binaries and scripts.** There’s no declared, portable way
    to say “this project has a script called `fetch-data` that a project
    tool can run by name” (e.g. with `rig run fetch-data`).

## Solution

We propose `rproj.toml` as a new schema, borrowing ideas from cargo, uv,
and the prior `rproj.toml` draft, but not bound to any one of them.

Key design decisions:

1.  **Authoritative manifest.** `rproj.toml` is what a user edits.
    `DESCRIPTION` is generated from it as needed and should not be
    hand-edited once a project has an `rproj.toml`.

2.  **Filename `rproj.toml`.** This matches the filename already
    proposed for this purpose. The schema itself is new.

3.  **Two-axis optional dependencies.** `[optional-dependencies]` covers
    extra functionality a downstream consumer opts in (e.g. with
    `mypkg[extra]`). `[dependency-groups]` covers *local*, unpublished
    groups (test, doc, website, dev). Both use the same dependency-entry
    syntax as `[dependencies]`.

4.  **Binaries and named entry-point scripts.** A `[[bin]]` table
    declares a name, a script path, and an optional description. Tools
    may provide functionality to execute and/or install scripts. E.g.
    `rig run <name>` executes a script. How a script is run (with
    `Rscript` or otherwise, from which working directory, with which
    library) is up to the tool.

5.  **Bare version strings mean caret ranges.** `dplyr = "1.2.3"` is
    equivalent to `^1.2.3`, meaning `>= 1.2.3, < 2.0.0`, including
    zero-prefix nuances (`^0.2.3` means `>= 0.2.3, < 0.3.0`, `^0.0.3`
    means `>= 0.0.3, < 0.0.4`). An exact pin uses `=1.2.3`. Other ranges
    use `~`, explicit `>=`/`<`, or comma-separated AND.

6.  **`[config.<name>]` and `[tool.<name>]` for other tools’ config.**
    `[config.<name>]` (see the full example below) is for settings that
    end up in `DESCRIPTION` as `Config/<Name>/<key>`, e.g. `roxygen2` or
    `testthat` options that these packages read.

    `[tool.<name>]` is a separate, unstructured namespace: any section
    under `[tool.*]` is free-form config owned by one tool. A tool reads
    and writes only its own `[tool.<name>]` section and leaves the other
    sections alone. E.g. `rig` only uses `[tool.rig]`. This lets one
    file replace several tool-specific TOML files in a project.

    `<name>` must be a name the tool owns, to avoid collisions: a CRAN
    package name (e.g. `[tool.lintr]`) if the tool is a CRAN package, or
    a reverse-domain identifier the tool controls (e.g.
    `[tool."dev.posit.air"]`).

7.  **Round-trip fidelity.** Tools that write `rproj.toml`
    (e.g. `rig proj add`) must preserve tables, keys, and comments they
    don’t own, including `[tool.*]` sections and anything else
    unrecognized. A tool edits only the tables it’s responsible for and
    leaves the rest of the file, including formatting and comments,
    untouched.

8.  **Unknown keys when reading.** A tool that reads `rproj.toml`
    ignores `[tool.*]` sections it does not own. For unknown keys
    elsewhere in the file it should warn rather than fail, so that files
    written for a newer version of this specification still work with
    older tools.

### A full example

``` toml
[project]
type = "package"                          # project (not installed), or package
name = "mypkg"
version = "0.1.0"
title = "A Modern Thing"                  # → Title
description = "One to three sentences."   # → Description
license = "MIT + file LICENSE"            # SPDX-ish, passed through
keywords = ["cli", "tools"]

authors = [
  { name = "Gábor Csárdi", email = "gabor@posit.co", roles = ["aut", "cre"],
    orcid = "0000-0001-7098-9676" },
  { name = "Posit Software, PBC", roles = ["cph", "fnd"], ror = "03wc8by49" },
]

[project.urls]                            # → URL (joined) + BugReports
homepage   = "https://mypkg.example.org"
source     = "https://github.com/me/mypkg"
bugreports = "https://github.com/me/mypkg/issues"

[dependencies]                            # → Imports (the 90% case)
R        = ">= 4.5"                       # R itself is a dep (→ Depends: R (>= 4.5))
cli      = ">= 3.6.5"
dplyr    = "1.1.0"                        # bare = caret → >= 1.1.0, < 2.0.0
rlang    = "~1.1.0"                       # >= 1.1.0, < 1.2.0
tidyr    = "=1.3.0"                       # exact
readr    = "*"                            # any version
methods  = { attach = true }              # → Depends (force attach)
special  = { version = ">= 1.0", repository = "posit" }   # pin repo
limma    = { version = "*", repository = "bioc" }         # Bioconductor only
ts       = { git = "https://github.com/gaborcsardi/ts", rev = "main" }
urlpkg   = { url = "https://ex.com/urlpkg.tar.gz" }
localpkg = { path = "../localpkg" }

[linking-dependencies]                    # → LinkingTo (compile-time)
Rcpp = ">= 1.0"

# Optional dependencies (consumer opts in) → Config/Needs/Optional/<name> + Suggests
[optional-dependencies.viz]
ggplot2 = "*"
plotly  = ">= 4.10"

[optional-dependencies.db]
DBI       = "*"
RPostgres = "*"

# Local deps, dev → Suggests, others → Config/Needs/<name>.
[dependency-groups.dev]
testthat  = ">= 3.0"
knitr     = { vignette-builder = true }   # → Suggests, VignetteBuilder
rmarkdown = "*"

[dependency-groups.website]               # → Config/Needs/website
pkgdown  = "*"
asciicast = "*"

[dependency-groups.tidy]                  # → Config/Needs/tidy
include-groups = ["dev", "website"]       # reserved key: pull in other groups
devtools = "*"
lintr    = "*"

[[repository]]
name = "posit"
url  = "https://packagemanager.posit.co/cran/latest"

[[repository]]
name = "CRAN"
url  = "https://cran.r-project.org"

[[repository]]
name    = "bioc"                          # built-in Bioconductor entry, no url
version = "3.23"                          # pin the release, or `enabled = false`

[build]
byte-compile       = true                 # ByteCompile
needs-compilation  = true                 # NeedsCompilation
lazy-data          = true                 # LazyData
biarch             = true                 # Biarch

[[bin]]
name = "fetch-data"
path = "scripts/fetch.R"
description = "Download the raw data"

[[bin]]
name = "report"
path = "scripts/report.R"

[config.roxygen2]                         # → Config/roxygen2/*
markdown = true
r6 = false
version = "7.3.3"

[config.testthat]                         # → Config/testthat/*
edition = 3
parallel = true

[description]                             # verbatim passthrough
License_is_FOSS = "yes"                   # → License_is_FOSS: yes

[tool.rig]                                # rig's own settings
exclude-newer = "2026-06-01"              # ignore package versions published later
prefer-binary = true                      # prefer older binaries over newer sources

[tool."dev.posit.air"]                    # free-form, owned by air, other tools leave it alone
line-length = 88

[tool."dev.posit.jarl"]
enable = ["lints:all"]

[tool.lintr]                              # lintr is a CRAN package, so it owns this name
linters = "linters_with_defaults()"

[workspace]
members = ["packages/*", "apps/dashboard"]
exclude = ["packages/experimental"]

[workspace.dependencies]                  # shared versions inherited by members
cli = ">= 3.6.5"
```

Member manifests inherit shared versions with `{ workspace = true }`:

``` toml
[dependencies]
cli     = { workspace = true }            # version from [workspace.dependencies]
sibling = { path = "../sibling" }         # or resolves to a workspace member
```

### Version-constraint dialect

| Manifest string               | Meaning                                 |
|-------------------------------|-----------------------------------------|
| `"1.2.3"` (bare) ≡ `"^1.2.3"` | caret / compatible: `>= 1.2.3, < 2.0.0` |
| `"^0.2.3"`                    | `>= 0.2.3, < 0.3.0` (zero-nuance)       |
| `"^0.0.3"`                    | `>= 0.0.3, < 0.0.4` (zero-nuance)       |
| `"=1.2.3"`                    | exact                                   |
| `"~1.2.3"`                    | `>= 1.2.3, < 1.3.0` (tilde / patch)     |
| `">= 1.2"`, `"< 2.0"`         | single bound                            |
| `">= 1.0, < 2.0"`             | comma = AND (multiple constraints)      |
| `"*"` or `""`                 | any version                             |

### Dependency-class mapping (`rproj.toml` → `DESCRIPTION`)

| `rproj.toml` | `DESCRIPTION` field |
|----|----|
| `[project].type` | `Type:` (default `project` = not installed, `package`) |
| `[dependencies]` | `Imports` |
| dep entry `attach = true` | `Depends` |
| dep entry `vignette-builder = true` | names it in `VignetteBuilder:` (dep itself still emitted) |
| `[dependencies].R` | `Depends: R (>= x)` |
| `[linking-dependencies]` | `LinkingTo` (may also be in `[dependencies]` → merged types) |
| `[optional-dependencies.<extra>]` | `Config/Needs/Optional/<extra>` + `Suggests` |
| `[dependency-groups.dev]` | `Suggests` (R CMD check reads them) |
| `[dependency-groups.<other>]` | `Config/Needs/<other>` |
| `[dependencies]` entry `enhances = true` | `Enhances` |
| inline `{ git / url / path }` source | dep line **+** `Remotes:` entry |
| `[config.<name>]` | `Config/<name>/<key>` |
| `[tool.<name>]` | (none; free-form config, each tool reads only its own section) |
| `[[repository]]` | (none; used for dependency resolution only) |
| `[description]` | verbatim fields |
| `authors = [...]` | `Authors@R` (generated `person()` vector, incl. ORCID/ROR comments) |

### Repositories

1.  `[[repository]]` entries are listed in order of precedence, the
    first entry has the highest.
2.  Every entry needs a `name` and a `url`, except the built-in
    Bioconductor entry, see below.
3.  If there are no `[[repository]]` entries, the tool picks the default
    repositories, e.g. a CRAN mirror.
4.  A dependency entry with `repository = "<name>"` must be installed
    from the repository called `<name>`, and from no other repository.

The name `bioc` is reserved for Bioconductor. Bioconductor is enabled by
default, and the Bioconductor release is the one that belongs to the R
version in use. A `[[repository]]` entry with `name = "bioc"` has no
`url`, and it may have:

- `version`, to pin a Bioconductor release, e.g. `version = "3.23"`, or
- `enabled = false`, to turn off Bioconductor.

At most one `bioc` entry is allowed. Other entries cannot have `version`
or `enabled`.

### Dependency groups

`include-groups` is a reserved key in `[dependency-groups.<name>]`; it
cannot clash with a package name, because R package names cannot contain
`-`. It lists other groups whose dependencies are also part of this
group. Every listed group must exist, and the includes must not form a
cycle; tools report both as errors.

## Implementations

[`rig`](https://github.com/r-lib/rig) is the reference implementation.
Support for most of `rproj.toml` is in its development version (many
features are already in the released 0.10.0 version):

1.  `rig proj init` writes a minimal `rproj.toml` skeleton.
2.  `rig proj import` creates `rproj.toml` from a `DESCRIPTION` file.
3.  `rig proj deps`/`tree`/`lock`/`sync` read and interpret
    `rproj.toml`, including version constraints.
4.  `rig run` support for `[[bin]]` entries.
5.  Workspace support (`[workspace]`, member resolution, shared
    dependency versions).
6.  Bioconductor packages, including the `bioc` `[[repository]]` entry
    and `repository = "bioc"` dependency pins.
7.  `[tool.rig]` settings: `exclude-newer` (ignore package versions
    published after a date) and `prefer-binary` (prefer older binaries
    over newer sources).
8.  Self-contained scripts, with `rig run script.R`.

Currently the following features are missing from rig:

1.  Custom repositories for the project. Only PPM’s CRAN and
    Bioconductor repositories are supported currently.
2.  Per dependency repository pins are not supported, except for
    `repository = "bioc"`.
3.  R projects (e.g. repositories with `rproj.toml`) are not supported
    as dependencies.

Other tools, e.g. package installers or development tools, can implement
all of this specification, or only part of it, e.g. only scaffolding
`rproj.toml` or `rproj.toml` and a virtual environment (see Tidyup 10),
or generating `DESCRIPTION` from `rproj.toml`.

## Backwards compatibility

`rproj.toml` is additive: a package or project without one continues to
work exactly as it does today, driven by `DESCRIPTION` alone. Adding an
`rproj.toml` to an existing package does not require removing
`DESCRIPTION`. There are no breaking changes proposed to `DESCRIPTION`’s
own semantics, and no existing package or project is required to adopt
`rproj.toml`.

## How to teach

`rproj.toml` would be taught as the modern entry point for new projects
and packages, analogous to how `usethis::create_package()` currently
scaffolds a `DESCRIPTION`. Tools would provide the equivalent
scaffolding command for `rproj.toml`, e.g. `rig proj init`. Existing
users would not need to learn anything new unless they choose to adopt
it. Documentation may frame it as an alternative, more expressive way to
declare dependencies, not a required migration.

## Open issues

None known at this point.

## Unresolved questions

1.  **The name of `[tool.rig]`.** `rig` is not a CRAN package, and `rig`
    is not a reverse-domain identifier, so `[tool.rig]` does not follow
    the naming rule for `[tool.<name>]`. Options: rename it (e.g.
    `[tool."io.github.r-lib.rig"]`), allow an exception for existing
    tools, or loosen the rule.

2.  **`rproj.toml` and `DESCRIPTION` out of sync.** A package may have
    both files. `rproj.toml` is authoritative, but it is not decided
    what a tool should do if `DESCRIPTION` differs from the one
    generated from `rproj.toml`: regenerate it, warn, or fail.

3.  **Version constraints for R version numbers.** R package versions
    can have two or more components, separated by `.` or `-`,
    e.g. `1.2`, `1.2-3`, `1.2.3.9000`. The caret and tilde rules above
    are defined for three components only. The rules for other versions
    need to be specified, so that all tools resolve the same constraint
    the same way.

## Alternatives

### Extending `DESCRIPTION` itself instead of a new manifest

An alternative to a new TOML file would be to extend `DESCRIPTION`’s own
DCF format with new fields (dependency groups, richer version
constraints, workspace members). We rejected this because CRAN policy
constrains what fields can appear in a submitted package’s
`DESCRIPTION`.

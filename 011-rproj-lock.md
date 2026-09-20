
# Tidyup 11: `rproj.lock`, a lock file for R projects

**Champion**: Gábor Csárdi

**Co-champion**: TBD

**Status**: Draft

## Abstract

This tidyup proposes `rproj.lock`, a TOML lock file that resolves an
`rproj.toml` manifest to a concrete, fully reproducible sets of package
versions and builds for multiple `(R version, platform)` targets, and
specifies how `rig` writes and consumes it (`rig proj lock`/`sync`).

## Motivation

- **No canonical resolved-state format.**

  `DESCRIPTION` and `rproj.toml` express constraints, not a solve.
  Reproducing an install today means re-running dependency resolution
  against whatever CRAN snapshot happens to be live, which drifts.

- **`renv.lock` doesn’t record enough to be safe.**

  It lists package name, version, and a repository, but not which build
  (source vs binary) was chosen, nor the exact download URL. Two
  machines reading the same `renv.lock` can legitimately fetch different
  artifacts.

- **No R-version pinning tied to the solve.**

  A lock file needs to record the R version its solve is valid for,
  since binary packages and `LinkingTo` compatibility are tied to a
  specific R ABI. Nothing today ties a set of resolved package versions
  to the R version they were solved against.

- **No cache-addressable build identity.**

  Binary artifacts differ by platform, R version, and compiler ABI.
  Without recording the target filename and hash, a second solve on a
  different machine cannot verify it is installing byte-identical
  packages.

## Solution

We propose `rproj.lock`, a TOML file written by `rig proj lock` next to
`rproj.toml`, and read by `rig proj sync` to install a project’s
`.rvenv` (see [Tidyup 10](010-r-virtual-environments.md)). It is tracked
in version control, the same role `Cargo.lock`/`uv.lock` play.

Key design decisions:

- **One file, an array of targets.**

  `[[targets]]` is keyed by `r_version` + `platform`. By default
  `rig proj lock` solves for this machine plus three other common
  platforms (macOS arm64, Windows x86_64, GNU Linux x86_64).

- **A recorded fingerprint of what each target was solved against.**

  `[[targets.direct_dependencies]]` records one entry per manifest
  direct dependency this target was solved against (`name` and
  `constraint`, the latter formatted the same way `rproj.toml` itself
  writes a version requirement), filled in after the solve. It names
  only the manifest’s own roots, not the full transitive `packages`
  list, so it changes only when `rproj.toml`’s own dependencies change,
  not when a transitive dependency’s version moves.

- **Every resolved package carries what installing it needs, nothing
  more.**

  Each `[[targets.packages]]` entry records the resolved version,
  whether the build is source or binary, its own dependency list (for
  offline re-verification without re-solving), the exact source URL(s),
  the on-disk cache target path, repository metadata needed to detect
  drift without querying a repository, and the dependency group(s)
  (`"main"` for a hard dependency, plus the name of any
  `[dependency-groups.*]` table, e.g. `"test"`) that need the package.
  `groups` is what lets `rig proj sync --no-dev` decide what to install
  without re-solving: it filters `[[targets.packages]]` down to entries
  whose `groups` contains `"main"`.

  Base packages (`utils`, `methods`, …) and R itself are not resolved as
  packages: a manifest dependency on one of them does not produce a
  `[[targets.packages]]` entry. It is instead satisfied implicitly by
  the target’s `r_version`.

- **Binary vs source is a first-class, recorded decision.**

  `binary = true`/`false`, `sources` (the URL(s) considered), and
  `target` (the resolved cache path,
  `bin/<platform>/<Rx.y>/<pkg>_<version>-<hash>.tgz` for a binary, an
  analogous `src/...` path otherwise) make the solver’s binary-vs-source
  choice ([Tidyup 9](009-rproj-toml.md)’s “Source and binary packages”)
  reproducible, not just its outcome.

- **Metadata carries repository-specific provenance.**

  `[targets.packages.metadata]` is an open table for fields a repository
  publishes that don’t fit the fixed schema, e.g. `RemoteHash`.
  `rig proj sync` writes each entry into the installed package’s
  `DESCRIPTION`, mirroring how `remotes`/`pak` record `Remotes`-adjacent
  provenance fields today, rather than inventing a new format.

### A full example

Two packages of a larger `rproj.lock`, showing a transitive binary
dependency with its own dependency and a package’s repository metadata:

``` toml
version = 4

[[targets]]
r_version = "4.1"
platform = "macos-arm64"

[[targets.direct_dependencies]]
name = "openssl"
constraint = ">= 2.0.0"

[[targets.packages]]
package = "openssl"
version = "2.4.0"
binary = true
platform = "macos-arm64"
dependencies = ["askpass"]
sources = ["https://p3m.dev/cran/2026-04-17/bin/macosx/big-sur-arm64/contrib/4.1/openssl_2.4.0.tgz"]
target = "bin/macosx/big-sur-arm64/4.1/openssl_2.4.0-fd4daad0.tgz"
metadata = { RemoteHash = "75ccbde2e52fc3f13c146eab3d30aa3d3a80ab66fa203b9b40d09b1f2a0fcd85" }
groups = ["main"]

[[targets.packages]]
package = "askpass"
version = "1.2.1"
binary = true
platform = "macos-arm64"
dependencies = ["sys"]
sources = ["https://p3m.dev/cran/2024-10-05/bin/macosx/big-sur-arm64/contrib/4.1/askpass_1.2.1.tgz"]
target = "bin/macosx/big-sur-arm64/4.1/askpass_1.2.1-8ea2713a.tgz"
metadata = { RemoteHash = "6c2106a74c44a748f2cea795d9686e27a0058a90debcfd8558b62b06aec0c7dd" }
groups = ["main"]
```

### Field reference (`[[targets]]`)

| Field | Meaning |
|----|----|
| `r_version` | The R version this target was solved for. |
| `platform` | The target’s platform, e.g. `"macos-arm64"`. |
| `direct_dependencies` | Fingerprint of the manifest’s direct dependencies this target was solved against (name + version constraint), used to check whether the target still satisfies `rproj.toml` without re-solving. |
| `packages` | The solved packages, one entry per installable dependency (direct or transitive). |

### Field reference (`[[targets.direct_dependencies]]`)

| Field | Meaning |
|----|----|
| `name` | A manifest direct dependency’s name. |
| `constraint` | Its version requirement, formatted the same way as in `rproj.toml`, e.g. `">= 2.0.0"` or `"*"`. |

### Field reference (`[[targets.packages]]`)

| Field | Meaning |
|----|----|
| `package` | The resolved package name. |
| `version` | The resolved version. |
| `binary` | Whether `target` is a binary build (`true`) or a source tarball (`false`). |
| `platform` | The build’s platform (mirrors the enclosing `[[targets]]`, since a binary is platform-specific). |
| `dependencies` | The package’s own runtime dependency names, recorded so the graph can be re-verified without re-solving. |
| `sources` | The URL(s) the package was resolved from. |
| `target` | The cache-relative path the artifact is (or will be) stored at, unique per build. |
| `metadata` | Open table of repository-specific provenance fields, e.g. `RemoteHash`. `rig proj sync` writes these into the installed package’s `DESCRIPTION`. |
| `groups` | Dependency group(s) that need this package: `"main"` for a hard dependency, plus every `[dependency-groups.*]` name that (transitively) needs it. |

## Implementation

Support for `rproj.lock` is implemented in
[`rig`](https://github.com/r-lib/rig), version 0.10.0 (currently beta).

1.  `rig proj lock` reads `rproj.toml`, solves against the configured
    repositories without running R, and writes `rproj.lock`.
2.  `--r-version`, `--no-dev`, `--platform`, `--prefer-binary`, and
    `--no-cache` control the solve (R version, dev dependencies, target
    platform, binary/version trade-off, and cache freshness,
    respectively). `--renv` additionally writes an `renv.lock`.
3.  `rig proj sync` reads `rproj.lock` and installs the matching target
    into a project’s `.rvenv/lib` ([Tidyup
    10](010-r-virtual-environments.md)), installing the locked R version
    first if it is missing.
4.  `rig proj tree` reads `rproj.lock` to print the resolved dependency
    graph, walking `dependencies` from `rproj.toml`’s direct
    dependencies down.

## Backwards compatibility

`rproj.lock` is new and additive: a project without one is solved and
installed on demand by `rig proj sync`, which runs `rig proj lock` first
if the file is missing. Nothing about `DESCRIPTION`, `renv.lock`, or
existing `rig` behavior changes. `--renv` lets a project keep publishing
an `renv.lock` alongside `rproj.lock` during a transition.

## How to teach

`rproj.lock` would be taught as the `Cargo.lock`/`uv.lock` analog:
commit it, don’t hand-edit it, and re-run `rig proj lock` to update it
after changing `rproj.toml`. `rig proj sync` should be taught as the one
command that makes a checkout match its lock file, the same role
`cargo build`/`uv sync` play for their ecosystems.

## Open issues

None known at this point.

## Unresolved questions

None known at this point.

## Alternatives

### Extending `renv.lock` instead of a new format

An alternative would be to add the missing fields (binary vs source,
cache target path) to `renv.lock` itself. We rejected this because
`renv.lock`’s schema is renv’s own and not owned by this proposal, and
because `renv.lock` has no notion of a manifest to resolve against in
the first place (Tidyup 9’s rejection of `DESCRIPTION`-as-manifest
applies here too). `rproj.lock` complements `renv.lock` instead:
`rig proj lock --renv` can still emit one for tools that only understand
renv’s format.

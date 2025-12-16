
# Tidyup 9: R project requirements

**Champion**: Thomas

**Co-Champion**: Hadley

**Status**: Draft

## Abstract

Historically, R has had a single way of describing the requirements of a
project: the `DESCRIPTION` file. This is a text file in the dcf format
which is used to describe metadata for an R package, including any
dependencies the package may have. While this file format could also be
used to describe non-package R project, it has a number of deficiencies
that makes it reasonable to consider a new format altogether. These
includes:

- Insufficient syntax for dependency version requirements
- Depends/Imports/Suggests/Enhances does not align well with the
  dependency levels needed for projects in general
- No way to specify the source of the package, fallbacks if not
  accessible, etc

It would be possible to add to the syntax and spec of the DESCRIPTION
file but we would then end up with two divergent specifications with
resulting confusion.

In parallel with this, there have been efforts to create ways of
capturing the exact environment of a project for the sake of
reproduction. The current state of the art for this is renv, which
encodes the state in an renv.lock file. However, a lock file is
orthogonal to describing requirements as it fixes all dependencies to a
specific version making graceful upgrades of dependencies difficult.

This document intends to sketch out a new file specification for
describing *requirements* of R projects. The goal is to support:

- Rich version constraints of dependencies
- Global and per-package repositories for dependencies
- Meaningful levels of dependencies, including:
  - Testing
  - Development
  - Platform
- R and system requirements
- General project metadata

Further, from the spec it should be possible to derive renv lockfile and
a DESCRIPTION file (but not the other way since our spec will be
richer). The spec should be able to exist in a separate file as well as
inlined in a script file as a header.

Initial support for the spec should include:

- pak, for deriving and installing dependencies according to the
  requirements
- rig, for setting up the correct R version, as well as installing
  dependencies. Should probably also be able to derive an renv.lock file
- Connect and Connect Cloud should accept this as an alternative to
  manifest.json

## Prior art

Project requirements are not a problem unique to R and many programming
languages have different ways of solving it. We want to highlight just a
few:

### pyproject.toml in uv (Python)

TBD

### cargo.toml

TBD

### rproject.toml in rv (R)

TBD

## Proposal

Below is an example `rproj.toml` file showcasing the various parts of
the proposed specification

``` toml
# Project metadata
[project]
name = "example-project"
version = "0.1.0"
description = "An example R project"
authors = [
  {name = "Jane Doe", email = "jane@example.com", role = "creator"},
  {name = "John Smith", email = "john@example.com", role = "contributor"}
]
license = "MIT"
readme = "README.md"
homepage = "https://github.com/user/example-project"
repository = "https://github.com/user/example-project"
keywords = ["data-science", "machine-learning"]

# R version requirements
[r]
version = ">= 4.1.0, < 4.5.0"
options.warn_level = 1
options.defaultPackages = ["utils", "stats", "datasets", "graphics", "grDevices", "methods"]

# Repository configuration
[[repositories]]
name = "CRAN"
url = "https://cran.r-project.org"
priority = 100

[[repositories]]
name = "Posit"
url = "https://packagemanager.posit.co/cran/latest"
priority = 50
auth.type = "token"
auth.env_var = "POSIT_TOKEN"

[[repositories]]
name = "BioConductor"
url = "https://bioconductor.org/packages/release/bioc"
priority = 25

# Core dependencies (required for the project)
[dependencies]
dplyr = "^1.1.0"        # Compatible with 1.1.0 up to but not including 2.0.0
ggplot2 = "~3.4.0"      # Patch updates only (3.4.x)
readr = ">= 2.1.0"      # 2.1.0 or higher
tidyr = "1.3.0"         # Exact version
shiny = { version = "1.7.4", repository = "CRAN" }

# Package from GitHub
github_pkg = { git = "https://github.com/user/github_pkg", tag = "v1.0.0" }

# Local package
local_pkg = { path = "../local_pkg" }

# Package from URL
url_pkg = { url = "https://example.com/url_pkg.tar.gz", hash = "sha256:a1b2c3..." }

# Development dependencies (not needed for production)
[additional-dependencies.dev]
testthat = "^3.1.0"
roxygen2 = "^7.2.0"
devtools = "^2.4.0"
lintr = "^3.0.0"

# Testing dependencies
[additional-dependencies.test]
mockery = "^0.4.3"
covr = "^3.6.1"

# Documentation dependencies
[additional-dependencies.doc]
pkgdown = "^2.0.0"
knitr = "^1.40"

# Platform-specific dependencies
[additional-dependencies.platform.windows]
winpackage = "^1.0.0"

[additional-dependencies.platform.macos]
macpackage = "^1.0.0"

[additional-dependencies.platform.linux]
linuxpackage = "^1.0.0"

# System requirements (non-R dependencies)
[system-dependencies]
python = ">= 3.8.0"
node = ">= 16.0.0"
pandoc = ">= 2.18"
gdal = { version = ">= 3.0.0", optional = true }

# Custom project configuration
[config]
library_path = "renv/library"  # Custom library path
data_dir = "data"  # Data directory
results_dir = "results"  # Results directory
auto_install = true  # Auto-install missing packages
renv_integration = true  # Enable renv integration
```

### The project table

This table contains metadata about the specific project. None of these
fields are relevant to how the project should be executed or how the
environment it should be executed in looks like. The specific elements
of the table are as follows:

| Element | Description |
|----|----|
| name | The name of the project as a single string with no spaces |
| version | The current version of the project |
| description | 1-3 sentences describing the project |
| authors | An array of inline tables giving name, email, and role of the authors of the project |
| license | The license this project is distributed under |
| readme | A pointer to the file that serves as this projects readme document |
| homepage | The url of this projects homepage |
| repository | The location of the repository for this project |
| keywords | An array of keywords describing the project |

### The r table

This table describe the R environment that the project should be
executed in. Specifically the version constraints on the R runtime, but
also additional settings in R

| Element | Description |
|----|----|
| version | The version of R to use for executing the project. See the version syntax in the description of the \[dependencies\] table |
| options | A subtable holding any R `options()` that should be applied to the session before executing the project |
| env | A subtable holding environment variables that should be set in the R session prior to executing the project |

### The repositories table array

repositories is an array of repository configurations which define the
various locations the dependencies can come from. Each entry into the
array must be conform to the following spec

| Element | Description |
|----|----|
| name | The name of the repository. Will be used when referencing the repository in dependencies |
| url | The url pointing to the repository. The repository must conform to CRAN guidelines for R repositories |
| priority | An integer. Repositories will be tried in descending order so the repository with the highest priority will come first |
| auth | A subtable giving optional authentication settings for the repository. The subtable must include a `type` element defining how authentication must be done, along with any additional elements relevant to the type |

### The dependencies table

This table gives hard R package dependencies for the project which will
be required for any mode of execution (see [The additional-dependencies
table](#the-additional-dependencies-table)). Dependencies are given as
named entries to the table with the name giving the name of the
dependency. The value can be either a single string giving the version
constraint of the dependency or a subtable with more specific
information of the nature of the dependency

#### Version Specification Reference

| Format | Meaning | Example |
|----|----|----|
| `"x.y.z"` or `"^x.y.z"` | Compatible updates | `"1.2.3"`/`"^1.2.3"` → `>= 1.2.3, < 2.0.0` |
| `"=x.y.z"` | Exact version | `"=1.2.3"` |
| `">= x.y.z"` | Greater than or equal | `">=1.2.3"` |
| `"< x.y.z"` | Less than | `"<2.0.0"` |
| `"~x.y.z"` | Patch updates only | `"~1.2.0"` → `>=1.2.0, <1.3.0` |
| `"range1, range2"` | Multiple constraints | `">= 1.0.0, < 2.0.0"` |

#### Subtable syntax

When providing the dependency as a subtable it can follow one of several
schemas:

##### Version and repository

If you want to specify the exact repository to retrieve the dependency
from use a subtable with the following schema

| Element | Description |
|----|----|
| version | The version requirement of the package as described above |
| repository | The name of the repository to retrieve it from, referencing a repository given in the repositories array |

##### Git package

If you want to use a dependency from a git repository use a subtable
with the following schema

| Element | Description |
|----|----|
| git | A url pointing to the package in the given repository. The terminal `.git` part of the url is optional |
| branch | Which branch to get. If omitted the default branch will be used (unless rev or tag is set) |
| tag | A specific git tag to use |
| rev | Any identifier not branch or tag. Often a commit hash, but some repositories also allow you to reference pull requests etc. |

##### Package from url

If the dependency is located outside of git but accessible as a build
tarball online you can use the following subtable

| Element | Description |
|----|----|
| url | The url pointing to the package |
| hash | A checksum for the package given as a combination of algorithm and hash, separated by `:`. |

##### Local package

A local package can be stated as a dependency with this subtable

| Element | Description |
|----|----|
| path | A path pointing to either the source directory of the dependency or a build tarball. The path can either be absolute or relative to the rproj.toml file |

### The additional-dependencies table

The dependencies table gives strong dependencies for the project under
any mode of use. However, there may be dependencies that are only
relevant for certain scopes of use. These can be given as subtables to
the additional-dependencies table. The schema of the subtables matches
that of the dependencies table. While you are free to define new types
of additional dependencies, a set of predefined types are provided and
should be used if they match the need:

- `additiona-dependencies.test` provides dependencies only relevant
  during testing of the package

- `additiona-dependencies.dev` provides dependencies only relevant
  during development of the package

- `additiona-dependencies.doc` provides dependencies only relevant when
  rendering the documentation of the package

- `additiona-dependencies.platform.<OS>` provides dependencies only
  relevant when running on the given OS

### The system-dependencies table

The dependencies and additional-dependencies table only concern
themselves with R package dependencies. There may be system library
requirements as well which can be stated under the system-dependencies
table. This table contain named entries with the name giving the system
library and the value giving either the version constraint or a subtable
with additional information about the dependency. The version string is
subject to the same syntax as given [Version Specification
Reference](#version-specification-reference). If providing a subtable it
must follow the given schema

| Element | Description |
|----|----|
| version | The version requirement of the system library as described above |
| optional | A boolean indicating if the requirement is hard (defaults to true) |
| libname | A subtable giving the exact name of the library for specific distributions if it cannot be resolved automatically. |
| needed-for | An array of strings referencing the subtables in the additional-dependencies table. If provided it will only be required in those modes. If missing the library will be installed for all modes. |

### The config table

This table allows you to set specific runtime settings for your project.
Schema TBD

## renv.lock Integration

Rather than defining a new lockfile format, this proposal leverages the
existing `renv.lock` format. The current `renv.lock` format is already
well-established in the R ecosystem and provides:

- Exact versions of all direct and transitive dependencies
- Source information (repository URLs)
- Package hashes for verification
- Platform awareness

Tools implementing this standard would:

1.  Read dependency intent from `rproj.toml`

2.  Resolve dependencies according to specified constraints

3.  Write exact resolution results to `renv.lock`

4.  Use `renv.lock` for reproducible installations

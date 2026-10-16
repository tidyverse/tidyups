
# Tidyup 12: Extended metadata for R package repositories

**Champion**: Gábor Csárdi<br> **Co-champion**: TBD<br> **Status**:
Draft

## Abstract

This tidyup specifies a set of files that an R package repository can
serve, in addition to the usual `PACKAGES` files, to describe its full
history: every source package version it ever published, the packages it
removed, and every binary build of every package version. Tools that
lock or reproduce R projects need this information, and today they can
only get it by scraping several services. The format is plain DCF and
TSV, compressed with zstd, so it is easy to produce and easy to read.

Posit Package Manager (P3M) data for CRAN and Bioconductor is published
in this format at `https://ppm.r-pkg.org` and
`https://ppm-bioc.r-pkg.org`. `rig` is the reference client, and uses
these files to solve and lock `rproj.toml` projects, see [Tidyup
9](009-rproj-toml.md) and [Tidyup 11](011-rproj-lock.md).

## Motivation

- **`PACKAGES` only describes the present.**

  A CRAN-like repository’s `PACKAGES` file lists the current version of
  each package. Older versions live in `src/contrib/Archive/`, which has
  no index of its own, so a resolver cannot see their dependencies
  without downloading and unpacking every tarball. A dependency solver
  that may need to go back to an older version, for example to satisfy
  an upper bound, or to reproduce a project as it was on a given day,
  needs all of them.

- **No record of when a version was published.**

  Reproducing a project “as of” a date needs the date each version first
  appeared. `PACKAGES` does not have it. Snapshot repositories have it
  implicitly, one full `PACKAGES` file per day, but finding the first
  snapshot of a version means searching through them.

- **No record of removed packages.**

  CRAN archives packages regularly. A tool cannot tell an archived
  package from a typo, or tell the user when and why it disappeared,
  without the archive date.

- **Binaries are scattered across many `PACKAGES` files.**

  P3M serves binaries for many Linux distributions, macOS and Windows,
  for many R versions, from daily snapshots. Finding out which builds
  exist for a package means asking one `PACKAGES` file per platform, R
  version and date. A lock file that covers several platforms and R
  versions needs all of those answers at once.

- **Binaries depend on the versions they were built against.**

  A binary of a package with `LinkingTo` dependencies includes code from
  the headers of those dependencies. If the installed version of a
  `LinkingTo` dependency differs, the binary may not work. `PACKAGES`
  does not record which versions a binary was built against.

## Solution

A repository with extended metadata has a base URL, and serves these
files under it:

    <base>/ALLPACKAGES.zst
    <base>/ALLPACKAGES
    <base>/ARCHIVEDPACKAGES.zst
    <base>/ARCHIVEDPACKAGES
    <base>/binaries/<package>.tsv.zst

The plain text files have the same content as their `.zst` counterparts.
Clients download the compressed files in full, and use the plain text
files to fetch only the bytes appended since their last download, see
[Updates](#updates).

The extended metadata does not replace the repository: package files are
still downloaded from the URLs listed in the metadata, and the
repository may still serve regular `PACKAGES` files for tools that do
not know about this format.

The current instances are:

| Repository | Base URL |
|----|----|
| CRAN | `https://ppm.r-pkg.org` |
| Bioconductor release `<v>` | `https://ppm-bioc.r-pkg.org/<v>`, e.g. `https://ppm-bioc.r-pkg.org/3.22` |

### `ALLPACKAGES`

`ALLPACKAGES` is a DCF file, in the same format as `PACKAGES`, with one
record for every source package version the repository ever published,
archived or not. For CRAN this is around 200,000 records.

    Package: cli
    Version: 3.6.6
    Depends: R (>= 3.4)
    Imports: utils
    Suggests: callr, covr, crayon, digest, glue (>= 1.6.0), grDevices, htmltools, htmlwidgets, knitr, methods, processx, ps (>= 1.3.4.9000), rlang (>= 1.0.2.9003), rmarkdown, rprojroot, rstudioapi, testthat (>= 3.2.0), tibble, whoami, withr
    License: MIT + file LICENSE
    NeedsCompilation: yes
    SHA256: 27b5e74a6b2a8264457ce8412ded40aa7b660e01894bea4bfde956bbc4b5b294
    SHA256Original: b2b58d6dd82f5798b335e39c00591686a01fd3e94399ef898e146173e36f18f9
    Snapshot: 2026-04-10
    DownloadURL: https://p3m.dev/cran/2026-04-10/src/contrib/cli_3.6.6.tar.gz

Fields copied from the package’s `DESCRIPTION` file, if present:

- `Package`, `Version` (always present).
- `Depends`, `Imports`, `LinkingTo`, `Suggests`, `Enhances`.
- `License`, `License_is_FOSS`, `License_restricts_use`.
- `NeedsCompilation`, `Priority`, `OS_type`, `Archs`,
  `SystemRequirements`.

Fields added by the repository, always present:

- `SHA256`: the SHA-256 hash of the file at `DownloadURL`.
- `SHA256Original`: the SHA-256 hash of the original source tarball, as
  published by the upstream repository (CRAN or Bioconductor). This is
  different from `SHA256` if the repository modifies the tarball before
  serving it. P3M does: it changes the `Repository` field of the
  `DESCRIPTION` file. `SHA256Original` identifies the upstream tarball,
  so tools can compare it with the hash of a package installed from
  another source. The binary indices use this hash as well.
- `Snapshot`: the date the version first appeared in the repository, as
  `YYYY-MM-DD`.
- `DownloadURL`: a URL of the source tarball that does not change over
  time. For P3M it points to the snapshot of the `Snapshot` date,
  e.g. `https://p3m.dev/cran/2026-04-10/src/contrib/cli_3.6.6.tar.gz`.

Rules for readers:

- Every field is on a single line, there are no continuation lines.

- Records are in no particular order. The initial file is sorted, and
  later records are appended to it, see [Updates](#updates).

- The same `(Package, Version)` pair may appear more than once. This
  happens when the upstream repository replaced a tarball without
  changing its version number. The records differ in `SHA256`,
  `SHA256Original`, `Snapshot` and `DownloadURL`, and possibly in the
  other fields. There are about 350 such pairs for CRAN.

- Lines starting with `#` are comments, and must be skipped. The server
  adds one after each update, with the time of the update, e.g.
  `# 2026-10-15 08:17:46`.

- A few very old records have dependency fields that do not parse,
  e.g. `Depends: tcltk (butonlyfortheGUI), allotherfunctionsarestandalone)`.
  Readers should skip these records, not the whole file.

- Versions published before the first P3M snapshot all have
  `Snapshot: 2017-10-10`, and a `DownloadURL` that points into
  `src/contrib/Archive/` of that snapshot. So for these versions the
  publication date is not known, only that it was not later than
  2017-10-10.

### `ARCHIVEDPACKAGES`

`ARCHIVEDPACKAGES` lists the packages that the upstream repository
removed. It has the same format and the same fields as `ALLPACKAGES`,
plus one more field:

- `Archived`: the date the package was archived, as `YYYY-MM-DD`.

There is one record per archived package, describing the last version
published before it was archived.

    Package: gpclib
    Version: 1.5-6
    Depends: R (>= 3.0.0), methods
    Imports: graphics
    License: file LICENSE
    License_restricts_use: yes
    NeedsCompilation: yes
    SHA256: 5ef1de6f69ca88b8d9960b3c637d566b59d7401aa4ec1d3dc30bb248e9126c00
    SHA256Original: fa4e3e1f74e327b185e873052f826312ed6f71cc907a49be4c1e224ace7b5286
    Snapshot: 2020-03-02
    DownloadURL: https://p3m.dev/cran/2020-03-02/src/contrib/gpclib_1.5-6.tar.gz
    Archived: 2023-04-06

A package is “current” if it has a record in `ALLPACKAGES` and does not
have one in `ARCHIVEDPACKAGES`. Packages that were archived before the
first P3M snapshot have `Archived: 2017-10-10`.

`ARCHIVEDPACKAGES` is updated the same way as `ALLPACKAGES`, so it has
comment lines, too.

### `binaries/<package>.tsv.zst`

There is one binary index per package, a zstd-compressed, tab-separated
file. The first line is a header with the column names, and each other
line is one downloadable file: the source tarball or a binary build.

    package version platform       arch   r_version sha256  url                                                                                       linkingto
    cli     3.6.6   source         *      *         b2b58d… https://p3m.dev/cran/2026-04-10/src/contrib/cli_3.6.6.tar.gz
    cli     3.6.6   macos          arm64  4.6       b2b58d… https://p3m.dev/cran/2026-04-10/bin/macosx/sonoma-arm64/contrib/4.6/cli_3.6.6.tgz
    cli     3.6.6   windows        x86_64 4.6       b2b58d… https://p3m.dev/cran/2026-04-10/bin/windows/contrib/4.6/cli_3.6.6.zip
    cli     3.6.6   noble          x86_64 4.6       b2b58d… https://p3m.dev/cran/2026-04-10/bin/linux/noble-x86_64/4.6/src/contrib/cli_3.6.6.tar.gz
    cli     3.6.6   manylinux_2_28 arm64  4.6       b2b58d… https://p3m.dev/cran/2026-04-10/bin/linux/manylinux_2_28-arm64/4.6/src/contrib/cli_3.6.6.tar.gz

Columns:

- `package`: the package name.

- `version`: the package version, exactly as in `ALLPACKAGES`.

- `platform`: `source` for the source tarball. Otherwise `macos`,
  `windows`, a Linux distribution name, e.g. `jammy`, `noble`,
  `bookworm`, `rhel9`, `opensuse156`, or `manylinux_2_28` for a generic
  build that works on any Linux distribution with glibc 2.28 or later.

- `arch`: `*` for source tarballs, otherwise `x86_64` or `arm64` (not
  `aarch64`).

- `r_version`: `*` for source tarballs, otherwise the minor R version
  the binary was built for, e.g. `4.5`.

- `sha256`: the `SHA256Original` hash of the source tarball of the
  version, from `ALLPACKAGES`. It is the same on every row of a version.
  It is *not* the hash of the file at `url`, so it cannot be used to
  check a download. It identifies the upstream source the binary was
  built from.

- `url`: the download URL of the file, which does not change over time.

- `linkingto`: empty for source tarballs and for packages without
  `LinkingTo` dependencies. Otherwise the source versions of the
  `LinkingTo` dependencies the binary was built against, as a
  comma-separated list of `<package>@<version>=<sha256>` entries, where
  `<sha256>` is the `SHA256Original` hash of that dependency version:

      cpp11@0.5.4=6e0053ce…,systemfonts@1.3.2=828a1780…,textshaping@1.0.5=12209d55…

  This list is the transitive closure of `LinkingTo` dependencies, not a
  copy of the package’s `LinkingTo` field. E.g. ragg has
  `LinkingTo: systemfonts, textshaping`, and its list also includes
  cpp11, because systemfonts and textshaping link to it.

Rules for readers:

- Find columns by their name in the header, not by their position. New
  columns may be added later. The `linkingto` column is optional.

- Several rows may have the same `version`, `platform`, `arch` and
  `r_version`. This happens when a package was rebuilt because a
  `LinkingTo` dependency changed, and the `linkingto` column is what
  tells them apart. A client that picks a binary for a lock file needs
  to pick the row whose `linkingto` versions match the versions it
  resolved, or fall back to the source tarball.

- The binaries of a version may come from several snapshots, e.g. a
  binary for a new R version or a new Linux distribution is added later,
  so the date in `url` is not necessarily the `Snapshot` date of the
  version.

- The macOS URL includes the macOS version the binary was built on,
  e.g. `sonoma-arm64` or `big-sur-x86_64`. The `platform` column is
  `macos` for all of them.

- A package without an index, e.g. a Bioconductor package in the CRAN
  repository, returns HTTP 404.

There is no index of all binaries, and no listing of the `binaries/`
directory. Clients download the indices of the packages they need, one
request per package.

### Updates

`ALLPACKAGES` and `ARCHIVEDPACKAGES` only ever grow: new records are
appended to the end, and existing records are not changed. The server
serves both the compressed and the plain text files with `ETag` and
`Accept-Ranges: bytes`.

A client can update its copy cheaply:

1.  On the first download, fetch the `.zst` file, parse it, and record
    the length of the uncompressed text and a hash of its last bytes
    (`rig` uses the last 512 bytes).
2.  To update, send a `Range` request for the plain text file, starting
    at the recorded length minus the size of that window.
3.  If the server answers 416 (range not satisfiable), the file has not
    changed. If the returned bytes start with the same bytes as before,
    the file only grew: parse the rest. If they do not, or the server
    ignored the `Range` header, download the `.zst` file in full.

The `.zst` files are a single zstd frame, so they cannot be fetched by
range.

The binary indices are small, and are replaced, not appended to. Clients
should revalidate them with `If-None-Match`.

The files are updated about once a day. Clients should cache them for a
day before checking for updates.

## Implementations

### Server

The CRAN and Bioconductor instances are generated from P3M data and
updated daily.

### `rig`

`rig` uses the extended metadata for:

- solving dependencies, using all versions in `ALLPACKAGES`, not just
  the current ones;
- `--exclude-newer <date>`: hiding the versions with a `Snapshot` date
  after `<date>`;
- listing available packages, hiding archived packages, and telling the
  user when a package was archived;
- picking binaries for each target of a lock file, matching `platform`,
  `arch`, `r_version` and `linkingto`;
- recording `SHA256Original` in the lock file and in installed packages,
  to tell whether an installed package is the same as the locked one.

`rig` stores the parsed `ALLPACKAGES` and `ARCHIVEDPACKAGES` in a SQLite
database, and each binary index in a binary cache file, so they are not
parsed again on every run.

A repository in `rig`’s configuration declares its extended metadata
with the `metadata` field, which is the base URL. It may contain `%v`,
which `rig` replaces with the Bioconductor version. For testing, these
environment variables override the CRAN URLs:

- `RIG_ALLPACKAGES_URL` (default
  `https://ppm.r-pkg.org/ALLPACKAGES.zst`),
- `RIG_ARCHIVEDPACKAGES_URL` (default
  `https://ppm.r-pkg.org/ARCHIVEDPACKAGES.zst`),
- `RIG_BINARIES_URL` (default `https://ppm.r-pkg.org/binaries`),

and `RIG_BIOC_METADATA_URL` (default `https://ppm-bioc.r-pkg.org`)
overrides the Bioconductor base URL.

## Backwards compatibility

The extended metadata is served in addition to the regular repository,
under a different URL. Existing tools are not affected.

## How to teach

Most users will not see these files. Tool authors can start from the
[`ALLPACKAGES`](#allpackages) section, and from the examples, since the
files are plain text after decompression:

``` sh
curl -s https://ppm.r-pkg.org/binaries/cli.tsv.zst | zstd -dc | head
```

## Open issues

- The `SHA256` field of `ALLPACKAGES` and the `sha256` column of the
  binary indices have different meanings: the first is the hash of the
  file served, the second is the hash of the upstream tarball. Should
  the column be renamed to `sha256_original`, and should the binary
  indices also have the hash of the served file, so clients can check
  their downloads?

## Unresolved questions

- What happens to an `ARCHIVEDPACKAGES` record if the package comes back
  to CRAN? Since the file is only appended to, the record stays. Readers
  could compare the `Snapshot` date of the latest `ALLPACKAGES` record
  with the `Archived` date, but this should be part of the
  specification.

- If a `(Package, Version)` pair appears more than once in
  `ALLPACKAGES`, which record should readers use? The one with the
  latest `Snapshot` date is the natural choice, but a lock file that
  records `SHA256Original` may need the older one.

- Should other repositories, e.g. R-universe, serve the same files? The
  format does not depend on P3M, except that `Snapshot` and stable
  download URLs need a repository that keeps old files.

## Alternatives

### Per-day `PACKAGES` files

P3M already serves a full `PACKAGES` file for every day. A client could
download these, but finding all versions of a package, or the first
snapshot of a version, would need thousands of downloads.

### An index of all binaries

A single file listing every binary of every package would be very large
for CRAN (cli alone has over 3,000 rows), and almost all of it would be
irrelevant for any one project. One small file per package means a
client only downloads what it needs, and the requests can run in
parallel.

### JSON instead of DCF and TSV

JSON would be easier to parse with general tools, but DCF is what R and
existing R tools already read, and `ALLPACKAGES` is a valid `PACKAGES`
file. TSV keeps the binary indices small and fast to parse.

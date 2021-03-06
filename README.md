# OBS Source Service `obs-service-go_modules`

## Contents

- [Overview](#overview)
- [Usage for packagers](#usage-for-packagers)
- [Compression format support](#compression-format-support)
- [OBS Source Service Build Mode support](#obs-source-service-build-mode-support)
- [Building Go applications with vendored dependency modules](#building-go-applications-with-vendored-dependency-modules)
- [Example](#example)
- [Example `_service` configuration](#example-_service-configuration)
- [Transition note](#transition-note)
- [openSUSE RPM packages built using `obs-service-go_modules`](#opensuse-rpm-packages-built-using-obs-service-go_modules)
- [Dependencies](#dependencies)
- [FAQ](#faq)
- [Support](#support)
- [Contributing](#contributing)
- [License](#license)

## Overview

This is the git repository for
[`devel:languages:go/obs-service-go_modules`](https://build.opensuse.org/package/show/devel:languages:go/obs-service-go_modules),
an [Open Build Service (OBS)](https://build.opensuse.org)
[Source Service](https://openbuildservice.org/help/manuals/obs-user-guide/cha.obs.source_service.html)
to download, verify, and vendor Go module dependency sources.
The authoritative source is https://github.com/openSUSE/obs-service-go_modules.

Using
[`go.mod` and `go.sum`](https://github.com/golang/go/wiki/Modules)
present in a Go application,
`obs-service-go_modules` will call Go tools in sequence:

```
go mod download
go mod verify
go mod vendor
```

`obs-service-go_modules` will create a `vendor.tar.gz` archive or other supported compression type
containing the `vendor/` directory populated by `go mod vendor`.
The archive is generated in the rpm package directory, and can be committed to
[OBS](https://build.opensuse.org) to facilitate offline Go application package builds
for [openSUSE](https://www.opensuse.org),
[SUSE](https://www.suse.com), and numerous other distributions.

## Usage for packagers

Presently it is assumed the Go application source is distributed as a compressed tarball named
`app-0.1.0.tar.gz` or other supported compression type, unpacking to `app-0.1.0/`.
The compression type can be specified using the `compression` parameter,
and defaults to `gz` (gzip).
`obs-service-go_modules` will autodetect tarball archives of the form `app-0.1.0.tar.gz`,
where the RPM packaging uses spec file `app.spec` sharing the base name `app`.

Create a `_service` file containing:

```
<services>
  <service name="go_modules" mode="disabled">
  </service>
</services>
```

The archive name can alternatively be specified using service parameter `archive`.

```
<services>
  <service name="go_modules" mode="disabled">
  <param name="archive">app-0.1.0.tar.xz</param>
  </service>
</services>

```
Run `osc` command locally:

```
osc service disabledrun
```

See [Example](#example) below for typical output with a complete `_service` file.

## Compression format support

`obs-service-go_modules` reads and writes compressed tar archives
using [`libarchive`](https://libarchive.org/) via the Python3 `ctypes` wrapper
[`python3-libarchive-c`](https://github.com/Changaco/python-libarchive-c).
While `libarchive` supports numerous compression formats,
`obs-service-go_modules` usage recommends limiting selections to
`gz` (default), `xz` and `zstd`.
Tables of representative compression method relative sizes and timings
with Go vendored dependency sources (`vendor/`) are shown below.

### Compression sizes with Go dependency sources

| project    | version  | uncompressed | gz  | xz   | zstd |
| ---------- | -------- | ------------ | --- | ---- | ---- |
| hugo       | v0.100.2 | 75M          | 11M | 7.5M | 8.3M |
| kubernetes | v1.24.1  | 129M         | 17M | 12M  | 14M  |

### Compression timing with Go dependency sources

| project    | version  | gz   | xz    | zstd |
| ---------- | -------- | ---- | ----- | ---- |
| hugo       | v0.100.2 | 1.8s |  6.3s | 0.3s |
| kubernetes | v1.24.1  | 2.9s | 10.9s | 0.4s |

The above are an average of five runs on Intel i7-6820HQ CPU.
The `zstd` format has a clear advantage in speed with reasonable compression ratio,
and is likely to become the default compression method in a future release.
Decompression timings are closely matched among `gz`, `xz`, and `zstd` compression methods.

## OBS Source Service Build Mode support

OBS Source Services can run in one of several modes as shown in
[OBS Documentation: Using Source Services: Modes of Services](https://openbuildservice.org/help/manuals/obs-user-guide/cha.obs.source_service.html#sec.obs.sserv.mode).

Currently the recommended mode is `disabled` (aliased as `manual`) which implies:

- The service is run locally via explicit CLI call `osc service disabledrun`

- `obs-service-go_modules` and its dependencies
  `python3`, `libarchive` and `python3-libarchive-c`
  are installed on the local machine.
  In particular, `python3-libarchive-c` packages
  are not available in all SUSE repositories at this time.

- The resulting `vendor.tar.gz` or other supported compression type
  should be committed and referenced as an RPM `Source:`.

- `Source:` file names are as given in the RPM `.spec`, no `_service:` prefix is applied.

If and when `obs-service-go_modules` is available on
[OBS](https://build.opensuse.org),
additional modes including `Default` will be supported.
`Default` runs server side after each commit
and locally before every local build.
This will enable tighter integration with the
[tar_scm / obs_scm](https://github.com/openSUSE/obs-service-tar_scm) source service,
including uncommitted server-managed `.obscpio` source and vendor archives.

## Building Go applications with vendored dependency modules

Go commands support building with vendored dependencies,
but it is no longer on by default.
Upstream has stated vendoring is not going away.
To ensure the top-level `vendor/` directory is used by go build, either:

- pass the argument `go build -mod=vendor` to each invocation

- set environment variable `GOFLAGS=-mod=vendor` to apply the setting to all invocations

More information about additional controls is available at:
[Go Module Knobs](https://github.com/thepudds/go-module-knobs/blob/master/README.md),
[Go Wiki: Modules: How do I use vendoring](https://github.com/golang/go/wiki/Modules#how-do-i-use-vendoring-with-modules-is-vendoring-going-away) and
[Go Wiki: Modules: Old vs. new behavior](https://github.com/golang/go/wiki/Modules#when-do-i-get-old-behavior-vs-new-module-based-behavior)

## Example

Using the [hugo](https://github.com/gohugoio) static site generator as
an example of a Go application with a large number of Go module
dependencies, `obs-service-go_modules` produces `vendor.tar.gz`:

```
$ ls -1
hugo-0.57.2.tar.gz
hugo.changes
hugo.spec
_service
_servicedata

$ osc service disabledrun
INFO:obs-service-go_modules:Autodetecting archive since no archive param provided in _service
INFO:obs-service-go_modules:Archive autodetected at /path/to/prj/pkg/hugo-0.57.2.tar.gz
INFO:obs-service-go_modules:Using archive hugo-0.57.2.tar.gz
INFO:obs-service-go_modules:Extracting hugo-0.57.2.tar.gz to /path/to/tmpdir
INFO:obs-service-go_modules:Using go.mod found at /path/to/tmpdir/hugo-0.57.2/go.mod
INFO:obs-service-go_modules:go mod download
go: finding cloud.google.com/go v0.39.0
go: finding contrib.go.opencensus.io/exporter/aws v0.0.0-20181029163544-2befc13012d0
go: finding contrib.go.opencensus.io/exporter/ocagent v0.4.12
go: finding contrib.go.opencensus.io/exporter/stackdriver v0.11.0
go: finding contrib.go.opencensus.io/integrations/ocsql v0.1.4
go: finding contrib.go.opencensus.io/resource v0.0.0-20190131005048-21591786a5e0
go: finding github.com/Azure/azure-amqp-common-go v1.1.4
go: finding github.com/Azure/azure-pipeline-go v0.1.9
go: finding github.com/Azure/azure-sdk-for-go v27.3.0+incompatible
(elided: 193 additional entries)
go: finding gopkg.in/fsnotify.v1 v1.4.7
go: finding gopkg.in/resty.v1 v1.12.0
go: finding gopkg.in/tomb.v1 v1.0.0-20141024135613-dd632973f1e7
go: finding gopkg.in/yaml.v2 v2.2.2
go: finding honnef.co/go/tools v0.0.0-20190106161140-3f1c8253044a
go: finding pack.ag/amqp v0.11.0
INFO:obs-service-go_modules:go mod verify
INFO:obs-service-go_modules:all modules verified
INFO:obs-service-go_modules:go mod vendor
INFO:obs-service-go_modules:Vendor go.mod dependencies to vendor.tar.gz

$ ls -1
hugo-0.57.2.tar.gz
hugo.changes
hugo.spec
_service
_servicedata
vendor.tar.gz
```

## Example `_service` configuration

OBS Source Services
[`obs-service-tar_scm`](https://github.com/openSUSE/obs-service-tar_scm),
[`obs-service-set_version`](https://github.com/openSUSE/obs-service-set_version) and
[`obs-service-recompress`]()
can be used together to automate Go application source archive handling and support local development workflows:

```
<services>
  <service name="tar_scm" mode="disabled">
    <param name="url">git://github.com/gohugoio/hugo.git</param>
    <param name="scm">git</param>
    <param name="exclude">.git</param>
    <param name="revision">v0.57.2</param>
    <param name="versionformat">@PARENT_TAG@</param>
    <param name="changesgenerate">enable</param>
    <param name="versionrewrite-pattern">v(.*)</param>
  </service>
  <service name="set_version" mode="disabled">
    <param name="basename">hugo</param>
  </service>
  <service name="recompress" mode="disabled">
    <param name="file">*.tar</param>
    <param name="compression">gz</param>
  </service>
  <service name="go_modules" mode="disabled">
  </service>
</services>
```

Persistent state for changelog generation is stored in `_servicedata`.

## Transition note

Until such time as `obs-service-go_modules` is available on
[OBS](https://build.opensuse.org), `vendor.tar.gz` should
be committed along with the Go application release tarball.

## openSUSE RPM packages built using `obs-service-go_modules`

- [hugo](https://build.opensuse.org/package/show/devel:languages:go/hugo)
- [go-modiff](https://build.opensuse.org/package/show/devel:languages:go/go-modiff)
- [gohack](https://build.opensuse.org/package/show/devel:languages:go/gohack)
- [mod](https://build.opensuse.org/package/show/devel:languages:go/mod)
- [mgit](https://build.opensuse.org/package/show/devel:languages:go/mgit)

## Dependencies

`obs-service-go_modules` requires:

- `python3`
- `python-libarchive-c` ctypes wrapper for the `libarchive` C library (added in `v0.5.0`)

The Python standard library supports only gzipped tar archives.
The `libarchive` dependency was chosen to support additional compression types
including `xz` and the `cpio_newc` used by `.obscpio` archives.
Supported compression types are intentionally limited to
`gz`, `xz`, `zstd` and `cpio_newc` to preserve future flexibility
in the event eliminating the `python-libarchive-c` dependency is desirable.

## FAQ

### Q: Does `vendor.tar.gz` need to be committed to OBS package?

A: Currently yes.
As long as  `obs-service-go_modules` is run locally via `osc service disabledrun`,
then `vendor.tar.gz` should be committed and referenced as an additional `Source:`.
If and when `obs-service-go_modules` is available on
[OBS](https://build.opensuse.org),
additional strategies should be possible such as a `vendor.cpio`
where the vendored dependencies are managed on the fly
and will not need to be committed to OBS package revisions.
The single source of truth `go.mod` and `go.sum` always remain with the application source code.

### Q: Does `obs-service-go_modules` update dependencies to newer versions?

A: No. Go modules use
[Minimum Version Selection](https://github.com/golang/go/wiki/Modules#faqs--minimal-version-selection),
selecting the minimum (oldest) version of a Go module that satisfies all `go.mod` entries in the transitive dependency set.
Go modules are relatively new and real-world use remains to be seen,
but the expectation is that dependency versions will increment at a measured pace
driven by upstream projects making releases with a well-tested dependency set.
It is a design goal that there should be no surprise updates pulled in,
and the dependency set selected remains repeatable over time.
These characteristics should be quite favorable for distribution packagers.

### Q: Does `obs-service-go_modules` cache Go module downloads to save time and bandwidth?

A: Yes. For local use with `osc service disabledrun`,
`obs-service-go_modules` uses the standard module cache `~/go/pkg/mod`.
Subsequent runs of `obs-service-go_modules` with a populated cache will finish in less time.

### Q: Would `obs-service-go_modules` installed on OBS cache Go module downloads to conserve server resources?

A: Not directly.
The Go module cache `~/go/pkg/mod` would not persist between OBS build runs.
Running a private [Go proxy](https://proxy.golang.org) inside OBS could accomplish this,
as well as provide protections against third-party service outages and
upstream Go modules being removed by the author.

## Support

`obs-service-go_modules` intends to be compatible with most upstream Go projects that use best practice source layouts and module conventions.
If you are packaging a Go application with an uncommon project layout or nonstandard `go.mod` usage pattern that presents compatibility problems,
please file an [issue](https://github.com/openSUSE/obs-service-go_modules/issues) with description and reference the specific upstream tag or commit.
While it may not be possible to support every unique upstream layout,
a maintainer will evaluate feasibility of adding support for that special case or improve output messages to clearly indicate the error.

## Contributing

In keeping with [support](#support) objectives,
feature ideas are welcome,
particularly those that improve idiomatic OBS and RPM usage,
packaging automation and commonality among Go application package sources.
It is also a goal to keep manual configuration parameters in `_service` to a minimum for maintainability.
Please file proposals as an [issue](https://github.com/openSUSE/obs-service-go_modules/issues)
to discuss feasibility and feature design leading to a subsequent implementation via pull request.

When creating a pull request,
please enable the permission:
"[Maintainers are allowed to edit this pull request](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/working-with-forks/allowing-changes-to-a-pull-request-branch-created-from-a-fork#enabling-repository-maintainer-permissions-on-existing-pull-requests)"
if it is not on by default.

## License

GNU General Public License v2.0 or later

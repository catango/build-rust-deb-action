[![Linting](https://github.com/catango/build-rust-deb-action/actions/workflows/lint.yml/badge.svg)](https://github.com/catango/build-rust-deb-action/actions/workflows/lint.yml)
[![Testing](https://github.com/catango/build-rust-deb-action/actions/workflows/test.yml/badge.svg)](https://github.com/catango/build-rust-deb-action/actions/workflows/test.yml)

# Build Debian Packages GitHub Action

This action builds rust Debian packages in a clean, flexible environment.

It is mainly a shell wrapper around `cargo deb`, using a configurable
Docker image to install build dependencies in and build packages. Resulting
.deb files are accessible in an artifacts directory, specified by the output of the action.

cargo deb specific configuration in Cargo.toml is required for proper build. Please read the official 
[cargo-deb documentation](https://crates.io/crates/cargo-deb).


## Usage
### Basic Example
```yaml
on: push

jobs:
  build-debs:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: catango/build-rust-deb-action@v1
```

### Input Parameters
All input parameters have a default value or are optional.

#### `apt-opts`
Extra options to be passed to `apt-get` when installing build dependencies and
extra packages.

Optional and empty by default.

#### `before-build-hook`
Shell command(s) to be executed after installing the build dependencies and right
before `dpkg-buildpackage` is executed. A single or multiple commands can be
given, same as for a
[`run` step](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#jobsjob_idstepsrun)
in a workflow.

The hook is executed with `sh -c` as the root user *inside* the build
container. The working directory is the workspace. The package contents from
the build dependencies and [`extra-build-deps`](#extra-build-deps) are
available.

Optional and empty by default.

Example use case:
```yaml
- uses: catango/build-rust-deb-action@v1
  with:
    before-build-hook: debchange --controlmaint --local="+ci${{ github.run_id }}~git$(git rev-parse --short HEAD)" "CI build"
    extra-build-deps: devscripts git
```
#### `docker-image`
Name of a Debian-based Docker image to use as build container or path of a
Dockerfile in the workspace to build a temporary container from.

Defaults to `debian:stable-slim`.

#### `extra-docker-args`
Additional command-line arguments passed to `docker run` when the build
container is started. This might be needed if specific volumes or network
settings are required.

Optional and empty by default.

#### `extra-repo-keys`
Extra keys for APT to trust in the build environment. Useful in combination
with [`extra-repos`](#extra-repos).

The parameter can be used to pass either one or multiple ASCII-armored keys, or
a newline-separated list of paths to key files in ASCII-armored or binary
format. Paths to key files must be relative to the workspace.

Optional and empty by default.

#### `extra-repos`
Extra APT repositories to configure as sources in the build environment.

Entries can be given in either format supported by APT: one-line style or
deb822 style, see
[`man sources.list`](https://manpages.debian.org/sources.list.5).

Optional and empty by default.

#### `host-build-deps`
Extra packages to be installed on host as additional build tools (e.g. cmake) 
or required dependencies for buildtools, compiled in rust for host environment.

By default, these packages are installed without their recommended
dependencies. To change this, pass `--install-recommends` in
[`apt-opts`](#apt-opts).

Optional and empty by default.

#### `target-build-deps`
Extra packages to be installed as “build dependencies” for build target. This is required
if rust packages are linked to dependent system libraries (e.g. alsa). 

By default, these packages are installed without their recommended
dependencies. To change this, pass `--install-recommends` in
[`apt-opts`](#apt-opts).

Optional and empty by default.

#### `target-arch`
The architecture packages are built for. Cross building is automatically set up for target architecture
adhering debian best practices 
[in the Debian wiki](https://wiki.debian.org/CrossCompiling#Building_with_dpkg-buildpackage).

Optional and defaults to the amd64.

Basic example for a cross-build:
```yaml
- uses: catango/build-rust-deb-action@v1
  with:
    target-arch: i386
```

#### `setup-hook`
Shell command(s) to be executed after setting-up the build environment and
right before installing the build dependencies. A single or multiple commands
can be given, same as for a
[`run` step](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#jobsjob_idstepsrun)
in a workflow.

The hook is executed with `sh -c` as the root user *inside* the build
container. The working directory is the workspace.

Optional and empty by default.

#### `source-dir`
Directory relative to the workspace that contains the package sources,
especially the `debian/` subdirectory.

Defaults to the workspace.

#### `rust-features`
Additional rust package features of the package to be built  
[in the Cargo wiki](https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html).

Optional and empty by default.

#### `rust-version`
Rust version the package shall be built with. Please enter full rust version tag (e.g. `1.82.0`) 

Optional and empty by default.

#### `rust-buildtools`
Additional Rust build tools, required to build the package. Specific version of tools can be specified 
with rust syntax (tool@version)

Optional and empty by default.

### Output Parameters

#### `artifact-dir`
Directory, where build artifacts are uploaded.

### Environment Variables
Environment variables work as you would expect. So you can use e.g. the
`DEB_BUILD_OPTIONS` variable:
```yaml
- uses: catango/build-rust-deb-action@v1
  env:
    DEB_BUILD_OPTIONS: noautodbgsym
```

## Motivation
There are other GitHub actions that build rust packages for debian. At the time of
writing, all of them had one or multiple limitations:
 * Hard-coding too specific options,
 * hard-coding one specific distribution as build environment,
 * installing unnecessary packages as build dependencies,

This action’s goal is to not have any of these limitations.


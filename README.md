Protocol Buffers - Google's data interchange format
===================================================

[![OpenSSF Scorecard](https://api.securityscorecards.dev/projects/github.com/protocolbuffers/protobuf/badge)](https://securityscorecards.dev/viewer/?uri=github.com/protocolbuffers/protobuf)

Copyright 2008 Google LLC

## Cygwin Port (`cygwin/v34.1` branch)

This branch adds Cygwin support to Protocol Buffers v34.1 (C++ 7.34.1, released 2026-03-19).

Protobuf does not officially support Cygwin. Six patches fix PE/COFF platform differences -- weak symbol semantics, text-mode I/O, `jmp_buf` sizing, and linker constructor behavior -- that prevent protoc and the C++ runtime from functioning. Four regression tests guard against silent breakage. The patches are intended as a stopgap until upstream support exists.

This port exists to support `grpcio` (Python gRPC bindings) on Cygwin. The dependency chain is:

    psutil -> pypinfo -> grpcio -> protobuf -> abseil-cpp

### Prerequisites

- GCC 13+, CMake 3.16+, Cygwin x86_64
- **Abseil LTS 20250512.1 for Cygwin** must be installed first. Protobuf v34.1 requires exactly this version. The Cygwin system package (`libabsl-devel`) ships 20260107, which is too new -- protobuf pins its abseil dependency and will not build against a newer release. The upstream abseil source also won't compile on Cygwin without patches. Use the pre-built release or build from source:
  - Release: <https://github.com/phdye-cygwin/abseil-cpp/releases/tag/20250512.1-cygwin>
  - Source: <https://github.com/phdye-cygwin/abseil-cpp/tree/cygwin/support>
  - Installs to `/usr/local`, so it won't conflict with the system package in `/usr`.

### Building on Cygwin

```bash
mkdir build && cd build
cmake ~/repo/protobuf \
  -DCMAKE_PREFIX_PATH=/usr/local \
  -DCMAKE_CXX_STANDARD=17 \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_CXX_FLAGS="-D_GLIBCXX_USE_CXX11_ABI=1 -DGTEST_HAS_PTHREAD=1" \
  -Dprotobuf_ABSL_PROVIDER=package
cmake --build . --parallel 8
```

The key flag is `-Dprotobuf_ABSL_PROVIDER=package`, which tells CMake to use the installed Abseil rather than fetching one. `-DCMAKE_PREFIX_PATH=/usr/local` points `find_package(absl)` at the Cygwin Abseil install.

Both `-D` flags in `CMAKE_CXX_FLAGS` must match the Abseil build:

- **`_GLIBCXX_USE_CXX11_ABI=1`** -- Cygwin's libstdc++ defaults to the old COW string ABI (`sizeof(std::string)` = 8). Protobuf requires the SSO ABI (`sizeof(std::string)` = 32).
- **`GTEST_HAS_PTHREAD=1`** -- googletest omits Cygwin from its pthread platform list, causing `Mutex` layout mismatches in test binaries. Only needed when building tests (`-Dprotobuf_BUILD_TESTS=ON`).

### Building with a custom prefix

```bash
cmake --install . --prefix /path/to/prefix
```

### Binary release

Pre-built protoc, C++ runtime, and UPB libraries for Cygwin x86_64
(GCC 13, Release mode) are available on the
[Releases](https://github.com/phdye-cygwin/protobuf/releases) page.

The [patched Abseil](https://github.com/phdye-cygwin/abseil-cpp/releases/tag/20250512.1-cygwin)
must be installed first. Both tarballs install to `/usr/local`.

```bash
# Install Abseil (dependency)
cd / && tar xzf abseil-cpp-20250512.1-cygwin-x86_64.tar.gz

# Install Protobuf
cd / && tar xzf protobuf-v34.1-cygwin-x86_64.tar.gz
```

### Patch summary

**Fixes** (6 files):

| File | Purpose |
|------|---------|
| `src/google/protobuf/port_def.inc` | Disable `PROTOBUF_ATTRIBUTE_WEAK` on Cygwin -- PE/COFF weak symbols collapse all definitions into one, breaking per-type vtable dispatch |
| `src/google/protobuf/compiler/plugin.cc` | Set binary mode on plugin stdin/stdout -- Cygwin defaults to text mode, corrupting protobuf wire data with `\r\n` translation |
| `src/google/protobuf/compiler/command_line_interface_tester.cc` | Remove stale `#if !defined(__CYGWIN__)` guards around stdout capture |
| `src/google/protobuf/compiler/command_line_interface_unittest.cc` | Remove stale `#if !defined(__CYGWIN__)` guard in `PrintFreeFieldNumbers` test |
| `upb/port/def.inc` | Use plain `setjmp`/`longjmp` on Cygwin -- `sigjmp_buf` is larger than `jmp_buf` (34 vs 32 longs); the `sigsetjmp` path overflows UPB's buffers |
| `upb/mini_table/generated_registry.c` | Tolerate duplicate extension registration -- per-TU constructors on PE/COFF register the same extensions multiple times |

**Regression tests** (4 files):

| File | Purpose |
|------|---------|
| `src/google/protobuf/port_test.cc` | Platform config, type sizes, binary round-trip |
| `src/google/protobuf/descriptor_unittest.cc` | Distinct descriptor pointers (catches weak symbol collapse) |
| `src/google/protobuf/wire_format_unittest.cc` | Wire data survives binary file I/O without CRLF corruption |
| `src/google/protobuf/code_generator_unittest.cc` | Plugin wire data contains bytes that text-mode I/O would corrupt |

**Build and docs** (3 files):

| File | Purpose |
|------|---------|
| `src/google/protobuf/BUILD.bazel` | Add missing `:protobuf_lite` dep to `port_test` |
| `src/README.md` | Cygwin build instructions |
| `README.md` | Port documentation |

### Upstream dependency

Protobuf cannot build on Cygwin without the patched Abseil:

- Repo: <https://github.com/phdye-cygwin/abseil-cpp>
- Branch: `cygwin/support`
- Tag: `20250512.1-cygwin`

### Known limitations

- **UPB plugin tests**: `protoc-gen-upb` occasionally crashes (SIGSEGV) when invoked by protoc during certain test scenarios. The protoc compiler and C++ runtime are unaffected.
- **Subprocess tests**: A small number of tests that fork/exec child processes may hang on Cygwin and require timeout enforcement.
- **Bazel**: Only CMake is supported. Bazel on Cygwin is not tested.

### Test results

The existing protobuf test suite (745 tests in the CMake build) passes on Cygwin.

---

Overview
--------

Protocol Buffers (a.k.a., protobuf) are Google's language-neutral,
platform-neutral, extensible mechanism for serializing structured data. You
can learn more about it in [protobuf's documentation](https://protobuf.dev).

This README file contains protobuf installation instructions. To install
protobuf, you need to install the protocol compiler (used to compile .proto
files) and the protobuf runtime for your chosen programming language.

Working With Protobuf Source Code
---------------------------------

Most users will find working from
[supported releases](https://github.com/protocolbuffers/protobuf/releases) to be
the easiest path.

If you choose to work from the head revision of the main branch your build will
occasionally be broken by source-incompatible changes and insufficiently-tested
(and therefore broken) behavior.

If you are using C++ or otherwise need to build protobuf from source as a part
of your project, you should pin to a release commit on a release branch.

This is because even release branches can experience some instability in between
release commits.

### Bazel with Bzlmod

Protobuf supports
[Bzlmod](https://bazel.build/external/module) with Bazel 8 +.
Users should specify a dependency on protobuf in their MODULE.bazel file as
follows.

```
bazel_dep(name = "protobuf", version = <VERSION>)
```

Users can optionally override the repo name, such as for compatibility with
WORKSPACE.

```
bazel_dep(name = "protobuf", version = <VERSION>, repo_name = "com_google_protobuf")
```

### Bazel with WORKSPACE

Users can also add the following to their legacy
[WORKSPACE](https://bazel.build/external/overview#workspace-system) file.

Note that with the release of 30.x there are a few more load statements to
properly set up rules_java and rules_python.

```
http_archive(
    name = "com_google_protobuf",
    strip_prefix = "protobuf-VERSION",
    sha256 = ...,
    url = ...,
)

load("@com_google_protobuf//:protobuf_deps.bzl", "protobuf_deps")

protobuf_deps()

load("@rules_java//java:rules_java_deps.bzl", "rules_java_dependencies")

rules_java_dependencies()

load("@rules_java//java:repositories.bzl", "rules_java_toolchains")

rules_java_toolchains()

load("@rules_python//python:repositories.bzl", "py_repositories")

py_repositories()
```

Protobuf Compiler Installation
------------------------------

The protobuf compiler is written in C++. If you are using C++, please follow
the [C++ Installation Instructions](src/README.md) to install protoc along
with the C++ runtime.

For non-C++ users, the simplest way to install the protocol compiler is to
download a pre-built binary from our [GitHub release page](https://github.com/protocolbuffers/protobuf/releases).

In the downloads section of each release, you can find pre-built binaries in
zip packages: `protoc-$VERSION-$PLATFORM.zip`. It contains the protoc binary
as well as a set of standard `.proto` files distributed along with protobuf.

If you are looking for an old version that is not available in the release
page, check out the [Maven repository](https://repo1.maven.org/maven2/com/google/protobuf/protoc/).

These pre-built binaries are only provided for released versions. If you want
to use the github main version at HEAD, or you need to modify protobuf code,
or you are using C++, it's recommended to build your own protoc binary from
source.

If you would like to build protoc binary from source, see the [C++ Installation Instructions](src/README.md).

Protobuf Runtime Installation
-----------------------------

Protobuf supports several different programming languages. For each programming
language, you can find instructions in the corresponding source directory about
how to install protobuf runtime for that specific language:

| Language                             | Source                                                      |
|--------------------------------------|-------------------------------------------------------------|
| C++ (include C++ runtime and protoc) | [src](src)                                                  |
| Java                                 | [java](java)                                                |
| Python                               | [python](python)                                            |
| Objective-C                          | [objectivec](objectivec)                                    |
| C#                                   | [csharp](csharp)                                            |
| Ruby                                 | [ruby](ruby)                                                |
| Go                                   | [protocolbuffers/protobuf-go](https://github.com/protocolbuffers/protobuf-go)|
| PHP                                  | [php](php)                                                  |
| Dart                                 | [dart-lang/protobuf](https://github.com/dart-lang/protobuf) |
| JavaScript                           | [protocolbuffers/protobuf-javascript](https://github.com/protocolbuffers/protobuf-javascript)|

Quick Start
-----------

The best way to learn how to use protobuf is to follow the [tutorials in our
developer guide](https://protobuf.dev/getting-started).

If you want to learn from code examples, take a look at the examples in the
[examples](examples) directory.

Documentation
-------------

The complete documentation is available at the [Protocol Buffers doc site](https://protobuf.dev).

Support Policy
--------------

Read about our [version support policy](https://protobuf.dev/version-support/)
to stay current on support timeframes for the language libraries.

Developer Community
-------------------

To be alerted to upcoming changes in Protocol Buffers and connect with protobuf developers and users,
[join the Google Group](https://groups.google.com/g/protobuf).

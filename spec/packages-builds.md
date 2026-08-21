# Packages, Manifests, and Builds

## Concepts

A **workspace manifest** is one `dot.yaml` file and its source tree. It defines
zero or one library and zero or more executable targets. The workspace itself
has no source-visible name.

A **library** contains named modules. Packaging a library creates one immutable,
named, versioned **package**. A **module** is the source import, compilation, and
top-level-private unit. An **executable** contains one private source module and
exactly one `::main`.

Version 1 does not support multiple libraries or nested workspace members in one
manifest. A repository may contain multiple independent manifests in different
directories and connect them with path dependencies.

## Manifest encoding and top-level schema

`dot.yaml` is UTF-8 YAML 1.2. It must contain a mapping with exactly these
recognized top-level keys:

```yaml
format: 1
language: 1
workspace:                         # optional
  dependencies: []
library: ...                       # optional
executables: {}                    # optional
```

`format` and `language` are required integers and must both equal `1` for this
specification. Unknown keys at any schema level are errors. YAML aliases,
anchors, custom tags, duplicate mapping keys, and merge keys are forbidden so
the manifest has one canonical data model.

At least one of `library` or a non-empty `executables` mapping is required.
Relative paths are resolved from the directory containing `dot.yaml`.

## Workspace dependency catalog

The optional workspace catalog declares every external package that a local
module or executable may opt into:

```yaml
workspace:
  dependencies:
    - package: compression
      version: 1.2.0
      registry: local

    - package: command_line
      path: ../command_line/dot.yaml
```

`dependencies` defaults to an empty list. Each entry has one of two forms.

### Registry dependency

```yaml
- package: package_identifier
  version: 1.2.0
  registry: local                 # optional; defaults to local
  when: ...                       # optional
```

`package` is a Dot identifier. `version` is one exact semantic-version string;
ranges and wildcards are invalid. `registry` is a configured registry name
string and defaults to `local`.

A semantic version is `MAJOR.MINOR.PATCH`, optionally followed by
`-PRERELEASE` and/or `+BUILD`. The three core components are non-negative
decimal integers without leading zeroes except the single digit `0`.
Prerelease/build parts are dot-separated non-empty ASCII alphanumeric or hyphen
identifiers. A numeric prerelease identifier has no leading zero. Exact package
identity includes prerelease and build text as written; no precedence or range
selection is performed in version 1.

### Path dependency

```yaml
- package: package_identifier
  path: ../dependency/dot.yaml
  when: ...
```

The path must identify another version-1 manifest containing exactly one
library. `package` must equal that library's package name. The dependency's
version comes from its manifest. `version` and `registry` are invalid in a path
entry.

Workspace package names must be unique and must not equal the package name of a
library in the same manifest. A target condition that excludes an entry makes
that package unavailable on the selected target; opting into or importing it is
then an error.

## Library schema

```yaml
library:
  package: image
  version: 0.1.0

  modules:
    core:
      sources:
        - src/image/core.dot
      dependencies: []

    png:
      sources:
        - src/image/png/*.dot
      dependencies:
        - compression

    _implementation:
      sources:
        - src/image/internal/**/*.dot
      dependencies: []
```

`package` is a Dot identifier and is the library's source import and registry
name. `version` is an exact semantic version. `modules` is a non-empty mapping
from module identifier to module configuration.

A module configuration recognizes:

```yaml
sources: []        # required, non-empty
dependencies: []   # optional, defaults empty
link: []           # optional, defaults empty
```

`dependencies` contains unique package identifiers from the workspace catalog.
It does not list other modules in the current library; relative source imports
determine internal module edges.

A module whose name begins with `_` is private to the library. Every other
module is public and included by `import package_name;`. Private modules may be
imported only with `.module_name` from the same library.

No library module may define `::main`.

## Executable schema

```yaml
executables:
  image_convert:
    sources:
      - tools/image_convert/*.dot
    dependencies:
      - image
      - command_line
    link: []
```

Executable names are Dot identifiers used by build commands but are not source
names. Each configuration recognizes the same `sources`, `dependencies`, and
`link` keys as a module. An executable name must differ from the current
library's package name so a build-target name is unambiguous. Sources form one
private non-importable module.

An executable may list the current manifest's library package name even though
that package does not appear in the workspace external catalog. It then imports
that library in source normally. No other undeclared dependency name is valid.

Exactly one accepted `::main` must exist across the executable's source files.

## Source patterns

Source entries are relative, forward-slash-separated paths or glob patterns.
They must resolve within the manifest directory after symlink resolution; `..`
may be written only when the canonical result remains inside that directory.

Version-1 glob metacharacters are:

- `*`: zero or more non-`/` characters;
- `?`: exactly one non-`/` character; and
- `**`: zero or more complete path components when it occupies a component.

Character classes, brace expansion, backslash escaping, and platform-native
separator semantics are unsupported. A literal `*` or `?` filename therefore
cannot be selected.

Each pattern is expanded, paths are normalized to forward slashes, and the
combined result is sorted by Unicode code-point order. A pattern matching no
regular file is an error. Duplicate resolved files in one target are errors. A
source file cannot belong to two modules of one library. A file may be compiled
separately as part of two distinct executable targets, which creates independent
private modules. A file selected for a library module may not also be selected
for an executable in the same manifest; common executable code must either be
placed in the library or deliberately selected only into the executable
modules.

Only files ending in `.dot` are accepted as source entries.

## Target conditions

Dependency and native-link entries may contain:

```yaml
when:
  os:
    - linux
    - macos
  arch: x86_64
```

`os` and `arch` each accept either one string or a non-empty list of unique
strings. Values within one list are alternatives; when both fields appear, both
must match. An empty `when` is invalid.

The selected target has exactly one canonical OS and architecture string, also
exposed to compile-time code. An implementation documents its supported names
and must recognize at least `linux`, `macos`, and `windows` for OS and `x86_64`
and `aarch64` for architecture when it supports those targets. A condition value
unknown to the selected compiler is an error rather than silently false.

## Native link inputs

`link` is an ordered list. Each entry contains exactly one action plus optional
`when`:

```yaml
link:
  - library: pthread
    when:
      os: linux

  - framework: Cocoa
    when:
      os: macos

  - search_path: native/lib
  - option: -Wl,--as-needed
```

Actions are:

- `library`: one linker library name without a path;
- `framework`: one platform framework name;
- `search_path`: a relative directory inside the workspace/package snapshot;
- `option`: one opaque backend linker argument.

Absolute search paths are invalid in a portable manifest. Package creation must
include files beneath a referenced search path that are needed by the declared
package or fail with a missing-native-input diagnostic. Link actions are applied
in manifest list order after filtering target conditions. Raw options are
implementation/target-specific and become part of the package content hash.

A native input does not make a Dot package importable; it only affects final
linking of targets that use the module/package.

## Dependency and import rules

A module may source-import only packages listed in its own `dependencies` after
target filtering. Executables follow the same rule. One target's opt-in does not
grant another target access.

Relative imports establish current-library module edges. External and relative
module graphs, and the package graph, must be acyclic. The compiler/build tool
must diagnose a cycle with a concrete cycle path.

Only one version of a public package name may occur in one resolved program.
Encountering two versions is a resolution error even if their imported modules
would not collide.

Ordinary transitive dependencies are resolved and linked as needed but are not
source-importable without direct opt-in. Explicit source re-exports make those
declarations available through the re-exporting package surface as defined in
[Program Structure](program-structure.md#imports).

## Lockfile

`dot.lock` is generated UTF-8 YAML with this canonical logical schema:

```yaml
format: 1
language: 1
target:
  os: linux
  arch: x86_64

packages:
  - package: compression
    version: 1.2.0
    source:
      registry: local
    content_hash: sha256:<lowercase-hex>
    dependencies: []

  - package: command_line
    version: 0.4.0
    source:
      path: ../command_line/dot.yaml
    content_hash: sha256:<lowercase-hex>
    dependencies: []
```

Package entries are sorted by package name. `dependencies` is the sorted list of
direct resolved package names. A source has exactly one of `registry` or `path`.
Paths are normalized relative to the root manifest directory when possible.

The content hash covers the immutable package snapshot: normalized manifest,
selected Dot sources, declared native input files/options, and dependency
identity metadata. The exact byte canonicalization is build-tool-defined but
must be deterministic and documented.

Normal builds use an existing compatible lockfile and fail if source manifests,
target, selected conditions, path contents, or registry content disagree.
An explicit update operation rewrites it. Committing the lockfile is recommended
for reproducible executables and packages.

## Packages

`dot package` snapshots the current library. A version-1 package contains:

- its normalized manifest library/module data;
- selected Dot sources;
- package dependency metadata and relevant lock identities;
- declared native inputs; and
- a content hash.

It is source-based. Compiler-generated C++, object files, and backend-specific
binary interfaces may be cached or included as optional acceleration artifacts,
but they are never the normative package interface and may be discarded.

Package entries in a registry are immutable by `(package, version,
content_hash)`. Installing different contents under an existing package and
version is an error. A registry may store multiple identical-hash copies only as
one logical entry.

Path dependencies are for active development and do not require publishing.

## Local registry and build cache

The `local` registry location is toolchain/user configuration and is not written
as an absolute path into a portable manifest. `dot install` places the package
snapshot created by `dot package` into that registry.

The build cache is distinct. It may store generated C++, object files,
reflection indexes, and executables keyed by at least:

- source/package content hashes;
- complete resolved dependency graph;
- language and manifest versions;
- compiler/backend version;
- target OS, architecture, and ABI;
- build profile and relevant options; and
- native link inputs.

An ordinary build may populate the cache but never changes a named registry
package.

## Commands

A conforming version-1 build tool provides these logical operations; exact CLI
formatting may add options without changing semantics:

- `dot build [target]`: compile the named library package target or executable;
  with no target, compile the library when present and every executable in the
  manifest, without publishing.
- `dot run <executable> [-- arguments...]`: build and execute one manifest
  executable, passing arguments after the separator.
- `dot package`: create the current library's immutable package snapshot.
- `dot install`: package and install the current library into the configured
  local registry.

Running `package` or `install` without a library is an error. A future remote
publish operation is outside version 1.

## Target and profile

A build selects one target OS, architecture, and ABI and one profile. `debug`
and `release` are required profile names; implementations may add others. The
selection is available as `build.target.os`, `build.target.arch`, and
`build.profile` during compile-time evaluation.

All modules in one linked program use the same target and language version. A
source package is recompiled for that target. Target-specific `#repr("C")`
layout and native links are resolved only after selection.

## Bootstrap C++ backend

The initial implementation targets C++20 and initially supports Clang. A backend
must select options that preserve Dot semantics, including:

- exact fixed-width integer operations without accidental promotions changing
  overflow behavior;
- IEEE binary32/binary64, signed zero, NaN, infinity, subnormals, and no unsafe
  reassociation/reciprocal transforms;
- atomic shared ownership counts;
- deterministic left-to-right evaluation;
- Dot exception and destructor behavior; and
- required target C ABI mappings.

Compiler extensions such as `__int128` may implement `i128`. Generated ordinary
Dot layout, symbols, and calling convention are private implementation details.
Only `extern "C"` and `#repr("C")` establish target ABI promises.

# cxon — C and C++ builds configured with JSON

cxon is a small build tool for C and C++ projects. Describe your sources, compiler options, and dependencies in `cxon.json`, then run `cxon` to compile and link them.

The project is intended for C++ beginners and people learning how build systems work. It keeps configuration focused on a small set of essential fields.

## Features

- GNU, LLVM/Clang, and MSVC toolchains, with explicit compiler paths and custom toolchain selection.
- Parallel source compilation and timestamp-based object caching.
- Executable, static-library, shared-library, and object-library target types.
- Recursive modules built in dependency order, with library outputs passed to their parents.
- `compile_commands.json` export.
- JSON Schema integration for editor completion and validation.

## Installation

Install from crates.io with Cargo:

```sh
cargo install cxon
```

Prebuilt packages are available on the [GitHub releases page](https://github.com/CoraBlack/cxon/releases).

To install the current checkout, including the custom toolchain support documented below:

```sh
cargo install --path .
```

You also need a C/C++ toolchain. By default, cxon looks for its tools on `PATH`; explicit compiler paths can be configured as described below.

## Quick start

Create `main.cpp`:

```cpp
#include <iostream>

int main() {
    std::cout << "Hello from cxon!\n";
}
```

Create `cxon.json` in the same directory:

```json
{
    "project": "hello",
    "toolchain": "gnu",
    "sources": ["./main.cpp"]
}
```

Run cxon from that directory:

```sh
cxon
```

You can also pass a project directory or a path to its configuration:

```sh
cxon ./path/to/project
cxon ./path/to/project/cxon.json
```

Intermediate files go into `build/`, and the executable goes into `output/` by default. Use `toolchain: "llvm"` for Clang or `toolchain: "msvc"` for MSVC, with flags appropriate to that compiler.

## Configuration

`cxon.json` must be valid JSON: comments and trailing commas are not supported. Relative paths are resolved from the directory containing that configuration file.

This example builds a GNU C++ executable and exports its compilation commands:

```json
{
    "$schema": "https://corablack.github.io/cxon_schema/cxon.schema.json",
    "project": "HelloWorld",
    "target_name": "hello",
    "target_type": "executable",
    "toolchain": "gnu",
    "build_dir": "build",
    "output_dir": "bin",
    "threads": 4,
    "flags": ["-Wall", "-Wextra"],
    "cxxflags": ["-std=c++17"],
    "sources": ["./main.cpp"],
    "export_compile_commands": true,
    "export_compile_commands_path": "./build"
}
```

### Project and build settings

| Field | Description | Default |
| --- | --- | --- |
| `project` | Project name. Required. | — |
| `toolchain` | `gnu`, `llvm`, `msvc`, or `custom`. Required. | — |
| `target_name` | Output name before the toolchain's extension is applied. | `project` |
| `target_type` | `executable`, `static_lib`, `shared_lib`, or `object_lib`. | `executable` |
| `sources` | Array of source file paths. List files explicitly. | None |
| `modules` | Array of child module paths or objects with a `path` field. | None |
| `build_dir` | Directory for intermediate object files. | `./build` |
| `output_dir` | Directory for the final output. | `./output` |
| `threads` | Number of compilation workers. Use a positive integer. | Logical CPU count minus one, with a minimum of one |
| `export_compile_commands` | Export compilation commands after the root project builds. | `false` |
| `export_compile_commands_path` | Directory for `compile_commands.json`. | `build_dir` |

Each configuration must contain a nonempty `sources` or `modules` array. Target types and toolchain names are case-insensitive.

### Compiler and linker settings

| Field | Description |
| --- | --- |
| `flags` | Compiler arguments shared by C and C++ sources. |
| `cflags` | Additional arguments for C sources. |
| `cxxflags` | Additional arguments for C++ sources. |
| `defines` | Preprocessor definitions, such as `["DEBUG", "VERSION=1"]`. |
| `include` | Existing header search directories. |
| `link` | Existing library search directories. |
| `libs` | Library names passed with the selected toolchain's library flag. For GNU/LLVM, `["m"]` becomes `-lm`. |
| `cc_prefix` | Prefix for the C compiler name, such as `aarch64-linux-gnu-`. |
| `cxxc_prefix` | Prefix for the C++ compiler name. The field is spelled `cxxc_prefix`. |
| `cc_path` | C compiler executable or a directory containing it. |
| `cxx_path` | C++ compiler executable or a directory containing it. |

Argument fields are arrays of strings. The legacy `cc` and `cxx` fields are not used to select compilers; use `cc_path` and `cxx_path` instead.

### Explicit compiler paths and custom toolchains

For `gnu`, `llvm`, and `msvc`, `cc_path` and `cxx_path` optionally override compiler lookup on `PATH`. For `custom`, both fields are required, even if the project contains only one source language.

For example, with an MSYS2 UCRT64 installation on Windows:

```json
{
    "project": "custom_hello",
    "toolchain": "custom",
    "cc_path": "C:/msys64/ucrt64/bin/gcc.exe",
    "cxx_path": "C:/msys64/ucrt64/bin",
    "cxxflags": ["-std=c++17"],
    "sources": ["./main.cpp"]
}
```

Replace these paths with your installation paths. Relative compiler paths are resolved from the project directory. Windows paths can use forward slashes, as above, or escaped backslashes.

When a path names a directory, cxon searches for the selected toolchain's compiler name. With `custom`, it tries `gcc`, `clang`, `cc`, then `cl` for C, and `g++`, `clang++`, `c++`, then `cl` for C++. Configured prefixes apply to these names; Windows lookup also checks `.exe` files. An invalid explicit path produces an error instead of falling back to `PATH`.

Custom toolchains reuse an existing command-line convention based on the resolved C++ compiler filename: names containing `clang` use LLVM conventions, `cl` or `cl.exe` uses MSVC, and other names use GNU. This does not provide arbitrary compiler command templates.

When linking uses the C++ compiler, cxon uses the selected `cxx_path`. For a custom toolchain that needs a separate linker or archiver, cxon looks beside the C++ compiler, for example for `ar` when building a GNU/LLVM static library. The selected tool's directory is prepended to the child process's `PATH` so companion tools and runtime libraries can be found.

### Modules

Each module has its own `cxon.json`. Add dependencies using either supported form:

```json
{
    "project": "app",
    "toolchain": "gnu",
    "sources": ["./main.cpp"],
    "modules": [
        "./modules/core",
        { "path": "./modules/utils" }
    ]
}
```

Module paths are relative to the declaring configuration. Dependencies build before their parents, and direct child static-library, shared-library, and object-library outputs become parent linker inputs. Executable outputs are not linked into parents.

All modules in a tree must use the same `toolchain` value. Each module configures its own sources, flags, include paths, and compiler paths. Cyclic dependencies are rejected.

### Editor integration and caching

Add the `$schema` field shown above to enable schema-aware editor support. The schema is maintained in the separate [cxon_schema repository](https://github.com/CoraBlack/cxon_schema).

Set `export_compile_commands` on the root project to export commands collected during the build, including module compilations. The export path is a directory, not a filename.

The object cache checks source and object timestamps. Header changes, compiler changes, and flag changes do not automatically invalidate cached objects. Remove the relevant project's `build_dir` contents before rebuilding after those changes. Cached sources also do not contribute commands to the current export, so use a clean build when you need a complete `compile_commands.json`.

## Examples

Run the included examples from the repository root:

```sh
cargo run -- ./example/hello_world
cargo run -- ./example/static_lib
cargo run -- ./example/shared_lib
```

The [custom toolchain example](example/custom-toolchain) compiles C and C++ together; adjust its compiler paths before running `cargo run -- ./example/custom-toolchain`. The [snake example](example/snake) demonstrates multiple modules and requires SDL3 headers and libraries.

## Development

```sh
cargo build
cargo test
cargo fmt --all -- --check
cargo clippy --all-targets --all-features -- -D warnings
cargo build --release
```

Multiple targets within a single configuration and platform-specific configuration remain future work.

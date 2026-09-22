For multi-file projects, it is recommended to use a YAML configuration file to manage project build options. The configuration file is usually named `project.yml`, but any `*.yml` filename can be used, for example `myproject.yml`; simply pass the corresponding path via a command-line argument when compiling.

Command-line arguments have the highest priority, followed by YAML configuration, and finally default values.

## Complete example

```yaml
# Output file path (equivalent to -o/--output)
output: build/myapp

# If you only need to specify the artifact name, you can also use name
# name does not change the output directory
# name: myapp

# Build mode: bin (executable) / lib (dynamic library) / ext (PHP extension)
# Under WASM, you can also use library to generate a Component with a WIT export interface
mode: bin

# Optimization level, parallel jobs, and debug switch
optimize: 2
job: 8
debug: false

# Source list (files or directories)
sources:
  - src/
  - lib/utils.php
  - native/helper.cpp
  - if: PHP_VERSION_ID >= 80400
    path: compat/php84.php
  - if: PHP_VERSION >= "8.4.0"
    path: compat/php84-version.php

# Ignored files or directories
ignore:
  - src/tests/
  - src/vendor/

# Package dependencies and read-only resources into the release executable
embedded-files:
  - vendor/
  - resources/

# Build directory (relative paths are resolved relative to the directory containing this YAML file)
build-dir: build/out

# C++ standard version
cxx-std: c++20

# Target CPU / cross-compilation platform
march: native
target-platform: aarch64-linux-gnu

# WASI 0.2 output: only component or browser is allowed
# wasm: component
# wasm-browser-dir: generated
# library mode can configure the WIT identifier
# wasm-package: app:calculator@1.0.0
# wasm-world: calculator

# Custom C++ compiler (gcc/g++/clang/clang++, etc.)
cpp-compiler: clang++

# Additional C++ compilation options
cxx-flags:
  - -fvisibility=hidden
  - -Wno-unused-parameter

# Additional linker options
ld-flags:
  - -lcurl
  - -lssl

# Additional C++ header search paths (equivalent to -I)
include-paths:
  - /opt/mylib/include
  - ../shared/headers

# Preprocessor macro definitions (equivalent to -D)
defines:
  - ENABLE_LOGGING=1
  - DEBUG_LEVEL=3

# Runtime / build behavior
sanitize: address
profile: false
no-literal-strings: false
no-progress: true
no-console: false
dry: false
lto: true
format: true

# Libraries to link (equivalent to -l)
link-libs:
  - curl
  - ssl

# Library search paths (equivalent to -L)
link-paths:
  - /usr/local/lib
  - /opt/custom/lib

# Runtime PHP extension dependencies (written into the Zend module dependency table)
# Can be abbreviated as ext-deps, but the two names cannot appear at the same time
extension-dependencies:
  - pdo_mysql
  - curl

# Windows resource configuration (Windows platform only)
resource:
  icon: assets/app.ico
```

## Configuration option reference

| Option | Type | Description |
|--------|------|------|
| `output` | `string` | Output file path, equivalent to `-o` / `--output`. Can include a directory part, for example `build/myapp` |
| `name` | `string` | Only sets the output filename, does not change the output directory. For example, `name: myapp` generates `./myapp` in the current working directory |
| `mode` / `build-mode` / `type` | `string` | Build mode, equivalent to `-m`. Host builds support `bin`/`binary`/`cli`, `lib`/`library`/`shared`, and `ext`/`extension`; WASM builds use the command mode or the `lib`/`library`/`reactor` library component mode |
| `optimize` | `integer` | Optimization level, equivalent to `-O`, range `0` ~ `3` |
| `job` | `integer` | Number of parallel compile jobs, equivalent to `-j` / `--job` |
| `debug` | `boolean` | Enables debug mode, equivalent to `-d` / `--debug` |
| `profile` | `boolean` | Enables CPU profiling, equivalent to `--profile`. Linux only |
| `no-literal-strings` | `boolean` | Disables string literal optimization, equivalent to `--no-literal-strings` |
| `no-progress` | `boolean` | Disables progress bar output, equivalent to `--no-progress` |
| `no-console` | `boolean` | Hides the console window, equivalent to `--no-console`. Windows only |
| `sanitize` | `string` | Enables a sanitizer, equivalent to `--sanitize`, e.g. `address`, `undefined` |
| `sources` | `array` | List of source files or directories. Supports `.php`, `.cpp`, `.c`, `.s`, `.m`, `.mm`. Can use `if` + `path` to load conditionally by PHP version or operating system |
| `ignore` | `array` | Files or directories to exclude. Each entry must be an explicit path — **wildcards and regular expressions are not supported**. A directory entry excludes everything below it; entries that do not exist are silently skipped |
| `embedded-files` | `array` | Files or directories recursively packaged into a `bin` executable. PHP files not successfully AOT-compiled from `sources` receive build-time opcodes and are executed lazily by ZendVM at runtime |
| `build-dir` | `string` | Build directory, equivalent to `--build-dir`. Supports relative and absolute paths; relative paths are resolved relative to the directory containing the current YAML file |
| `dry` | `boolean` | Dry-run mode, equivalent to `--dry` |
| `cxx-std` | `string` | C++ standard version, equivalent to `--cxx-std` |
| `march` | `string` | Target CPU instruction set (e.g. `native`, `x86-64-v3`), equivalent to `--march` |
| `target-platform` | `string` | Cross-compilation target platform triple, equivalent to `--target-platform` |
| `wasm` | `string` | Enables WASI 0.2 builds. Only `component` or `browser` is allowed; boolean values are not accepted |
| `wasm-browser-dir` | `string` | Output directory for Jco browser modules when `wasm: browser`; relative paths are resolved relative to the directory containing the YAML file |
| `wasm-package` | `string` | WIT package for a library component, in the format `namespace:name@major.minor.patch`; defaults to a value generated from the project name |
| `wasm-world` | `string` | WIT world name for a library component, using lowercase hyphenated naming; defaults to the project name |
| `cpp-compiler` | `string` | Custom C++ compiler (e.g. `g++`, `clang++`) |
| `cxx-flags` | `string` or `array` | Additional C++ compilation options, appended to the compile command |
| `ld-flags` | `string` or `array` | Additional linker options, appended to the link command |
| `include-paths` | `array` | Additional C++ header search directories, equivalent to `-I` |
| `defines` | `array` | Preprocessor macro definitions, equivalent to `-D`. Each entry is in the `name=value` format |
| `lto` | `boolean` | Enables link-time optimization, equivalent to `--lto` |
| `format` | `boolean` | Enables clang-format formatting, equivalent to `--format` |
| `link-libs` | `array` | Libraries to link, equivalent to `-l`. Each entry is a library name (without the `lib` prefix and the `.so`/`.a` suffix) |
| `link-paths` | `array` | Library search paths, equivalent to `-L`. Each entry is a directory path |
| `extension-dependencies` / `ext-deps` | `array` | Required PHP extension module names. `ext-deps` is an abbreviated alias; the two cannot appear at the same time. The compiler writes them into `zend_module_entry.deps`, which Zend checks when loading the TypePHP module |
| `resource` | `object` | Windows platform resource configuration (icons, etc.) |

## Embedding PHP dependencies and runtime resources

`embedded-files` packages Composer `vendor`, PHP files that AOT does not yet support, and read-only resources such as configuration and templates into the executable. The release host does not need to run `composer install` or carry a disk `vendor` tree. The build host needs PHP CLI and OPcache matching the target PHP; the runtime does not depend on OPcache.

```yaml
mode: bin
sources:
  - main.php
  - src

embedded-files:
  - vendor
  - resources
```

Composer autoload is used normally. `vendor/autoload.php` and later class files are loaded lazily from the executable's memory tables. See [Embedding Dependencies and Resources in an Executable](embedded-files.md) for the relationship among `sources`, `ignore`, and `embedded-files`, separate development and release configurations, caching, performance, and complete limitations.

## PHP extension dependencies

When a program depends on PHP extensions such as `pdo_mysql` or `curl`, you can declare them via `extension-dependencies`:

```yaml
name: database_app
mode: ext
sources:
  - src

extension-dependencies:
  - pdo_mysql
  - curl
```

You can also use the fully equivalent abbreviated name:

```yaml
ext-deps:
  - pdo_mysql
  - curl
```

A single project can use only one of the two names. The following configuration is invalid; the compiler reports an error immediately when reading the YAML:

```yaml
extension-dependencies:
  - pdo_mysql
ext-deps:
  - curl
```

The compiler builds the required dependency table in the generated Zend module, equivalent to generating `ZEND_MOD_REQUIRED` for each entry. When the TypePHP extension is loaded, Zend first confirms that these modules are already loaded; when a required module is missing, the TypePHP module will not start normally. Duplicate names are deduplicated, and surrounding whitespace is removed from names.

Both `extension-dependencies` and `ext-deps` must be arrays, and each entry must be a non-empty string. All dependencies in the current configuration are required; optional, conflicts, or version-relation declarations are not supported.

This configuration applies to `bin`, `lib`, and `ext` builds on the Host platform: all three modes generate a TypePHP Zend module entry, where `ext` is loaded directly by PHP and `bin`/`lib` are registered by the embedded runtime. The WASM Runtime's extension set is fixed when the runtime component is built, and cannot dynamically add missing PHP extensions via this configuration.

What you fill in here are **PHP module names**, usually the same names used by `extension_loaded()`, for example:

```php
extension_loaded('pdo_mysql');
extension_loaded('curl');
```

Do not fill in operating system package names, dynamic library filenames, or linker options; for example, `php8.4-curl`, `libcurl.so`, and `-lcurl` are not PHP module names.

`extension-dependencies` only declares the load order and runtime required relationships; it does not install, enable, or statically link extensions. During deployment, you still need to load the corresponding extensions in advance via `php.ini` or SAPI configuration:

```ini
extension=pdo_mysql
extension=curl
extension=database_app
```

`extension-dependencies` and native C/C++ linking configuration are two independent mechanisms:

| Configuration | Target | Stage | Example |
|------|----------|----------|------|
| `extension-dependencies` | PHP/Zend extension modules | Zend module loading and startup | `pdo_mysql`, `curl` |
| `link-libs` | native link libraries | C/C++ linking | `curl`, `ssl` |
| `link-paths` | native library search directories | C/C++ linking | `/usr/local/lib` |

If a project both directly calls PHP's `curl` extension and links libcurl in custom C++ source code, you may need to configure both `extension-dependencies: [curl]` and `link-libs: [curl]`; the two do not substitute for each other.

## WASM projects

A project run with Wasmtime is configured as a Component:

```yaml
name: hello
mode: bin
wasm: component
sources:
  - src
build-dir: build
output: dist/hello.wasm
```

To use the browser runtime, explicitly select the browser profile:

```yaml
name: browser-app
mode: bin
wasm: browser
sources:
  - src
output: component/browser-app.wasm
wasm-browser-dir: generated
```

To call TypePHP functions multiple times from JavaScript or another Component Host, use library mode:

```yaml
name: calculator
mode: library
wasm: browser
wasm-package: app:calculator@1.0.0
wasm-world: calculator

sources:
  - src

output: component/calculator.wasm
wasm-browser-dir: generated
```

Library mode requires at least one [`#[WasmExport]`](wasm-export.md) function and does not use `main()` as the automatic entry point.

`wasm` is not a boolean switch; both `wasm: true` and `wasm: false` are invalid configurations. WASM projects default to `wasm32-wasip2` when `target-platform` is omitted. See [Compiling to WebAssembly](wasm.md) for detailed toolchain installation, running methods, browser Hosts, and platform limitations.

## The difference between name and output

Both `name` and `output` can affect the final artifact name, but their semantics differ:

- `name` only sets the artifact filename, not the output directory.
- `output` sets the full output path, equivalent to the command-line `-o` / `--output`.
- `name` is not resolved relative to the directory containing the YAML file; the artifact is generated in the current working directory where the compile command is run by default.
- If `output` includes a directory part, it sets both the output directory and the filename; relative paths are resolved relative to the directory containing the YAML file.
- When both `output` and `name` are configured, `output` takes precedence.

For example:

```yaml
name: tetris
sources:
  - main.php
```

Run:

```bash
./tpc examples/tetris-sdl/project.yml
```

This generates:

```text
./tetris
```

instead of:

```text
examples/tetris-sdl/tetris
```

If you want to output explicitly to the YAML directory, use `output`:

```yaml
output: tetris
sources:
  - main.php
```

## Priority

Command-line arguments > YAML configuration > default values.

For example, if the YAML sets `cxx-std: c++17` but the command line passes `--cxx-std=c++20`, the final value is `c++20`.

## Conditional sources

`sources` supports conditional loading. The string form remains compatible; use `if` and `path` when a condition is needed. `if` and `path` are mapping fields of the same source, order-independent; it is recommended to write `if` first so the load condition is seen first.

```yaml
sources:
  - main.php
  - if: PHP_VERSION_ID >= 80400
    path: compat/php84.php
  - if: PHP_VERSION >= "8.4.0"
    path: compat/php84-version.php
  - if: PHP_VERSION_ID >= 80200 && PHP_VERSION_ID < 80400
    path: compat/php82.php
  - if: PHP_OS_FAMILY == "Linux"
    path: platform/linux.php
```

The condition left-hand side supports only three constants: `PHP_VERSION_ID`, `PHP_VERSION`, and `PHP_OS_FAMILY`. Multiple conditions can be combined using `&&`, `||`, `!`, and parentheses.
When the condition is `false`, that `source` is skipped, and skipped entries do not require the file to exist. This capability applies only to `YAML` configuration files; there is no corresponding command-line argument.

### PHP_VERSION_ID
The condition right-hand side must be a valid PHP version ID number. `PHP_VERSION_ID` is first converted to a `PHP_VERSION` string for comparison, for example `80400` becomes `8.4.0`.

### PHP_VERSION
The condition right-hand side must be a valid PHP version number string. Supported operators: `<`, `lt`, `<=`, `le`, `>`, `gt`, `>=`, `ge`, `==`, `=`, `eq`, `!=`, `<>`, `ne`.
Internally uses `version_compare()`.

### PHP_OS_FAMILY
The condition right-hand side must be a valid operating system family name string: `Windows`, `BSD`, `Darwin`, `Solaris`, `Linux`, `Unknown`. Supported operators: only `==` and `!=`.

## Not using a YAML configuration file

You can also compile a single PHP file or directory directly, without creating a YAML configuration file:

```shell
# Compile a single file
./tpc hello.php -O2

# Use a custom YAML filename
./tpc myproject.yml -O2

# Compile an entire directory
./tpc myproject/ -O2 -o myapp
```

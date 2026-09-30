## Nano Native Compilation

Nano mode produces TypePHP native programs or shared libraries without carrying the PHP
interpreter or the Zend VM. PHP still runs the compiler and Composer still
installs build-time dependencies; neither becomes a runtime dependency of the
resulting program.

For Linux, macOS, iOS, and Android targets, TypePHP reads source manifests from
Composer packages and compiles PHP Nano, PHPX, and the generated C++ together.
The resulting artifact does not link `libphp` and cannot interpret or load PHP
source code at runtime.

### How it differs from regular mode

| Item | Regular mode | Nano mode (outside Windows) |
|---|---|---|
| PHP runtime | Links the complete `libphp` | Compiles PHP Nano sources into the application |
| Zend VM | Available through the full runtime | Not included |
| PHPX | Links a PHPX library | Compiles PHPX sources with `PHPX_NANO` |
| Extensions | Supplied by PHP configuration and dynamic libraries | Statically selected by Composer at build time |
| Host capabilities | Determined by the complete PHP build | Restricted to a local, portable capability set |
| Runtime dependencies | PHP/PHPX shared or static libraries | C/C++ standard libraries, POSIX, and toolchain runtime |

Nano does not reimplement PHP's fundamental data structures. It directly
reuses the PHP 8.6 sources for `zval`, `zend_string`, `zend_array`, objects,
exceptions, GC, function tables, and built-in modules, while removing the
interpreter and capabilities outside Nano's boundary. Ordinary classes,
inheritance, interfaces, exceptions, closures, and synchronous dynamic object
method calls therefore remain available.

### Installation

Install TypePHP and PHP Nano in the project. `swoole/typephp` already depends
on `swoole/phpx`, so PHPX does not need to be declared again:

```shell
composer require --dev swoole/typephp "swoole/php-nano:^1.0.2"
```

Source composition for `mode: lib`, Android, iOS, and macOS requires PHP Nano
1.0.2 or later.

Production Nano builds do not use CMake. TypePHP reads the Composer metadata
published by `swoole/php-nano` and `swoole/phpx`, then adds the listed `.c`,
`.cc`, and `.cpp` files to the application sources. C files are compiled as
C11, while PHPX and generated TypePHP code are compiled as C++17.

The PHP executable that starts `vendor/bin/tpc.php` must satisfy TypePHP's
version requirements. It runs only at build time. Nano's reuse of PHP 8.6
source code does not require PHP 8.6 to be installed on the target device.

### Compiling the first executable

Create `hello.php`:

```php
<?php

function main(int $argc, array $argv): void
{
    echo "Hello Nano\n";
    var_dump($argc, $argv);
}
```

Compile and run it:

```shell
vendor/bin/tpc.php --nano hello.php
./hello first second
```

The executable is written to the directory in which the compile command was
started, so this example produces `./hello`. Intermediate files use the normal
`build` directory. Regular compilation options continue to apply:

```shell
vendor/bin/tpc.php --nano hello.php -O2 -j8 -o build/hello
vendor/bin/tpc.php --nano hello.php --run -- first second
```

For release-size guidance on `strip`, source selection, and optimization
levels, see [reducing executable size](reduce-binary-size.md).

See [environment variables](environment-variables.md) for the difference
between Nano and the full PHP runtime, including `PHPRC`.

Nano and regular mode share argument parsing, code generation, parallel jobs,
progress reporting, the object cache, and output path rules. The first build
compiles many PHP Nano and PHPX sources and is substantially slower than an
incremental application build. Later builds reuse valid objects in `build`.
Use `--force` to ignore the cache or `--no-progress` to replace the progress bar
with one line per source file.

Single files, source directories, and `project.yml` inputs remain available.
Outside Windows, Nano supports two artifact types:

- `mode: bin` produces an executable with an operating-system `main` entry and
  invokes the global TypePHP `main()` function;
- `mode: lib` produces a shared library for an Android JNI, iOS/macOS, or other
  native host and does not generate an operating-system `main`.

`mode: ext` produces an extension for a complete PHP runtime and conflicts with
Nano's self-contained runtime model, so it is not supported. `bin` and `lib`
compile the same Nano runtime and capability set; the essential difference is
whether the artifact or its native host supplies `main`. Both additionally require:

- use C++17;
- do not combine it with `--full-static`;
- do not use `-l`, `-L`, or the corresponding external link-library settings.

PHP-bundled sources such as PCRE2, timelib, and libbcmath are compiled directly
into PHP Nano and are not additional dynamic link dependencies.

### Shared libraries and native hosts

Select `mode: lib` in YAML and enable Nano on the command line:

```yaml
name: my_runtime
mode: lib
output: build/libmy_runtime.so
cxx-std: c++17

sources:
  - src
  - bridge
```

```shell
vendor/bin/tpc.php project.yml --nano
```

A library project does not need a global `main()`. TypePHP exports a pair of C
ABI lifecycle functions derived from the project name. A native bridge can
include `typephp_runtime.h` and refer to them through
`TYPEPHP_RUNTIME_INIT(my_runtime)` and `TYPEPHP_RUNTIME_SHUTDOWN(my_runtime)`.
The initializer accepts `argc`/`argv` and returns `0` on success. The host must
not call generated TypePHP functions before initialization succeeds.

```cpp
#include <typephp_runtime.h>

TYPEPHP_RUNTIME_INIT_FUNCTION(my_runtime);
TYPEPHP_RUNTIME_SHUTDOWN_FUNCTION(my_runtime);

if (TYPEPHP_RUNTIME_INIT(my_runtime)(argc, argv) != 0) {
    // Initialization failed; do not enter TypePHP callbacks.
}

// At the end of the host lifetime, after all TypePHP callbacks have stopped:
TYPEPHP_RUNTIME_SHUTDOWN(my_runtime)();
```

Initialize once per process or native-library lifetime, not before every UI
event or business call. Invoke shutdown only after the host can no longer enter
any TypePHP callback; a mobile application whose process is reclaimed directly
by the operating system may keep the runtime alive until process exit. Nano
runs MINIT/RINIT once during initialization and RSHUTDOWN/MSHUTDOWN once during
shutdown, so it does not introduce a PHP web request's repeated lifecycle loop.

### Available capabilities

Nano includes restricted subsets of Core, date, hash, json, pcre, random,
Reflection, SPL, standard, and filter. The main retained facilities are:

- strings, arrays, objects, resources, exceptions, and garbage collection;
- ordinary classes, inheritance, interfaces, closures, and synchronous dynamic calls;
- output buffering and console I/O;
- local `file` and `glob` streams, plus file, directory, stat, and hash-file operations;
- date/time, clocks, sleeping, and random values;
- serialization, JSON, regular expressions, and commonly used standard functions;
- the Zend INI registry and module INI values, without scanning or loading `php.ini`;
- TypePHP `BigInt`, `Decimal`, and `BigFloat`.

The `ctype_*` functions are TypePHP/PHPX compiler intrinsics and do not require
a registered ctype extension.

Nano high-precision types use the PHP 8.6 bundled libbcmath, whereas regular
mode uses GMP, mpdecimal, and MPFR. The APIs remain available, but the precision
model, rounding, extreme ranges, formatting, and performance are not guaranteed
to match digit for digit. See
[Arbitrary Precision Math](math.md#high-precision-backend-in-nano-mode).

### Unsupported capabilities

On every platform, `--nano` excludes the following capabilities. Their syntax
forms and statically known direct calls are rejected at compile time:

- `eval`, `include`, `include_once`, `require`, and `require_once`;
- anonymous classes;
- `yield`, `yield from`, Generator, and Fiber;
- backtick execution and process/command APIs such as `shell_exec`, `exec`,
  `system`, `passthru`, `popen`, `proc_open`, and `fork`;
- sockets, DNS, network clients and servers, and remote streams;
- `dl`, dynamic extension loading, and runtime extension discovery;
- signals, execution timers, virtual-memory mapping, system logging, and other
  host-control capabilities.

Unsupported public functions and classes are normally absent from Nano's
function table. TypePHP reports statically known direct calls before generating
C++. Local file access does not imply remote URLs, process pipes, or
user-defined stream wrappers.

### Static Composer extensions

Nano retains the Zend extension lifecycle but has no runtime dynamic loader.
Additional extensions must be Composer packages named `swoole/php-ext-*` and
must publish exact sources, include directories, ABI information, and their
`zend_module_entry` through `extra.typephp-native`.

After such a package is installed, TypePHP discovers it at build time, adds its
sources to the application, and generates a fixed extension registry. The
original MINIT, RINIT, RSHUTDOWN, and MSHUTDOWN lifecycle remains in use, but
the extension set cannot change after the application is built. Extensions
must still satisfy Nano's dependency and capability boundary.

Here, “extension” means a Composer native package statically compiled into the
Nano runtime, not a `mode: ext` artifact. Nano still cannot produce an extension
for dynamic loading into an external PHP runtime.

### WASI

Nano can be combined with a WASI target:

```shell
vendor/bin/tpc.php --nano --wasm hello.php
wasmtime hello.wasm first second
```

The Component and browser profiles can also be selected explicitly:

```shell
vendor/bin/tpc.php --nano --wasm=component hello.php
vendor/bin/tpc.php --nano --wasm=browser hello.php
```

WASI is a smaller subset of Native Nano. It retains only console I/O, clocks,
entropy, arguments, and access to preopened files and directories that the
target runtime can represent. Native filesystem operations that WASI cannot
represent are removed or rejected. See [Compiling to WebAssembly](wasm.md) for
the WASI SDK, Wasmtime, browser output, and directory-permission setup.

### Platform differences

- **Linux and macOS:** PHP Nano and PHPX sources are compiled directly into the
  final artifact.
- **iOS and Android:** `bin`/`lib` compile PHP Nano and PHPX sources directly
  with the corresponding SDK/NDK toolchain and do not require prebuilt
  `libphp.a` or `libphpx.a`. A complete application still needs its platform
  SDK, native entry point, and UI bridge. See [Native Applications](mobile-native.md).
- **Windows:** the PHP Nano source-composition backend is not used. `--nano`
  still rejects dynamic PHP, external commands, and other Nano capabilities,
  but the program continues to link the complete `php.dll` and `phpx.dll`
  through import libraries. A Windows Nano artifact is therefore not a
  PHP-DLL-free program.

To check whether a final artifact unexpectedly links PHP or another shared
library, use the target platform's binary dependency inspection tools and
compare the result with the sources and link inputs shown in the build log.

### Common errors

- **`swoole/php-nano` is not installed:** run
  `composer require --dev "swoole/php-nano:^1.0.2"` in the current project and use that
  project's `vendor/bin/tpc.php`.
- **Native ABI mismatch:** update or downgrade `swoole/typephp`, `swoole/phpx`,
  and `swoole/php-nano` so Composer resolves compatible versions. Do not mix a
  different project's `vendor` directory into the build.
- **`ext` mode is not supported:** use `-m bin` for an executable or `-m lib`
  for a native-hosted library. Nano cannot produce a dynamic PHP extension.
- **The first build appears slow:** the complete Nano/PHPX source set is being
  compiled. Keep the `build` directory for subsequent cached builds, or use
  `--no-progress` to observe each source file.
- **A function works in regular mode but is rejected by Nano:** it depends on
  networking, process control, SAPI, or another removed host capability. Use a
  retained local API or select regular mode.

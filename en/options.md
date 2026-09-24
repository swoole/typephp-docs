## Compilation Options

The TypePHP compiler supports specifying compilation options via command-line arguments and a [YAML configuration file](project-yml.md). Command-line arguments have the highest priority, followed by YAML configuration, and finally default values.

### Usage

```shell
./tpc <file/dir/config.yml> [options]
```

## Command-line arguments

### `-O <level>` — optimization level

- Range: `0` ~ `3`, default `0`
- `-O0`: no optimization, fastest compile speed, suitable for debugging
- `-O2`: the commonly used release optimization level, balancing compile time and runtime performance
- `-O3`: the highest optimization level, with aggressive function inlining and vectorization

```shell
./tpc app.php -O2
```

### `-o, --output <file>` — output file path

Uses the base name of the entry file or directory by default. Building a binary generates an executable file; building an extension generates a `.so`/`.dll`.

`--output` can specify just a filename or include a directory. The `output` field in YAML configuration is equivalent to this argument; `name` only sets the filename and does not change the output directory.

```shell
./tpc src/ -o myapp
./tpc src/ -o build/myapp
```

### `-m, --mode <mode>` — build mode

- `bin` (default): compile into a standalone executable
- `lib`: compile into a dynamic library that can be linked by other TypePHP projects
- `ext`: compile into a PHP extension (`.so`/`.dll`)

```shell
./tpc myext/ -m ext -o myext
./tpc project.yml -m lib
```

### `--sapi <embed|cli|fpm>` — PHP SAPI

Selects the PHP SAPI used by a `mode: bin` executable. The default is `embed`.
Separate multiple targets with commas:

```shell
./tpc project.yml --sapi=embed,cli,fpm --php-builder
```

`cli` and `fpm` are not modes and always require `php-builder`. Multiple targets
produce separate programs with `-embed`, `-cli`, and `-fpm` suffixes.

### `--entry <file>` — CLI entry script

Selects the embedded PHP primary script executed by Zend VM when the CLI SAPI
starts. It is required whenever `sapi` contains `cli` and cannot be used without
selecting `cli`.

```shell
./tpc src/functions.php --sapi=cli --entry=bin/app.php --php-builder
```

### `--php-builder[=<config>]` — build PHP from php-src

Enables a private static PHP build that does not depend on the host PHP runtime.
Without a value, it uses the default `{}` configuration:

```shell
./tpc hello.php --php-builder
```

An explicit configuration uses YAML field names separated by semicolons. Quote
the complete value:

```shell
./tpc project.yml \
    --sapi=cli \
    --entry=bin/app.php \
    --php-builder='extensions: [swoole, mongodb]; zts: on'
```

`sapi` is not a `php-builder` field and must always use the independent `--sapi`
option. See [PHP Builder and SAPI Targets](php-builder.md) for platform limits,
caching, extension collection, and the complete semantics.

### `--proxy <url>` — network proxy

Sets an HTTP(S) or SOCKS proxy for PHP downloads, PECL extensions, and other
network operations. It is a global option, not a `php-builder` field:

```shell
./tpc project.yml --proxy=socks5h://127.0.0.1:1080 --php-builder
```

### `-d, --debug` — debug mode

Automatically disables optimization and adds debug symbols (`-g`), making it easier to debug the generated binary with GDB/LLDB.

```shell
./tpc app.php -d
```

### `--sanitize <type>` — Sanitizer

Enables compiler sanitizers in the generated C++ code, for detecting memory errors and undefined behavior.

- `address` — AddressSanitizer (out-of-bounds access, use-after-free, etc.)
- `undefined` — UndefinedBehaviorSanitizer (integer overflow, null pointers, etc.)

```shell
./tpc app.php --sanitize=address
```

### `--profile` — CPU profiling

Based on gperftools (Linux only). Inserts profiling probes into the generated binary and automatically links `-lprofiler`; after running, it generates a `{target}.prof` data file. When enabled, it automatically forces recompilation of the misc files to ensure the compile macro takes effect.

```shell
./tpc app.php --profile
./app                                    # generates app.prof after running
```

Use the compiler's built-in pprof analysis (default web mode, automatically opens the browser):

```shell
./tpc app.prof
```

### `-j, --job <num>` — number of parallel compile jobs

Controls the number of C++ files compiled simultaneously, similar to `make -j`. The default is `4`. On machines with more CPU cores, this can be increased as appropriate.

```shell
./tpc app.php -j8
```

### `--no-literal-strings` — disable string literal optimization

By default, the compiler optimizes literal strings to `const char*` pointers to improve performance. In some scenarios (such as when `zend_string` compatibility is required), this optimization may need to be disabled.

```shell
./tpc app.php --no-literal-strings
```

### `-I <dir>, --include-path <dir>` — C++ header search path (repeatable)

Appends the specified directory to the end of the C++ compiler's include search path. The system default paths (PHP header files, PHPX header files) always take priority, with user paths appended after them.

Can be specified multiple times to add multiple directories:

```shell
# Add multiple include directories
./tpc app.php -I /opt/mylib/include -I ../shared/headers -O2

# Equivalent long form
./tpc app.php --include-path /opt/mylib/include --include-path ../shared/headers
```

### `-D <macro>, --define <macro>` — C++ preprocessor macro (repeatable)

Equivalent to using `#define` in C++ code, for conditional compilation, feature toggles, and similar scenarios. Uses the `name=value` format; the value after `=` is optional.

The compiler automatically adapts: GCC/Clang use `-D<macro>`, MSVC uses `/D<macro>`.

```shell
# Define macros to control feature toggles
./tpc app.php -D ENABLE_LOGGING=1 -D DEBUG_LEVEL=3 -O2

# Equivalent long form
./tpc app.php --define ENABLE_LOGGING=1 --define DEBUG_LEVEL=3
```

### `--march <arch>` — target CPU instruction set

Specifies the target CPU instruction set for the generated code, equivalent to GCC's `-march=<arch>`.

Common values:
  - `native` — auto-optimize for the current CPU
  - `x86-64-v3` / `x86-64-v4` — x86-64 microarchitecture levels
  - `armv8-a` / `armv9-a` — ARM architectures

```shell
# Optimize for the current machine
./tpc app.php --march=native -O2

# Specify the target architecture
./tpc app.php --march=x86-64-v3 -O2
```

> Applies only to GCC/Clang; use `--cxx-flags` with MSVC.

### `--target-platform <triple>` — cross-compilation target platform

Cross-compiles, appending the `--target=<triple>` flag to the compile and link commands. Suitable for generating binaries for a different architecture/platform than the current host.

Common target triple examples:
  - `aarch64-linux-gnu` — ARM64 Linux
  - `aarch64-linux-android24` — Android API 24+, arm64-v8a
  - `arm64-apple-ios15.0` — iPhoneOS arm64, iOS 15+
  - `x86_64-w64-mingw32` — Windows x86-64 (MinGW)
  - `arm-linux-gnueabihf` — ARM32 Linux (hard float)

```shell
# Compile an ARM64 Linux binary on an x86-64 host
./tpc app.php --target-platform aarch64-linux-gnu -O2

# Cross-compile to Windows (MinGW)
./tpc app.php --target-platform x86_64-w64-mingw32 -O2
```

> **Note**: GCC cross-compilation usually also requires installing the corresponding cross-compilation toolchain (such as `g++-aarch64-linux-gnu`). Clang has built-in cross-compilation support and only needs `--target`. You can specify the cross-compiler path via the YAML `cpp-compiler` option.

Android and iPhoneOS additionally require their platform SDK, a matching PHPX SDK, sysroot, and linker settings. See [Android, iOS, and macOS Native Applications](mobile-native.md).

### `--nano` — compile a native program without the Zend VM

On Linux, macOS, iOS, and Android targets, this compiles PHP Nano, PHPX, and
the generated code as one C11/C++17 source build. The resulting program does
not link `libphp`. Windows continues to link the complete PHP/PHPX DLLs while
applying the same Nano language and capability restrictions.

```shell
vendor/bin/tpc.php --nano hello.php
./hello
```

Outside Windows, Nano currently supports only `bin` mode, cannot be combined
with `--full-static`, `-l`, or `-L`, and requires C++17. See
[Nano Native Compilation](nano.md) for dependency installation, the supported
API, static extensions, and WASI usage.

### `--wasm[=profile]` — compile to WASI 0.2

Compiles a TypePHP program into a WASI 0.2 Component. Bare `--wasm` defaults to the `component` profile:

```shell
./tpc --wasm hello.php
./tpc --wasm=component hello.php
```

Both forms generate only `hello.wasm` and do not require Jco. Run with Wasmtime:

```shell
wasmtime hello.wasm
```

To generate a browser ESM, use explicitly:

```shell
./tpc --wasm=browser hello.php
```

The browser profile requires `jco` to be in `PATH`. The default WASM mode target is `wasm32-wasip2`; see [Compiling to WebAssembly](wasm.md) for detailed environment requirements, `project.yml` configuration, and running methods.


### `--lto` — Link Time Optimization

Allows the compiler to optimize across translation units during the linking stage; combined with `-O2`/`-O3`, it can further improve runtime performance and reduce binary size. The compiler automatically adapts: GCC/Clang use `-flto`, MSVC uses `/GL` + `/LTCG`.

⚠️ Increases link time; recommended only for release builds.

```shell
# Enable LTO in production
./tpc app.php -O2 --lto
```

### `--format` — clang-format code formatting

When enabled, the compiler invokes `clang-format -i` to auto-format each generated C++ file. Because formatting consumes extra time, it is disabled by default. Requires `clang-format` to be installed on the system.

```shell
# Enable code formatting
./tpc app.php --format
```

### `-l <lib>, --link-lib <lib>` — link library (repeatable)

Specifies a library to link, equivalent to GCC's `-l<lib>`. The actual flag passed to the linker is `-l<lib>`. Can be specified multiple times to link multiple libraries.

```shell
# Link multiple libraries
./tpc app.php -lcurl -lssl -lcrypto -O2

# Equivalent long form
./tpc app.php --link-lib curl --link-lib ssl --link-lib crypto
```

### `-L <dir>, --link-path <dir>` — library search path (repeatable)

Adds a search path for library files, equivalent to GCC's `-L<dir>`. The actual flag passed to the linker is `-L<dir>`. Used to link libraries located in non-standard paths.

```shell
# Add library search paths and link
./tpc app.php -L/usr/local/lib -L/opt/custom/lib -lmycustom -O2

# Equivalent long form
./tpc app.php --link-path /usr/local/lib --link-path /opt/custom/lib
```

### `-f, --force` — force recompilation

Forces recompilation of all files even when a compilation cache exists.

```shell
./tpc app.php -f
```

### `--cxx-std <version>` — C++ standard version

Defaults to `c++17`. If your code or dependent libraries require a specific C++ version, specify it via this option.

```shell
./tpc app.php --cxx-std=c++20
```

### `--build-dir <dir>` — build directory

Sets the directory for build intermediates such as generated C++ code (`.cc`), header files, and object files (`.o`). The default is `<project root>/build`.

```shell
./tpc app.php --build-dir /tmp/mybuild
```

### `--dry` — dry-run mode

Performs only the PHP → C++ transpilation step, generating all C++ source files, but does **not** perform compilation and linking. Suitable for reviewing the generated C++ code or pre-checking the transpilation result.

```shell
./tpc app.php --dry
```

### `--no-console` — hide the console window (Windows only)

On Windows, links with `/SUBSYSTEM:WINDOWS`, so the program does not show a console window at startup; suitable for GUI applications.

```shell
./tpc gui-app.php --no-console
```


### `--no-color` — disable ANSI color output

Disables ANSI color escape sequences in the terminal; all output is plain text. Suitable for log redirection, terminals that do not support color, or CI/CD pipelines.

```shell
./tpc app.php --no-color -O2
```

### `-v, --version` — show version information

### `-h, --help` — show help information

---

> 💡 For multi-file projects, it is recommended to use a [YAML configuration file](project-yml.md) to centrally manage build options. The filename can be `project.yml` or any `*.yml` passed to the compiler.

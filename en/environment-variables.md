# Environment variables

This page lists environment variables used by the current TypePHP compiler, its bundled build scripts, and generated programs at startup. Timing matters: `PHP_HOME` locates a PHP SDK during compilation, while `PHPRC` locates `php.ini` when a full PHP runtime starts. Arbitrary application settings read through `getenv()` are defined by the application, not by TypePHP.

## Common setup

Linux, using a host PHP Embed library to build and run an application:

```bash
export PHP_HOME=/opt/php-8.5
export PHPX_HOME=/opt/phpx
export PATH="$PHP_HOME/bin:$PATH"
export LD_LIBRARY_PATH="$PHP_HOME/lib:$PHPX_HOME/lib${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}"
./tpc project.yml

# Run a full-PHP executable with php.ini from this directory
PHPRC="$PWD/config/php" ./myapp
```

On macOS, `DYLD_LIBRARY_PATH` can sometimes help for command-line launches, but a released `.app` should bundle its native libraries and set their load paths correctly. An app launched from Finder does not inherit a temporary `export` from your shell. Windows PowerShell example:

```powershell
$env:PHP_HOME = 'C:\php'
$env:PHPX_HOME = 'C:\phpx'
$env:PATH = "C:\php;C:\phpx\lib;$env:PATH"
.\tpc.exe project.yml

$env:PHPRC = 'C:\myapp\config\php'
$env:PHP_INI_SCAN_DIR = 'C:\myapp\config\php.d'
.\myapp.exe
```

These paths are examples. `PHP_HOME` and `PHPX_HOME` help the compiler locate SDKs; they do not add libraries to the operating system's loader search path or choose the generated program's `php.ini`.

## Compiler and toolchain

| Variable | Stage | Purpose |
|---|---|---|
| `PHP_HOME` | Build | Prefix of a matching PHP installation, used for `php-config`, headers, the Embed library, and ABI checks. On Linux/macOS, TypePHP can also search the active PHP and `PATH`; on Windows it prefers this variable. During a php-builder build, the private prefix is passed to child processes. |
| `PHPX_HOME` | Build | PHPX source or installation directory. Without it, TypePHP tries the Composer-installed `swoole/phpx`. Nano and cross-compilation also use it to find PHPX sources and target SDKs. |
| `HOME` | Build, Linux/macOS | Private PHP sources, php-builder caches, and automatic installations normally live under `~/.typephp`. |
| `PATH` | Build and run | Locates `php-config`, C/C++ compilers, `curl`, WASI tools, and other executables. On Windows it also helps locate runtime DLLs. |
| `PATHEXT` | Build, Windows | Extensions used when looking up a tool without a filename extension in `PATH`; defaults to `.COM;.EXE;.BAT;.CMD`. |
| `LLVM_HOME` | Build, Windows Clang | If `lld-link` is absent from `PATH` and beside the compiler, TypePHP tries `$LLVM_HOME/bin/lld-link.exe` and `$LLVM_HOME/x64/bin/lld-link.exe`. It does not fall back to MSVC `link.exe`. |
| `INCLUDE`, `LIB`, `LIBPATH` | Build, Windows | MSVC/Windows SDK header and library search paths for Microsoft ABI builds. LLVM builds can set `INCLUDE` and `LIB` directly without running `vcvars64.bat`; see [Windows setup](windows.md). |
| `TYPEPHP_STRICT_CACHE=1` | Build | Enables stricter PCH dependency checks; Windows already enables this by default. The check may take longer. |
| `CC`, `CXX`, `AR` | Build PHP/SDKs | Standard C compiler, C++ compiler, and archiver settings for lower-level php-src and SDK builds. To choose the compiler for a TypePHP application, use `--compiler` or the project setting; do not assume `CXX` overrides it. |
| `PKG_CONFIG_PATH` | Build PHP/native dependencies | Lets `pkg-config` find `.pc` files outside standard prefixes; this is a toolchain setting, not a TypePHP-specific setting. |

For downloads, use `./tpc --proxy=<URL>`. TypePHP passes this to its downloader and sets `HTTP_PROXY`, `HTTPS_PROXY`, `ALL_PROXY`, and the corresponding lowercase names for child processes. Setting only the conventional environment variables may affect `curl` and package managers, but the PHP-stream download path in TypePHP does not guarantee that it will read them. Use `--proxy` when downloads must all use the same proxy.

## `php.ini`: `PHPRC` and `PHP_INI_SCAN_DIR`

Regular Embed programs and php-builder Embed/CLI/FPM programs use a full PHP runtime and can load PHP configuration at startup. The PHP process that runs the compiler also loads its own configuration; **its `php.ini` is not automatically embedded into the generated program**.

| Variable | Purpose |
|---|---|
| `PHPRC` | Main configuration location, either a directory or a configuration file path. For a directory, PHP first considers a SAPI-specific file, then `php.ini`; the Embed SAPI, for example, looks for `php-embed.ini` first. A readable file path loads that file directly. CLI/FPM configuration-path options may take precedence. |
| `PHP_INI_SCAN_DIR` | Overrides the additional `.ini` scan directories. Files are loaded in name order. Separate directories with `:` on Linux/macOS or `;` on Windows. An empty path component also scans the compiled-in default directory; an empty string disables additional directory scanning. |

To keep the main and additional configuration in separate directories:

```bash
PHPRC=/opt/myapp/conf \
PHP_INI_SCAN_DIR=/opt/myapp/conf.d \
./myapp
```

`/opt/myapp/conf` may contain `php.ini`. If it also contains `php-embed.ini`, the Embed SAPI loads that file first. To retain the compiled-in additional scan directory too, use `PHP_INI_SCAN_DIR="/opt/myapp/conf.d:"` on Linux/macOS, or a trailing `;` on Windows.

Check the actual configuration from a regular-mode program's `main()`:

```php
function main(): void
{
    var_dump(php_ini_loaded_file());
    var_dump(php_ini_scanned_files());
    var_dump(ini_get('memory_limit'));
}
```

`php --ini` describes **that PHP CLI process**, not necessarily the generated TypePHP executable. `--nano` has a separate startup path and does not load `php.ini`, so `PHPRC` and `PHP_INI_SCAN_DIR` do not configure Nano output. Nano retains some built-in INI entries, which does not imply file loading. See the [official PHP configuration-file reference](https://www.php.net/manual/en/configuration.file.php) for search order and scan-directory rules.

Use a dedicated configuration directory for a private php-builder runtime. Reusing the host PHP's `conf.d` can load `extension=...` entries that point to dynamic extensions built for an incompatible PHP ABI.

## Runtime libraries and application variables

| Variable | Platform | Purpose |
|---|---|---|
| `LD_LIBRARY_PATH` | Linux | Helps a dynamically linked `tpc` or application find `libphp.so`, `libphpx.so`, and other shared libraries. A system library directory or rpath is another deployment choice. |
| `DYLD_LIBRARY_PATH` | macOS | One possible search path for command-line launched dynamic programs. Released `.app` bundles should use bundled Frameworks and correct Mach-O load paths instead of depending on a shell variable. |
| `PATH` | Windows | Helps locate runtime DLLs. The application directory is also a usual place to ship them. |
| Application-defined variables | Generated program | An application may read variables passed by its parent through `getenv()`. Set them **when starting the generated program**; values supplied only when compiling are not baked into the executable. |

php-builder removes the dependency on host `libphp`/`libphpx` shared libraries, but other system shared libraries may still be required. Nano does not use `libphp`. FPM workers clear inherited environment variables by default. Pass required application variables through `env[NAME] = value` in the FPM pool configuration, or configure `clear_env` deliberately. See the [official PHP-FPM configuration reference](https://www.php.net/manual/en/install.fpm.configuration.php).

## Cross-compilation and platform scripts

| Variable | Scenario | Purpose |
|---|---|---|
| `ANDROID_NDK_HOME`, `ANDROID_NDK_ROOT` | Android | NDK root; `ANDROID_NDK_HOME` takes precedence. |
| `TYPEPHP_ANDROID_API` | Android | API level when the target triple contains no API number; defaults to `24`. An explicit `--target-platform=aarch64-linux-android<number>` takes precedence. |
| `PHPX_ANDROID_SDK_DIR` | Regular Android build | Overrides the default `$PHPX_HOME/android/arm64-v8a` SDK path; Nano source-composition builds do not read it. |
| `PHPX_ANDROID_BUILD_DIR`, `PHPX_ANDROID_JOBS`, `PHPX_SDK_JOBS` | Build PHPX Android/native SDKs | Set the build directory and job count for `phpx/android/build.sh`, and the job count for `phpx/sdk/build-native.sh`. Regular `tpc` builds do not read them. |
| `TYPEPHP_ANDROID_NM` | Android Nano | Overrides the `llvm-nm` command used for dependency auditing. |
| `PHPX_IOS_PLATFORM`, `PHPX_IOS_SDK_DIR`, `PHPX_IOS_BUILD_DIR`, `PHPX_IOS_DEPLOYMENT_TARGET`, `PHPX_IOS_JOBS` | Build the PHPX iOS SDK | Control `phpx/ios/build.sh`. When compiling an iOS application, TypePHP looks under `$PHPX_HOME/ios/<target>`; it does not use these variables to relocate that target. |
| `TYPEPHP_IOS_PROVISIONING_PROFILE`, `TYPEPHP_IOS_CODE_SIGN_IDENTITY`, `TYPEPHP_IOS_BUNDLE_IDENTIFIER` | Repository iOS example packaging | Used by `examples/apple-native/package-ios-app.sh`; these are not general `tpc` signing options. |
| `PATH` | WASI | `--wasm` finds WASI Clang, LLVM tools, and Wasmtime through `PATH`; the browser profile also requires Jco, and library mode requires wit-bindgen. |
| `PHPX_WASM_BUILD_DIR`, `PHPX_WASM_JOBS`, `PHPX_WASM_DEPS_WORK_DIR`, `PHPX_WASM_DEPS_PREFIX` | Build the PHPX WASI SDK | Control scripts under `phpx/wasm/`, not regular `tpc --wasm` options. |
| `TYPEPHP_WASM_SDK_BUILD_DIR`, `TYPEPHP_WASM_JOBS` | Build the TypePHP WASI SDK | Set default build directory and job count in `compiler/wasm/build-sdk.sh`; corresponding CLI options take precedence. |
| `XDG_CACHE_HOME` | Directly run output from a lower-level WASI build script | Controls Wasmtime's cache directory with `TYPEPHP_WASM_RUN=1`; the script defaults to `/tmp/typephp-wasmtime-cache`. |

## Compiler-internal variables

The WASI wrapper and PHP-version restart logic pass these variables to child processes. Ordinary projects do not need to set them. Changing them can bypass toolchain checks or break a build.

| Variable | Internal purpose |
|---|---|
| `TYPEPHP_SAPI_PHP_VERSION` | Prevents repeated restarts when the source compiler switches to the PHP CLI matching a php-builder target. |
| `TYPEPHP_WASM_INTERNAL_COMPILE`, `TYPEPHP_GENERATED_SOURCE_LIST` | WASI scripts call the compiler recursively to generate C++ and return the source list. |
| `TYPEPHP_WASM_INTERFACE_MANIFEST`, `TYPEPHP_WASM_INTERFACE_WIT`, `TYPEPHP_WASM_INTERFACE_ADAPTER`, `TYPEPHP_WASM_INTERFACE_ASYNC_EXPORTS` | WASI component/library interface generation and adaptation. |
| `TYPEPHP_WASI_CC`, `TYPEPHP_WASI_CXX`, `TYPEPHP_WASI_AR`, `TYPEPHP_WASI_RANLIB`, `TYPEPHP_WASI_NM`, `TYPEPHP_WASI_LD`, `TYPEPHP_WASI_TARGET`, `TYPEPHP_WASI_CLANG_VERSION` | Detected WASI tool paths, target, and compiler version. |
| `TYPEPHP_WASMTIME`, `TYPEPHP_WASMTIME_VERSION`, `TYPEPHP_JCO`, `TYPEPHP_JCO_VERSION`, `TYPEPHP_WIT_BINDGEN` | WASI execution, browser adaptation, and WIT tool information. |
| `TYPEPHP_WASM_BROWSER`, `TYPEPHP_WASM_BROWSER_DIR`, `TYPEPHP_WASM_PROGRAM_BUILD_DIR`, `TYPEPHP_WASM_MODE`, `TYPEPHP_WASM_PACKAGE`, `TYPEPHP_WASM_WORLD`, `TYPEPHP_WASM_NANO` | WASI build mode and output parameters generated from `--wasm` and the project configuration. |
| `TYPEPHP_WASM_RUN` | Direct-run switch recognized only by lower-level `wasm/build-*-program.sh` scripts; the public `tpc --wasm` command does not expose it as a stable interface. |
| `TYPEPHP_PACKAGE_VERSION`, `TYPEPHP_BUILD_ID` | Release or preview version while packaging the TypePHP compiler itself; they do not affect application builds or runtime. |
| `PROCESSOR_ARCHITECTURE`, `PROCESSOR_ARCHITEW6432` | OS-provided architecture identifiers used when packaging the TypePHP compiler on Windows; they are not application build settings to override. |
| `PHPX_SDK_VERSION` | Overrides the archive version when packaging a PHPX SDK; it does not affect a TypePHP application build or runtime. |

Example-specific settings such as `TYPEPHP_TASKBOARD_*` and `MACDEPLOYQT` belong to those applications and packaging scripts; they are not global TypePHP settings.

Related guides: [installation](install.md), [PHP Builder](php-builder.md),
[Nano](nano.md), [native applications](mobile-native.md), and [WASM](wasm.md).

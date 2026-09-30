## TypePHP Windows Toolkit

Windows users are recommended to download the TypePHP Windows toolkit directly. The toolkit already includes the DLL files, PHP, PHPX, and the compiled `tpc.exe` required for running and compiling, so there is no need to compile `libphp` or PHPX separately on Windows.

## Preparation

1. Install `VS17 (Visual Studio 2022)`; the Community edition download link is: <https://visualstudio.microsoft.com/zh-hans/vs/community/>. Be sure to check `Desktop development with C++` during installation
2. Download the `TypePHP Windows` toolkit from <https://github.com/swoole/typephp/releases> and extract it to `D:\workspace\typephp-windows-x64`


## Environment variables
- PHP_HOME=D:\workspace\typephp-windows-x64
- PHPX_HOME=D:\workspace\typephp-windows-x64\phpx
- Path+=D:\workspace\typephp-windows-x64

The `TypePHP Windows` package includes a full PHP 8.4 ZTS, PHPX, native DLLs, and `tpc.exe`. You can load more PHP extensions provided in the toolkit via `php.ini`.

## Compiling a PHP program

Place your `PHP` code in the `D:\workspace\typephp-windows-x64` directory and open an `x64 Native Tools Command Prompt for VS 2022` window. Run the following commands:

```bash
cd D:\workspace\typephp-windows-x64
tpc.exe examples\hello.php
hello.exe
```

## Compiling with Clang and LLVM

Windows builds can use `clang++` and `lld-link`. Existing php-windows packages
are compiled with MSVC, so use the **Microsoft ABI** and matching MSVC/Windows
SDK headers and `.lib` files. LLVM supplies the compiler, linker, resource,
and inspection executables. See [LLVM's MSVC compatibility documentation](https://clang.llvm.org/docs/MSVCCompatibility.html).

### Installation and environment

Install:

- LLVM for Windows x64, including `clang++`, `lld-link`, `llvm-rc`, and `llvm-readobj`.
- MSVC x64 headers/libraries from Visual Studio Build Tools, and the Windows SDK.
- A matching PHP/PHPX toolkit. Architecture, PHP version, and NTS/ZTS configuration must agree across PHP, PHPX, and dependencies.

Set the SDK paths explicitly in PowerShell. Adjust versions and directories to
your installation:

```powershell
$vc = 'C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\VC\Tools\MSVC\14.44.35207'
$sdk = 'C:\Program Files (x86)\Windows Kits\10'
$sdkVersion = '10.0.26100.0'
$env:LLVM_HOME = 'D:\workspace\tools\llvm-23.1.2'
$env:PHP_HOME = 'D:\workspace\typephp-windows-x64'
$env:PHPX_HOME = "$env:PHP_HOME\phpx"
$env:INCLUDE = "$vc\include;$sdk\Include\$sdkVersion\ucrt;$sdk\Include\$sdkVersion\shared;$sdk\Include\$sdkVersion\um;$sdk\Include\$sdkVersion\winrt"
$env:LIB = "$vc\lib\x64;$sdk\Lib\$sdkVersion\ucrt\x64;$sdk\Lib\$sdkVersion\um\x64"
$env:LIBPATH = ''
$env:PATH = "$env:LLVM_HOME\bin;$env:PHP_HOME;$env:PHPX_HOME\lib;$env:PHPX_HOME\build;$env:SystemRoot\System32;$env:SystemRoot\System32\WindowsPowerShell\v1.0;$env:SystemRoot"

clang++ -dumpmachine
lld-link --version
```

The target should be `x86_64-pc-windows-msvc`. This configuration does not run
`vcvars64.bat` or add MSVC executable directories to PATH. Check tool visibility
with `Get-Command cl.exe,link.exe,dumpbin.exe,rc.exe -ErrorAction SilentlyContinue`.

### Build commands

```powershell
& "$env:PHP_HOME\tpc.exe" examples\hello.php --compiler clang++ -o hello.exe
.\hello.exe

# DLL and PHP extension
& "$env:PHP_HOME\tpc.exe" project.yml --compiler clang++ -m lib -o app.dll
& "$env:PHP_HOME\tpc.exe" project.yml --compiler clang++ -m ext -o php_app.dll

# Standalone Nano executable
& "$env:PHP_HOME\tpc.exe" examples\hello.php --compiler clang++ --nano -o hello_nano.exe
```

For a source checkout, replace `tpc.exe` with `php bin/tpc.php`. Nano also needs
PHPX and php-nano sources resolvable through Composer. Normal builds need matching
PHP/PHPX DLLs at runtime; Nano bin/lib artifacts do not depend on external
PHP/PHPX DLLs.

The Windows Clang backend uses `lld-link`, `llvm-rc` for resources, and
`llvm-readobj` for Nano import checks. Options include `--lto`, `--debug`, and
`--no-console`. The last option selects the Windows GUI subsystem for desktop
applications such as Qt programs. Use Clang-compatible custom `cxx-flags`,
such as `-I` and `-D`; do not copy MSVC-specific options such as
`/Zc:__cplusplus` or `/permissive-` directly into a `clang++` build.

### Verified scope

Verified on Windows x64 with LLVM 23.1.2 and PHP 8.4.23:

| Runtime | Compiler host PHP | bin | lib | ext |
| --- | --- | --- | --- | --- |
| Normal NTS | NTS | Pass | Pass | Pass |
| Normal ZTS | ZTS | Pass | Pass | Pass |
| Nano NTS | NTS | Pass | Pass | Explicitly rejected; unsupported |
| Nano NTS | ZTS | Pass | Pass | Explicitly rejected; unsupported |

Both NTS and ZTS PHPX libraries were rebuilt using `clang-cl`, `lld-link`,
`llvm-lib`, `llvm-rc`, `llvm-mt`, and standalone CMake/Ninja. Tests retained
Microsoft headers/libraries and excluded MSVC executables. These results apply
to the listed versions and x64 architecture; the current Nano runtime is NTS.

See the [Windows Clang tests](https://github.com/swoole/typephp/tree/main/tests/windows/clang)
for the matrix, scripts, and reproduction instructions.

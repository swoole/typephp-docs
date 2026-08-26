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

## TypePHP Windows 工具包

Windows 用户推荐直接下载 TypePHP Windows 工具包。工具包已经包含运行和编译所需的 DLL 文件、PHP、PHPX 以及编译好的 `tpc.exe`，不需要在 Windows 上单独编译 `libphp` 或 PHPX。

## 准备工作

1. 安装`VS17 (Visual Studio 2022)`，社区版下载地址：<https://visualstudio.microsoft.com/zh-hans/vs/community/>，安装时务必勾选 `使用 C++ 的桌面开发`
2. 从 <https://github.com/swoole/typephp/releases> 下载 `TypePHP Windows` 工具包，解压至 `D:\workspace\typephp-windows-x64`


## 环境变量
- PHP_HOME=D:\workspace\typephp-windows-x64
- PHPX_HOME=D:\workspace\typephp-windows-x64\phpx
- Path+=D:\workspace\typephp-windows-x64

`TypePHP Windows` 软件包中包含完整版的 PHP 8.4 ZTS、PHPX、本地 DLL 和 `tpc.exe`。可以通过 `php.ini` 加载工具包中提供的更多 PHP 扩展。

## 编译 PHP 程序

将`PHP`代码放置于`D:\workspace\typephp-windows-x64`目录下，启动`x64 Native Tools Command Prompt for VS 2022`窗口。执行下面的命令：

```bash
cd D:\workspace\typephp-windows-x64
tpc.exe examples\hello.php
hello.exe
```

## 使用 Clang 和 LLVM 编译

Windows 也可以使用 `clang++` 编译、`lld-link` 链接。现有 php-windows
包由 MSVC 编译，因此仍需使用 **MSVC ABI**，以及匹配的 MSVC、Windows SDK
头文件和 `.lib`。LLVM 替换的是编译、链接、资源处理等可执行工具。
Clang 对这一 ABI 的支持见 [LLVM 官方说明](https://clang.llvm.org/docs/MSVCCompatibility.html)。

### 安装与环境

准备以下组件：

- LLVM Windows x64，包括 `clang++`、`lld-link`、`llvm-rc`、`llvm-readobj`。
- Visual Studio Build Tools 的 MSVC x64 头文件和库，以及 Windows SDK。
- 匹配的 PHP/PHPX 工具包；PHP、PHPX、第三方库的架构、PHP 版本和 NTS/ZTS 类型必须一致。

在 PowerShell 中显式设置 SDK 路径。下面的版本号和安装目录需要根据本机调整：

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

目标应为 `x86_64-pc-windows-msvc`。以上配置无需运行 `vcvars64.bat`，PATH
中也没有添加 MSVC 的 EXE 目录。可以用
`Get-Command cl.exe,link.exe,dumpbin.exe,rc.exe -ErrorAction SilentlyContinue`
检查这些工具是否可见。

### 编译命令

```powershell
& "$env:PHP_HOME\tpc.exe" examples\hello.php --compiler clang++ -o hello.exe
.\hello.exe

# DLL 和 PHP 扩展
& "$env:PHP_HOME\tpc.exe" project.yml --compiler clang++ -m lib -o app.dll
& "$env:PHP_HOME\tpc.exe" project.yml --compiler clang++ -m ext -o php_app.dll

# Nano 独立可执行文件
& "$env:PHP_HOME\tpc.exe" examples\hello.php --compiler clang++ --nano -o hello_nano.exe
```

从源码运行编译器时，将 `tpc.exe` 换成 `php bin/tpc.php`。Nano 还需要可由
Composer 解析的 PHPX、php-nano 源码。普通模式运行时仍需匹配的 PHP/PHPX
DLL；Nano 的 bin/lib 产物不依赖外部 PHP/PHPX DLL。

Windows Clang 后端使用 `lld-link`，资源文件使用 `llvm-rc`，Nano 的导入检查
使用 `llvm-readobj`。编译选项支持 `--lto`、`--debug`、`--no-console`，
其中 `--no-console` 将可执行文件设为 Windows GUI 子系统，适合 Qt 等桌面程序。
项目中的自定义 `cxx-flags` 应使用 Clang 支持的形式，例如 `-I`、`-D`；
MSVC 项目里的 `/Zc:__cplusplus`、`/permissive-` 等参数不能直接照搬到 `clang++`。

### 已验证范围

Windows x64、LLVM 23.1.2、PHP 8.4.23 下已验证：

| 运行时 | 编译器宿主 PHP | bin | lib | ext |
| --- | --- | --- | --- | --- |
| 普通 NTS | NTS | 通过 | 通过 | 通过 |
| 普通 ZTS | ZTS | 通过 | 通过 | 通过 |
| Nano NTS | NTS | 通过 | 通过 | 明确拒绝，不支持 |
| Nano NTS | ZTS | 通过 | 通过 | 明确拒绝，不支持 |

NTS/ZTS 两套 PHPX 也已通过 `clang-cl`、`lld-link`、`llvm-lib`、`llvm-rc`、
`llvm-mt` 和独立 CMake/Ninja 重建。测试保留 Microsoft 头文件与库，排除了
MSVC 的 EXE 工具。此结果的范围是上述版本和 x64 架构；Nano 当前运行时为 NTS。

完整矩阵、测试脚本与复现步骤见
[Windows Clang 测试](https://github.com/swoole/typephp/tree/main/tests/windows/clang)。

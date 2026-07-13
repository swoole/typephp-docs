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

## 准备工作

1. 安装`VS17 (Visual Studio 2022)`，社区版下载地址：<https://visualstudio.microsoft.com/zh-hans/vs/community/>
2. 下载`Swoole-Compiler-Windows`版本，解压至 `D:\workspace\swoole-compiler-windows-x64`


## 环境变量
- PHP_HOME=D:\workspace\swoole-compiler-windows-x64
- PHPX_HOME=D:\workspace\swoole-compiler-windows-x64\phpx
- Path+=D:\workspace\swoole-compiler-windows-x64

`Swoole-Compiler-Windows` 软件包中包含了完整版的 `PHP 8.4 ZTS`工具，可配置`php.ini`加载更多`PHP`扩展。

## 编译 PHP 程序

将`PHP`代码放置于`D:\workspace\swoole-compiler-windows-x64`目录下，启动`x64 Native Tools Command Prompt for VS 2022`窗口。执行下面的命令：

```bash
cd D:\workspace\swoole-compiler-windows-x64
swoole_compiler.exe examples\hello.php
hello.exe
```


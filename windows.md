## 准备

1. 安装`VS17 (Visual Studio 2022)`
1. 安装`php-sdk-binary-tools-2.3.0`
1. 安装`cmake`
1. 下载`PHP 8.4.20 ZTS Windows`版本
1. 下载`PHPX`代码
1. 安装`composer`

## 环境变量
- PHP_HOME=D:\workspace\php-8.4.20
- PHPX_HOME=D:\workspace\phpx
- Path=+D:\workspace\php-8.4.20 +D:\workspace\phpx\build

## 编译 PHPX
需要进入 php-sdk 环境

```bash
cd D:\workspace\php-sdk-binary-tools-2.3.0
phpsdk-vs17-x64.bat
```

编译
```bash
cd D:\workspace\phpx
mkdir build
cmake -DCMAKE_BUILD_TYPE=Release ..
nmake phpx
```

编译成功后，在`phpx/build`目录下会有`php8ts.dll`、`phpx.dll`两个文件

## 编译 PHP 程序
将 `swoole_compiler.exe` 程序放置于 `D:\workspace\compiler` 目录下。先使用`composer`安装依赖。

```bash
php D:\workspace\php-8.4.20\composer.phar install
swoole_compiler.exe examples\hello.php
hello.exe
```


## 参考
- PHP Windows 开发文档：https://wiki.php.net/internals/windows/stepbystepbuild_sdk_2 
- PHP-SDK：https://github.com/php/php-sdk-binary-tools 

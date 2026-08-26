# Composer 安装

可以使用 `Composer` 快速安装 `TypePHP`：

```bash
composer require swoole/typephp
```

`tpc.php` 由当前 `PHP` 解释器运行。需要 `PHP 8.4` 或 `PHP 8.5`。
`TypePHP` 项目需要支持 `C++17` 的 `GCC` 或 `Clang`，请检查当前是否满足要求。

> 建议使用 `require-dev`，因为 `tpc.php` 是开发期和构建期工具

## 确认安装
```bash
vendor/bin/tpc.php --version
vendor/bin/tpc.php --help
```

## 编译 PHP 项目

```bash
vendor/bin/tpc.php project.yml
vendor/bin/tpc.php hello.php
```

如果系统已经准备好相互兼容的 `libphp.so` 和 `libphpx.so`，`TypePHP` 会直接使用，
若系统缺少 `libphp.so` 或 `libphpx.so`，在 `Linux` 环境下 `TypePHP` 会自动构建。

## 环境要求

- 自动构建 `PHPX` 时需要 `CMake 3.24` 或更高版本；缺失时安装器可以通过系统包管理器安装。
- `PHP` 开发头文件，推荐提供 `php-config`。
- 自动构建 `PHPX` 时需要 `gmp`、`mpfr` 等编译依赖。

Ubuntu / Debian 常用依赖：

```bash
sudo apt-get update
sudo apt-get install -y \
    build-essential cmake pkg-config \
    libgmp-dev libmpfr-dev
```

Fedora / RHEL 系常用依赖：

```bash
sudo dnf install -y \
    gcc gcc-c++ make cmake pkgconf-pkg-config \
    gmp-devel mpfr-devel
```

具体包名可能因发行版版本不同而变化。已经准备好本地库时，不需要为了自动构建再次安装这些依赖。

`tpc.php` 不会在检查本地库之前要求 GCC 或 CMake。只有用户确认自动构建缺失库后，安装器才检测 `apt-get`、`dnf` 或 `yum`，并询问是否安装 GCC/G++、make、CMake、pkg-config 及其他缺失开发包。完成本地库处理后，编译器才执行最终工具链校验。

团队项目应提交 `composer.json` 和 `composer.lock`。其他开发者可以执行：

```bash
composer install
vendor/bin/tpc.php project.yml
```

## 全局安装

```bash
composer global require swoole/typephp
composer global config bin-dir --absolute
```

将命令输出的 `Composer` 全局 `bin` 目录加入 `PATH` 后，可以执行：

```bash
tpc.php --version
tpc.php project.yml
```

项目安装可以通过 `composer.lock` 固定编译器版本，正式项目优先使用项目级 `vendor/bin/tpc.php`。

## 优先使用已有本地库

如果已经自行编译好 `PHP Embed SAPI` 和 `PHPX`，建议显式设置安装目录：

```bash
export PHP_HOME=/opt/php-8.4
export PHPX_HOME=/opt/phpx

vendor/bin/tpc.php project.yml
```

对应目录应包含：

```text
$PHP_HOME/bin/php
$PHP_HOME/bin/php-config
$PHP_HOME/lib/libphp.so

$PHPX_HOME/CMakeLists.txt
$PHPX_HOME/include/
$PHPX_HOME/lib/libphpx.so
```

只要这些文件存在，`tpc.php` 会直接进入项目编译，不下载 PHP 源码，也不执行 PHPX CMake 构建。

`libphpx.so` 必须使用 `PHP_HOME` 指向的同一套 PHP 头文件和 `php-config` 编译，避免 PHP ABI 不一致。

## Composer 检查流程

每次编译都会先检查已有本地库，只在缺失时提供自动构建选项：

```text
系统 PHP
  → vendor/bin/tpc.php
  → 查找已有 libphp.so
      └─ 缺失时询问是否自动构建
  → 查找已有 libphpx.so
      └─ 缺失时询问是否使用同一 PHP 自动构建
  → 编译 TypePHP 项目
```

自动构建必须从 `tpc.php` 启动。自举二进制 `tpc` 在进入主程序前就需要系统动态链接器加载 `libphp.so` 和 `libphpx.so`；如果库不存在，二进制无法启动，也就没有机会运行安装器。

## 缺失时自动构建 libphp.so

Linux 二进制模式需要 PHP Embed SAPI 提供的 `libphp.so` 或 `libphp.a`。很多发行版只提供 PHP CLI/FPM，能够运行 `php` 不代表已经安装 Embed 库。

正常执行编译命令即可触发检查：

```bash
vendor/bin/tpc.php project.yml
```

缺少 Embed 库时会询问：

```text
The current PHP installation does not provide libphp.so: /usr
Build a private PHP embed library now? [Y/n] y
PHP version [当前运行 tpc.php 的完整 PHP_VERSION]:
Install directory [/home/user/.typephp]:
```

安装器会依次完成：

1. 从 PHP.net 获取稳定版发布信息；
2. 默认选择与当前运行 Composer 和 `tpc.php` 的 PHP 完全相同的版本，例如当前为 PHP 8.4.14，则默认构建 PHP 8.4.14；用户也可以手动输入其他 PHP 8.4.x 或 8.5.x 稳定版本；
3. 从 PHP.net 下载官方 `.tar.xz` 源码包；
4. 使用官方 SHA-256 校验源码包；
5. 读取当前 `php-config --configure-options`，保留当前扩展和构建配置；
6. 替换安装前缀并加入 `--enable-embed=shared`；
7. 检测 `apt-get`、`dnf` 或 `yum`，经确认后安装缺少的开发包；
8. 最多使用 8 个并行任务构建并安装 PHP；
9. 合并当前 ini，生成私有 PHP 使用的 `php.ini`。

默认安装目录：

```text
~/.typephp/
├── bin/
│   ├── php
│   └── php-config
├── include/
│   └── php/
├── lib/
│   ├── libphp.so
│   ├── php.ini
│   └── loaded-extensions.txt
└── var/
    └── build/
```

安装成功后，当前 `tpc.php` 进程会设置新的 `PHP_HOME` 并继续原编译任务。

### php.ini 与扩展

安装器会合并当前主 `php.ini` 和扫描目录中的 ini 文件。

新旧 PHP 主次版本相同时，例如均为 PHP 8.4，会尝试复制当前配置引用的共享扩展。找不到的扩展会在新 ini 中注释，避免 PHP 因加载不存在的 `.so` 而无法启动。

跨主次版本时，例如从 PHP 8.4 构建 PHP 8.5，不复制现有二进制扩展，因为不同 PHP 模块 API 的扩展不能直接复用。

## 缺失时自动构建 libphpx.so

Composer 会安装 `swoole/phpx` 源码。TypePHP 按以下顺序定位 PHPX：

1. `PHPX_HOME`；
2. Composer `InstalledVersions` 返回的 `swoole/phpx` 目录；
3. TypePHP 源码仓库内的 `vendor/swoole/phpx`。

如果希望使用自行检出的 PHPX：

```bash
export PHPX_HOME=/path/to/phpx
vendor/bin/tpc.php project.yml
```

目标目录必须包含 `CMakeLists.txt`、`include/` 和 `src/`。

缺少 `<PHPX>/lib/libphpx.so` 时会询问：

```text
The PHPX installation does not provide libphpx.so: /path/to/phpx
Build libphpx.so now? [Y/n]
```

确认后执行等价于：

```bash
cmake \
    -S /path/to/phpx \
    -B /path/to/phpx/build \
    -DCMAKE_BUILD_TYPE=Release \
    -DBUILD_TESTS=OFF \
    -Dphp_dir="$PHP_HOME"

cmake \
    --build /path/to/phpx/build \
    --parallel 8 \
    --target phpx
```

实际并行数为 CPU 数量与 8 的较小值。成功后生成：

```text
<PHPX>/lib/libphpx.so
```

### PHP ABI 一致性

PHPX 本身不依赖 `libphp.so`，也不需要在构建时链接 Embed 库。它依赖 PHP 的头文件和 `php-config`。为了保证最终运行时 ABI 一致，PHPX 应使用项目最终采用的同一套 PHP 构建。自动安装器会同时设置：

```text
PHP_HOME=<发现或自动安装 libphp.so 的 PHP 前缀>
PATH=$PHP_HOME/bin:$PATH
-Dphp_dir=$PHP_HOME
```

并验证该 PHP 前缀提供：

```text
$PHP_HOME/bin/php
$PHP_HOME/bin/php-config
```

如果 `libphp.so` 刚刚自动安装到 `~/.typephp`，PHPX 会使用 `~/.typephp/bin/php` 和 `~/.typephp/bin/php-config` 构建。如果用户拒绝构建 Embed 库但仍选择构建 PHPX，PHPX 可以直接使用当前 Composer PHP 对应的 `php-config`，两项构建并不存在依赖关系。

更换 PHP 版本、ZTS/NTS 或 Debug ABI 后，应删除旧 `libphpx.so` 并重新构建。

## 后续复用

后续编译可显式设置 PHP 和 PHPX：

```bash
export PHP_HOME="$HOME/.typephp"
export PHPX_HOME=/path/to/phpx
vendor/bin/tpc.php project.yml
```

再次选择相同 PHP 目录和版本时，安装器会询问是否复用已有 `libphp.so`。存在 `libphpx.so` 时不会重复执行 CMake。

## CI 与非交互环境

非交互环境不会自动确认下载、系统包安装或 CMake 构建。请在镜像准备阶段生成两个库，然后设置：

```bash
export PHP_HOME=/opt/typephp-php
export PHPX_HOME=/opt/phpx

composer install --no-interaction
vendor/bin/tpc.php project.yml --no-progress
```

不要跨操作系统、CPU 架构、PHP 主次版本或 PHP ABI 直接复用这些本地库。

## 手工构建 PHPX

自动构建失败时，可以使用同一 PHP 前缀手工执行：

```bash
cmake -S "$PHPX_HOME" -B "$PHPX_HOME/build" \
    -DCMAKE_BUILD_TYPE=Release \
    -DBUILD_TESTS=OFF \
    -Dphp_dir="$PHP_HOME"

cmake --build "$PHPX_HOME/build" --parallel 4 --target phpx
```

确认结果：

```bash
test -f "$PHPX_HOME/lib/libphpx.so"
```

## 第一次编译

创建 `hello.php`：

```php
<?php

function main(): void
{
    echo "Hello TypePHP\n";
}
```

编译并运行：

```bash
vendor/bin/tpc.php hello.php -O2 -o hello
./hello
```

完整构建参数见[编译项目](compile.md)和[命令行参数](options.md)。

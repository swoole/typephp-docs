# 安装 TypePHP

TypePHP 推荐直接使用 GitHub Releases 提供的、已经自举编译完成的 `tpc` 二进制程序。本教程以 Linux 为例，假设机器上已经自行编译并安装好相互兼容的 `libphp.so` 和 `libphpx.so`。

如果使用 Windows 工具包或希望通过 PHP Composer 包运行 `tpc.php`，请阅读本页末尾的其他安装方式。

## 1. 下载 tpc

打开 TypePHP Releases 页面，选择与当前操作系统和 CPU 架构匹配的软件包：

<https://github.com/swoole/typephp/releases>

下载后解压，例如：

```bash
mkdir -p "$HOME/typephp"
cd "$HOME/typephp"
tar -xf /path/to/typephp-linux-x86_64.tar.gz
chmod +x tpc
```

实际压缩包名称以 Releases 页面为准。不要使用其他操作系统或 CPU 架构的二进制文件。

TypePHP 生成的是当前平台的原生可执行程序，不是 PHP 字节码。

## 2. 检查 PHP Embed

TypePHP 二进制模式需要 PHP Embed SAPI。推荐使用 PHP 8.4，并确保 PHP、头文件、`php-config` 和 `libphp.so` 来自同一次 PHP 构建。

设置 PHP 安装前缀：

```bash
export PHP_HOME=/opt/php-8.4
```

检查 PHP 版本和配置：

```bash
"$PHP_HOME/bin/php" -v
"$PHP_HOME/bin/php-config" --version
"$PHP_HOME/bin/php-config" --php-sapis
"$PHP_HOME/bin/php-config" --lib-dir
```

`--php-sapis` 应包含 `embed`，PHP lib 目录中应存在：

```bash
test -f "$PHP_HOME/lib/libphp.so"
```

典型的 PHP configure 配置包含：

```text
--enable-embed=shared
```

如果使用 ZTS，则 PHPX 和 `tpc` 所使用的 PHP 头文件、库也必须属于同一 ZTS 构建。不要混用不同 PHP 主次版本、ZTS/NTS 或 Debug/Release ABI 的文件。

## 3. 检查 PHPX

PHPX 需要使用上一步同一套 PHP 编译。PHPX 依赖 GMP 和 MPFR；以 Ubuntu / Debian 为例：

```bash
sudo apt-get update
sudo apt-get install -y \
    build-essential cmake pkg-config \
    libgmp-dev libmpfr-dev
```

如果尚未编译 PHPX，可以执行：

```bash
git clone https://github.com/swoole/phpx.git /opt/phpx

cmake -S /opt/phpx -B /opt/phpx/build \
    -DCMAKE_BUILD_TYPE=Release \
    -DBUILD_TESTS=OFF \
    -Dphp_dir="$PHP_HOME"

cmake --build /opt/phpx/build --parallel 4 --target phpx
```

设置 PHPX 路径并检查结果：

```bash
export PHPX_HOME=/opt/phpx
test -f "$PHPX_HOME/lib/libphpx.so"
```

`PHPX_HOME` 必须指向 PHPX 源码根目录，其中应包含：

```text
$PHPX_HOME/CMakeLists.txt
$PHPX_HOME/include/
$PHPX_HOME/src/misc/
$PHPX_HOME/lib/libphpx.so
```

除了链接 `libphpx.so`，TypePHP 还会使用 PHPX 的头文件和 `src/misc` 源文件，因此不能只保留单独一个 `.so` 文件。

## 4. 配置环境变量

当前终端可以直接设置：

```bash
export PHP_HOME=/opt/php-8.4
export PHPX_HOME=/opt/phpx
export PATH="$HOME/typephp:$PATH"
export LD_LIBRARY_PATH="$PHP_HOME/lib:$PHPX_HOME/lib${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}"
```

如果希望长期使用，可将这些配置加入 `~/.bashrc` 或对应 shell 配置文件。

也可以使用系统动态库配置。创建 `/etc/ld.so.conf.d/typephp.conf`：

```text
/opt/php-8.4/lib
/opt/phpx/lib
```

然后刷新缓存：

```bash
sudo ldconfig
```

`LD_LIBRARY_PATH` 和 `ldconfig` 选择一种即可。开发环境通常使用前者，固定部署环境可使用后者。

## 5. 验证动态库

在运行 `tpc` 前检查动态链接结果：

```bash
ldd "$HOME/typephp/tpc" | grep -E 'libphp(x)?\.so'
```

正常情况下应看到 `libphp.so` 和 `libphpx.so` 都解析到预期目录，例如：

```text
libphpx.so => /opt/phpx/lib/libphpx.so
libphp.so  => /opt/php-8.4/lib/libphp.so
```

如果显示 `not found`，请先修正 `LD_LIBRARY_PATH` 或 `ldconfig`。二进制 `tpc` 在进入主程序前就需要加载这两个库，无法在缺库时自行修复环境。

## 6. 查看编译器

```bash
tpc --version
tpc --help
```

也可以在解压目录中直接执行：

```bash
./tpc --version
./tpc --help
```

基本命令格式：

```text
tpc <PHP 文件、目录或 project.yml> [选项]
```

## 7. 编译第一个程序

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
tpc hello.php -O2 -o hello
./hello
```

预期输出：

```text
Hello TypePHP
```

复杂项目建议使用 `project.yml`：

```yaml
name: my-app
version: 1.0.0

sources:
  - src
  - main.php

build-dir: build
```

```bash
tpc project.yml -O2
```

完整说明见[编译项目](compile.md)、[命令行参数](options.md)和 [project.yml 配置](project-yml.md)。

## 常见问题

### tpc: command not found

使用完整路径运行，或把 `tpc` 所在目录加入 `PATH`：

```bash
export PATH="$HOME/typephp:$PATH"
```

### error while loading shared libraries

使用 `ldd` 查看缺少的库：

```bash
ldd "$HOME/typephp/tpc" | grep 'not found'
```

确认 `PHP_HOME`、`PHPX_HOME` 和动态库搜索路径指向正确目录。

### libphpx.so 与 PHP ABI 不一致

使用 `$PHP_HOME/bin/php-config` 重新配置并编译 PHPX：

```bash
cmake -S "$PHPX_HOME" -B "$PHPX_HOME/build" \
    -DCMAKE_BUILD_TYPE=Release \
    -DBUILD_TESTS=OFF \
    -Dphp_dir="$PHP_HOME"

cmake --build "$PHPX_HOME/build" --parallel 4 --target phpx
```

## 其他安装方式

- [Composer](composer.md)：适用于 Linux 环境中已经有 PHP 和 Composer 的用户。优先使用已有 `libphp.so` / `libphpx.so`，缺少时可以由 `vendor/bin/tpc.php` 交互式自动构建。
- [Windows 环境](windows.md)：直接使用 TypePHP Windows 工具包，其中包含所需 DLL、PHP、PHPX 和已经编译好的 `tpc.exe`。

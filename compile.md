# 编译项目

推荐使用 TypePHP 自举编译生成的二进制编译器：

```text
./tpc
```

基本格式：

```bash
./tpc <PHP 文件、目录或 project.yml> [选项]
```

如果已经把二进制所在目录加入 `PATH`，也可以直接使用 `tpc`。Composer 安装方式见[Composer 安装](composer.md)。

## 单文件编译

PHP 文件必须提供全局 `main()` 入口：

```php
<?php

function main(): void
{
    echo "Hello TypePHP\n";
}
```

编译：

```bash
./tpc hello.php
```

默认生成与入口文件同名的可执行程序：

```bash
./hello
```

指定优化等级和输出路径：

```bash
./tpc hello.php -O2 -o build/hello
```

编译成功后立即运行：

```bash
./tpc hello.php -O2 --run
```

## 编译目录

传入目录时，编译器会递归扫描其中的 PHP 文件：

```bash
./tpc src/ -O2 -o app
```

目录模式适合结构简单的项目。需要精确控制源码、忽略文件、C++ 源码和链接参数时，应使用 `project.yml`。

## 使用 project.yml

```yaml
name: my-app
version: 1.0.0

sources:
  - src
  - main.php

ignore:
  - src/DevelopmentOnly.php

build-dir: build
```

执行：

```bash
./tpc project.yml -O2
```

配置文件路径可以使用任意文件名，只要扩展名是 `.yml` 或 `.yaml`。相对路径以配置文件所在目录为基准。

完整字段见 [project.yml 配置](project-yml.md)。

## 构建模式

默认 `bin` 模式生成可执行程序：

```bash
./tpc project.yml -m bin
```

`lib` 模式生成可供其他 TypePHP 项目链接的动态库：

```bash
./tpc project.yml -m lib
```

创建、发布和引用动态库的完整步骤见[TypePHP 动态库](library.md)。

`ext` 模式生成 PHP 扩展：

```bash
./tpc extension.yml -m ext -o my_extension
```

Linux `bin` 模式需要 `libphp.so` 或 `libphp.a`。二进制 `tpc` 自身也依赖 `libphp.so` 和 `libphpx.so`，所以它不能在缺库时自行启动安装器。Composer 用户应通过 `vendor/bin/tpc.php` 自动准备这些库，详见[Composer 安装](composer.md)。`ext` 模式不触发 Embed PHP 安装器。

## 检查生成代码

`--dry` 只生成 C++ 文件，不调用 C++ 编译器和链接器：

```bash
./tpc project.yml --dry
```

生成目录默认为项目的 `build`，也可指定：

```bash
./tpc project.yml --dry --build-dir /tmp/typephp-build
```

## 常用命令

```bash
# 查看帮助和版本
./tpc --help
./tpc --version

# 发布构建
./tpc project.yml -O2 -j8

# 调试构建
./tpc project.yml --debug

# 强制忽略缓存重新编译
./tpc project.yml --force

# 生成并立即运行
./tpc project.yml --run
```

所有参数见[命令行参数](options.md)。

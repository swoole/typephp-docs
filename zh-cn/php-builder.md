# PHP Builder 与 SAPI

TypePHP 将产物类型、PHP 进程接口和 PHP 运行时来源分成三个独立配置：

| 配置 | 表达的语义 | 可选值 |
|---|---|---|
| `mode` | 生成什么产物 | `bin`、`lib`、`ext` |
| `sapi` | `bin` 可执行文件使用哪种 PHP SAPI | `embed`、`cli`、`fpm`，可选择多个 |
| `php-builder` | 是否从 php-src 构建私有 PHP 运行时 | 包含 `extensions`、`zts` 的映射 |

`cli` 和 `fpm` 不是构建模式。它们与 `embed` 一样是 SAPI，生成的都是 ELF、Mach-O 等平台原生可执行文件。`sapi` 和 `php-builder` 只适用于 `mode: bin`。

## 如何选择

| SAPI | 未配置 `php-builder` | 配置 `php-builder` | `entry` |
|---|---|---|---|
| `embed` | 链接宿主机的 `libphp`，保持原有构建方式 | 从 php-src 构建并静态链接私有 Embed 运行时 | 不使用 |
| `cli` | 不支持 | 构建包含 TypePHP 项目的 PHP CLI 程序 | 必须配置 |
| `fpm` | 不支持 | 构建包含 TypePHP 项目的 PHP-FPM 程序 | 不需要 |

`php-builder` 当前支持 Linux 和 macOS。Windows 可以使用工具包或宿主机提供的 Embed 运行时，但目前不能通过 `php-builder` 从源码生成私有 SAPI 运行时。

`php-builder` 不依赖宿主机的 PHP 运行时，却仍使用操作系统提供的底层开发库。PHP、PHPX 和所选 PHP 扩展会静态链接进产物，libc、libxml、zlib、OpenSSL 等仍由系统提供。因此它不同于需要专用 SDK 的 `--full-static`。

## YAML 配置

```yaml
name: myapp
mode: bin
php-version: 8.5
sapi: [embed, cli, fpm]
entry: bin/console.php

php-builder:
  extensions: [swoole, mongodb]
  zts: on

sources:
  - src
  - bin/console.php
```

配置规则：

- `sapi` 可以是单个值，如 `sapi: cli`，也可以是列表；默认值是 `embed`。
- `php-builder` 必须是映射。使用默认配置时写成 `php-builder: {}`，不能只写空的 `php-builder:`。
- `extensions` 默认是 `[]`，用于显式补充需要编入 PHP 的扩展。
- `zts` 默认是 `off`，推荐明确填写 `on` 或 `off`。
- `php-builder` 中不能配置 `sapi`；SAPI 始终由顶层 `sapi` 选择。
- `sapi` 中只要包含 `cli` 就必须设置 `entry`；没有 `cli` 时设置 `entry` 会报错。
- YAML 中的 `entry` 相对于 `project.yml` 所在目录解析。
- 命令行参数覆盖 YAML 中的同名配置。

旧的 `mode: cli`、`mode: fpm` 和 `php-builder.sapi` 均不受支持。

## 命令行配置

不带值的 `--php-builder` 等价于 `{}`：

```bash
./tpc hello.php --php-builder
```

显式配置使用与 YAML 相同的字段，字段之间用分号分隔。整个值必须使用 shell 引号包住：

```bash
./tpc project.yml \
    --sapi=embed,cli,fpm \
    --entry=bin/console.php \
    --php-builder='extensions: [swoole, mongodb]; zts: on'
```

下面两种写法等价：

```yaml
php-builder:
  extensions: [swoole, mongodb]
  zts: on
```

```bash
--php-builder='extensions: [swoole, mongodb]; zts: on'
```

不要把 `sapi` 写进 `--php-builder`，应单独使用 `--sapi=cli`、`--sapi=fpm` 或逗号分隔的多个值。

## Embed SAPI

### 使用宿主机 libphp

```yaml
mode: bin
sapi: embed
```

`embed` 是默认 SAPI，因此可以省略。产物链接 `PHP_HOME` 或当前 PHP 安装中的 `libphp.so`、`libphp.dylib` 或平台对应库，行为与原有 TypePHP 二进制模式一致。

Linux/macOS 上找不到 Embed 库时，交互式终端会询问是否启用 `php-builder`。CI 等非交互环境必须显式配置 `--php-builder`，否则构建会终止。

### 使用私有静态 Embed 运行时

```yaml
mode: bin
sapi: embed
php-builder: {}
```

等价命令：

```bash
./tpc hello.php --sapi=embed --php-builder
```

此方式保持 Embed 原有的模块注册和请求生命周期，只把宿主机动态 `libphp` 换成从 php-src 构建的静态 `libphp.a`。

## CLI SAPI

CLI 目标使用 PHP 官方 CLI SAPI，并把 TypePHP 生成的模块静态编入程序。`entry` 是启动时由 Zend VM 执行的 PHP 主脚本，其他 `sources` 仍走 TypePHP AOT 编译流程。

```yaml
name: worker
mode: bin
sapi: cli
entry: bin/worker.php
php-builder:
  extensions: [pcntl, sockets]
  zts: off
sources:
  - src
  - bin/worker.php
```

即使 `entry` 同时出现在 `sources` 中，编译器也会自动将它从 AOT 源码集合中排除，并以内嵌 PHP 脚本保存，避免重复编译。入口可以包含普通 PHP 顶层执行代码，不需要 Embed 模式使用的全局 `main()`。

也可以通过命令行把 AOT 源码和入口脚本分开指定：

```bash
./tpc src/functions.php \
    --sapi=cli \
    --entry=bin/worker.php \
    --php-builder
```

运行生成程序时，参数会传给内嵌入口：

```bash
./worker --queue emails
```

## FPM SAPI

FPM 目标将 TypePHP 项目作为内建 PHP 模块链接进 PHP-FPM 主程序。它没有固定的 CLI 入口，所以不配置 `entry`。请求脚本、pool 和监听地址仍按普通 PHP-FPM 方式配置，请求脚本可以调用 TypePHP AOT 模块导出的函数和类。

```yaml
name: myapp-fpm
mode: bin
sapi: fpm
php-builder:
  extensions: [pdo, pdo_mysql, opcache]
  zts: off
sources:
  - src
```

启动方式与普通 PHP-FPM 一致，例如：

```bash
./myapp-fpm --fpm-config /etc/myapp/php-fpm.conf
```

只有 FPM 目标时不能使用 `tpc --run`，应在准备好 FPM 配置后单独启动。

## 同时生成多个 SAPI

```yaml
name: myapp
mode: bin
sapi: [embed, cli, fpm]
entry: bin/console.php
php-builder:
  extensions: []
  zts: on
```

单个 SAPI 使用原输出名；多个 SAPI 自动添加后缀：

| 配置 | 生成文件 |
|---|---|
| `sapi: cli`、`name: myapp` | `myapp` |
| `sapi: [embed, cli, fpm]`、`name: myapp` | `myapp-embed`、`myapp-cli`、`myapp-fpm` |
| 多 SAPI 加 `-o build/myapp.bin` | `build/myapp-embed.bin`、`build/myapp-cli.bin`、`build/myapp-fpm.bin` |

多目标配合 `--run` 时，会运行 `sapi` 顺序中第一个 `embed` 或 `cli` 目标；FPM 不会成为 `--run` 目标。

## 扩展依赖收集

`php-builder` 会合并以下来源，并自动去重：

1. `php-builder.extensions` 显式配置；
2. YAML 的 `extension-dependencies` 或 `ext-deps`；
3. AOT 源码实际引用的 PHP 内部函数和类所属扩展；
4. `embedded-files` 所包含 Composer 项目的 `composer.json` 中的 `ext-*` 依赖。

```yaml
php-builder:
  extensions: [swoole]
extension-dependencies:
  - curl
embedded-files:
  - vendor
```

扩展名会规范化，例如 `ext-curl` 变为 `curl`、`pdo_mysql` 变为 `pdo-mysql`。php-src 自带扩展会自动转换成对应的 `--enable-*` 或 `--with-*` configure 参数；不在 php-src 中的扩展会从 PECL 获取最新稳定源码，并注入派生的源码树。缓存的官方 php-src 不会被修改。

所有 PHP 扩展都以静态方式编入私有运行时，不会在目标机器上加载宿主 PHP 的扩展 `.so`。扩展需要的 C/C++ 开发库仍应由操作系统提供。

## PHP 版本、ZTS 和工具链

- `php-version: 8.4` 或 `8.5` 同时选择解析语法和 php-src 发布分支。首次下载时使用该分支最新稳定补丁版本，已有完整缓存时优先复用。
- `zts: on` 构建线程安全 PHP，`zts: off` 构建 NTS。切换 ZTS 会选择不同的运行时缓存。
- `--compiler` 同时影响 PHP builder 工具链。GCC 使用配套的 `gcc`/`g++`/`gcc-ar`，Clang 使用 `clang`/`clang++`/`llvm-ar`，不会要求 GCC 用户安装 LLVM。
- `-j`/`--job` 同时控制 TypePHP 和 php-src 的并行编译任务数。

构建机需要 C/C++ 编译器、`make`、CMake、pkg-config，以及所选 PHP/PECL 扩展需要的系统开发库。外部 PECL 扩展还可能需要 autoconf 等生成工具。

## 缓存与构建日志

下载和构建内容位于 `~/.typephp`：

```text
~/.typephp/
├── archives/          # PHP、PECL 源码包
├── src/               # 未修改的官方 php-src
├── pecl/              # 解压后的 PECL 扩展
├── php-builder-src/   # 注入外部扩展后的派生源码
└── php-builder/       # 已构建的私有运行时
```

`php-8.5.10-e90c19d6ad02c94f` 末尾的哈希是兼容性指纹。它包含 PHP 补丁版本、configure 参数、SAPI 集合、ZTS、外部扩展版本、操作系统、CPU 架构和工具链，不包含项目名称或路径。因此并非每个项目复制一份 PHP 源码；配置兼容的项目会共享运行时，扩展集合满足要求的现有运行时也可复用。

同一运行时目录使用文件锁，避免并发构建破坏缓存。完整 PHP 编译输出保存在该运行时的 `build/.typephp-build.log`。默认终端显示进度条和构建阶段；使用 `--no-progress` 时改为适合 CI 的逐行日志。

## 网络代理

`--proxy` 是全局网络配置，不属于 `php-builder`。PHP 发布元数据、php-src、PECL 元数据和源码包，以及 TypePHP 的其他网络操作都会使用它：

```bash
./tpc project.yml \
    --proxy=socks5h://127.0.0.1:1080 \
    --php-builder='extensions: [mongodb]; zts: off'
```

支持 HTTP(S) 代理；安装了 `curl` 时也支持 SOCKS 代理。下载器调用外部 `curl`
时会通过 `--proxy` 显式传递该设置。

## 常见错误

| 错误 | 处理方法 |
|---|---|
| `` `php-builder` must be a mapping `` | 使用 `php-builder: {}`，或填写 `extensions`/`zts` 映射。 |
| `The cli and fpm SAPIs require php-builder` | 为 CLI/FPM 显式启用 `php-builder`。 |
| `` `sapi` containing cli requires an `entry` PHP file `` | 为 CLI 指定存在的 PHP 入口文件。 |
| `` `entry` requires `sapi` to contain cli `` | 删除 `entry`，或在 `sapi` 中加入 `cli`。 |
| `` `sapi` is only supported with `mode: bin` `` | `lib` 和 `ext` 不能配置 `sapi`。 |
| configure 找不到头文件或库 | 安装对应扩展所需的操作系统开发包后重试。 |

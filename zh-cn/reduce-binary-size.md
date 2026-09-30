# 缩小 Nano 与 php-builder 可执行文件

TypePHP 的 `bin` 产物既包含应用代码，也可能包含 PHP Nano、PHPX 或静态 PHP
运行时。先确认比较对象：`build/` 是编译中间文件和缓存，不是发布包；macOS
`.app`、Windows 发布目录还可能含有 Qt、系统运行库、资源等文件。比较产物时，
分别记录可执行文件大小和完整发布目录大小，并在每次改动后运行相同的功能测试。

| 构建方式 | 主要体积来源 | 优先检查 |
|---|---|---|
| `--nano` | 选中的 Nano/PHPX 源码、应用代码、原生依赖 | 动态调用、无用源码和资源、优化级别 |
| `--php-builder` | 静态 PHP/PHPX、扩展、应用代码、嵌入文件 | SAPI、扩展、ZTS、`embedded-files` |

最直接的发布步骤是：Linux/macOS 对可执行文件的副本执行下文的 `strip`；
使用 `embedded-files` 打包 `vendor` 时，先在干净的发布工作目录运行
`composer install --no-dev`，只安装生产依赖。

## 先做发布构建

不要带 `-d` / `--debug`、`--sanitize` 或 `--profile` 发布。`-d` 会关闭优化并
生成调试信息。TypePHP 的 `-O` 只接受 `0` 到 `3`，默认是 `0`；可以先试 `-O1`，
再与 `-O2` 的大小和性能比较。更高优化级别不保证更小的文件。`--lto` 可让链接器
跨编译单元优化，但会增加构建时间，最终大小也应实测。

```bash
# Nano：只用所需源码构建，先试 -O1；-o 指定输出文件
./tpc app.php --nano -O1 --lto -o app-small

# 私有 PHP 运行时：默认只选择 Embed SAPI
./tpc app.php --sapi=embed \
  --php-builder='extensions: []; zts: off' \
  -O1 --lto -o app-small
```

`--lto` 作用于 TypePHP 此次编译和链接的对象；它不会用 LTO 重新构建已缓存的
php-builder 静态 PHP/PHPX 运行时。若工具链或某个原生库不支持 LTO，去掉
`--lto` 后再比较。Nano 在各平台已经自动启用未引用代码段清理（Windows 为
`/OPT:REF /OPT:ICF`，Linux 为 `--gc-sections`，macOS 为 `-dead_strip`），无需手动
重复添加这些链接选项。

## Nano：减少被选中的运行时功能

Nano 根据源码实际使用的函数、类和类型选择扩展及部分标准库功能。检查项目的
`sources`，用 `ignore` 排除不会进入产物的源码，并删去不需要的原生库链接输入。
动态函数/方法调用会让编译器保守地保留更多内置 Nano 功能；能明确写出调用目标时，
编译器通常可以选择更小的功能集合。不要为缩小产物而删除程序确实需要的功能。

Nano 不支持 `embedded-files`，因此缩小普通模式的嵌入文件配置不会影响 Nano
产物。它也不依赖 `libphp`，`php-builder.extensions` 与 `php-builder.zts` 不适用。

## php-builder：控制运行时与资源

1. 如果程序只需要 Embed SAPI，使用默认的 `sapi: embed`；不要额外请求 `cli`
   或 `fpm`。这些目标有自己的入口和运行时代码。
2. 不需要线程安全 PHP 时使用 `zts: off`（默认值）。确实需要 ZTS 时保留
   `zts: on`。
3. `extensions` 只列出应用确实需要的额外扩展。TypePHP 还会从项目源码、YAML
   和 Composer 元数据合并扩展需求；`extensions: []` **不表示禁用所有扩展**。
   PHP 核心扩展与构建器固定启用的功能仍会存在。
4. 如果使用 `embedded-files`，先在干净的发布工作目录执行
   `composer install --no-dev`，再编译；这样 `vendor` 不会包含 `require-dev`
   中的开发依赖。只列出发布时实际需要的文件，不要把测试、文档、样例和大资源
   目录无差别嵌入。`ignore` 只影响 `sources`，不会从 `embedded-files` 中删除
   文件。详见[嵌入文件](embedded-files.md)。

例如发布配置确实需要打包整个生产版 `vendor` 时：

```yaml
embedded-files:
  - vendor
```

在干净的发布工作目录中执行：

```bash
composer install --no-dev
./tpc project-release.yml -O1
```

这里的 `--no-dev` 控制 `vendor` 的内容；若要继续缩小产物，可在
`embedded-files` 中仅列出实际需要的子目录。别把缺失的运行时依赖误当成开发包。

例如，项目确实使用 SQLite 时可以保留 `pdo_sqlite`；不能为缩小体积把它删除：

```yaml
mode: bin
sapi: embed
optimize: 1
lto: true
php-builder:
  extensions: [pdo_sqlite]
  zts: off
```

php-builder 会从 `~/.typephp/php-builder/` 复用兼容的私有运行时，并可能选择已安装
扩展的超集。缩减 `extensions` 后若日志显示复用运行时，检查对应缓存目录的
`runtime.json` 中 `enabled_extensions` 和 `configure_options`；此时文件大小不变
不能证明配置无效。要验证新的最小运行时，请在隔离的构建环境中使用干净的缓存，
不要直接删除仍被其他构建使用的缓存。`-O1` / `--lto` 不会改变已缓存 PHP 本身的
编译参数。

## 发布前去掉不需要的符号

保留原始构建产物用于排错，对**副本**做符号裁剪，并重新运行程序：

Linux（GNU binutils）：

```bash
cp app-small app-small.release
strip --strip-unneeded app-small.release
```

macOS（在 Mac 上执行，且在签名前操作）：

```bash
cp app-small app-small.release
xcrun strip -S -x app-small.release
```

macOS `.app` 应在裁剪可执行文件和内含原生库后再签名；修改已签名的包会使签名
失效。Windows MSVC 的调试信息主要保存在独立的 PDB 中：发布目录只放需要的
`.exe`、DLL 和资源文件，按需另行保存 PDB 供崩溃分析。不要用 Unix `strip`
处理 MSVC 产物。

最后同时检查可执行文件和发布目录。例如 Linux/macOS 可以用
`ls -lh app-small.release`、`du -sh release/`；Windows PowerShell 可以用
`(Get-Item .\app-small.exe).Length`。如果依赖动态库，较小的 `.exe` 不等于较小
的完整发布包。`--full-static` 会将更多运行库放进可执行文件，通常不适合作为
减小单个文件的手段。

对于多文件项目，推荐使用 YAML 配置文件管理项目构建参数。配置文件通常命名为 `project.yml`，也可以使用任意 `*.yml` 文件名，例如 `myproject.yml`，编译时通过命令行参数传入对应路径即可。

命令行参数优先级最高，其次是 YAML 配置，最后是默认值。

## 完整示例

```yaml
# 输出文件路径（等价于 -o/--output）
output: build/myapp

# 若只需要指定产物名称，也可以使用 name
# name 不改变输出目录
# name: myapp

# 构建模式：bin（可执行文件）/ lib（动态库）/ ext（PHP 扩展）
# WASM 下还可使用 library，生成具有 WIT 导出接口的 Component
mode: bin

# 优化级别、并行任务和调试开关
optimize: 2
job: 8
debug: false

# 源码列表（文件或目录）
sources:
  - src/
  - lib/utils.php
  - native/helper.cpp
  - if: PHP_VERSION_ID >= 80400
    path: compat/php84.php
  - if: PHP_VERSION >= "8.4.0"
    path: compat/php84-version.php

# 忽略的文件或目录
ignore:
  - src/tests/
  - src/vendor/
  - ext-gd          # 忽略特定扩展依赖

# 构建目录（相对路径基于当前 YAML 文件所在目录解析）
build-dir: build/out

# C++ 标准版本
cxx-std: c++20

# 目标 CPU / 交叉编译平台
march: native
target-platform: aarch64-linux-gnu

# WASI 0.2 输出：仅允许 component 或 browser
# wasm: component
# wasm-browser-dir: generated
# library 模式可配置 WIT 标识
# wasm-package: app:calculator@1.0.0
# wasm-world: calculator

# 自定义 C++ 编译器（gcc/g++/clang/clang++ 等）
cpp-compiler: clang++

# 额外的 C++ 编译选项
cxx-flags:
  - -fvisibility=hidden
  - -Wno-unused-parameter

# 额外的链接选项
ld-flags:
  - -lcurl
  - -lssl

# 额外的 C++ 头文件搜索路径（等价于 -I）
include-paths:
  - /opt/mylib/include
  - ../shared/headers

# 预处理器宏定义（等价于 -D）
defines:
  - ENABLE_LOGGING=1
  - DEBUG_LEVEL=3

# 运行时/构建行为
sanitize: address
profile: false
no-literal-strings: false
no-progress: true
no-console: false
dry: false
lto: true
format: true

# 要链接的库（等价于 -l）
link-libs:
  - curl
  - ssl

# 库搜索路径（等价于 -L）
link-paths:
  - /usr/local/lib
  - /opt/custom/lib

# Windows 资源配置（仅 Windows 平台）
resource:
  icon: assets/app.ico
```

## 配置项说明

| 配置项 | 类型 | 说明 |
|--------|------|------|
| `output` | `string` | 输出文件路径，等价于 `-o` / `--output`。可包含目录部分，例如 `build/myapp` |
| `name` | `string` | 仅设置输出文件名，不改变输出目录。例如 `name: myapp` 会在当前工作目录生成 `./myapp` |
| `mode` / `build-mode` / `type` | `string` | 构建模式，等价于 `-m`。Host 构建支持 `bin`/`binary`/`cli`、`lib`/`library`/`shared` 和 `ext`/`extension`；WASM 构建使用命令模式或 `lib`/`library`/`reactor` library component 模式 |
| `optimize` | `integer` | 优化级别，等价于 `-O`，取值范围 `0` ~ `3` |
| `job` | `integer` | 并行编译任务数，等价于 `-j` / `--job` |
| `debug` | `boolean` | 启用调试模式，等价于 `-d` / `--debug` |
| `profile` | `boolean` | 启用 CPU 性能分析，等价于 `--profile`。仅 Linux 支持 |
| `no-literal-strings` | `boolean` | 禁用字符串字面量优化，等价于 `--no-literal-strings` |
| `no-progress` | `boolean` | 禁用进度条输出，等价于 `--no-progress` |
| `no-console` | `boolean` | 隐藏控制台窗口，等价于 `--no-console`。仅 Windows 生效 |
| `sanitize` | `string` | 启用 Sanitizer，等价于 `--sanitize`，例如 `address`、`undefined` |
| `sources` | `array` | 源码文件或目录列表。支持 `.php`、`.cpp`、`.c`、`.s`、`.m`、`.mm`。可使用 `if` + `path` 按 PHP 版本或操作系统条件加载 |
| `ignore` | `array` | 忽略的路径或扩展名。路径支持文件/目录；`ext-<name>` 格式用于忽略特定扩展依赖 |
| `build-dir` | `string` | 构建目录，等价于 `--build-dir`。支持相对路径和绝对路径；相对路径基于当前 YAML 文件所在目录解析 |
| `dry` | `boolean` | 干运行模式，等价于 `--dry` |
| `cxx-std` | `string` | C++ 标准版本，等价于 `--cxx-std` |
| `march` | `string` | 目标 CPU 指令集（如 `native`, `x86-64-v3`），等价于 `--march` |
| `target-platform` | `string` | 交叉编译目标平台三元组，等价于 `--target-platform` |
| `wasm` | `string` | 启用 WASI 0.2 构建。只允许 `component` 或 `browser`，不接受布尔值 |
| `wasm-browser-dir` | `string` | `wasm: browser` 时 Jco 浏览器模块的输出目录；相对路径以 YAML 所在目录为基准 |
| `wasm-package` | `string` | library component 的 WIT package，格式为 `namespace:name@major.minor.patch`；默认根据项目名称生成 |
| `wasm-world` | `string` | library component 的 WIT world 名称，使用小写连字符命名；默认使用项目名称 |
| `cpp-compiler` | `string` | 自定义 C++ 编译器（如 `g++`、`clang++`） |
| `cxx-flags` | `string` 或 `array` | 额外的 C++ 编译选项，会追加到编译命令中 |
| `ld-flags` | `string` 或 `array` | 额外的链接选项，会追加到链接命令中 |
| `include-paths` | `array` | 额外的 C++ 头文件搜索目录，等价于 `-I` |
| `defines` | `array` | 预处理器宏定义，等价于 `-D`。每项为 `name=value` 格式 |
| `lto` | `boolean` | 启用链接时优化，等价于 `--lto` |
| `format` | `boolean` | 启用 clang-format 格式化，等价于 `--format` |
| `link-libs` | `array` | 要链接的库，等价于 `-l`。每项为库名（不含 `lib` 前缀和 `.so`/`.a` 后缀） |
| `link-paths` | `array` | 库搜索路径，等价于 `-L`。每项为目录路径 |
| `resource` | `object` | Windows 平台资源配置（图标等） |

## WASM 项目

使用 Wasmtime 运行的项目配置为 Component：

```yaml
name: hello
mode: bin
wasm: component
sources:
  - src
build-dir: build
output: dist/hello.wasm
```

使用浏览器运行时显式选择 browser profile：

```yaml
name: browser-app
mode: bin
wasm: browser
sources:
  - src
output: component/browser-app.wasm
wasm-browser-dir: generated
```

需要从 JavaScript 或其他 Component Host 多次调用 TypePHP 函数时，使用 library 模式：

```yaml
name: calculator
mode: library
wasm: browser
wasm-package: app:calculator@1.0.0
wasm-world: calculator

sources:
  - src

output: component/calculator.wasm
wasm-browser-dir: generated
```

library 模式至少需要一个 [`#[WasmExport]`](wasm-export.md) 函数，不使用 `main()` 作为自动入口。

`wasm` 不是 boolean 开关，`wasm: true` 和 `wasm: false` 都是无效配置。WASM 项目省略 `target-platform` 时默认使用 `wasm32-wasip2`。详细的工具链安装、运行方法、浏览器 Host 和平台限制见[编译到 WebAssembly](wasm.md)。

## name 与 output 的差异

`name` 和 `output` 都可以影响最终产物名称，但语义不同：

- `name` 只设置产物文件名，不设置输出目录。
- `output` 设置完整输出路径，等价于命令行 `-o` / `--output`。
- `name` 不会按 YAML 文件所在目录解析；产物默认生成在执行编译命令时的当前工作目录。
- `output` 如果包含目录部分，会同时设置输出目录和文件名；相对路径按 YAML 文件所在目录解析。
- 同时配置 `output` 和 `name` 时，以 `output` 为准。

例如：

```yaml
name: tetris
sources:
  - main.php
```

执行：

```bash
./tpc examples/tetris-sdl/project.yml
```

会生成：

```text
./tetris
```

而不是：

```text
examples/tetris-sdl/tetris
```

如果希望显式输出到 YAML 目录下，应使用 `output`：

```yaml
output: tetris
sources:
  - main.php
```

## 优先级

命令行参数 > YAML 配置 > 默认值。

例如，YAML 中设置了 `cxx-std: c++17`，但命令行传入 `--cxx-std=c++20`，最终使用 `c++20`。

## 条件 sources

`sources` 支持按条件加载。字符串写法保持兼容；需要条件时使用 `if` 和 `path`。`if` 与 `path` 是同一条 source 的映射字段，顺序无关；推荐将 `if` 写在前面，便于先看到加载条件。

```yaml
sources:
  - main.php
  - if: PHP_VERSION_ID >= 80400
    path: compat/php84.php
  - if: PHP_VERSION >= "8.4.0"
    path: compat/php84-version.php
  - if: PHP_VERSION_ID >= 80200 && PHP_VERSION_ID < 80400
    path: compat/php82.php
  - if: PHP_OS_FAMILY == "Linux"
    path: platform/linux.php
```

条件左值仅支持 `PHP_VERSION_ID`、`PHP_VERSION`、`PHP_OS_FAMILY` 3 种常量。支持使用 `&&`、`||`、`!` 和括号组合多个条件。
条件为 `false` 时该 `source` 会被跳过，跳过项不要求文件存在。此能力仅适用于 `YAML` 配置文件，不提供对应命令行参数。

### PHP_VERSION_ID
条件右值必须是合法的 PHP 版本 ID 数字。`PHP_VERSION_ID` 会先转换为 `PHP_VERSION` 字符串再进行比较，例如 `80400` 转为 `8.4.0`。

### PHP_VERSION
条件右值必须是合法的 PHP 版本号字符串。操作符支持：`<`、`lt`、`<=`、`le`、`>`、`gt`、`>=`、`ge`、`==`、`=`、`eq`、`!=`、`<>`、`ne`。
内部使用 `version_compare()` 处理。

### PHP_OS_FAMILY
条件右值必须是合法的操作系统族名字符串：`Windows`、`BSD`、`Darwin`、`Solaris`、`Linux`、`Unknown`。操作符仅支持：`==`、`!=`。

## 不使用 YAML 配置文件

也可以直接编译单个 PHP 文件或目录，无需创建 YAML 配置文件：

```shell
# 编译单个文件
./tpc hello.php -O2

# 使用自定义 YAML 文件名
./tpc myproject.yml -O2

# 编译整个目录
./tpc myproject/ -O2 -o myapp
```

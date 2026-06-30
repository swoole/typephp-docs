对于多文件项目，推荐使用 YAML 配置文件管理项目构建参数。配置文件通常命名为 `project.yml`，也可以使用任意 `*.yml` 文件名，例如 `myproject.yml`，编译时通过命令行参数传入对应路径即可。

命令行参数优先级最高，其次是 YAML 配置，最后是默认值。

## 完整示例

```yaml
# 输出文件名（等价于 -o/--output）
output: build/myapp

# 构建模式：bin(可执行文件) / ext(扩展) / binary / extension / cli
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
| `output` / `name` | `string` | 输出文件名，等价于 `-o` / `--output`。可包含目录部分，例如 `build/myapp` |
| `mode` / `build-mode` / `type` | `string` | 构建模式，等价于 `-m`。支持 `bin`/`binary`/`cli`（可执行文件）和 `ext`/`extension`（扩展） |
| `optimize` | `integer` | 优化级别，等价于 `-O`，取值范围 `0` ~ `3` |
| `job` | `integer` | 并行编译任务数，等价于 `-j` / `--job` |
| `debug` | `boolean` | 启用调试模式，等价于 `-d` / `--debug` |
| `profile` | `boolean` | 启用 CPU 性能分析，等价于 `--profile`。仅 Linux 支持 |
| `no-literal-strings` | `boolean` | 禁用字符串字面量优化，等价于 `--no-literal-strings` |
| `no-progress` | `boolean` | 禁用进度条输出，等价于 `--no-progress` |
| `no-console` | `boolean` | 隐藏控制台窗口，等价于 `--no-console`。仅 Windows 生效 |
| `sanitize` | `string` | 启用 Sanitizer，等价于 `--sanitize`，例如 `address`、`undefined` |
| `sources` | `array` | 源码文件或目录列表。支持 `.php`、`.cpp`、`.c`、`.s`、`.m`、`.mm` |
| `ignore` | `array` | 忽略的路径或扩展名。路径支持文件/目录；`ext-<name>` 格式用于忽略特定扩展依赖 |
| `build-dir` | `string` | 构建目录，等价于 `--build-dir`。支持相对路径和绝对路径；相对路径基于当前 YAML 文件所在目录解析 |
| `dry` | `boolean` | 干运行模式，等价于 `--dry` |
| `cxx-std` | `string` | C++ 标准版本，等价于 `--cxx-std` |
| `march` | `string` | 目标 CPU 指令集（如 `native`, `x86-64-v3`），等价于 `--march` |
| `target-platform` | `string` | 交叉编译目标平台三元组，等价于 `--target-platform` |
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

## 优先级

命令行参数 > YAML 配置 > 默认值。

例如，YAML 中设置了 `cxx-std: c++17`，但命令行传入 `--cxx-std=c++20`，最终使用 `c++20`。

## 不使用 YAML 配置文件

也可以直接编译单个 PHP 文件或目录，无需创建 YAML 配置文件：

```shell
# 编译单个文件
php bin/compiler.php hello.php -O2

# 使用自定义 YAML 文件名
php bin/compiler.php myproject.yml -O2

# 编译整个目录
php bin/compiler.php myproject/ -O2 -o myapp
```

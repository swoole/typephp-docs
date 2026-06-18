## 编译参数

AOT 编译器支持命令行参数和 `project.yml` 配置文件两种方式指定编译选项。命令行参数优先级最高，其次是 YAML 配置，最后是默认值。

### 用法

```shell
php bin/compiler.php <file/dir/project.yml> [options]
```

### 命令行参数

#### 基本参数

**`-O <level>`** — 设置 GCC 优化级别。

- 取值范围：`0` ~ `3`，默认 `0`
- `-O0`：不优化，编译速度最快，适合调试
- `-O2`：常用的发布优化级别，平衡编译时间和运行性能
- `-O3`：最高优化级别，激进的函数内联和向量化

```shell
php bin/compiler.php app.php -O2
```

**`-o, --output <file>`** — 指定输出文件名。

默认使用入口文件或目录的基本名。构建二进制时生成可执行文件，构建扩展时生成 `.so`/`.dll`。

```shell
php bin/compiler.php src/ -o myapp
```

**`-m, --mode <mode>`** — 构建模式。

- `bin`（默认）：编译为独立的可执行文件
- `ext`：编译为 PHP 扩展（`.so`/`.dll`）

```shell
php bin/compiler.php myext/ -m ext -o myext
```

#### 调试与诊断

**`-d, --debug`** — 启用调试模式。

自动禁用优化并添加调试符号（`-g`），便于使用 GDB/LLDB 调试生成的二进制文件。

```shell
php bin/compiler.php app.php -d
```

**`--sanitize <type>`** — 启用 Sanitizer。

在生成的 C++ 代码中启用编译器 Sanitizer，用于检测内存错误和未定义行为。

- `address` — AddressSanitizer（内存越界、use-after-free 等）
- `undefined` — UndefinedBehaviorSanitizer（整数溢出、空指针等）

```shell
php bin/compiler.php app.php --sanitize=address
```

#### 性能相关

**`--profile`** — 启用 CPU 性能分析（基于 gperftools，仅 Linux）。

在生成的二进制中插入性能分析探针并自动链接 `-lprofiler`，运行后生成 `{target}.prof` 数据文件。启用后自动强制重编译 misc 文件以确保编译宏生效。

```shell
php bin/compiler.php app.php --profile
./app                                    # 运行后生成 app.prof
```

使用编译器内置的 pprof 分析功能（默认 web 模式，自动打开浏览器）：

```shell
php bin/compiler.php app.prof
```

**`-j, --job <num>`** — 并行编译任务数。

控制同时编译的 C++ 文件数量，类似 `make -j`。默认值为 `4`。在 CPU 核心数较多的机器上可适当增大。

```shell
php bin/compiler.php app.php -j8
```

**`--no-literal-strings`** — 禁用字符串字面量优化。

默认情况下，编译器将字面量字符串优化为 `const char*` 指针以提升性能。某些场景下（如需要 `zend_string` 兼容性）可能需要关闭此优化。

```shell
php bin/compiler.php app.php --no-literal-strings
```

#### 其他

**`-f, --force`** — 强制重新编译。

即使存在编译缓存也强制重新编译所有文件。

```shell
php bin/compiler.php app.php -f
```

**`--cxx-std <version>`** — 指定 C++ 标准版本。

默认使用 `c++17`。如果你的代码或依赖的库需要特定的 C++ 版本，可通过此参数指定。

```shell
php bin/compiler.php app.php --cxx-std=c++20
```

**`--build-dir <dir>`** — 指定构建目录。

设置生成的 C++ 代码（`.cc`）、头文件、目标文件（`.o`）等构建中间产物的存放目录。默认目录为 `<项目根>/build`。

```shell
php bin/compiler.php app.php --build-dir /tmp/mybuild
```

**`--dry`** — 干运行模式。

仅执行 PHP → C++ 代码的转译步骤，生成所有 C++ 源文件，但**不执行**编译和链接。适用于仅需审查生成的 C++ 代码或预先检查转译结果的场景。

```shell
php bin/compiler.php app.php --dry
```

**`--no-console`** — 隐藏控制台窗口（仅 Windows）。

Windows 平台下，使用 `/SUBSYSTEM:WINDOWS` 链接，程序启动时不显示控制台窗口，适用于 GUI 应用程序。

```shell
php bin/compiler.php gui-app.php --no-console
```

**`-v, --version`** — 显示版本信息。

**`-h, --help`** — 显示帮助信息。

### project.yml 配置

对于多文件项目，推荐使用 `project.yml` 配置文件管理项目构建参数。`project.yml` 放置在项目根目录。

#### 完整示例

```yaml
# 项目名称（输出文件名）
name: myapp

# 构建模式：bin(可执行文件) / ext(扩展) / binary / extension / cli
build-mode: bin

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

# 构建目录（生成的 C++ 代码存放位置，也可用 --build-dir 参数）
build-dir: /path/to/custom/build

# C++ 标准版本
cxx-std: c++17

# 自定义 C++ 编译器（gcc/g++/clang/clang++ 等）
cpp-compiler: g++

# 额外的 C++ 编译选项
cxx-flags:
  - -march=native
  - -mtune=native

# 额外的链接选项
ld-flags:
  - -lcurl
  - -lssl

# Windows 资源配置（仅 Windows 平台）
resource:
  icon: assets/app.ico
```

#### 配置项说明

| 配置项 | 类型 | 说明 |
|--------|------|------|
| `name` | `string` | 输出文件名，等价于 `-o` |
| `build-mode` / `type` | `string` | 构建模式，等价于 `-m`。支持 `bin`/`binary`/`cli`（可执行文件）和 `ext`/`extension`（扩展） |
| `sources` | `array` | 源码文件或目录列表。支持 `.php`、`.cpp`、`.c`、`.s`、`.m`、`.mm` |
| `ignore` | `array` | 忽略的路径或扩展名。路径支持文件/目录；`ext-<name>` 格式用于忽略特定扩展依赖 |
| `build-dir` / `build_dir` | `string` | 构建目录，等价于 `--build-dir`。设置生成的 C++ 代码和中间文件的存放目录 |
| `cxx-std` / `cxx_std` | `string` | C++ 标准版本，等价于 `--cxx-std` |
| `cpp-compiler` / `cpp_compiler` | `string` | 自定义 C++ 编译器（如 `g++`、`clang++`） |
| `cxx-flags` / `cxxflags` | `string` 或 `array` | 额外的 C++ 编译选项，会追加到编译命令中 |
| `ld-flags` / `ldflags` | `string` 或 `array` | 额外的链接选项，会追加到链接命令中 |
| `resource` | `object` | Windows 平台资源配置（图标等） |

#### 优先级

命令行参数 > `project.yml` > 默认值。

例如，`project.yml` 中设置了 `cxx-std: c++17`，但命令行传入 `--cxx-std=c++20`，最终使用 `c++20`。

#### 不使用 project.yml

也可以直接编译单个 PHP 文件或目录，无需创建 `project.yml`：

```shell
# 编译单个文件
php bin/compiler.php hello.php -O2

# 编译整个目录
php bin/compiler.php myproject/ -O2 -o myapp
```

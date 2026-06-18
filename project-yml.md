对于多文件项目，推荐使用 `project.yml` 配置文件管理项目构建参数。`project.yml` 放置在项目根目录。

命令行参数优先级最高，其次是 YAML 配置，最后是默认值。

## 完整示例

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

# 额外的 C++ 头文件搜索路径（等价于 -I）
include-paths:
  - /opt/mylib/include
  - ../shared/headers

# 预处理器宏定义（等价于 -D）
defines:
  - ENABLE_LOGGING=1
  - DEBUG_LEVEL=3

# 启用链接时优化
lto: true

# 启用 clang-format 代码格式化
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
| `name` | `string` | 输出文件名，等价于 `-o` |
| `build-mode` / `type` | `string` | 构建模式，等价于 `-m`。支持 `bin`/`binary`/`cli`（可执行文件）和 `ext`/`extension`（扩展） |
| `sources` | `array` | 源码文件或目录列表。支持 `.php`、`.cpp`、`.c`、`.s`、`.m`、`.mm` |
| `ignore` | `array` | 忽略的路径或扩展名。路径支持文件/目录；`ext-<name>` 格式用于忽略特定扩展依赖 |
| `build-dir` | `string` | 构建目录，等价于 `--build-dir`。设置生成的 C++ 代码和中间文件的存放目录 |
| `cxx-std` | `string` | C++ 标准版本，等价于 `--cxx-std` |
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

命令行参数 > `project.yml` > 默认值。

例如，`project.yml` 中设置了 `cxx-std: c++17`，但命令行传入 `--cxx-std=c++20`，最终使用 `c++20`。

## 不使用 project.yml

也可以直接编译单个 PHP 文件或目录，无需创建 `project.yml`：

```shell
# 编译单个文件
php bin/compiler.php hello.php -O2

# 编译整个目录
php bin/compiler.php myproject/ -O2 -o myapp
```

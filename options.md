## 编译参数

AOT 编译器支持命令行参数和 [project.yml 配置](project-yml.md) 两种方式指定编译选项。命令行参数优先级最高，其次是 YAML 配置，最后是默认值。

### 用法

```shell
php bin/compiler.php <file/dir/project.yml> [options]
```

## 命令行参数

### `-O <level>` — 优化级别

- 取值范围：`0` ~ `3`，默认 `0`
- `-O0`：不优化，编译速度最快，适合调试
- `-O2`：常用的发布优化级别，平衡编译时间和运行性能
- `-O3`：最高优化级别，激进的函数内联和向量化

```shell
php bin/compiler.php app.php -O2
```

### `-o, --output <file>` — 输出文件名

默认使用入口文件或目录的基本名。构建二进制时生成可执行文件，构建扩展时生成 `.so`/`.dll`。

```shell
php bin/compiler.php src/ -o myapp
```

### `-m, --mode <mode>` — 构建模式

- `bin`（默认）：编译为独立的可执行文件
- `ext`：编译为 PHP 扩展（`.so`/`.dll`）

```shell
php bin/compiler.php myext/ -m ext -o myext
```

### `-d, --debug` — 调试模式

自动禁用优化并添加调试符号（`-g`），便于使用 GDB/LLDB 调试生成的二进制文件。

```shell
php bin/compiler.php app.php -d
```

### `--sanitize <type>` — Sanitizer

在生成的 C++ 代码中启用编译器 Sanitizer，用于检测内存错误和未定义行为。

- `address` — AddressSanitizer（内存越界、use-after-free 等）
- `undefined` — UndefinedBehaviorSanitizer（整数溢出、空指针等）

```shell
php bin/compiler.php app.php --sanitize=address
```

### `--profile` — CPU 性能分析

基于 gperftools（仅 Linux）。在生成的二进制中插入性能分析探针并自动链接 `-lprofiler`，运行后生成 `{target}.prof` 数据文件。启用后自动强制重编译 misc 文件以确保编译宏生效。

```shell
php bin/compiler.php app.php --profile
./app                                    # 运行后生成 app.prof
```

使用编译器内置的 pprof 分析功能（默认 web 模式，自动打开浏览器）：

```shell
php bin/compiler.php app.prof
```

### `-j, --job <num>` — 并行编译任务数

控制同时编译的 C++ 文件数量，类似 `make -j`。默认值为 `4`。在 CPU 核心数较多的机器上可适当增大。

```shell
php bin/compiler.php app.php -j8
```

### `--no-literal-strings` — 禁用字符串字面量优化

默认情况下，编译器将字面量字符串优化为 `const char*` 指针以提升性能。某些场景下（如需要 `zend_string` 兼容性）可能需要关闭此优化。

```shell
php bin/compiler.php app.php --no-literal-strings
```

### `-I <dir>, --include-path <dir>` — C++ 头文件搜索路径（可重复）

将指定目录追加到 C++ 编译器的 include 搜索路径末尾。系统默认路径（PHP 头文件、PHPX 头文件）始终优先，用户路径追加在其后。

允许多次指定以添加多个目录：

```shell
# 添加多个 include 目录
php bin/compiler.php app.php -I /opt/mylib/include -I ../shared/headers -O2

# 长格式等价写法
php bin/compiler.php app.php --include-path /opt/mylib/include --include-path ../shared/headers
```

### `-D <macro>, --define <macro>` — C++ 预处理器宏（可重复）

等价于在 C++ 代码中使用 `#define`，用于条件编译、功能开关等场景。使用 `name=value` 格式，`=` 后的值可选。

编译时自动适配编译器：GCC/Clang 使用 `-D<macro>`，MSVC 使用 `/D<macro>`。

```shell
# 定义宏控制功能开关
php bin/compiler.php app.php -D ENABLE_LOGGING=1 -D DEBUG_LEVEL=3 -O2

# 长格式等价写法
php bin/compiler.php app.php --define ENABLE_LOGGING=1 --define DEBUG_LEVEL=3
```

### `--lto` — 链接时优化（Link Time Optimization）

允许编译器在链接阶段跨编译单元进行优化，配合 `-O2`/`-O3` 可进一步提升运行时性能和减小二进制体积。编译器自动适配：GCC/Clang 使用 `-flto`，MSVC 使用 `/GL` + `/LTCG`。

⚠️ 会增加链接时间，建议仅在发布构建时使用。

```shell
# 生产环境启用 LTO
php bin/compiler.php app.php -O2 --lto
```

### `--format` — clang-format 代码格式化

启用后，编译器在每个 C++ 文件生成后调用 `clang-format -i` 进行自动格式化。由于格式化会消耗额外时间，默认关闭。需要系统已安装 `clang-format`。

```shell
# 启用代码格式化
php bin/compiler.php app.php --format
```

### `-f, --force` — 强制重新编译

即使存在编译缓存也强制重新编译所有文件。

```shell
php bin/compiler.php app.php -f
```

### `--cxx-std <version>` — C++ 标准版本

默认使用 `c++17`。如果你的代码或依赖的库需要特定的 C++ 版本，可通过此参数指定。

```shell
php bin/compiler.php app.php --cxx-std=c++20
```

### `--build-dir <dir>` — 构建目录

设置生成的 C++ 代码（`.cc`）、头文件、目标文件（`.o`）等构建中间产物的存放目录。默认目录为 `<项目根>/build`。

```shell
php bin/compiler.php app.php --build-dir /tmp/mybuild
```

### `--dry` — 干运行模式

仅执行 PHP → C++ 代码的转译步骤，生成所有 C++ 源文件，但**不执行**编译和链接。适用于仅需审查生成的 C++ 代码或预先检查转译结果的场景。

```shell
php bin/compiler.php app.php --dry
```

### `--no-console` — 隐藏控制台窗口（仅 Windows）

Windows 平台下，使用 `/SUBSYSTEM:WINDOWS` 链接，程序启动时不显示控制台窗口，适用于 GUI 应用程序。

```shell
php bin/compiler.php gui-app.php --no-console
```

### `-v, --version` — 显示版本信息

### `-h, --help` — 显示帮助信息

---

> 💡 多文件项目推荐使用 [project.yml 配置文件](project-yml.md) 统一管理项目构建参数。

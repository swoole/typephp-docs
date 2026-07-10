# 调试指南

TypePHP 编译器将 PHP 代码翻译为 C++ 后编译成原生二进制，因此调试工具链基于 GDB/LLDB，不支持 Xdebug、Zend 断点等运行时调试器。

---

## 前置条件

调试前需确保以下三项配置到位：

1. **关闭优化** — 优化等级设为 `-O0`，否则函数可能被内联、变量可能被优化消除
2. **开启调试符号** — 编译时添加 `--debug` 参数，等效于传递 `-g -O0` 给 C++ 编译器
3. **PHPX / PHP 使用 Debug 构建**（推荐）— 便于在 PHP 内部函数中设置断点

```bash
# PHPX Debug 构建
cmake . -DCMAKE_BUILD_TYPE=Debug && make -j$(nproc)

# PHP Debug 构建
./configure --enable-debug && make -j$(nproc)
```

---

## 编译命令

### 基本用法

```bash
# 生成可调试的二进制文件
./tpc app.php --debug

# 使用 project.yml
./tpc project.yml --debug
```

`--debug` 自动设置 `-O0` 并追加 `-g` 编译标志，关闭所有优化。

### 其他调试相关参数

| 参数 | 作用 |
|------|------|
| `-d`, `--debug` | 禁用优化，添加调试符号 |
| `-O0` | 单独设置优化级别为 0（等同于 `--debug` 的优化部分） |
| `--sanitize=address` | 启用 AddressSanitizer，检测内存越界、use-after-free 等 |
| `--sanitize=undefined` | 启用 UndefinedBehaviorSanitizer，检测整数溢出、空指针等 |
| `-j <num>` | 并行编译任务数（等同于 `make -j`），默认 4 |

```bash
# ASAN + 调试符号，快速定位内存错误
./tpc app.php --debug --sanitize=address

```

### 查看生成的 C++ 代码

编译后的 `.cc` 和 `.o` 文件位于 `build/` 目录。可以查阅生成的 C++ 源码来理解编译器翻译结果：

```bash
ls build/*.cc
# 输出示例：
# build/main.cc
# build/src__utils.cc
# build/src__database.cc
```

---

## GDB / LLDB 调试

### 启动

```bash
gdb ./hello
```

LLDB（macOS）：

```bash
lldb ./hello
```

### 函数符号命名规则

所有 AOT 编译的函数在二进制中以 C 链接符号导出，命名规则如下：

#### 普通函数

```
php_{函数名}
```

若源码中使用命名空间，则斜杠 `/` 替换为双下划线 `__`。**函数名全部小写**。

```php
// PHP 源码
function my_add(int $a, int $b): int { ... }
Foo\Bar\baz();

// 对应符号
php_my_add
php_foo__bar__baz
```

#### 类方法

```
php_{类名}__{方法名}
```

若类有命名空间，命名空间作为前缀用 `__` 分隔。**全部小写**。

```php
// PHP 源码
class UserService {
    public function create(array $data): int { ... }
}

// 对应符号
php_userservice__create
```

#### 内置函数与方法

| 类型 | 符号格式 | 示例 |
|------|----------|------|
| 内置函数 | `zif_{函数名}` | `zif_array_merge` |
| 内置方法 | `zim_{类名}_{方法名}` | `zim_arrayobject_count` |

> 内置函数/方法的符号格式取决于具体扩展的实现，上述为常见命名惯例。

### 断点设置示例

```bash
(gdb) b php_my_add         # 在 my_add 函数入口设断点
(gdb) b php_userservice__create  # 在 UserService::create 入口设断点
(gdb) b main.cc:42         # 在生成的 C++ 文件指定行设断点
(gdb) r                    # 运行
```

```bash
Breakpoint 1, php_my_add (a=1, b=2) at /home/swoole/workspace/aot/build/examples/myext/test.cc:9
9       php::Int tmp_var_0 = 0;
```

### 查看变量

#### 原生 C++ 类型

`int64_t`、`double`、`bool` 等可直接用 `print` 查看：

```bash
(gdb) print a
$1 = 1
(gdb) print b
$2 = 2
```

#### PHP 类型（php::Variant / php::Array / php::String 等）

使用 `.print()` 方法输出 PHP 风格的可读表示：

```bash
(gdb) call env.print()
         array(61) {
           ["SHELL"]=>
           string(9) "/bin/bash"
           ["SESSION_MANAGER"]=>
           string(71) "local/swoole-26:@/tmp/.ICE-unix/4392,unix/swoole-26:/tmp/.ICE-unix/4392"
           ...
         }
```

`.print()` 方法在 Debug 构建中可用（PHpx 内部在 `DEBUG` 宏下编译），输出格式与 PHP `var_dump()` 类似。

常用调试调用：

```bash
(gdb) call var.print()      # 打印 php::Variant 值
(gdb) call arr.print()      # 打印 php::Array 内容
(gdb) call str.toCString()  # 获取 php::String 的 C 字符串
(gdb) call obj.print()      # 打印 php::Object 内容
```

#### Box 对象（BigInt / Decimal / BigFloat 等）

Box 对象不支持直接用 `print` 查看，需要通过静态方法转换：

```bash
# 查看 BigInt 的值
(gdb) call php::BigInt::toString(bi).print()

# 查看 Decimal 的值
(gdb) call php::Decimal::toString(dec).print()
```

### 常用 GDB 命令速查

| 命令 | 说明 |
|------|------|
| `b <sym>` | 在符号处设置断点 |
| `b <file>:<line>` | 在文件行号设置断点 |
| `b <sym> if <cond>` | 条件断点：`b php_my_func if a > 10` |
| `r` / `run` | 启动程序 |
| `c` / `continue` | 继续执行 |
| `n` / `next` | 单步跳过（不进入函数） |
| `s` / `step` | 单步进入函数 |
| `finish` | 执行到当前函数返回 |
| `p <var>` / `print` | 打印变量值 |
| `info locals` | 查看当前栈帧所有局部变量 |
| `info args` | 查看当前函数参数 |
| `bt` / `backtrace` | 查看调用堆栈 |
| `frame <n>` | 切换到第 n 层栈帧 |
| `x/10x <ptr>` | 以十六进制查看指针指向的 10 个字 |
| `watch <var>` | 监视变量变化 |
| `disas` | 反汇编当前函数 |

### LLDB 对应命令

| 操作 | GDB | LLDB |
|------|-----|------|
| 断点 | `b php_foo` | `b php_foo` |
| 运行 | `r` | `r` |
| 单步跳过 | `n` | `thread step-over` |
| 单步进入 | `s` | `thread step-in` |
| 打印变量 | `p var` | `frame variable var` |
| 查看堆栈 | `bt` | `bt` |
| 调用方法 | `call obj.print()` | `expression obj.print()` |

---

## 内存错误检测

### AddressSanitizer (ASAN)

```bash
# 编译时启用
./tpc app.php --sanitize=address

# 环境变量控制行为
export ASAN_OPTIONS=detect_leaks=1:abort_on_error=1:halt_on_error=1
./app
```

常用 ASAN 选项：

| 选项 | 说明 |
|------|------|
| `detect_leaks=1` | 程序退出时检测内存泄漏 |
| `abort_on_error=1` | 首次错误即 abort（生成 core dump） |
| `halt_on_error=1` | 首次错误即退出（不生成 core） |
| `log_path=/tmp/asan` | 报告输出路径前缀 |

ASAN 可检测：

- 堆/栈/全局缓冲区溢出
- use-after-free / use-after-return
- 双重释放
- 内存泄漏（需 `detect_leaks=1`）

### Valgrind (Linux)

```bash
# 完整泄漏检测
valgrind --leak-check=full \
         --show-leak-kinds=all \
         --track-origins=yes \
         ./app

# 只对特定程序使用
valgrind --leak-check=full --log-file=vg.log ./app
```

> **注意**：PHP 的 `zend_alloc` 会干扰 Valgrind 检测。如果怀疑泄漏来自 phpx 运行时层，可在编译时禁用 ZendMM，或先设置 `USE_ZEND_ALLOC=0`。

### 泄漏常见原因

| 模式 | 原因 |
|------|------|
| Box 对象不释放 | PHP 变量未 unset / 循环引用 / GC 未触发 |
| C++ `new` 无 `delete` | 手写的 C++ 扩展代码未管理好生命周期 |
| 数组元素泄漏 | `php::Array` 持有大量元素，生命周期过长 |

---

## 常见问题排查

### 1. 段错误 (Segfault)

**排查步骤**：

```bash
# 1. 用 GDB 定位崩溃点
gdb ./app
(gdb) run
# 崩溃后
(gdb) bt         # 查看调用堆栈
(gdb) frame 0    # 定位崩溃帧
(gdb) info locals

# 2. 启用 ASAN 重新构建
./tpc app.php --sanitize=address
./app
```

**常见原因**：

- `nullptr` 解引用 — 检查 `.toBox<T>()` 返回值是否为空
- 数组越界 — 检查 `operator[]` 的索引范围
- use-after-free — Box 对象提前释放但仍有引用
- 类型转换错误 — `toObject()` 指定的类与实际类型不匹配

### 2. 编译错误

TypePHP 编译器在翻译阶段就可能报错（比 C++ 编译器更早），常见编译期错误：

| 错误 | 原因 |
|------|------|
| `Cannot re-assign variable from X to Y` | 变量类型不可变，声明为 `int` 后不能再赋值为 `string` |
| `Undefined variable` | 变量必须先定义后使用，不支持 `isset()` 检测未定义变量 |
| `declare(strict_types=0) is not allowed` | 仅支持严格模式 `declare(strict_types=1)` |
| `Cannot convert float to Decimal/BigInt` | 浮点字面量不能直接转换为高精度类型，需使用字符串 |
| 函数调用参数数量不匹配 | 必须传够函数声明中的所有参数 |

### 3. 运行时行为差异

AOT 二进制与 ZendPHP 的行为差异（非 bug，是 AOT 编译的固有特性）：

- **类型错误是硬错误** — ZendPHP 会隐式转换类型，TypePHP 编译器直接报 Fatal Error
- **除法行为** — `$a / $b` 在 AOT 中与 C++ 行为一致：整数除法结果仍为整数（除非 `any()`）
- **字符串拼接** — 非字符串与字符串拼接需要显式转换，无自动 `toString()`
- **未定义变量** — ZendPHP 发 Warning，AOT 直接编译报错

### 4. Box 对象相关

```bash
# Box 对象看起来是 "resource" 类型，调试时确认
(gdb) call box.isResource()
$1 = true

# 确认 toBox 返回非空
(gdb) call box.toBox<BigInt>().toString(box).print()

# 检查是否是预期的 Box 子类（通过类型信息）
# Box 内部有 type_info / extra_info 字段可用于识别
```

---

## 调试生成的 C++ 代码

当需要深入理解编译器行为或排查代码生成问题时，可以直接检查和调试生成的 `.cc` 文件。

### 查看生成的代码

```bash
ls build/*.cc
cat -n build/main.cc | head -100
```

### 在生成的代码中设置断点

```bash
gdb ./app
(gdb) b build/main.cc:42     # 在生成代码的指定行设断点
(gdb) r
```

### 理解生成代码的关键标识

| 生成代码模式 | 对应 PHP 源码 |
|-------------|-------------|
| `tmp_var_N` | 编译器生成的临时变量 |
| `php_my_func()` | PHP 函数 `my_func()` |
| `php_myclass__mymethod()` | 类 `MyClass::myMethod()` |
| `php::toBigInt(expr)` | BigInt 类型转换 |
| `php::toDecimal(expr)` | Decimal 类型转换 |
| `php::BigInt::add(a, b)` | BigInt 加法运算 |
| `php::zend_call("func", args)` | 动态函数调用 |

---

## 多线程 / 协程调试

TypePHP 编译器支持 Swoole/Swow 协程。协程调试时注意：

```bash
# 查看所有线程
(gdb) info threads

# 切换到指定线程
(gdb) thread <n>

# 所有线程的调用堆栈
(gdb) thread apply all bt
```

---

## 环境变量

| 变量 | 作用 |
|------|------|
| `ASAN_OPTIONS` | 控制 AddressSanitizer 行为 |
| `USE_ZEND_ALLOC=0` | 禁用 PHP 内存管理器（配合 Valgrind） |
| `GDBHISTFILE` | GDB 命令历史文件路径 |

---

## 快速诊断流程

**崩溃**：
1. `--debug` 重新构建
2. GDB 获取 `bt` 堆栈
3. 检查崩溃帧的 `info locals`
4. ASAN 辅助定位

**内存问题**：
1. `--sanitize=address` 重新构建
2. 或 Valgrind `--leak-check=full`
3. 重点检查 Box 对象生命周期和数组引用

**行为不符合预期**：
1. 检查 `build/*.cc` 生成的代码是否与预期一致
2. 确认变量类型是否符合预期（`info locals`）
3. 在关键函数入口设断点，单步执行比对

**扩展加载失败**：
1. `ldd` 检查依赖库
2. `php -d display_startup_errors=1 -d extension=<path>` 查看详细错误
3. `nm -D` 确认符号导出

---

*本文档最后更新：2026-06-03*

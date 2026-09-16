Nano 模式用于生成不携带 PHP 解释器和 ZendVM 的 TypePHP 原生程序。PHP 仍用于
运行编译器，Composer 仍用于安装编译期依赖；它们不会成为最终程序的运行时依赖。

在 Linux、macOS、iOS 和 Android 目标上，TypePHP 会读取 Composer 包提供的源码
清单，把 PHP Nano、PHPX 和生成的 C++ 一起编译。最终程序不链接 `libphp`，也不能
在运行时解释或加载 PHP 源码。

## 与普通模式的区别

| 项目 | 普通模式 | Nano 模式（非 Windows） |
|---|---|---|
| PHP 运行时 | 链接完整的 `libphp` | PHP Nano 源码直接参与应用编译 |
| ZendVM | 可由完整运行时提供 | 不包含 |
| PHPX | 链接 PHPX 库 | PHPX 源码以 `PHPX_NANO` 模式参与编译 |
| 扩展 | 由 PHP 配置和动态库提供 | 编译期由 Composer 静态选择 |
| 系统能力 | 取决于完整 PHP 构建 | 仅保留本地、可移植的能力集合 |
| 运行时依赖 | PHP/PHPX 共享库或静态库 | C/C++ 标准库、POSIX 和工具链运行库 |

Nano 不是重新实现一套 PHP 数据结构。它直接复用 PHP 8.6 的 `zval`、
`zend_string`、`zend_array`、对象、异常、GC、函数表和内置模块等源码，只移除
解释执行和不符合 Nano 能力边界的部分。因此普通类、继承、接口、异常、闭包和
同步的动态对象方法调用仍然可用。

## 安装

项目需要安装 TypePHP 和 PHP Nano。`swoole/typephp` 已依赖 `swoole/phpx`，无需
重复声明 PHPX：

```shell
composer require --dev swoole/typephp swoole/php-nano
```

Nano 的生产构建不使用 CMake。TypePHP 直接读取 `swoole/php-nano` 与
`swoole/phpx` 的 Composer 元数据，将其中列出的 `.c`、`.cc` 和 `.cpp` 文件加入
应用的 sources。C 文件按 C11 编译，C++ 与 TypePHP 生成代码按 C++17 编译。

当前用于启动 `vendor/bin/tpc.php` 的 PHP 必须满足 TypePHP 的版本要求。它只在
编译期运行；Nano 内部复用 PHP 8.6 源码，并不要求目标设备安装 PHP 8.6。

## 编译第一个程序

创建 `hello.php`：

```php
<?php

function main(int $argc, array $argv): void
{
    echo "Hello Nano\n";
    var_dump($argc, $argv);
}
```

编译并运行：

```shell
vendor/bin/tpc.php --nano hello.php
./hello first second
```

默认可执行文件生成在执行编译命令时的当前目录，本例为 `./hello`。中间文件保存
在通常的 `build` 目录中；也可以继续使用普通编译的参数：

```shell
vendor/bin/tpc.php --nano hello.php -O2 -j8 -o build/hello
vendor/bin/tpc.php --nano hello.php --run -- first second
```

Nano 与普通模式共用参数解析、代码生成、并行任务、进度显示、对象缓存和输出路径
规则。第一次需要编译 PHP Nano 和 PHPX 的大量源码，时间会明显长于普通应用增量
编译；后续构建会复用 `build` 中未失效的目标文件。使用 `--force` 才会忽略缓存，
使用 `--no-progress` 可将进度条改为逐文件日志。

单文件、源码目录和 `project.yml` 的输入方式均保持不变。非 Windows Nano 当前只
支持可执行文件，即 `-m bin`；还要求：

- 使用 C++17；
- 不与 `--full-static` 组合；
- 不使用 `-l`、`-L` 或对应的外部链接库配置。

PHP Nano 中 PHP 自带的 PCRE2、timelib 和 libbcmath 等源码会直接编入程序，不是
额外的动态链接依赖。

## 可用能力

Nano 内置经过裁剪的 Core、date、hash、json、pcre、random、Reflection、SPL、
standard 和 filter。主要保留：

- 字符串、数组、对象、资源、异常和垃圾回收；
- 普通类、继承、接口、闭包以及同步动态调用；
- 输出缓冲和控制台输入输出；
- `file` 与 `glob` 本地文件 stream，以及文件、目录、stat 和 hash 文件操作；
- 日期时间、时钟、休眠和随机数；
- 序列化、JSON、正则表达式和常用标准函数；
- Zend INI 注册表和模块 INI 值，但不会扫描或加载 `php.ini`；
- TypePHP 的 `BigInt`、`Decimal` 和 `BigFloat`。

`ctype_*` 是 TypePHP/PHPX 的编译器内建能力，不需要注册 ctype 扩展。

高精度类型在 Nano 中使用 PHP 8.6 内嵌的 libbcmath，而普通模式分别使用 GMP、
mpdecimal 和 MPFR。接口保持可用，但精度模型、舍入、极端数值范围、格式化和性能
不保证逐位一致，详见[高精度运算](math.md#nano-模式的高精度后端)。

## 不支持的能力

所有平台的 `--nano` 都不支持以下能力。对应语法和可静态识别的直接调用会在编译期
报错：

- `eval`、`include`、`include_once`、`require` 和 `require_once`；
- 匿名类；
- `yield`、`yield from`、Generator 和 Fiber；
- 反引号命令执行，以及 `shell_exec`、`exec`、`system`、`passthru`、`popen`、
  `proc_open`、`fork` 等进程或外部命令 API；
- socket、DNS、网络客户端与服务端、远程 stream；
- `dl`、动态扩展加载和运行时扩展发现；
- 信号、执行计时器、虚拟内存映射和系统日志等宿主控制能力。

不支持的公开函数和类原则上不会注册到 Nano 函数表；能够静态识别的直接调用会由
TypePHP 在生成 C++ 前报错。本地文件能力并不意味着支持远程 URL、进程管道或用户
自定义 stream wrapper。

## 静态 Composer 扩展

Nano 保留 Zend 扩展生命周期，但不提供运行时动态加载。额外扩展必须使用
`swoole/php-ext-*` 命名的 Composer 包，并在包的 `extra.typephp-native` 中声明
准确的源码、头文件目录、ABI 和 `zend_module_entry`。

安装扩展包后，TypePHP 会在编译时发现它，将扩展源码加入应用，并生成固定的扩展
注册表。原有的 MINIT、RINIT、RSHUTDOWN 和 MSHUTDOWN 生命周期保持不变；扩展
集合在程序构建完成后不可改变。扩展仍必须满足 Nano 的依赖和能力限制。

## WASI

Nano 可以与 WASI 目标组合：

```shell
vendor/bin/tpc.php --nano --wasm hello.php
wasmtime hello.wasm first second
```

也可以显式选择 Component 或浏览器输出：

```shell
vendor/bin/tpc.php --nano --wasm=component hello.php
vendor/bin/tpc.php --nano --wasm=browser hello.php
```

WASI 是 Native Nano 的更小能力子集，仅保留目标运行时可表达的控制台、时钟、
随机数、参数和预打开目录中的文件访问。不支持的 Native 文件操作在 WASI 下也会
被移除或报错。WASI SDK、Wasmtime、浏览器输出和目录授权方式见
[编译到 WebAssembly](wasm.md)。

## 平台差异

- **Linux 与 macOS**：PHP Nano 和 PHPX 源码直接加入最终程序构建。
- **iOS 与 Android**：PHP Nano 运行时接受对应 SDK/NDK 工具链编译；完整应用还需
  平台 SDK、入口与 UI 层配置，参见[原生应用](mobile-native.md)。
- **Windows**：不使用 PHP Nano 源码组合后端。`--nano` 仍禁止动态 PHP、外部命令
  和其他 Nano 能力，但程序继续通过 import library 链接完整的 `php.dll` 与
  `phpx.dll`，因此 Windows Nano 产物不是“不依赖 PHP DLL”的程序。

当目标是检查最终程序是否意外链接 PHP 或其他共享库时，可使用平台提供的二进制
依赖检查工具；同时应以编译日志中列出的 sources 和 link inputs 为准。

## 常见错误

- **提示未安装 `swoole/php-nano`**：在当前项目执行
  `composer require --dev swoole/php-nano`，并使用该项目的
  `vendor/bin/tpc.php`。
- **提示 ABI 不一致**：更新或降级 `swoole/typephp`、`swoole/phpx` 与
  `swoole/php-nano`，让 Composer 解析到相互兼容的版本，不要混用其他项目的
  `vendor` 目录。
- **提示只支持 `bin` 模式**：Nano 源码组合当前不能用于 `-m lib` 或 `-m ext`。
- **首次编译看似较慢**：这是在编译完整的 Nano/PHPX 源码。保留 `build` 目录以便
  下次复用缓存；需要逐文件观察时使用 `--no-progress`。
- **函数在普通模式可用、Nano 报不支持**：该函数依赖网络、进程、SAPI 或其他被
  移除的宿主能力，应改用保留的本地 API，或选择普通模式。

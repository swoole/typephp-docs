# 编译到 WebAssembly

TypePHP 可以将 PHP 源码提前编译为 WASI 0.2（Preview 2）Component。PHP、PHPX、TypePHP Runtime 以及高精度运算库会静态链接到同一个 `.wasm` 文件中，因此部署程序时不需要目标机器安装 PHP，也不需要携带 Linux `.so` 动态库。

同一个 Component 可以运行在两类环境中：

- **Wasmtime**：适合命令行工具、服务端任务和自动化测试。
- **Chrome 等支持 JSPI 的浏览器**：先由 Jco 将 Component 转换为 ESM，再通过 Worker 中的 WASI Host 运行。

TypePHP 只支持 WASI 0.2，不支持 WASI 0.1（Preview 1）。当前实现为 NTS、单线程模型。

## 先构建一个 Component

创建 `hello.php`：

```php
<?php

function main(): void
{
    echo "Hello TypePHP/WASI\n";
}
```

执行：

```bash
tpc --wasm hello.php
```

裸 `--wasm` 默认等价于 `--wasm=component`，只生成：

```text
hello.wasm
```

运行：

```bash
wasmtime hello.wasm
```

预期输出：

```text
Hello TypePHP/WASI
```

Component 是最简单的入门方式，不需要 Node.js 或 Jco。

## 环境要求

### 需要安装哪些工具

当前版本要求：

| 工具 | 版本要求 | 是否必须 | 用途 |
|---|---:|---|---|
| [WASI SDK](https://github.com/WebAssembly/wasi-sdk/releases) | 33 或更高 | 是 | 提供 LLVM/Clang/LLD 22 和 `wasm32-wasip2` 工具链 |
| [Wasmtime](https://docs.wasmtime.dev/cli-install.html) | 47 或更高 | 是 | 运行和测试 WASI 0.2 Component |
| [Jco](https://bytecodealliance.github.io/jco/) | 1 或更高 | 仅 `browser` profile | 将 Component 转换为浏览器 ESM |
| [wit-bindgen](https://github.com/bytecodealliance/wit-bindgen) | 固定为 `wit-bindgen-cli 0.60.0` | 仅 `mode: library` | 为 `#[WasmExport]` 生成 Canonical ABI 绑定 |
| [WABT](https://github.com/WebAssembly/wabt) | 推荐最新稳定版 | 否 | 调试 Jco 生成的 Core Wasm，不参与 TypePHP 构建 |

普通 command component 不需要 Rust、Cargo 或 `wit-bindgen`。只有使用 `mode: library` / `#[WasmExport]` 时，才需要安装固定版本的 `wit-bindgen-cli`。PHPX 发行包只提供目标端头文件和静态库，不再携带宿主平台的 `wit-bindgen` 二进制。

WASI SDK 和 Wasmtime 的可执行文件必须位于 `PATH`。browser profile 还要求 `jco`，library 模式还要求 `wit-bindgen` 位于 `PATH`。TypePHP 不探测 `/opt` 等固定目录，也没有 WASI 专用的工具链环境变量。

当前 TypePHP WASM 构建入口面向 Linux 和 macOS；macOS 需要确保 `PATH` 中有 Bash 4.0+ 和 `realpath`（可通过 Homebrew 安装新版 Bash 与 GNU coreutils）。Windows 用户目前应在 WSL 中构建。WASI SDK、Wasmtime、Jco 或 WABT 本身提供 Windows 版本，并不代表当前 TypePHP Windows 工具包已经支持 WASM 构建。

### 安装 WASI SDK

从 [WASI SDK Releases](https://github.com/WebAssembly/wasi-sdk/releases) 下载与宿主操作系统和 CPU 架构匹配的 33 或更高稳定版。不要下载 GitHub 自动生成的 `Source code` 压缩包。

Linux x86-64 可以下载 `.deb` 直接安装：

```bash
sudo apt install ./wasi-sdk-33.0-x86_64-linux.deb
export PATH="/opt/wasi-sdk/bin:$PATH"
```

也可以在 Linux 或 macOS 解压对应的 `.tar.gz`，然后把 `bin` 目录加入 `PATH`。例如解压到 `/opt/wasi-sdk-33.0` 后：

```bash
export PATH="/opt/wasi-sdk-33.0/bin:$PATH"
```

需要永久生效时，将相同目录写入 shell 配置文件。Windows/WSL 用户应在 WSL 内安装 Linux 版 WASI SDK，并从 WSL 执行 `tpc`。

### 安装 Wasmtime

Linux 和 macOS 可使用 [Wasmtime 官方安装脚本](https://docs.wasmtime.dev/cli-install.html)：

```bash
curl https://wasmtime.dev/install.sh -sSf | bash
```

根据安装程序的提示重新打开终端，或把 `$HOME/.wasmtime/bin` 加入 `PATH`。

### 浏览器开发：使用 npm 安装 Jco

只有 `wasm: browser` 或 `--wasm=browser` 需要 Jco。最简单的安装方式只有一条 npm 命令，不需要 Cargo：

```bash
npm install --global @bytecodealliance/jco
```

Linux 和 macOS 均可使用这条命令。npm 的全局可执行文件目录需要位于 `PATH`；Node.js 的标准安装程序通常会自动完成配置。

如果希望通过 `package-lock.json` 固定项目使用的 Jco 版本，也可以改为项目本地安装：

```bash
npm install --save-dev @bytecodealliance/jco
```

项目本地安装时，建议从 npm script 中运行 `tpc`，npm 会自动把 `node_modules/.bin` 加入命令的 `PATH`。也可以手动将该目录加入当前终端的 `PATH`。如果尚未安装 npm，请先安装 [Node.js](https://nodejs.org/en/download)。

Jco 的官方用法与 Component 转译示例见 [Jco Example Workflow](https://bytecodealliance.github.io/jco/example.html)。

### 验证工具链

执行：

```bash
wasm32-wasip2-clang++ --version
wasm32-wasip2-clang++ --print-target-triple
wasm-component-ld --version
wasmtime --version
jco --version  # 仅 browser profile 需要
```

目标 triple 应为：

```text
wasm32-unknown-wasip2
```

TypePHP 构建时还会检查 `wasm32-wasip2-clang`、`llvm-ar`、`llvm-ranlib` 和 `llvm-nm`。browser profile 检查 `jco`，library 模式检查 `wit-bindgen` 及其精确版本。版本或目标不符合要求时会在开始编译前报错。WABT 不是 TypePHP 的构建依赖，也不能替代 WASI SDK。

### 可选：安装 WABT 调试 Core Wasm

[WABT（WebAssembly Binary Toolkit）](https://github.com/WebAssembly/wabt) 是一组 Core WebAssembly 分析工具。TypePHP、WASI SDK、Jco 和 Wasmtime 都不会依赖它，普通用户可以不安装。

需要分析生成代码时，从 [WABT Releases](https://github.com/WebAssembly/wabt/releases) 下载与宿主操作系统和 CPU 架构匹配的最新稳定版，解压后将 `bin` 目录加入 `PATH`。例如安装到 `/opt/wabt-1.0.41`：

```bash
export PATH="/opt/wabt-1.0.41/bin:$PATH"
wasm-objdump --version
```

常用工具：

| 命令 | 用途 |
|---|---|
| `wasm-objdump` | 查看 section、import、export 和指令反汇编 |
| `wasm2wat` | 将 Core Wasm 转换为便于阅读的 WAT 文本 |
| `wasm-validate` | 验证 Core Wasm 的二进制格式和指令类型 |
| `wasm-stats` | 统计指令、函数和 section |
| `wasm-strip` | 删除 name 或其他自定义 section；TypePHP 产物通常已经完成 stripping |

TypePHP 输出的 `app.wasm` 是 Component Model 容器，而 WABT 主要面向 Core Wasm。browser profile 经 Jco 转换后，可以检查其中的 core module：

```bash
wasm-objdump -x generated/program.core.wasm
wasm2wat generated/program.core.wasm -o program.wat
wasm-validate generated/program.core.wasm
wasm-stats generated/program.core.wasm
```

不要因为旧版 WABT 无法识别新异常处理指令，就判断 TypePHP 产物一定损坏。系统软件源中的 WABT 可能明显落后于 WASI SDK。分析完整 WASI 0.2 Component 时，应优先使用 [wasm-tools](https://github.com/bytecodealliance/wasm-tools) 或 Wasmtime；WABT 更适合分析 Jco 拆出的 `program.core.wasm`。

### TypePHP WASI 运行库

TypePHP 发布包提供与编译器版本绑定的 WASI 静态库和目标端头文件。它们统一安装在 PHPX 目录中：

```text
<PHPX_HOME>/wasm/
└── wasm32-wasip2/
    ├── include/
    ├── lib/
    │   ├── libphp.a
    │   ├── libphpx.a
    │   ├── libgmp.a
    │   ├── libgmpxx.a
    │   ├── libmpfr.a
    │   ├── libmpdec.a
    │   └── libmpdec++.a
    └── .typephp-wasi-sdk-abi
```

TypePHP 按以下顺序定位 PHPX：

1. `PHPX_HOME` 环境变量。
2. Composer 安装的 `swoole/phpx` 包目录。
3. 当前 TypePHP 安装目录中的 `vendor/swoole/phpx`。

自定义 PHPX 位置时，只需使用已有的 `PHPX_HOME`：

```bash
export PHPX_HOME=/opt/phpx
```

不要分别指定或替换某一个 `.a`。这些头文件、静态库和 ABI 标记必须来自同一套 TypePHP/PHPX 发行包；混用版本可能产生链接错误或运行时内存问题。

普通应用构建不会重新编译 PHP、PHPX、GMP、MPFR 或 mpdecimal，也不需要安装 Autoconf、Bison、re2c 或 PHP 源码。

### wit-bindgen 的安装方式

`wit-bindgen` 与 Jco 的职责不同：前者在 `mode: library` 下生成 Wasm Component 的 C/C++ Guest 绑定，后者把已经链接完成的 Component 转换为浏览器 ESM。因此安装 Jco 不能替代 `wit-bindgen`。

command component 用户不需要安装 `wit-bindgen`：

- `mode: bin` 不会调用它。
- `mode: library` 必须使用 `PATH` 中的 `wit-bindgen-cli 0.60.0`。
- `tpc` 在编译前校验版本，其他版本会被拒绝，以免生成与当前 adapter 不兼容的 C ABI。

使用 library 模式时，先安装 [Rustup](https://rustup.rs/)，再执行：

```bash
cargo install wit-bindgen-cli --version 0.60.0 --locked
wit-bindgen --version
```

版本输出必须为：

```text
wit-bindgen-cli 0.60.0
```

Cargo 会把宿主可执行文件安装到用户的 Cargo bin 目录，通常是 `$HOME/.cargo/bin`。确认该目录位于 `PATH`；不能用目标为 Wasm 的程序代替宿主工具。

## 命令行模式

### Component 模式

以下两条命令等价：

```bash
tpc --wasm app.php
tpc --wasm=component app.php
```

默认输出为当前目录中的 `app.wasm`。生成的 C++ 和目标文件位于 TypePHP 默认构建目录，也可以单独指定：

```bash
tpc --wasm app.php --build-dir build/wasm
```

单文件 WASM 命令只接受输入文件、`--wasm` profile 和 `--build-dir`。输出路径、源码列表等项目选项应放入 `project.yml`，不能沿用 Host 模式的 `-o`、`-O`、`-j` 等命令行参数。WASI Runtime 和生成代码使用发布构建的优化设置。

### Browser 模式

需要浏览器产物时显式使用：

```bash
tpc --wasm=browser app.php
```

该命令先生成 `app.wasm`，再调用 Jco 生成 `app.browser/`。因此 `jco` 必须位于 `PATH`：

```bash
npm install --global @bytecodealliance/jco
```

浏览器目录中的文件是 Jco 生成的 Component 绑定，不是完整网页。应用仍需提供 HTML、Worker、WASI Preview 2 Shim，以及 stdin、stdout、环境变量、文件系统和网络能力的 Host 配置。建议从 TypePHP 仓库的 `examples/wasm-hello` 示例开始修改。

## 使用 project.yml

多文件项目推荐使用 `project.yml`。

WASM 支持 `bin`/`binary`/`cli` 命令模式，也支持通过 `mode: library` 和 `#[WasmExport]` 导出 WIT 接口。它不支持生成 PHP 动态扩展。

### Wasmtime 项目

```yaml
name: report-tool
mode: bin
wasm: component

sources:
  - src

build-dir: build
output: dist/report-tool.wasm
```

构建时不需要再传入 `--wasm`：

```bash
tpc project.yml
wasmtime dist/report-tool.wasm
```

### 浏览器项目

```yaml
name: browser-app
mode: bin
wasm: browser

sources:
  - src

build-dir: build
output: component/browser-app.wasm
wasm-browser-dir: generated
```

### 导出函数供 JavaScript 调用

需要由 JavaScript 多次调用 TypePHP 函数时，使用 library component：

```yaml
name: calculator
mode: library
wasm: browser
wasm-package: app:calculator@1.0.0
wasm-world: calculator

sources:
  - src

build-dir: build
output: component/calculator.wasm
wasm-browser-dir: generated
```

使用 `#[WasmExport]` 标记需要发布的函数：

```php
#[WasmExport]
function add(int $left, int $right): int
{
    return $left + $right;
}

#[WasmExport(name: 'greet-user')]
function greetUser(string $name): string
{
    return "Hello, {$name}";
}
```

JavaScript 必须先创建 runtime，再调用导出方法：

```js
const component = await instantiate(null, wasi.getImportObject());
const runtime = await component.api.createRuntime();

try {
    console.log(await runtime.add(20n, 22n));
    console.log(await runtime.greetUser('TypePHP'));
} finally {
    runtime[Symbol.dispose]();
}
```

当前导出 ABI 支持 `bool`、`int`、`float`、`string`、对应的 nullable 类型和 `void`。PHP `int` 在 JavaScript 中对应 `bigint`。数组和对象目前需要编码为 JSON 字符串。完整的类型限制、异常行为、NTS 串行调用规则和 ZendVM 生命周期参见 [`WasmExport`](wasm-export.md)。

`wasm` **只能**是 `component` 或 `browser`。以下配置无效：

```yaml
wasm: true
```

`target-platform` 通常不需要配置。只要启用了 WASM，TypePHP 默认使用：

```yaml
target-platform: wasm32-wasip2
```

如果显式配置，则只接受 `wasm32-wasip2` 或等价的 `wasm32-unknown-wasip2`。WASI Preview 1 的 `wasm32-wasi` 和 `wasm32-wasip1` 会被拒绝。

所有相对路径都以 `project.yml` 所在目录为基准。命令行可临时覆盖输出 profile：

```bash
tpc project.yml --wasm=component
tpc project.yml --wasm=browser
```

## 在 Wasmtime 中运行

### 参数和环境变量

TypePHP 的 `main()` 可以接收参数：

```php
<?php

function main(int $argc, array $argv): void
{
    echo "argc={$argc}\n";
    var_dump($argv);
    var_dump(getenv('APP_ENV'));
}
```

传递参数和环境变量：

```bash
wasmtime --env APP_ENV=production app.wasm first second
```

WASI Host 负责提供 `$argc`、`$argv`、环境变量以及 stdin/stdout/stderr。

### 文件系统

PHP stream 和常用文件函数可以访问 WASI Host 授权的目录，例如 `file_get_contents()`、`file_put_contents()`、`fopen()`、`scandir()` 和 `mkdir()`。

WASI 默认不会让程序访问整个宿主文件系统。运行时应明确预开放目录：

```bash
mkdir -p data
wasmtime --dir ./data::/data app.wasm
```

程序中使用 Guest 路径：

```php
file_put_contents('/data/result.txt', "generated by TypePHP\n");
$contents = file_get_contents('/data/result.txt');
```

没有授权的路径会出现 `No such file or directory` 或权限错误。这是 WASI 沙箱行为，不是 PHP 文件函数失效。

### HTTP/HTTPS GET

WASI 版本支持通过 PHP 的同步 `file_get_contents()` 发起 HTTP/HTTPS GET：

```php
<?php

function main(): void
{
    $context = stream_context_create([
        'http' => [
            'method' => 'GET',
            'timeout' => 10,
            'ignore_errors' => true,
        ],
    ]);

    $body = file_get_contents('https://www.example.com/', false, $context);
    if ($body === false) {
        fwrite(STDERR, "request failed\n");
        return;
    }
    echo $body;
}
```

Wasmtime 需要显式允许 WASI HTTP：

```bash
wasmtime -S http app.wasm
```

当前 HTTP stream 的限制：

- 只支持 GET 和只读 stream。
- 支持 `http.timeout` 和 `http.ignore_errors`。
- 不提供 Curl API。
- 不开放原始 TCP/UDP socket。

PHP API 仍然是同步的。Wasmtime 会阻塞等待 Host I/O；浏览器中由 JSPI 挂起 Wasm 调用栈并等待 Promise，不会通过忙等占用 CPU。

### 时间和安全随机数

以下能力由 WASI Host 提供，在 Wasmtime 和浏览器中使用同一套 PHP API：

```php
$now = time();
$formatted = date('Y-m-d H:i:s T', $now);
$token = bin2hex(random_bytes(16));
$number = random_int(1, 1000);
```

## 在浏览器中运行

浏览器不能直接使用 `WebAssembly.instantiate()` 执行 WASI 0.2 Component。TypePHP 的 browser profile 使用 Jco 将同一个 Component 转换为 ESM，并要求浏览器支持：

- `WebAssembly.Suspending`
- `WebAssembly.promising`

这两项 JSPI API 用于在同步 PHP 调用等待异步浏览器 I/O 时挂起和恢复 Wasm 栈。

推荐直接运行完整示例：

```bash
cd examples/wasm-hello
npm ci
npm run wasm
npm run dev
```

该示例展示：

- 参数、环境变量和标准输入输出。
- `time()`、`date()`、`random_int()` 和 `random_bytes()`。
- 内存文件系统以及可选的 OPFS 快照持久化。
- `file_get_contents()` HTTP/HTTPS GET。
- PHP 扩展列表和三种高精度类型。

浏览器网络请求仍受 CORS、CSP 和 Mixed Content 规则限制。WASI HTTP 能力不会绕过浏览器安全策略。

示例把 TypePHP 放在独立 module Worker 中运行。文件操作发生在同步内存文件系统上；启用持久化时，Worker 在程序启动和退出阶段通过 OPFS 恢复和保存快照，避免每次 PHP 文件访问都跨越异步 JavaScript 边界。

## PHP 扩展

当前 WASI Runtime 静态内建以下扩展：

```text
Core
date
pcre
bcmath
bz2
calendar
ctype
dom
exif
fileinfo
filter
hash
json
lexbor
libxml
mbstring
openssl
PDO
pdo_sqlite
random
Reflection
SimpleXML
sodium
uri
SPL
tokenizer
standard
xml
xmlreader
xmlwriter
zip
zlib
```

TypePHP 应用本身也会注册一个与项目名称对应的运行时模块，因此 `get_loaded_extensions()` 的实际结果通常还会出现一个 `typephp_*` 条目；它属于当前应用，不是额外的 PHP 标准扩展。

程序可以获取实际加载列表：

```php
foreach (get_loaded_extensions() as $extension) {
    echo $extension, "\n";
}
```

WASI 产物不支持运行时加载 `.so` 或 `.dll` 动态扩展。只有随 TypePHP WASI SDK 静态编译的扩展可用。

## 高精度类型

WASI 产物包含 TypePHP 的语言级高精度类型：

- `BigInt`：基于 GMP。
- `BigFloat`：基于 MPFR。
- `Decimal`：基于 mpdecimal。

使用方式与原生 TypePHP 一致：

```php
$big = std::bigInt('123456789012345678901234567890');
$decimal = std::decimal('199.95') * std::decimal('1.0825');
$float = std::bigFloat('3.1415926535897932384626433832795');

echo $big->toString(), "\n";
echo $decimal->toString(), "\n";
echo $float->toString(), "\n";
```

Wasm 使用 32 位指针，但 `zend_long` 保持 64 位。高精度运算保持任意精度语义，不过缺少原生平台汇编优化，吞吐量可能低于 Linux 原生构建。

## 当前限制

WASI 运行环境不等同于完整 Linux。当前明确不支持：

- Fiber 和 Generator，包括 `yield`。
- 进程创建和管理，例如 `proc_open()`、`exec()`、`system()` 和 `shell_exec()`。
- 反引号 shell 执行语法。
- 原始 TCP/UDP socket，以及 `stream_socket_*()`、`socket_*()`、`fsockopen()`。
- `pcntl_*()`、`posix_*()` 和信号处理。
- Curl、MySQL 等未静态编入 WASI Runtime 的扩展。
- 动态加载 PHP 扩展。
- 多线程和 ZTS。

对于进程、shell、原始 socket 和信号等已明确列入限制的 API，编译器会在静态识别后直接报 Fatal Error，避免程序退化为链接错误，或在 Wasmtime 与 Chrome 中产生不同语义。未静态编入 Runtime 的 PHP 扩展同样不可使用。

PHP 异常和 Zend bailout 仍受支持。时间、随机数、标准流、参数、环境变量、受授权文件系统和 HTTP GET 由 WASI 0.2 Host 提供。

WASI 中的 OpenSSL 扩展采用 crypto-only 构建，保留加密、解密、摘要、签名、密钥和证书处理，不包含 TLS stream transport。HTTP/HTTPS 请求由 WASI HTTP Component 提供，与 OpenSSL 扩展相互独立。PDO SQLite 支持内存数据库和预开放目录中的数据库文件，但不支持运行时加载 SQLite 动态扩展。

## 产物与部署

Component 模式部署时通常只需要：

```text
app.wasm
```

目标环境需要兼容 WASI 0.2 和 Component Model 的 Runtime，例如满足版本要求的 Wasmtime。`.wasm` 不依赖构建机器的 Linux glibc，也不携带 Linux `.so`。

Browser 模式还需要部署：

- Jco 生成的 ESM 和 core Wasm 文件。
- 应用 HTML、CSS 和 JavaScript。
- WASI Preview 2 Shim 与 Worker Host。

不要只复制 Jco 生成目录后直接双击 HTML。应通过 Vite、Nginx 或其他 HTTP Server 提供资源；生产部署还需要正确配置 Wasm MIME 类型以及应用使用的 CORS/CSP 策略。

## 常见错误

### Required WASI tool was not found in PATH

例如：

```text
Required WASI tool `wasm32-wasip2-clang++` was not found in PATH
```

将 WASI SDK 和 Wasmtime 的 `bin` 目录加入 `PATH`。browser 模式还需确保 `jco` 可找到。Component 模式不需要 Jco。

### WASI tool is too old

升级到满足本页最低版本的工具。系统软件源中的旧 WABT、LLVM 或 Wasmtime 不能替代当前 WASI SDK 工具链。

### TypePHP WASI SDK is missing or ABI-incompatible

确认安装了与当前 TypePHP 版本匹配的 PHPX 包，并检查：

```bash
test -f "$PHPX_HOME/wasm/wasm32-wasip2/.typephp-wasi-sdk-abi"
test -f "$PHPX_HOME/wasm/wasm32-wasip2/lib/libphp.a"
test -f "$PHPX_HOME/wasm/wasm32-wasip2/lib/libphpx.a"
```

不要从另一版本的 TypePHP/PHPX 单独复制 `.a` 文件覆盖。

### Required WASI tool `wit-bindgen` was not found in PATH

此错误只会在 `mode: library` 构建中出现。安装并验证固定版本：

```bash
cargo install wit-bindgen-cli --version 0.60.0 --locked
export PATH="$HOME/.cargo/bin:$PATH"
wit-bindgen --version
```

如果报告 incompatible version，请卸载其他来源的同名程序，或调整 `PATH`，确保首先找到 `wit-bindgen-cli 0.60.0`。

### 浏览器提示不支持 JSPI

升级浏览器并确认 `WebAssembly.Suspending` 和 `WebAssembly.promising` 存在。应用必须在 Worker 中使用支持 JSPI 的 Jco/WASI Host 运行。

### file_get_contents() 请求失败

在 Wasmtime 中确认使用了：

```bash
wasmtime -S http app.wasm
```

在浏览器中检查开发者工具中的 CORS、CSP、证书和 Mixed Content 错误。HTTP API 可用并不意味着浏览器允许访问任意地址。

### 文件函数报告 No such file or directory

Wasmtime 中使用 `--dir` 预开放目录，并确保程序访问的是 Guest 路径。浏览器中则需要由 Worker Host 配置文件系统；TypePHP 不会自动授予宿主文件系统访问权限。

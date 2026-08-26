# WasmExport 编译期 Attribute

`WasmExport` 将 TypePHP 函数发布为 WASI 0.2 Component 的 WIT 接口，使 JavaScript、Wasmtime Embed API 或其他 Component Host 可以直接调用 TypePHP。

它是编译期 Attribute，只负责生成 WIT、Canonical ABI adapter 和静态绑定，不会在运行时通过 PHP 反射查找函数。所有内建 Attribute 的共同命名空间规则参见[编译期注解](compile-time-attributes.md)。

## 第一个导出函数

```php
<?php

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

没有指定 `name` 时，TypePHP 会把 PHP camelCase 名称转换为小写连字符形式，例如 `greetUser` 导出为 `greet-user`。显式名称必须是合法且不重复的 WIT identifier，例如 `load-user`；名称只能包含小写 ASCII 字母、数字和分隔单词的单个连字符。

`WasmExport` 位于根命名空间。在其他 namespace 中应写为 `#[\WasmExport]`，或先使用 `use \WasmExport;` 导入。

## project.yml

导出函数必须使用 WASM library 模式构建：

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

配置含义：

- `mode: library`：生成由 Host 调用的 library component，不自动执行 `main()`。
- `wasm: component`：只生成 `.wasm` Component，适合自定义 Host 或 Wasmtime Embed API。
- `wasm: browser`：额外使用 Jco 生成浏览器 ESM。
- `wasm-package`：WIT package，格式必须是 `namespace:name@major.minor.patch`。
- `wasm-world`：WIT world 名称。

至少需要声明一个 `#[WasmExport]` 函数。完整 WASI SDK 和浏览器环境配置参见[编译到 WebAssembly](wasm.md)。

library 模式还要求 `wit-bindgen-cli 0.60.0` 位于 `PATH`：

```bash
cargo install wit-bindgen-cli --version 0.60.0 --locked
wit-bindgen --version
```

它只在构建期间生成当前应用的 Canonical ABI 绑定，不会链接进 PHPX 或最终 `.wasm`。使用 `mode: bin` 的普通 command component 不需要安装它。

## 支持的参数和返回类型

当前 ABI 支持：

| TypePHP 类型 | WIT 类型 | JavaScript 类型 |
|---|---|---|
| `bool` | `bool` | `boolean` |
| `int` | `s64` | `bigint` |
| `float` | `f64` | `number` |
| `string` | `string` | `string` |
| `?bool`、`?int`、`?float`、`?string` | `option<T>` | 对应类型或 `undefined` |
| `void` 返回值 | `result<_, typephp-error>` | `undefined`，失败时抛出异常 |

参数和返回值必须显式声明类型。当前不支持：

- `mixed`、未声明类型和联合类型。
- PHP `array`、对象、resource 和 callable。
- 引用参数、可变参数和带默认值的可选参数。
- 返回引用、Generator 和类方法。

PHP 没有 `array<int>` 这样的泛型参数语法，编译器无法仅凭 `array` 判断它应映射为哪一种 WIT `list<T>`。当前需要把数组或动态对象编码为 JSON 字符串：

```php
#[WasmExport(name: 'summarize-users')]
function summarizeUsers(string $json): string
{
    $users = json_decode($json, true, flags: JSON_THROW_ON_ERROR);

    return json_encode(
        ['count' => count($users)],
        JSON_THROW_ON_ERROR | JSON_UNESCAPED_UNICODE,
    );
}
```

```js
const result = await runtime.summarizeUsers(JSON.stringify([
    { name: 'Ada' },
    { name: 'Linus' },
]));

console.log(JSON.parse(result));
```

JSON 会增加编码、解析和中间字符串分配开销，适合动态结构或低频接口。WIT `list`、`record` 和对象 resource 的语言级类型映射尚未开放。

## JavaScript 调用

Jco 生成的模块导出 `api.createRuntime()`。library component 没有自动入口，必须先创建 runtime resource：

```js
import { instantiate } from './generated/program.js';

const component = await instantiate(null, wasi.getImportObject());
const runtime = await component.api.createRuntime();

try {
    const sum = await runtime.add(20n, 22n);
    const greeting = await runtime.greetUser('TypePHP');

    console.log(sum);       // 42n
    console.log(greeting);  // Hello, TypePHP
} finally {
    runtime[Symbol.dispose]();
}
```

PHP `int` 对应有符号 64 位整数，因此 JavaScript 必须传入和接收 `bigint`，例如 `20n`，不能传入普通的 `20`。

浏览器中的 TypePHP 导出是异步 JavaScript 函数。即使某个函数当前只计算 CPU 数据，也应始终使用 `await`；这样函数内部才能安全调用时间、文件、HTTP 等可能通过 JSPI 挂起的 WASI API。

## ZendVM 生命周期

`createRuntime()` 会初始化 PHP Embed SAPI、ZendVM、当前应用模块和一个 PHP request，并执行相应的 MINIT/RINIT。用户不应直接调用 Zend C API，也不需要自己调用 MINIT/RINIT。

同一 runtime 上的所有导出调用共享这个 request：

- 不会为每次调用重复执行 RINIT/RSHUTDOWN。
- 请求级全局变量、静态状态和内存池会持续到 runtime 被释放。
- 当前 WASI Runtime 是 NTS，同一 runtime 不允许并发或重入调用；调用者应串行 `await`。
- PHP 异常会转换为 WIT `result`，Jco 会将错误表现为 JavaScript 异常。
- Zend bailout 会使 runtime 进入 failed 状态，后续调用将被拒绝。

`runtime[Symbol.dispose]()` 会执行 RSHUTDOWN、模块清理和 PHP Embed shutdown。不要只依赖 JavaScript GC；直接终止 Worker 也不保证 PHP 关闭回调得到执行。

## 完整示例

TypePHP 仓库中的 `examples/wasm-hello` 展示了完整交互流程：

1. JavaScript 在 Worker 中实例化 Component。
2. 调用 `api.createRuntime()` 建立 ZendVM request。
3. 页面点击 PHP 扩展名称。
4. Worker 调用 `runtime.getExtensionInfo()`。
5. TypePHP 返回 JSON 字符串，JavaScript 解析并渲染扩展详情。

该示例同时展示了 JSPI、HTTP、文件系统、时间、随机数和 PHP 高精度类型。

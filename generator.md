# Yield 与 Generator

`yield` 用于按需产生数据。函数中出现 `yield` 或 `yield from` 后，调用函数会立即返回一个可迭代对象；函数体在开始迭代时才执行，并在每次产生值后暂停。

Generator 适合处理大文件、分页结果、数据流和连续计算结果。与先构造完整数组相比，它通常能显著降低峰值内存占用。

## 基本用法

```php
function numbers(): iterable
{
    yield 1;
    yield 2;
    yield 3;
}

function main(): void
{
    foreach (numbers() as $number) {
        echo $number, "\n";
    }
}
```

输出：

```text
1
2
3
```

没有指定 key 时，会从 `0` 开始生成整数 key。也可以显式指定 key：

```php
function statuses(): iterable
{
    yield 'draft' => '草稿';
    yield 'published' => '已发布';
}

foreach (statuses() as $code => $label) {
    echo $code, ': ', $label, "\n";
}
```

自动整数 key 会避开已经显式使用过的整数 key，其行为与 ZendPHP Generator 一致。

## 延迟执行与资源释放

Generator 函数体不会在调用函数时执行，而是在第一次迭代或读取当前值时执行：

```php
function read_lines(string $file): iterable
{
    $handle = fopen($file, 'r');

    try {
        while (($line = fgets($handle)) !== false) {
            yield rtrim($line, "\r\n");
        }
    } finally {
        fclose($handle);
    }
}

foreach (read_lines('app.log') as $line) {
    echo $line, "\n";
}
```

持有文件、连接或锁时，建议使用 `try/finally`。遍历完成、异常退出或 Generator 被释放时，`finally` 可用于清理资源。

## 使用 `yield from`

`yield from` 可以把另一个可迭代对象中的 key 和 value 继续产出：

```php
function local_users(): iterable
{
    yield 'alice';
    yield 'bob';
}

function all_users(): iterable
{
    yield from local_users();
    yield from ['carol', 'dave'];
}
```

TypePHP 的 `yield from` 支持：

- 数组；
- `Iterator`；
- `IteratorAggregate`；
- ZendPHP 返回的 `Generator`；
- TypePHP 返回的 `FiberGenerator`。

对于第三方扩展提供的特殊 `Traversable` 对象，若不能直接委托，建议先转换为数组，或使用明确实现 `Iterator`/`IteratorAggregate` 的适配对象。

### 获取被委托 Generator 的返回值

`yield from` 表达式的结果是被委托 Generator 的返回值：

```php
function child(): iterable
{
    yield 1;
    yield 2;
    return 3;
}

function parent_generator(): iterable
{
    $result = yield from child();
    echo $result, "\n"; // 3
}
```

委托普通数组或普通 Iterator 时，`yield from` 的结果为 `null`。

## Generator 的返回值

`yield` 产生迭代值，函数末尾的 `return` 是 Generator 完成后的返回值。正常执行完毕后，可以通过 `getReturn()` 读取：

```php
function task(): iterable
{
    yield 'starting';
    yield 'working';
    return 'done';
}

$generator = task();

foreach ($generator as $message) {
    echo $message, "\n";
}

echo $generator->getReturn(), "\n"; // done
```

Generator 尚未完成或因异常失败时，不应读取 `getReturn()`。

## `send()` 与 `throw()`

大多数业务代码只需要 `foreach`。需要双向通信时，可以用 `send()` 把值传回 Generator：

```php
function receiver(): iterable
{
    $name = yield 'ready';
    yield 'hello, ' . $name;
}

$generator = receiver();

echo $generator->current(), "\n";        // ready
echo $generator->send('TypePHP'), "\n"; // hello, TypePHP
```

`throw($exception)` 会在当前暂停位置向 Generator 内部抛出异常，可由函数体中的 `try/catch/finally` 处理：

```php
function worker(): iterable
{
    try {
        yield 'waiting';
    } catch (RuntimeException $e) {
        yield 'recovered';
    }
}

$generator = worker();
$generator->current();
echo $generator->throw(new RuntimeException()), "\n";
```

除非确实需要协作式控制流程，否则优先使用 `foreach` 和普通异常，代码更容易维护。

## 匿名函数和箭头函数

匿名函数和箭头函数中也可以使用 `yield`：

```php
$factory = function (int $count): iterable {
    for ($i = 0; $i < $count; $i++) {
        yield $i;
    }
};

$single = fn (): iterable => yield 'value';

foreach ($factory(3) as $value) {
    echo $value, "\n";
}
```

匿名 Generator 会正常捕获 `use` 变量；非静态闭包也可以使用绑定的 `$this`。

## 返回类型

面向 TypePHP 和 ZendPHP 共用的代码，推荐声明通用可迭代类型：

```php
function records(): iterable { yield 1; }
function recordsIterator(): Iterator { yield 1; }
function recordsTraversable(): Traversable { yield 1; }
```

也可以使用 `object`、`mixed`，或包含兼容类型的联合类型。

TypePHP 编译后的 Generator 实际类型是全局类 `\FiberGenerator`，因此仅在 TypePHP 中运行的代码也可以声明精确类型：

```php
function records(): \FiberGenerator
{
    yield 1;
}
```

`FiberGenerator` 位于全局命名空间，不是 `TypePHP\FiberGenerator`。不过 ZendPHP 本身没有这个类；需要让同一份源码直接运行在 ZendPHP 中时，不要使用 `\FiberGenerator` 返回类型，应使用 `iterable`、`Iterator` 或 `Traversable`。

## `FiberGenerator` 可用方法

通常只需用 `foreach`，也可以按需调用以下方法：

| 方法 | 用途 |
|---|---|
| `current(): mixed` | 获取当前 value |
| `key(): mixed` | 获取当前 key |
| `valid(): bool` | 判断当前位置是否有效 |
| `next(): void` | 继续执行到下一个 `yield` |
| `rewind(): void` | 开始迭代；已经越过第一个 `yield` 后不能重新倒带 |
| `send(mixed $value): mixed` | 向当前 `yield` 发送值并继续执行 |
| `throw(Throwable $exception): mixed` | 在当前暂停位置抛入异常 |
| `getReturn(): mixed` | 获取正常执行结束后的返回值 |

`FiberGenerator` 实现了 `Iterator`，可以传给接受 `Iterator`、`Traversable` 或 `iterable` 的代码。

不要直接 `new FiberGenerator()`，也不要继承、clone 或序列化该对象。Generator 对象应始终由包含 `yield` 的函数创建。

## 与 ZendPHP Generator 的差异

TypePHP 保持常用的迭代、异常和返回值行为，但返回对象不是 ZendPHP 内置的 `Generator`。

| 项目 | ZendPHP | TypePHP |
|---|---|---|
| 实际对象类型 | `Generator` | `FiberGenerator` |
| `instanceof Iterator` | `true` | `true` |
| `instanceof Generator` | `true` | `false` |
| 推荐的跨环境返回类型 | `iterable`/`Iterator`/`Traversable` | `iterable`/`Iterator`/`Traversable` |
| 精确返回类型 | `Generator` | `FiberGenerator` |
| `ReflectionGenerator` | 支持 | 不支持 |
| clone、序列化 | 不支持 | 不支持 |

因此不要编写依赖具体 Generator 类名的跨环境判断：

```php
// 跨环境代码不推荐
if ($result instanceof Generator) {
}

// 推荐
if ($result instanceof Traversable) {
}
```

`get_class()`、`var_dump()`、Reflection 信息、异常调用栈和调试器中显示的栈结构，也可能与 ZendPHP 不同。业务逻辑不应依赖这些展示细节。

## 当前限制

### 不支持引用 Generator

当前不支持：

- Generator 函数或方法按引用返回；
- 按引用 `yield`；
- `foreach ($generator as &$value)`；
- 依赖 `current()`、`send()`、`throw()` 或 `getReturn()` 返回引用。

以下代码不能编译：

```php
function &values(): iterable
{
    yield 1;
}

foreach (values() as &$value) {
}
```

需要修改原始数据时，建议产出对象，或显式调用负责修改数据的方法。

### Generator 参数限制

Generator 支持普通参数、默认值、类型声明、联合类型和方法中的 `$this`，但当前不支持按引用参数和可变参数：

```php
function invalidReference(&$value): iterable { yield $value; }
function invalidVariadic(...$values): iterable { yield 1; }
function invalidVariadicReference(&...$values): iterable { yield 1; }
```

参数类型在调用 Generator 函数时检查；Generator 函数体仍然延迟到开始迭代时执行。

## 使用建议

- 数据量较小且需要随机访问、排序或重复遍历时，优先返回数组。
- 处理大文件、分页数据或持续数据流时，优先使用 Generator。
- 只读迭代优先使用 `foreach`，不要手动管理迭代状态。
- 需要跨 TypePHP 与 ZendPHP 运行时，返回类型优先写 `iterable`、`Iterator` 或 `Traversable`。
- 使用 `try/finally` 释放 Generator 持有的文件、连接、锁等资源。
- 不要依赖具体类名、Reflection、调试输出或调用栈与 ZendPHP 完全一致。

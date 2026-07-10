`yield` 用来编写“按需产生数据”的函数。函数中出现 `yield` 或 `yield from` 后，调用函数不会一次性得到全部结果；它会返回一个可迭代对象，`foreach` 每前进一步，函数才继续执行并产生下一个值。

Generator 很适合处理大文件、分页数据、连续计算结果等不需要一次性放入数组的场景。

## 最简单的用法

在函数中使用 `yield` 返回一个值：

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

没有指定 key 时，PHP 会从 `0` 开始自动生成整数 key。也可以显式指定 key：

```php
function statuses(): iterable
{
    yield "draft" => "草稿";
    yield "published" => "已发布";
}

foreach (statuses() as $code => $label) {
    echo $code, ": ", $label, "\n";
}
```

## 延迟执行

Generator 的函数体会在开始迭代时才执行，而不是调用函数时立即执行。因此可以把昂贵操作放在 `yield` 之间：

```php
function read_lines(string $file): iterable
{
    $handle = fopen($file, "r");

    try {
        while (($line = fgets($handle)) !== false) {
            yield rtrim($line, "\r\n");
        }
    } finally {
        fclose($handle);
    }
}

foreach (read_lines("app.log") as $line) {
    echo $line, "\n";
}
```

建议在 Generator 中使用 `try/finally` 释放文件、连接等资源。正常遍历结束、异常中断或 Generator 被销毁时，`finally` 都应承担清理职责。

## 使用 `yield from`

`yield from` 用于把另一个可迭代值的内容继续产出。它适合把多个数据源组合成一个流：

```php
function all_users(): iterable
{
    yield from ["alice", "bob"];
    yield from ["carol", "dave"];
}

foreach (all_users() as $user) {
    echo $user, "\n";
}
```

也可以委托给另一个 Generator：

```php
function range_values(int $from, int $to): iterable
{
    for ($i = $from; $i <= $to; $i++) {
        yield $i;
    }
}

function even_then_odd(): iterable
{
    yield from range_values(2, 6);
    yield from range_values(1, 5);
}
```

`yield from` 支持数组、普通 `Iterator`、`IteratorAggregate`、PHP Generator 和 TypePHP Generator。对于少见的扩展内部 `Traversable` 对象，兼容性可能与原生 PHP 不完全一致；遇到这类对象时，建议先转换为数组或使用项目中明确实现 `Iterator` 的对象。

## Generator 的返回值

`yield` 产生的是迭代值，函数末尾的 `return` 则是 Generator 完成后的返回值。完成后可通过 `getReturn()` 读取：

```php
function task(): iterable
{
    yield "starting";
    yield "working";

    return "done";
}

$generator = task();

foreach ($generator as $message) {
    echo $message, "\n";
}

echo $generator->getReturn(), "\n"; // done
```

必须先让 Generator 正常执行完毕，再调用 `getReturn()`。

## `send()` 与 `throw()`

大多数业务代码只需要 `foreach`。如果需要双向协作，可以把 `yield` 当成表达式接收 `send()` 传入的值：

```php
function receiver(): iterable
{
    $name = yield "ready";
    yield "hello, " . $name;
}

$generator = receiver();
echo $generator->current(), "\n"; // ready
echo $generator->send("TypePHP"), "\n"; // hello, TypePHP
```

`throw($exception)` 可以把异常抛回 Generator 内部，Generator 可以在函数体中用 `try/catch/finally` 处理它。除非确实需要协作式控制流程，否则优先使用普通异常和 `foreach`，代码会更容易维护。

## 返回类型声明

TypePHP 中，Generator 返回的是可迭代对象。推荐使用以下返回类型：

```php
function records(): iterable { /* ... */ }
function records(): Iterator { /* ... */ }
function records(): Traversable { /* ... */ }
```

也可以使用 `object`、`mixed`，或包含上述类型的联合类型。

不要把 Generator 的返回类型写成 PHP 内置的 `Generator`：

```php
// 不支持
function records(): Generator
{
    yield 1;
}
```

TypePHP 返回的是 `TypePHP\FiberGenerator`。它实现了 `Iterator`，因此可以正常 `foreach`、调用 `current()`、`next()`、`valid()`、`send()`、`throw()` 和 `getReturn()`；但它不是 PHP 内置的 `Generator`。

因此不要依赖以下行为：

```php
$generator instanceof Generator; // 不要依赖，结果为 false
new ReflectionGenerator($generator); // 不支持
```

业务代码也不应直接创建、继承、clone 或序列化 `TypePHP\FiberGenerator`。

## 参数限制

Generator 可以使用普通参数、默认值、对象参数、联合类型参数和类方法中的 `$this`：

```php
class Report
{
    public function pages(int $count, string $prefix = "page"): iterable
    {
        for ($i = 1; $i <= $count; $i++) {
            yield $prefix . " " . $i;
        }
    }
}
```

目前不支持以下 Generator 参数形式：

```php
function invalid_one(&$value): iterable { yield $value; } // 不支持按引用参数
function invalid_two(...$values): iterable { yield 1; }   // 不支持可变参数
function invalid_three(&...$values): iterable { yield 1; } // 不支持按引用可变参数
```

参数类型会在调用 Generator 时检查；函数体仍按迭代进度延迟执行。

## 引用限制

TypePHP Generator 只产生普通值，不支持通过 Generator 传递元素引用。以下写法不支持：

```php
function &values(): iterable
{
    yield $value;
}

foreach (values() as &$value) {
}
```

也就是说，不支持：

- Generator 函数按引用返回；
- 按引用 `yield`；
- 对 Generator 使用按引用 `foreach`；
- 依赖 `current()`、`send()`、`throw()` 或 `getReturn()` 返回引用。

如果需要修改原始数据，请直接传递数组、对象，或使用明确的引用参数调用，而不是通过 Generator 的迭代值传递引用。

## 使用建议

- 数据量不大、需要随机访问时，优先返回数组。
- 需要边读取边处理、避免一次性占用大量内存时，使用 Generator。
- 只读迭代优先使用 `foreach`。
- 只有确实需要双向通信时才使用 `send()` 或 `throw()`。
- 不要依赖 Generator 的具体类名、反射行为、调试输出、序列化或 clone 行为与 PHP 内置 `Generator` 完全相同。

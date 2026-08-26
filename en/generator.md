# Yield and Generator

`yield` is used to produce data on demand. When `yield` or `yield from` appears in a function, calling the function immediately returns an iterable object; the function body only executes when iteration begins, and pauses after each value is produced.

Generator is suitable for processing large files, paginated results, data streams, and continuous computation results. Compared with building a complete array first, it usually significantly reduces peak memory usage.

## Basic Usage

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

Output:

```text
1
2
3
```

When no key is specified, integer keys are generated starting from `0`. Keys can also be specified explicitly:

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

Automatic integer keys will avoid integer keys that have already been explicitly used, consistent with the ZendPHP Generator behavior.

## Deferred Execution and Resource Release

The Generator function body does not execute when the function is called, but instead executes on the first iteration or when reading the current value:

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

When holding files, connections, or locks, using `try/finally` is recommended. When iteration completes, an exception exits, or the Generator is released, `finally` can be used to clean up resources.

## Using `yield from`

`yield from` can continue yielding the keys and values from another iterable object:

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

TypePHP's `yield from` supports:

- arrays;
- `Iterator`;
- `IteratorAggregate`;
- `Generator` returned by ZendPHP;
- `FiberGenerator` returned by TypePHP.

For special `Traversable` objects provided by third-party extensions, if they cannot be delegated directly, it is recommended to convert them to arrays first, or use an adapter object that explicitly implements `Iterator`/`IteratorAggregate`.

### Getting the Return Value of a Delegated Generator

The result of a `yield from` expression is the return value of the delegated Generator:

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

When delegating a plain array or a plain Iterator, the result of `yield from` is `null`.

## Generator Return Value

`yield` produces iteration values; the `return` at the end of the function is the Generator's return value after completion. After normal execution finishes, it can be read via `getReturn()`:

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

When the Generator has not completed or has failed due to an exception, `getReturn()` should not be read.

## `send()` and `throw()`

Most business code only needs `foreach`. When two-way communication is needed, `send()` can be used to pass a value back into the Generator:

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

`throw($exception)` throws an exception into the Generator at the current suspension point, which can be handled by `try/catch/finally` in the function body:

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

Unless cooperative control flow is truly needed, prefer `foreach` and ordinary exceptions, which makes the code easier to maintain.

## Anonymous Functions and Arrow Functions

`yield` can also be used in anonymous functions and arrow functions:

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

Anonymous Generators capture `use` variables normally; non-static closures can also use the bound `$this`.

## Return Types

For code shared between TypePHP and ZendPHP, declaring a general iterable type is recommended:

```php
function records(): iterable { yield 1; }
function recordsIterator(): Iterator { yield 1; }
function recordsTraversable(): Traversable { yield 1; }
```

You can also use `object`, `mixed`, or union types containing compatible types.

The actual type of a Generator compiled by TypePHP is the global class `\FiberGenerator`, so code that runs only in TypePHP can also declare the precise type:

```php
function records(): \FiberGenerator
{
    yield 1;
}
```

`FiberGenerator` is in the global namespace, not `TypePHP\FiberGenerator`. However, ZendPHP itself does not have this class; when the same source code needs to run directly in ZendPHP, do not use the `\FiberGenerator` return type, but use `iterable`, `Iterator`, or `Traversable`.

## Available `FiberGenerator` Methods

Usually you only need `foreach`, but the following methods can also be called as needed:

| Method | Purpose |
|---|---|
| `current(): mixed` | Get the current value |
| `key(): mixed` | Get the current key |
| `valid(): bool` | Check whether the current position is valid |
| `next(): void` | Continue execution to the next `yield` |
| `rewind(): void` | Start iteration; cannot rewind after passing the first `yield` |
| `send(mixed $value): mixed` | Send a value to the current `yield` and continue execution |
| `throw(Throwable $exception): mixed` | Throw an exception into the current suspension point |
| `getReturn(): mixed` | Get the return value after normal execution completes |

`FiberGenerator` implements `Iterator` and can be passed to code that accepts `Iterator`, `Traversable`, or `iterable`.

Do not `new FiberGenerator()` directly, and do not inherit from, clone, or serialize this object. Generator objects should always be created by functions containing `yield`.

## Differences from ZendPHP Generator

TypePHP maintains the common iteration, exception, and return value behaviors, but the returned object is not ZendPHP's built-in `Generator`.

| Item | ZendPHP | TypePHP |
|---|---|---|
| Actual object type | `Generator` | `FiberGenerator` |
| `instanceof Iterator` | `true` | `true` |
| `instanceof Generator` | `true` | `false` |
| Recommended cross-environment return type | `iterable`/`Iterator`/`Traversable` | `iterable`/`Iterator`/`Traversable` |
| Precise return type | `Generator` | `FiberGenerator` |
| `ReflectionGenerator` | Supported | Not supported |
| clone, serialize | Not supported | Not supported |

Therefore, do not write cross-environment checks that depend on a specific Generator class name:

```php
// Not recommended for cross-environment code
if ($result instanceof Generator) {
}

// Recommended
if ($result instanceof Traversable) {
}
```

`get_class()`, `var_dump()`, Reflection information, exception call stacks, and the stack structure shown in debuggers may also differ from ZendPHP. Business logic should not depend on these presentation details.

## Current Limitations

### Reference Generators Are Not Supported

Currently not supported:

- Generator functions or methods returning by reference;
- `yield` by reference;
- `foreach ($generator as &$value)`;
- Relying on `current()`, `send()`, `throw()`, or `getReturn()` returning references.

The following code cannot be compiled:

```php
function &values(): iterable
{
    yield 1;
}

foreach (values() as &$value) {
}
```

When you need to modify the original data, it is recommended to yield objects, or explicitly call the method responsible for modifying the data.

### Generator Argument Restrictions

Generator supports ordinary arguments, default values, type declarations, union types, and `$this` in methods, but currently does not support by-reference arguments and variadic arguments:

```php
function invalidReference(&$value): iterable { yield $value; }
function invalidVariadic(...$values): iterable { yield 1; }
function invalidVariadicReference(&...$values): iterable { yield 1; }
```

Argument types are checked when the Generator function is called; the Generator function body is still deferred until iteration begins.

## Usage Recommendations

- When the data volume is small and random access, sorting, or repeated traversal is needed, prefer returning arrays.
- When processing large files, paginated data, or continuous data streams, prefer using Generator.
- For read-only iteration, prefer `foreach` and do not manage iteration state manually.
- When needing to run across TypePHP and ZendPHP, prefer writing `iterable`, `Iterator`, or `Traversable` as the return type.
- Use `try/finally` to release resources such as files, connections, and locks held by the Generator.
- Do not rely on specific class names, Reflection, debug output, or call stacks being exactly identical to ZendPHP.

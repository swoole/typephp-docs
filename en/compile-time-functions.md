# Compile-time Functions

Compile-time functions are syntax entry points proprietary to the TypePHP compiler. The compiler recognizes these functions during the static compilation stage and directly generates the corresponding C++ code; they are generally not looked up at runtime as ordinary PHP functions.

Keyword methods are a separate syntax system and are not included in the compile-time function list in this document. See [Keyword Methods](keyword-method.md).

## Function List

| Function | Purpose |
|------|------|
| `std::any([$value])` | downgrades an expression to `mixed` / `any` / `php::Var`; defaults to `null` when omitted |
| `std::object($value, ClassName::class)` | validates an object and restores its concrete class information |
| `std::ref($value)` | explicitly passes a variable, array element, or object property by reference |
| `std::expected($condition)` | marks a condition as usually true, helping the compiler optimize the common branch |
| `std::unexpected($condition)` | marks a condition as usually false, helping the compiler optimize the rare branch |
| `std::int($value)` | creates a native int expression |
| `std::float($value)` | creates a native float expression |
| `std::bool($value)` | creates a native bool expression |
| `std::bigInt($value)` | creates a BigInt high-precision integer |
| `std::decimal($value)` | creates a Decimal high-precision decimal number |
| `std::bigFloat($value)` | creates a BigFloat high-precision floating-point number |
| `std::array($type, $size)` | creates a fixed-size StdArray container |
| `std::vector($type[, $size])` | creates a dynamic StdVector container |
| `std::map($keyType, $valueType)` | creates a hash StdMap container |
| `std::orderedMap($keyType, $valueType)` | creates an ordered StdOrderedMap container |

## Usage Positions and Name Spelling

These core functions serve different purposes and apply in different positions:

| Function | Usable position | Description |
|------|--------------|------|
| `std::any()` | any ordinary value expression position | usable in assignments, function arguments, return values, array elements, arithmetic expressions, conditional expressions, etc. |
| `std::object()` | any ordinary value expression position | checks the runtime object type and restores the concrete class known to the compiler. |
| `std::ref()` | call arguments only | used to pass a referenceable value to a parameter that requires a reference; it is not an ordinary value conversion function. |
| `std::expected()` | boolean condition expressions | used in conditions that are usually true, such as `if`, `elseif`, loops, or ternary expressions. |
| `std::unexpected()` | boolean condition expressions | used in conditions that are usually false, such as errors, out-of-bounds, or cache misses. |

Compile-time functions are static methods of the global `std` class. The `std` class and its method names are case-insensitive, following PHP class/method rules; `Type::*` type constants remain case-sensitive.

## std::any([$value])

`std::any()` downgrades the compile-time type of an expression to a dynamic type. After downgrading, the variable can receive any PHP value, but the compiler no longer generates native type, typed object, or Native Call optimizations for it.

The argument is optional. Calling `std::any()` without an argument creates a dynamic value initialized to `null`; this is useful when a variable needs dynamic storage before its real value is known.

`std::any()` is an ordinary value expression and can be placed in any position that requires a value. For example:

```php
function identity(mixed $value): mixed {
    return std::any($value);
}

function main(): void {
    $values = [std::any(1), std::any("two")];
    var_dump(std::any(3) + 1);
    var_dump(identity(std::any($values[0])));
}
```

```php
function main(): void {
    $value = std::any(10);

    $value = "string";
    var_dump($value);
}
```

`std::any()` is often used in scenarios where branches may produce different types:

```php
class FileLogger {}
class NullLogger {}

function create_logger(bool $debug) {
    if ($debug) {
        $logger = std::any(new FileLogger());
    } else {
        $logger = std::any(new NullLogger());
    }

    return $logger;
}
```

It can also be used to preserve PHP dynamic operation semantics:

```php
function main(): void {
    $a = std::any(10);
    $b = $a / 3;

    var_dump($b); // float(3.333333...)
}
```

## std::object($value, ClassName::class)

`std::object()` is the function form of `$value->toObject(ClassName::class)`.
It checks that `$value` is an instance of the requested class and restores that
concrete class information for subsequent compilation:

```php
$user = std::object($payload['user'], User::class);
echo $user->getName();
```

The class argument must be resolvable at compile time. Concrete class names,
string class-name literals, `self::class`, and `parent::class` are supported.
Subclass objects pass a parent-class or interface check.

Unlike the TypePHP-only keyword-method syntax, the static call can be made
portable to ordinary Zend PHP by providing a compatibility implementation on
the Zend PHP execution path:

```php
class std {
    public static function object(mixed $value, string $class): object {
        if (!$value instanceof $class) {
            throw new TypeError("Expected an instance of {$class}");
        }
        return $value;
    }
}
```

The same application code can then call `std::object()` under both Zend PHP and
TypePHP. TypePHP lowers the call directly to its checked object conversion and
does not perform a runtime static-method dispatch.

## std::ref($value)

`std::ref()` is used to pass references explicitly. It is mainly used in scenarios where the compiler cannot know at compile time whether a parameter is a reference, such as dynamic calls, closure calls, and variable function calls.

The argument to `std::ref()` must be an lvalue that can be referenced: a variable, array element, or object property. Literals, function return values, arithmetic expressions, and other temporary values cannot be passed.

`std::ref()` can only be used as an argument to a single function or method call:

```php
$callback(std::ref($value));
```

Do not use it as an ordinary value expression; for example, the following usages are not supported:

```php
return std::ref($value);
$list = [std::ref($value)];
$sum = std::ref($value) + 1;
```

Variable references:

```php
function main(): void {
    $fn = function (&$name): void {
        $name .= " compiler";
    };

    $name = "php";
    $fn(std::ref($name));

    var_dump($name); // string(12) "php compiler"
}
```

Array element references:

```php
function main(): void {
    $fn = function (&$value): void {
        $value = "changed";
    };

    $data = ["name" => "origin"];
    $fn(std::ref($data["name"]));

    var_dump($data["name"]); // string(7) "changed"
}
```

Object property references:

```php
class Box {
    public string $value = "origin";
}

function main(): void {
    $fn = function (&$value): void {
        $value = "changed";
    };

    $box = new Box();
    $fn(std::ref($box->value));

    var_dump($box->value); // string(7) "changed"
}
```

When the parameter information of static functions and built-in functions is known at compile time, the compiler handles reference parameters automatically, and `std::ref()` is not needed additionally:

```php
function main(): void {
    parse_str("hello=world", $result);
    var_dump($result["hello"]); // string(5) "world"
}
```

For native `T&` references, local aliases, dynamic-call write-back, and escape restrictions, see [Strongly Typed References](strong-references.md).

## std::expected($condition) and std::unexpected($condition)

`std::expected()` and `std::unexpected()` provide hints about branch probability to the compiler:

- `std::expected($condition)` indicates that the condition is true in most cases.
- `std::unexpected($condition)` indicates that the condition is false in most cases.

For example, marking the normal processing path as the common branch:

```php
function handle_request(bool $ready): int {
    if (std::expected($ready)) {
        // the vast majority of requests enter this branch
        return 1;
    }

    return 0;
}
```

Marking errors or other rare cases as the uncommon branch:

```php
function normalize_id(int $id): int {
    if (std::unexpected($id < 0)) {
        return 0;
    }

    return $id;
}
```

It can also be used in loop conditions:

```php
function countdown(int $remaining): void {
    while (std::expected($remaining > 0)) {
        echo $remaining, "\n";
        $remaining--;
    }
}
```

Both functions accept exactly one non-spread argument and return `bool`. They do not change the truth value of the condition, the number of evaluations, or the business logic; they only provide optimization hints.

Branch prediction hints should be used based on actual runtime behavior. If you cannot determine whether a branch is clearly more common, just use an ordinary conditional expression:

```php
if ($condition) {
    // no need to forcibly add std::expected() or std::unexpected()
}
```

## std::int($value)

`std::int()` explicitly creates a native int expression. It suits hot code that needs to clearly enter the native integer operation path.

```php
function main(): void {
    $sum = std::int(0);

    for ($i = std::int(0); $i < 1000; $i++) {
        $sum += $i;
    }

    var_dump($sum);
}
```

Converting from a dynamic value:

```php
function main(): void {
    $value = std::any("123");
    $id = std::int($value);

    var_dump($id + 1);
}
```

## std::float($value)

`std::float()` explicitly creates a native float expression.

```php
function main(): void {
    $x = std::float(1.5);
    $y = std::float(2);

    var_dump($x * $y);
}
```

Converting from a dynamic value:

```php
function main(): void {
    $value = std::any("3.14");
    $pi = std::float($value);

    var_dump($pi);
}
```

## std::bool($value)

`std::bool()` explicitly creates a native bool expression.

```php
function main(): void {
    $enabled = std::bool(1);

    if ($enabled) {
        echo "enabled\n";
    }
}
```

Converting from a dynamic value:

```php
function main(): void {
    $value = std::any("");
    $ok = std::bool($value);

    var_dump($ok); // bool(false)
}
```

## std::bigInt($value)

`std::bigInt()` creates a BigInt high-precision integer. The argument can be an integer or a string; for very long integers a string is recommended to avoid precision loss during PHP parsing.

```php
function main(): void {
    $a = std::bigInt(100);
    $b = std::bigInt("999999999999999999999999999999");

    var_dump(($a + $b)->toString());
}
```

BigInt cannot be constructed from a float; use a string or integer instead:

```php
function main(): void {
    $value = "3";
    $big = std::bigInt($value);

    var_dump($big->toString());
}
```

## std::decimal($value)

`std::decimal()` creates a Decimal high-precision decimal number. For finance, monetary amounts, and precise decimal calculations, a string argument is preferred.

```php
function main(): void {
    $price = std::decimal("19.99");
    $tax = std::decimal("0.08");
    $total = $price + ($price * $tax);

    var_dump($total->toString());
}
```

A floating-point literal can be passed directly; the compiler will try to use the source literal text to construct the Decimal:

```php
function main(): void {
    $rate = std::decimal(0.125);

    var_dump($rate->toString());
}
```

Constructing a Decimal from a float variable is not recommended, because the variable has already lost the source literal text:

```php
function main(): void {
    $raw = "0.1";
    $value = std::decimal($raw);

    var_dump($value->toString());
}
```

## std::bigFloat($value)

`std::bigFloat()` creates a BigFloat high-precision floating-point number, suitable for scientific computing or scenarios requiring a larger exponent range.

```php
function main(): void {
    $pi = std::bigFloat("3.141592653589793238462643383279502884197");
    $radius = std::bigFloat(10);
    $area = $pi * $radius * $radius;

    var_dump($area->toString());
}
```

Constructing from an integer or float value:

```php
function main(): void {
    $a = std::bigFloat(42);
    $b = std::bigFloat(1.5);

    var_dump(($a + $b)->toString());
}
```

## std::array($type, $size)

`std::array()` creates a fixed-size StdArray container. The size must be an integer literal, and the container can only be created at the first assignment of a variable in the top-level scope of a function; an existing variable cannot be reassigned to a new StdArray.

```php
function main(): void {
    $items = std::array(Type::Int, 3);

    $items[0] = 10;
    $items[1] = 20;
    $items[2] = 30;

    var_dump($items[1]);
}
```

Nested fixed-size arrays:

```php
function main(): void {
    $matrix = std::array(std::array(Type::Int, 3), 2);

    $matrix[0][0] = 1;
    $matrix[1][2] = 9;

    var_dump($matrix[1][2]);
}
```

Object-typed elements:

```php
class User {
    public function __construct(public string $name) {}
}

function main(): void {
    $users = std::array(User::class, 2);

    $users[0] = new User("rango");
    $users[1] = new User("swoole");

    var_dump($users[0]->name);
}
```

## std::vector($type[, $size])

`std::vector()` creates a dynamic StdVector container. The first argument is the element type, and the second optional argument is the initial size, which must be an integer literal.

```php
function main(): void {
    $numbers = std::vector(Type::Int);

    $numbers[] = 10;
    $numbers[] = 20;

    var_dump(count($numbers));
}
```

Specifying an initial size:

```php
function main(): void {
    $numbers = std::vector(Type::Int, 3);

    $numbers[0] = 1;
    $numbers[1] = 2;
    $numbers[2] = 3;

    var_dump($numbers[2]);
}
```

Object or interface typed elements:

```php
interface Task {
    public function id(): int;
}

class Job implements Task {
    public function __construct(private int $id) {}

    public function id(): int {
        return $this->id;
    }
}

function main(): void {
    $tasks = std::vector(Task::class);

    $tasks[] = new Job(1);
    $tasks[] = new Job(2);

    var_dump($tasks[0]->id());
}
```

## std::map($keyType, $valueType)

`std::map()` creates a hash StdMap container, corresponding to C++ `std::unordered_map` underneath. The key type only supports `Type::Int`, `Type::String`, or `Type::String`.

```php
function main(): void {
    $scores = std::map(Type::String, Type::Int);

    $scores["alice"] = 90;
    $scores["bob"] = 80;

    var_dump($scores["alice"]);
}
```

Integer keys:

```php
function main(): void {
    $users = std::map(Type::Int, Type::String);

    $users[1001] = "alice";
    $users[1002] = "bob";

    var_dump($users[1002]);
}
```

Object-typed values:

```php
class Connection {
    public function __construct(public string $name) {}
}

function main(): void {
    $pool = std::map(Type::String, Connection::class);

    $pool["main"] = new Connection("main");

    var_dump($pool["main"]->name);
}
```

## std::orderedMap($keyType, $valueType)

`std::orderedMap()` creates an ordered StdOrderedMap container, corresponding to C++ `std::map` underneath. The key type restriction is the same as `std::map()`.

```php
function main(): void {
    $items = std::orderedMap(Type::String, Type::Int);

    $items["b"] = 2;
    $items["a"] = 1;

    foreach ($items as $key => $value) {
        echo $key . ":" . $value . "\n";
    }
}
```

High-precision numeric values as the value:

```php
function main(): void {
    $balances = std::orderedMap(Type::Int, Type::Decimal);

    $balances[1] = std::decimal("19.99");
    $balances[2] = std::decimal("100.50");

    var_dump($balances[2]->toString());
}
```

## Type Description Arguments

The `$type`, `$keyType`, and `$valueType` of Std containers are not ordinary runtime variables but compile-time type descriptions. Common values are as follows:

| Type description | Meaning |
|----------|------|
| `Type::Int` | native int |
| `Type::Float` | native float |
| `Type::Bool` | native bool |
| `Type::BigInt` | BigInt |
| `Type::Decimal` | Decimal |
| `Type::BigFloat` | BigFloat |
| `Type::String` | string |
| `Type::Array` | array |
| `Type::Object` | object |
| `Type::Any` | any / mixed |
| `Type::Stream` | stream |
| `ClassName::class` | a specified class, abstract class, or interface |

## Usage Restrictions

`std::array()`, `std::vector()`, `std::map()`, and `std::orderedMap()` are container construction entry points that can only be used for the first assignment of a variable, and must be located in the top-level scope of a function:

```php
function main(): void {
    $numbers = std::vector(Type::Int); // correct

    // error: cannot re-create a container for an existing variable
    // $numbers = std::vector(Type::Int);
}
```

Error example:

```php
function main(bool $flag): void {
    if ($flag) {
        // error: cannot create a std container inside nested statement blocks such as if/while/for
        $numbers = std::vector(Type::Int);
    }
}
```

If you need to restore a Std container type from a `mixed` / `any` value, use the keyword methods `toStdArray()`, `toStdVector()`, `toStdMap()`, `toStdOrderedMap()`, which are not part of the compile-time functions listed in this document.

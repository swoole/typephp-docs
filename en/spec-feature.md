The TypePHP compiler adds some proprietary features beyond regular `PHP` syntax.

> In the documentation, the `any` type means the variable is untyped; the corresponding `PHP` type is `mixed`, and the `PHPX` type is `php::Var`

## Native Types and `use varint_types`

Inferred `int`, `float`, and `bool` locals use fixed native storage (`php::Int`, `php::Float`, and `php::Bool`) by default.

Use `use varint_types` only when a file needs Zend PHP integer widening semantics:

```php
use varint_types;

function divide(): mixed {
    $a = 10;       // inferred integers use php::Var in this file
    return $a / 3; // float(3.3333...)
}
```

This directive affects inferred integers only. Floats and booleans remain fixed native values. For one dynamic expression rather than an entire file, use `std::any($value)`.

## use bigint_types

Automatically declares all integer literals in the current file as the `BigInt` type, without needing to manually wrap them with `std::bigInt()`.

```php
declare(strict_types=1);
use bigint_types;

function main(): void {
    // Ordinary integer literals automatically become BigInt
    $a = 42;
    echo $a->toString();   // "42"

    // Operations between BigInts
    $b = $a + 10;
    echo $b->toString();   // "52"

    // Literal operations are also BigInt
    $c = 100 + 200;
    echo $c->toString();   // "300"
}
```

After using `use bigint_types`, all `Scalar_Int` literals are converted into `php::newBigInt(N)` calls at compile time. Without this directive, only very long integer literals of 19 digits or more are automatically recognized as BigInt (see [math.md §11](math.md#11-automatic-recognition-of-very-long-literals)).

### Relationship to the Default Type Mode

| Directive | Integer literal type | Applicable scenario |
|------|--------------|---------|
| None | `php::Int` (native int64) | Ordinary integer arithmetic |
| `use varint_types` | `php::Var` (Zend integer semantics) | Integer overflow-to-float and dynamic integer arithmetic |
| `use bigint_types` | `php::BigInt` (arbitrary precision) | When large integers or chained BigInt method calls are needed |

`use bigint_types` promotes integer literals to `php::BigInt`; ordinary inferred integers use the default native storage unless the file also selects `use varint_types`.

## use decimal_types

Automatically declares all floating-point literals in the current file as the `Decimal` type, without needing to manually wrap them with `std::decimal()`.

```php
declare(strict_types=1);
use decimal_types;

function main(): void {
    // Ordinary float literals automatically become Decimal
    $a = 3.1;
    echo $a->toString();   // "3.1"

    // Operations between Decimals
    $b = 2.5;
    $c = $a->add($b);
    echo $c->toString();   // "5.6"

    // Mixed operation: Int + Float literal → Decimal
    $d = 10 + 0.5;
    echo $d->toString();   // "10.5"
}
```

After using `use decimal_types`, all `Scalar_Float` literals are converted into `php::newDecimal(...)` calls at compile time. Without this directive, only floating-point literals with 16 or more significant digits are automatically recognized as Decimal.

> **Note**: because the PHP parser may already introduce binary floating-point error when parsing floating-point literals, it is recommended to still use the string form `std::decimal("...")` for high-precision requirements. `use decimal_types` is best suited for reducing boilerplate in projects that use Decimal arithmetic throughout.

### Using Together with `use bigint_types`

```php
declare(strict_types=1);
use bigint_types;
use decimal_types;

function main(): void {
    // Integer literal → BigInt
    $a = 100;
    echo $a->toString();   // "100"

    // Float literal → Decimal
    $b = 2.5;
    echo $b->toString();   // "2.5"

    // BigInt + Int literal → BigInt
    $c = $a + 50;
    echo $c->toString();   // "150"

    // Decimal + Float literal → Decimal
    $d = $b + 1.5;
    echo $d->toString();   // "4.0"
}
```

## toObject(ClassName::class)

`toObject()` is a compiler built-in keyword method used to reconstruct an object type with class information from an `any` / `mixed` type. After reading an element from an array or calling a function that returns the `any` type, the compiler loses the concrete class information; this method can be used to continue the type.

```php
$obj = $array['object']->toObject(App\Hello\Test::class);
$obj->foo();
```

The compiler can re-obtain the type of the `$obj` object and turn its method calls into `Native Call` instead of the dynamic `zend_call_function()` call, greatly improving performance.

> `toObject()` without arguments returns a `php::Object` with no class information, equivalent to the `(object)` cast. To reconstruct concrete class information, `ClassName::class` must be passed.

When extracting an element from an array, the variable's type defaults to `any`. Objects can use `$var->toObject(ClassName::class)` to continue the type; other types can use type conversion functions or type cast syntax to continue the type. The methods are as follows:

#### 1. Continue the Type Using Cast Syntax
- Integer: `$v = (int) $array[$key]`
- Float: `$v = (float) $array[$key]`
- Boolean: `$v = (bool) $array[$key]`
- String: `$v = (string) $array[$key]`
- Array: `$v = (array) $array[$key]`


Please note that the object (`object`) type is actually untyped in `PHP` and is almost equivalent to the `any` type. If `$v = (object) $array[$key]` is used, `$v` is declared as `php::Object` instead of `php::Var`, but the compiler cannot obtain the object's `class` information. Therefore, the `object` cast is meaningless to the compiler. Likewise, the `callable` and `iterator` types are of no help to the compiler.

#### 2. Continue the Type Using Conversion Functions
- Integer: `$v = intval($array[$key])`
- Float: `$v = floatval($array[$key])`
- Boolean: `$v = boolval($array[$key])`
- String: `$v = strval($array[$key])`

Please note there are only these `4` built-in conversion functions; `PHP` does not provide an `arrayval()` function, so if you need to declare a variable as the `array` type, you can only use cast syntax.

In addition to continuing the type of array elements, if the right-hand side of an assignment is the `any` type, the left-hand side is also declared as `any` by default; you can use the above methods to declare a more accurate type.

```php
$a = std::any(3.001);
// $b's type will be php::Int, and the value will be converted to 3
$b = intval($a);
```

## std::any($value)
The purpose of this function is to mark a variable's type as `php::Var` rather than a native type. For example:

```php
$a = std::any(123);
$b = 123;
```

Without the `any` function, the variable is declared as the `php::Int` type. It cannot be used for non-integer assignment, losing capabilities such as overflow detection.

```php
$a = 10; // $a's type is Int
$b = $a / 3;  // $b's value is 3, type is integer

$a = std::any(10); // $a's type is Var
$b = $a / 3;  // $b's value is 3.33333..., type is float
```

`std::any($value)` is equivalent to the keyword method `$value->toAny()`. Both only affect compile-time type inference and do not produce additional runtime type checks.


## std::ref($value)

Changes value passing into reference passing in dynamic calls. `std::ref()` accepts a **variable**, **array element**, or **object property** as its argument, and cannot accept an expression. `std::ref($value)` is equivalent to the keyword method `$value->toRef()`.

**Variable reference:**

```php
eval('function retval_test(&$name) { $name .= "refval test"; }');

$name = 'php ';
retval_test(std::ref($name));
echo $name; // Output: php refval test
```

**Array element reference:**

```php
eval('function array_ref_test(&$val) { $val = "modified"; }');

$arr = ['key' => 'original'];
array_ref_test(std::ref($arr['key']));
echo $arr['key']; // Output: modified
```

**Object property reference:**

```php
eval('function prop_ref_test(&$val) { $val = "modified"; }');

$obj = new stdClass();
$obj->prop = 'original';
prop_ref_test(std::ref($obj->prop));
echo $obj->prop; // Output: modified
```

`eval()` is a function that executes instructions at runtime; it dynamically generates functions. Because the compiler cannot know the argument types of these dynamic functions during the static compilation phase, it cannot automatically identify by-reference arguments, which is where the `std::ref()` function is needed to explicitly convert a value into reference passing.

The following usages are **incorrect**:

```php
// ❌ std::ref() cannot accept an expression
retval_test(std::ref("literal string"));
retval_test(std::ref($a + $b));
retval_test(std::ref(foo()));

// ❌ toRef() also cannot be used on a temporary expression
retval_test(($a + $b)->toRef());
retval_test(foo()->toRef());
```

However, the following code does not need `std::ref()`:
```php
class Request {
    public $data;
}

function main()
{
    $req = new Request;
    $req->data = ['get' => [],];

    parse_str("hello=world", $req->data['get']);
    var_dump($req->data['get']['hello']);
}
```

`parse_str()` is a built-in function whose argument information is available at compile time; its second argument is a reference type, so the compiler automatically changes the argument to reference passing without requiring the extra `std::ref()` function.


## The `toStream()` Keyword Method

`toStream()` is a compiler built-in keyword method used to reconstruct a variable as the Stream type. When reading an element from an array, the compiler cannot track its concrete type; calling `toStream()` reconstructs the Stream type, thereby enabling chained calls to Stream methods such as `write()`, `read()`, and `close()`.

This method is typically used for the pipe arrays returned by `proc_open()` or `stream_socket_pair()`.

```php
// stream_socket_pair returns an array of two stream elements
$sockets = stream_socket_pair(
    STREAM_PF_UNIX,
    STREAM_SOCK_STREAM,
    0
);

// The compiler cannot track the element types in the array, use toStream() to reconstruct
$client = $sockets[0]->toStream();
$server = $sockets[1]->toStream();

$client->write("hello");
echo $server->read(5);   // "hello"

$client->close();
$server->close();
```

`toStream()` is directly replaced by a `php::toStream()` call at compile time, with no runtime overhead.

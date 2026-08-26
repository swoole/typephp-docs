The TypePHP compiler adds some proprietary features beyond regular `PHP` syntax.

> In the documentation, the `any` type means the variable is untyped; the corresponding `PHP` type is `mixed`, and the `PHPX` type is `php::Var`

## use native_types

Requires the compiler to convert the `int`, `float`, and `bool` types to native types to improve computation performance.

```php
use native_types;

function foo() {
    $a = 1000;
    while($a--) {

    }
}
```

After using `use native_types`, when a local variable is assigned an integer, it is declared as `php::Int` instead of `php::Var`, yielding huge performance gains in computation-intensive scenarios. Without it, the default is `php::Var`, and integers are stored in `zval` structs.

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

### Difference from `use native_types`

| Directive | Integer literal type | Applicable scenario |
|------|--------------|---------|
| None | `php::Int` (native int64) | Ordinary integer arithmetic |
| `use native_types` | `php::Int` (native int64) | High-performance integer arithmetic |
| `use bigint_types` | `php::BigInt` (arbitrary precision) | When large integers or chained BigInt method calls are needed |

`use bigint_types` and `use native_types` can be used together. When used together, non-literal integer variables remain `php::Int`, but integer literals are promoted to `php::BigInt`.

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
$a = any(3.001);
// $b's type will be php::Int, and the value will be converted to 3
$b = intval($a);
```

## any($value)
The purpose of this function is to mark a variable's type as `php::Var` rather than a native type. For example:

```php
$a = any(123);
$b = 123;
```

Without the `any` function, the variable is declared as the `php::Int` type. It cannot be used for non-integer assignment, losing capabilities such as overflow detection.

```php
$a = 10; // $a's type is Int
$b = $a / 3;  // $b's value is 3, type is integer

$a = any(10); // $a's type is Var
$b = $a / 3;  // $b's value is 3.33333..., type is float
```

`any($value)` is equivalent to the keyword method `$value->toAny()`. Both only affect compile-time type inference and do not produce additional runtime type checks.


## objval($value, $className)

Reconstructs an object type with class information from a `mixed` / `any` variable; it is

the functional form of `$var->toObject(ClassName::class)`. `objval()` restores object type information at compile time, giving subsequent method calls the opportunity to generate Native Calls; at runtime it still validates that the actual object satisfies the target class, parent class, or interface constraints.

```php
$obj = $array['object'];
$typed = objval($obj, App\Hello\Test::class);
$typed->foo();  // The compiler can generate a Native Call
```

The second argument of `objval()` only supports string literals or `ClassName::class` constants.

```php
// ✅ Correct usage
$obj = objval($var, TestObjval::class);
$obj = objval($var, 'TestObjval');

// ❌ The second argument cannot be a variable
$obj = objval($var, $someClass);
```

## refval($value)

Changes value passing into reference passing in dynamic calls. `refval()` accepts a **variable**, **array element**, or **object property** as its argument, and cannot accept an expression. `refval($value)` is equivalent to the keyword method `$value->toRef()`.

**Variable reference:**

```php
eval('function retval_test(&$name) { $name .= "refval test"; }');

$name = 'php ';
retval_test(refval($name));
echo $name; // Output: php refval test
```

**Array element reference:**

```php
eval('function array_ref_test(&$val) { $val = "modified"; }');

$arr = ['key' => 'original'];
array_ref_test(refval($arr['key']));
echo $arr['key']; // Output: modified
```

**Object property reference:**

```php
eval('function prop_ref_test(&$val) { $val = "modified"; }');

$obj = new stdClass();
$obj->prop = 'original';
prop_ref_test(refval($obj->prop));
echo $obj->prop; // Output: modified
```

`eval()` is a function that executes instructions at runtime; it dynamically generates functions. Because the compiler cannot know the argument types of these dynamic functions during the static compilation phase, it cannot automatically identify by-reference arguments, which is where the `refval()` function is needed to explicitly convert a value into reference passing.

The following usages are **incorrect**:

```php
// ❌ refval() cannot accept an expression
retval_test(refval("literal string"));
retval_test(refval($a + $b));
retval_test(refval(foo()));

// ❌ toRef() also cannot be used on a temporary expression
retval_test(($a + $b)->toRef());
retval_test(foo()->toRef());
```

However, the following code does not need `refval()`:
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

`parse_str()` is a built-in function whose argument information is available at compile time; its second argument is a reference type, so the compiler automatically changes the argument to reference passing without requiring the extra `refval()` function.


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

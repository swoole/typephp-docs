# Type System

The TypePHP compiler extends standard `PHP` types with high-precision numeric types and strongly-typed containers, and performs type inference and checking at compile time. This document introduces all types supported by the compiler, type conversion methods, and usage limitations.

Object types can also be supplemented with instance methods that only take effect during the static compilation phase through ordinary functions in the same namespace. See [Extended Object Methods](object-extension-method.md) for details.

For scenarios with extreme requirements on object layout and call performance, where entering ZendVM's internal data structures is unnecessary, you can explicitly declare a [`#[Native]` Native Class](native-class.md). Native classes use a fixed C++ field layout and an independent tracing GC, with stricter static typing and interoperability boundaries than ordinary PHP objects.

## ZendPHP Type System Boundaries

ZendPHP is still essentially a dynamic, weakly-typed language. The parameter and return type declarations (often called type hints) provided by PHP primarily constrain function call boundaries, rather than establishing an unchangeable static type for variables inside a function.

Parameter types are checked when entering a function, but after an ordinary parameter enters the function body it is merely a `zval` and can be reassigned to any other type:

```php
declare(strict_types=1);

function change(int $value): void
{
    $value = 'text';
    $value = ['php', 'typephp'];
}

change(1); // valid
```

Even if a parameter is passed by reference, the parameter declaration itself only performs the entry check:

```php
function changeRef(int &$value): void
{
    $value = 'text';
}

$value = 1;
changeRef($value);
var_dump($value); // string(4) "text"
```

Return types are likewise checked when the function exits and the return value leaves the function. They do not restrict type changes of temporary variables inside the function body. Local variables also have no independent type declaration, and their type can change with each assignment.

`declare(strict_types=1)` only disables part of the implicit scalar conversions at function call boundaries, causing mismatched parameters or return values to throw a `TypeError`; it does not make local variables or parameters fixed types inside the function body, and therefore does not convert ZendPHP into a statically strongly-typed language.

TypePHP always compiles in strict mode. The declaration is accepted but redundant, while `declare(strict_types=0)` is rejected. TypePHP's fixed local types come from compile-time inference and storage rules, not from the PHP declaration itself.

In PHP's mutable storage, what truly carries type constraints persistently is a declared object property, including static properties. Property values can be modified, but the declared type of a property cannot change at runtime; every write is checked by ZendVM:

```php
class User
{
    public int $id = 0;
}

$user = new User();
$user->id = 42;     // valid
$user->id = 'text'; // TypeError
```

This constraint is attached to the property slot. Even if a property is passed to a function by reference, it cannot be bypassed through the reference:

```php
$user = new User();
changeRef($user->id); // TypeError: cannot write string to int property
```

Without an explicit default value, ZendPHP's typed property is not the type's zero value, nor `null`, but the uninitialized `IS_UNDEF` state. In order to generate fixed-type C++ property slots, TypePHP uses deterministic zero-value initialization for common fixed value types, so there is a clear compatibility difference here:

| Property declaration | ZendPHP without explicit default | TypePHP without explicit default |
|---|---|---|
| `public int $value;` | Uninitialized; reading throws `Error` | `0` |
| `public float $value;` | Uninitialized; reading throws `Error` | `0.0` |
| `public bool $value;` | Uninitialized; reading throws `Error` | `false` |
| `public string $value;` | Uninitialized; reading throws `Error` | Empty string `''` |
| `public array $value;` | Uninitialized; reading throws `Error` | Empty array `[]` |
| `public ?int $value;` | Uninitialized; does not automatically become `null` | Nullable dynamic storage; when an explicit default is needed, write `= null` |
| `public mixed $value;` | Uninitialized | Dynamic value, defaults to `null` |
| `public $value;` | `null` | `null` |

If the source code explicitly declares a default value, such as `public int $value = 10` or `public ?int $value = null`, the declared value is used instead of the implicit initial state in the table.

This difference affects direct reads, `isset()`, `??`, and Reflection initialization state. For example, in ZendPHP an uninitialized `public int $value` makes `isset($object->value)` return `false` and `$object->value ?? 10` return `10`; TypePHP's fixed slot already contains `0`, so you cannot rely on ZendPHP's uninitialized behavior. To express the "not yet set" state, explicitly use a nullable type with `= null`, rather than relying on a typed property without a default value. For more limitations, see [Compatibility Notes](compatible.md#fixed-value-type-properties).

PHP 8.4 also supports typed class constants, but class constants themselves are not writable and do not belong to the mutable variable storage model.

Ordinary PHP arrays likewise have no element types for keys and values. A single array can hold keys and values of different types at the same time, and its structure can change arbitrarily at runtime:

```php
$values = [];
$values[] = 1;
$values[] = 'text';
$values['user'] = new User();
$values[10] = ['nested' => true];
```

The `array` parameter or property type only means that the value itself must be a PHP array; it does not constrain the elements inside the array. Notations such as `list<int>` and `array<string, User>` in PHPDoc are only for IDEs and static analysis tools; ZendVM does not enforce these constraints.

Therefore, ZendPHP does not cover a complete strongly-typed system for variables, parameters, local values, and container elements. On top of PHP-compatible syntax, TypePHP adds compile-time type inference, fixed native types, [typed PHP arrays and type annotations](typed-arrays.md), [Std Strongly-Typed Containers](std-containers.md), [`#[Immutable]`](immutable.md), and [`#[Native]`](native-class.md) constraints; only these capabilities restrict type changes at compile time, or generate code with fixed C++ storage types.

## 1. Type Overview

The TypePHP compiler supports the following C++ storage types (`php::*`):

| Type constant | C++ type | Corresponding PHP type | Description |
|---------|----------|--------------|------|
| `TYPE_VAR` | `php::Var` | `mixed` | Dynamic type, stored using zval |
| `TYPE_INT` | `php::Int` | `int` | Native 64-bit integer |
| `TYPE_FLOAT` | `php::Float` | `float` | Native double-precision float |
| `TYPE_BOOL` | `php::Bool` | `bool` | Native boolean |
| `TYPE_STR` | `php::Str` | `string` | Native string |
| `TYPE_ARRAY` | `php::Array` | `array` | Native array |
| `TYPE_OBJECT` | `php::Object` | `object` | Generic object (no concrete class info) |
| `TYPE_STREAM` | `php::Stream` | `resource` | Stream resource type |
| `TYPE_RESOURCE` | `php::Resource` | `resource` | Generic resource type |
| `TYPE_BIGINT` | `php::BigInt` | — | Arbitrary-precision integer |
| `TYPE_DECIMAL` | `php::Decimal` | — | Decimal high-precision number |
| `TYPE_BIGFLOAT` | `php::BigFloat` | — | Binary high-precision float |
| `TYPE_STD_VECTOR` | `php::StdVector` | — | C++ `std::vector` |
| `TYPE_STD_ARRAY` | `php::StdArray` | — | C++ `std::array` (fixed length) |
| `TYPE_STD_MAP` | `php::StdMap` | — | C++ `std::unordered_map` |
| `TYPE_STD_ORDERED_MAP` | `php::StdOrderedMap` | — | C++ `std::map` (ordered) |
| `TYPE_ARGS` | `php::Args` | — | Variadic argument list |
| `TYPE_REF` | `php::Ref` | — | Reference type |
| `TYPE_VOID` | `void` | `void` | No return value |

## 2. Native Types

The compiler maps inferred `int`, `float`, and `bool` locals to native C++ types by default, eliminating `zval` wrapping overhead. Native scalar storage is fixed and is not silently promoted to `php::Var` later.

```php
declare(strict_types=1);

function sum(int $n): int {
    $total = 0;        // php::Int
    $pi = 3.14159;     // php::Float
    $flag = true;      // php::Bool
    $name = "hello";   // php::Str
    $items = [1, 2];   // php::Array
    return $total + $n;
}
```

Use `std::any($value)` when one expression needs dynamic storage. Use `use varint_types` when inferred integers throughout a file need Zend integer widening behavior; this directive does not change inferred `float` or `bool` storage.

### 2.1 Native Type and PHP Type Declaration Mapping

| PHP type declaration | AOT type | Description |
|-------------|----------|------|
| `int` | `php::Int` | |
| `float` / `double` | `php::Float` | |
| `bool` / `false` / `true` | `php::Bool` | |
| `string` | `php::Str` | |
| `array` | `php::Array` | |
| `object` | `php::Object` | No concrete class info |
| `void` / `never` | `void` | |
| `mixed` | `php::Var` | |
| `null` | `php::Var` | Degrades to dynamic type |
| `callable` | `php::Var` | Compiler cannot track |
| `iterable` | `php::Var` | Compiler cannot track |
| `stream` | `php::Stream` | TypePHP-specific |

> **Note**: The `null`, `callable`, and `iterable` type declarations degrade to `php::Var` at compile time and cannot benefit from the performance advantages of native types.

## 3. High-Precision Numeric Types

The TypePHP compiler provides three high-precision numeric types. See [math.md](math.md) for details.

### 3.1 Construction

```php
declare(strict_types=1);

// Construct via std:: factory functions
$a = std::bigInt("12345678901234567890");
$b = std::decimal("0.1");
$c = std::bigFloat("1.2345e100");

// Literals auto-recognized (ultra-long integers / high-precision floats)
$d = 12345678901234567890;    // 19+ digits auto-recognized as BigInt
$e = 0.1234567890123456;      // 16+ significant digits auto-recognized as Decimal
```

### 3.2 Declaration Directives

- **`use bigint_types`**: all integer literals in the file automatically become `BigInt`
- **`use decimal_types`**: all float literals in the file automatically become `Decimal`

```php
declare(strict_types=1);
use bigint_types;
use decimal_types;

$a = 42;      // php::BigInt (not php::Int)
$b = 3.14;    // php::Decimal (not php::Float)
```

### 3.3 Immutability

Big* types are **immutable** — every operation returns a new value and does not modify the original variable. Compound assignment operations (`+=`, `-=`, etc.) are expanded at compile time to `$a = BigInt::add($a, ...)`.

## 4. Std Strongly-Typed Containers

The TypePHP compiler directly maps C++ standard library containers, providing zero-overhead type-safe storage. Key types support only `Type::Int` and `Type::String`.

If the data must remain an ordinary PHP `array` but you want the compiler to check direct element writes on properties, you can use [typed PHP arrays and type annotations](typed-arrays.md). It does not convert the PHP array into a Std Container; the two have different storage models and dynamic boundaries.

```php
declare(strict_types=1);

// std::vector — dynamic array
$v = std::vector(Type::Int);
$v[] = 10;
$v[] = 20;

// std::array — fixed-length array (compile-time bounds checking)
$a = std::array(Type::Float, 5);
$a[0] = 3.14;

// std::orderedMap — ordered map
$m = std::orderedMap(Type::String, Type::Int);
$m["key"] = 100;

// std::map — hash map
$u = std::map(Type::Int, User::class);
$u[1] = new User(1);
```

### 4.1 Type Symbols

The root namespace `Type` provides compile-time type symbols with IDE completion and spelling checking. It differs from the compiler-internal `TypePHP\Type`.

| Type symbol | Purpose |
|---------|------|
| `Type::Int` | Marks integer elements |
| `Type::Float` | Marks float elements |
| `Type::Bool` | Marks boolean elements |
| `Type::BigInt` | Marks BigInt elements |
| `Type::BigFloat` | Marks BigFloat elements |
| `Type::Decimal` | Marks Decimal elements |
| `Type::String` | Marks string elements |
| `Type::Array` | Marks array elements |
| `Type::Object` | Marks object elements |
| `Type::Any` | Marks dynamic-type elements |
| `Type::Stream` | Marks Stream elements |

The container value type can also be any PHP class name (such as `User::class`); the compiler generates an independent C++ template instantiation for each concrete type.

## 5. Type Inference

The compiler performs type inference on expressions at compile time via `detectTypeOfExpr()`:

### 5.1 Literals

| Expression | Inferred type |
|--------|---------|
| Integer literal `123` | `php::Int` (`php::BigInt` when `bigint_types` is enabled) |
| Float literal `3.14` | `php::Float` (`php::Decimal` when `decimal_types` is enabled) |
| Ultra-long integer (≥19 digits) | Auto-recognized as `php::BigInt` |
| High-precision float (≥16 significant digits) | Auto-recognized as `php::Decimal` |
| Boolean literals `true` / `false` | `php::Bool` |
| String literal `"hello"` | `php::Str` |
| Array literal `[1, 2]` | `php::Array` |

### 5.2 Type Cast Expressions

| Expression | Inferred type |
|--------|---------|
| `(int) $x` | `php::Int` |
| `(float) $x` | `php::Float` |
| `(bool) $x` | `php::Bool` |
| `(string) $x` | `php::Str` |
| `(array) $x` | `php::Array` |
| `(object) $x` | `php::Object` (**no class info**) |

### 5.3 Binary Operations

The compiler determines the result type based on the left and right operand types by priority:

1. Either operand is `BigFloat` → result `BigFloat`
2. Either operand is `Decimal` → result `Decimal`
3. Either operand is `BigInt` → result `BigInt`
4. Either operand is `Float` → result `Float`
5. Either operand is `Int` → result `Int`
6. Otherwise degrade to `Var`

> Division exception: dividing two `Int`s still yields an `Int` (integer division); if not evenly divisible, it degrades to `Var`.

### 5.4 Function Return Values

- The return types of built-in functions (such as `fopen`, `stream_socket_client`) are determined by compiler built-in rules
- The return types of user-defined functions are inferred from the `declare` declaration
- Dynamic calls (functions defined via `call_user_func`, `eval`) degrade to `Var`

## 6. Type Conversion

> The `to*` keyword methods are the primary approach for type conversion. See the [Type Conversion](type-convert.md) document for details.

### 6.1 Automatic Type Promotion

When Big* types are mixed with ordinary Int/Float in operations, the compiler automatically performs type promotion:

| Source type | Target type | Promotion approach |
|--------|---------|---------|
| `Int` | `BigInt` | `php::newBigInt($n)` |
| `Float` | `Decimal` (literal) | `php::newDecimal("...")` |
| `Float` | `Decimal` (variable) | **Error** — must use string construction |
| `Float` | `BigInt` | **Error** — conversion forbidden |
| `Float` | `BigFloat` | `php::newBigFloat($f)` |
| `Int` | `Decimal` | `php::newDecimal(php::toString($n))` |
| `Int` | `BigFloat` | `php::newBigFloat($n)` |
| `BigInt` | `Decimal` | `php::newDecimal(php::BigInt::toString($n))` |
| `BigInt` | `BigFloat` | `php::BigFloat::newInstance(php::BigInt::toString($n))` |
| `Decimal` | `BigFloat` | `php::BigFloat::newInstance(php::Decimal::toString($n))` |

### 6.2 Manual Type Conversion

The compiler provides type conversion functions, which can also be used in ZendPHP via polyfills:

```php
// Construct Big* types via std:: factories (recommended)
$a = std::bigInt("1234567890");      // string → BigInt
$b = std::decimal("0.01");           // string → Decimal
$c = std::bigFloat("1.23e100");      // string → BigFloat

// Methods to convert Big* → ordinary types
$d = $a->toInt();                    // BigInt → Int (may overflow)
$e = $b->toFloat();                  // Decimal → Float
$f = $c->toString();                 // BigFloat → String

// Cross Big* type conversion
$g = std::bigInt($a->toString());    // Decimal → String → BigInt
$h = std::decimal($a->toString());   // BigInt → String → Decimal
```

### 6.3 Type Erasure

`std::any($value)` and `$value->toAny()` erase a definite type into dynamic `php::Var` storage. The value may then change PHP type and use Zend dynamic dispatch, but fixed-type and typed-object optimizations no longer apply.

```php
$value = std::any(new User());
$value = "dynamic";
```

Global/static dynamic slots and ordinary PHP array elements are also common type-erasure boundaries. See [Type Erasure and Recovery](type-erasure.md) for their storage rules and restrictions.

### 6.4 Type Recovery

A dynamic value returns to fixed storage through an explicit conversion or assertion. Scalar types use casts, conversion functions, `toInt()` and similar methods; objects use `toObject(ClassName::class)` to restore concrete class information with a runtime type check.

```php
$id = $payload['id']->toInt();
$user = $payload['user']->toObject(User::class);
$user->load($id);
```

Recovery does not remember the erased type; the program must state the type it currently expects. See [Type Erasure and Recovery](type-erasure.md) for the complete mapping of scalar, object, stream, high-precision, and Std-container recovery operations.


## 7. Type Limitations

### 7.1 Cross Big* Implicit Mixing Forbidden

The compiler blocks cross-type implicit mixing that may cause precision loss:

```php
$a = std::bigInt("100");
$b = std::decimal("2.5");
$c = $a + $b;  // ❌ Compile error: BigInt and Decimal cannot be operated on directly
```

Solution: explicitly convert to the same type before operating.

### 7.2 Float to Decimal Only for Literals

```php
$a = std::decimal("0.1");    // ✅ recommended
$b = std::decimal(0.1);      // ✅ literal — the compiler extracts the original text "0.1" as a string
$pi = 3.14159;
$c = std::decimal($pi);      // ❌ Compile error: cannot convert from a variable
```

> **Special logic**: when the argument to `std::decimal()` is a float literal, the compiler extracts `rawValue` directly from the AST as a string and passes it to the constructor, avoiding binary floating-point error. But a variable does not retain the original text, so it cannot be safely converted.

### 7.3 Float to BigInt Forbidden

```php
$a = 3.14;
$b = std::bigInt($a);       // ❌ Compile error: Cannot convert float to BigInt
$b = std::bigInt("3");      // ✅ use a string
```

### 7.4 Big* Types Do Not Support Increment/Decrement

Big* types are immutable, and `++` / `--` semantics do not match, so the compiler reports an error:

```php
$a = std::bigInt("100");
$a++;  // ❌ compile error
$a--;  // ❌ compile error
```

### 7.5 Union, Intersection, and Nullable Types Degrade to Var

PHP union types (`int|float`, `int|string`, etc.), intersection types (`A&B`), and nullable types (`?A`) degrade to `php::Var` at compile time and cannot benefit from the performance advantages of native types. The compiler still retains the runtime type check to avoid bypassing PHP type constraints when static handling is impossible.

### 7.6 object Type Does Not Retain Class Info

The `(object)` cast and `object` type declaration only produce `php::Object`; the compiler does not know the concrete class name and cannot optimize method calls. You must use `toObject(ClassName::class)` to reconstruct it.

### 7.7 Std Container Key Type Limitations

The key types of `std::orderedMap` and `std::map` only support:
- `Type::Int` — integer key
- `Type::String` / `Type::String` — string key

Other key types cause a compile error.

### 7.8 Std Containers Cannot Delete Elements in foreach

When a std container is in a `foreach` loop with locking enabled, `unset` cannot be performed on its elements.

### 7.9 `unset()` Restores a Fixed Local's Initial State

Calling `unset()` on a fixed typed local releases its current value and restores the type's initial state; it does not make the variable undefined or change its type.

| Local type | Value after `unset()` |
|---|---|
| `int` | `0` |
| `float` | `0.0` |
| `bool` | `false` |
| `string` | empty string |
| `array` | empty array |
| typed object | `null` |

An object has no separate "empty object" value, so `null` is its initial state. A typed object local may also be assigned `null`; any later non-null assignment must still satisfy its class constraint. Dynamic `php::Var` storage retains ordinary PHP undefined behavior.

### 7.10 Object Property Types Are Fixed; Fixed-Value Types Cannot Be unset or Changed to null

The TypePHP compiler requires object properties to always maintain the declared type, and they cannot be changed to other types at runtime.

```php
declare(strict_types=1);

class User {
    public int $id = 0;
    public Profile $profile;
}

$user = new User();
unset($user->id); // ❌ cannot rely on PHP's property unset semantics
$user->id = null; // ❌ fixed-value type property cannot be changed to null

unset($user->profile); // ✅ object property can enter the null/unset state
$user->profile = null; // ✅ object property can be explicitly set to null
```

In the PHP interpreter, `unset($obj->prop)` can put a property into the uninitialized state; assigning `null` to a fixed-value type property also changes the property to an empty value. From the AOT type system's perspective, this is equivalent to changing a property from the declared `int`, `float`, `bool`, `string`, `array` into `null`/uninitialized state. AOT does not allow these fixed-value type properties to change type, so the property always remains its declared type.

Concrete-class object properties use static object type rules: a non-null assignment must satisfy the `is-a` relationship, so a subclass object may be assigned to a base class property, while unrelated objects or base class objects may not be assigned to a subclass property. A non-nullable property can be `unset()`, but cannot be assigned `null`.

If the business needs a "no value" state, explicitly use a nullable type and assign it:

```php
class User {
    public ?int $id = null;
}

$user->id = null; // ✅ the type declaration allows null
```

### 7.11 Select Dynamic Integer Behavior

Use `std::any()` when one value needs dynamic assignment or Zend arithmetic semantics:

```php
$a = std::any(10);        // $a is of type php::Var, retaining overflow detection
$b = $a / 3;         // float division, result is 3.333...
```

For a whole file whose inferred integers need overflow-to-float and fractional division behavior, declare `use varint_types`; explicitly constructed `std::int()` values remain fixed native integers.

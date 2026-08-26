`to*` is a keyword method (`Keyword Method`) provided by the `TypePHP` compiler for explicitly converting the value of an expression or variable to a target type. Unlike ordinary universal methods, `to*` methods hold first-class status in the compiler — regardless of the receiver's type, the compiler performs the conversion according to built-in rules, skipping the type method table lookup.

All `to*` method calls are resolved into explicit C++ function calls at compile time. Basic scalar type conversions usually have no additional dispatch overhead; object type conversions, if they carry a class name, perform an object type check at runtime.

---

## 1. Basic Type Conversion

Called on any expression, converting the result to the corresponding native type:

```php
declare(strict_types=1);
use native_types;

function convert_basic(mixed $input): void
{
    $i = $input->toInt();          // → php::Int    equivalent to (int) $input
    $f = $input->toFloat();        // → php::Float  equivalent to (float) $input
    $s = $input->toString();       // → php::Str    equivalent to (string) $input
    $b = $input->toBool();         // → php::Bool   equivalent to (bool) $input
    $a = $input->toArray();        // → php::Array  equivalent to (array) $input
}
```

| Method | Target type | Generated code | Description |
|------|---------|---------|------|
| `toInt()` | `php::Int` | `php::toInt($expr)` | May overflow |
| `toFloat()` | `php::Float` | `php::toFloat($expr)` | |
| `toString()` | `php::Str` | `php::toString($expr)` | |
| `toBool()` | `php::Bool` | `php::toBool($expr)` | |
| `toArray()` | `php::Array` | `php::toArray($expr)` | See "Object to Array" below |
| `toAny()` | `php::Var` | `php::Var($expr)` | Degrades to dynamic type, equivalent to `any($expr)` |

### `toArray()` Object to Array

When the receiver is an **object**, `php::toArray()` first checks whether the object's class defines a `toArray()` method:

- **Has a `toArray()` method**: call that method; the return value must be an array, otherwise an exception is thrown
- **No `toArray()` method**: follow the original logic, converting the object's property table into an array

> `toArray()` is a TypePHP reserved keyword method, handled with zero-argument conversion semantics. Do not define a business method of the same name that requires arguments; such calls will not pass arguments like ordinary object methods.

```php
class User {
    public int $id;
    public string $name;

    public function toArray(): array {
        return [
            'uid' => $this->id,
            'display_name' => $this->name,
        ];
    }
}

function main(): void {
    $user = new User(1, 'admin');
    $arr = $user->toArray();  // → php::toArray($user)
    // calls User::toArray(), returning a custom array structure
    // $arr = ['uid' => 1, 'display_name' => 'admin']

    // object without a toArray() method → reads the property table
    $plain = (array) $other;  // → php::toArray($other)
    // returns the property table: ['id' => ..., 'name' => ...]
}
```

> **Note**: The lookup of the `toArray()` method is based on PHP's function table (case-insensitive), so both `toArray` and `toarray` match. If `toArray()` returns a non-array type, the compiler throws a `"toArray() method must return an array, got ..."` exception.

### Difference from PHP Casts

PHP's `(int)` / `(float)` cast syntax is also supported, but the `to*` methods have the following advantages:

- **Chained calls**: `$result->toString()->length()` completes conversion and operation in one step, without intermediate variables
- **Type inference**: the compiler precisely infers the `to*` return type, so subsequent method calls can enjoy targeted optimizations
- **Universality**: `to*` works on `mixed` types and Big* types alike

---

## 2. Dynamic Type and Reference Conversion

`toAny()` and `toRef()` are TypePHP-specific keyword methods used to replace the functional forms `any()` and `refval()`. The two are fully equivalent, but the method form is better suited for chained expressions.

### 2.1 `toAny()`

`toAny()` degrades an expression to the `php::Var` / `mixed` / `any` dynamic type, equivalent to `any($expr)`. It does not restore object class info and does not generate object type checks.

```php
declare(strict_types=1);
use native_types;

function any_example(object $value): void
{
    $a = any($value);
    $b = $value->toAny();  // equivalent to any($value)
}
```

Typical use cases:

- When you want to degrade from a typed object to a dynamic value, to avoid generating native calls according to a static object type.
- When you need operations to return to ZendVM/PHP dynamic semantics, such as integer division, mixed-type operations, etc.
- When a parameter type requires `mixed` / `any`, but the current expression is a native type or a typed object.

`toAny()` accepts no arguments:

```php
$value->toAny();       // ✅
$value->toAny($type);  // ❌ compile error
```

### 2.2 `toRef()`

`toRef()` explicitly converts an expression to a reference, equivalent to `refval($expr)`. It is mainly used for scenarios where the compiler cannot know at compile time whether a parameter is passed by reference, such as dynamic calls and closure calls.

```php
function append_text(&$value, string $suffix): void
{
    $value .= $suffix;
}

function ref_example(): void
{
    $name = 'AOT';

    append_text(refval($name), ' compiler');
    append_text($name->toRef(), ' runtime'); // equivalent to refval($name)
}
```

`toRef()` can only be used on lvalues the compiler can locate, such as variables, array elements, and object properties:

```php
$value->toRef();        // ✅ variable
$array['key']->toRef(); // ✅ array element
$object->prop->toRef(); // ✅ object property

(1 + 2)->toRef();       // ❌ constant/temporary expression cannot be converted to reference
foo()->toRef();         // ❌ call result cannot be converted to reference
```

Like `refval()`, `toRef()` accepts no arguments:

```php
$value->toRef();     // ✅
$value->toRef(true); // ❌ compile error
```

> **Tip**: When the parameter information of static functions and built-in functions is clear, the compiler can handle reference parameters automatically. Only scenarios where the parameter signature cannot be obtained at compile time, such as dynamic calls, closure calls, and variable function calls, require the explicit use of `toRef()` / `refval()`.

---

## 3. Stream Type Conversion

`toStream()` converts the value of an expression to `php::Stream`, after which you can directly call Stream universal methods (such as `write`, `read`, `close`).

```php
declare(strict_types=1);
use native_types;

function stream_example(): void
{
    // continue the Stream type after taking an element from an array
    $pair = stream_socket_pair(AF_UNIX, SOCK_STREAM, 0);
    $pair[0]->toStream()->write("hello");   // → fwrite($pair[0], "hello")
    $data = $pair[1]->toStream()->read(5);  // → fread($pair[1], 5)
    var_dump($data);                        // string(5) "hello"
}
```

> **Note**: `toStream()` does not automatically close the connection. Call `->close()` to release the resource after use.

---

## 4. High-Precision Numeric Type Conversion

Big* types (BigInt / Decimal / BigFloat) define more precise conversion paths; the compiler uses dedicated conversion functions for each source type to avoid precision loss.

### 4.1 Conversions on BigInt

```php
declare(strict_types=1);
use native_types;

function bigint_convert(): void
{
    $b = std::bigInt("12345678901234567890");

    $i = $b->toInt();            // php::BigInt::toInt($b) — may overflow
    $f = $b->toFloat();          // php::BigInt::toFloat($b)
    $s = $b->toString();         // php::BigInt::toString($b)
    $d = $b->toDecimal();        // php::newDecimal(php::BigInt::toString($b))
    $bf = $b->toBigFloat();      // php::BigFloat::newInstance(php::BigInt::toString($b))
}
```

### 4.2 Conversions on Decimal

```php
$d = std::decimal("123.456");

$i = $d->toInt();               // php::Decimal::toInt($d)
$f = $d->toFloat();             // php::Decimal::toFloat($d)
$s = $d->toString();            // php::Decimal::toString($d)
$b = $d->toBigInt();            // php::newBigInt(php::Decimal::toString($d))
$bf = $d->toBigFloat();         // php::BigFloat::newInstance(php::Decimal::toString($d))
```

### 4.3 Conversions on BigFloat

```php
$bf = std::bigFloat("1.23e100");

$i = $bf->toInt();              // php::BigFloat::toInt($bf)
$f = $bf->toFloat();            // php::BigFloat::toFloat($bf)
$s = $bf->toString();           // php::BigFloat::toString($bf)
$b = $bf->toBigInt();           // php::newBigInt(php::BigFloat::toString($bf))
$d = $bf->toDecimal();          // php::newDecimal(php::BigFloat::toString($bf))
```

### 4.4 Cross Big* Type Conversion Rules

Big* → String always goes through each type's `toString()` static method, avoiding binary floating-point error:

| Source type | Target type | Generated code |
|--------|---------|---------|
| BigInt | Decimal | `php::newDecimal(php::BigInt::toString($b))` |
| BigInt | BigFloat | `php::BigFloat::newInstance(php::BigInt::toString($b))` |
| Decimal | BigInt | `php::newBigInt(php::Decimal::toString($d))` |
| Decimal | BigFloat | `php::BigFloat::newInstance(php::Decimal::toString($d))` |
| BigFloat | BigInt | `php::newBigInt(php::BigFloat::toString($bf))` |
| BigFloat | Decimal | `php::newDecimal(php::BigFloat::toString($bf))` |

> All cross Big* conversions pass through a string intermediate state to ensure decimal precision is not lost.

---

## 5. Object Type Conversion

`toObject()` converts the value of an expression to `php::Object`. With no argument it yields a generic object (no concrete class info); passing `ClassName::class` reconstructs a type with class info.

```php
declare(strict_types=1);
use native_types;

function object_convert(mixed $input): void
{
    // no argument: generic object (no class info)
    $obj = $input->toObject();

    // with class name: the compiler can optimize subsequent method calls
    $user = $input->toObject(User::class);
    echo $user->getName();   // Native Call
}
```

| Usage | Generated code | Type info |
|------|---------|---------|
| `$x->toObject()` | `php::toObject($x)` | None (`php::Object`) |
| `$x->toObject(User::class)` | `php::toObject($x, ce_User)` | Yes (compiler knows the target class) |

> **Tip**: Both `toObject(ClassName::class)` and `objval($var, ClassName::class)` can be used for object type continuation; the former supports chained calls. Object conversions with a class name use PHP `instanceof` / `is-a` relationships for runtime checking. See [Object Type Conversion](object-type-conversion.md) for details.

---

## 6. Std Container Type Conversion

The `toStd*` methods convert a variable of type `php::Var` into the specified C++ standard library container type. These methods **must be called at the top-level scope** and **cannot reassign** an already-declared variable.

```php
declare(strict_types=1);
use native_types;

function std_convert(): void
{
    $data = get_data();  // returns mixed, internally holding a StdContainerBox

    // recover from mixed to std::vector<int>
    $v = $data->toStdVector(Type::Int);
    $v[] = 42;
    echo $v[0];  // 42

    // recover from mixed to std::map<string, int>
    $m = $data->toStdOrderedMap(Type::String, Type::Int);
    $m["key"] = 100;
}
```

| Method | Target type | Key type limitation | Description |
|------|---------|-----------|------|
| `toStdArray(type, size)` | `php::StdArray<T, N>` | index is `int` | Fixed-length array, size determined at compile time |
| `toStdVector(type)` | `php::StdVector<T>` | index is `int` | Dynamic array |
| `toStdOrderedMap(ktype, vtype)` | `php::StdOrderedMap<K, T>` | `int` or `string` | Ordered map |
| `toStdMap(ktype, vtype)` | `php::StdMap<K, T>` | `int` or `string` | Hash map |

### Usage Limitations

- **Must be at top-level scope**: `toStd*` can only be called in the outermost scope of a function body, not in nested blocks such as `if` / `for`
- **Cannot reassign**: a variable assigned via `toStd*` cannot be reassigned to another type
- **Source variable must already exist**: `toStd*` must act on a variable that is already defined and has a value

---

## 7. Type Inference

The compiler has precise built-in inference rules for the return types of `to*` methods. Regardless of the receiver's type, when a `to*` method is detected, the corresponding target type is returned directly:

```php
$val = get_any_value();
// compiler inference: $val->toString() returns php::Str, and length() is a universal method on Str
echo $val->toString()->length();
```

Type inference covers the following scenarios:
- **Chained calls**: `$a->toBigInt()->mul(3)->toString()` — the compiler infers level by level: BigInt → BigInt → Str
- **Dynamic degradation**: `$obj->toAny()` — the compiler treats subsequent values as `php::Var`
- **Explicit reference**: `$value->toRef()` — the compiler passes the argument by reference
- **Conditional branches**: in `if`, the return types of `to*` in the two branches are inferred independently
- **Function return values**: `return $x->toString()` → the function return type is `php::Str`

---

## 8. void Type Behavior

When an expression whose return type is `void` / `never` is used as a value, it is treated as `null`. Continuing to call `to*` methods on such an expression has no practical meaning; the result is equivalent to performing the corresponding conversion on `null`.

```php
function bar(): void
{
    var_dump(__FUNCTION__);
}

$value = bar()->toAny();     // equivalent to any(null)
$text = bar()->toString();   // equivalent to php::toString(null)
```

---

## 9. Relationship with Universal Methods

`to*` methods are **language keywords**, not ordinary universal methods:

| Feature | `to*` keyword method | Ordinary universal method |
|------|-----------------|-------------|
| Method lookup | Directly matches the built-in table `KEYWORD_METHOD_MAP` | Looks up the `UNIVERSAL_METHODS` table by type |
| Receiver type | Any type is allowed (including `mixed`) | Must match a registered type |
| Argument validation | Special handling at compile time | Universal `min_args` / `max_args` validation |
| Code generation | Dedicated logic via `genToConvertCall()` | Dispatched by handler type |

This design allows `to*` methods to work unambiguously on `mixed` / `any` types, and is the core mechanism for type continuation (recovering a static type from a dynamic type).

---

## 10. Comprehensive Example

```php
declare(strict_types=1);
use native_types;

function comprehensive_convert(): void
{
    // chained conversion + high-precision arithmetic
    $a = std::bigInt("100");
    $result = $a->toDecimal()
                ->mul(std::decimal("0.05"))
                ->toBigFloat()
                ->add(std::bigFloat("1.0"))
                ->toString();
    var_dump($result);  // precise decimal result string

    // chained Stream operations
    $pair = stream_socket_pair(AF_UNIX, SOCK_STREAM, 0);
    $pair[0]->toStream()->write("ping");
    $response = $pair[1]->toStream()->read(4);
    var_dump($response);  // string(4) "ping"

    // Std container extraction
    $raw = get_container();                 // returns mixed
    $vec = $raw->toStdVector(Type::Int);
    $vec[] = 10;
    $vec[] = 20;
    echo $vec->count();                     // 2
}
```

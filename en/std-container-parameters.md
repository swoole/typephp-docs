# Type Annotations

`StdVector`, `StdMap`, `StdOrderedMap`, `StdList`, and `StdDict` are collectively called "type annotations". They use PHP Attribute syntax to declare a parameter or property's container kind and key/value types. The first three describe Box-wrapped C++ containers: the compiler checks the incoming Box and restores its reference automatically, without a `toStd*()` call in the function body. `StdList` / `StdDict` describe [typed PHP arrays](typed-arrays.md).

The PHP parameter or property type may be omitted or declared as a compatible storage type:

| Type annotation | Compatible PHP type |
|---|---|
| `StdVector` / `StdMap` / `StdOrderedMap` | `box` |
| `StdList` / `StdDict` | `array` |

Explicit `mixed`, `any`, nullable types, unions, and other incompatible types are rejected. The Box-container parameter rules below apply to the first three annotations; see [typed PHP arrays](typed-arrays.md) for list/dict reference parameters and other rules.

## Basic Usage

```php
function append(#[StdVector(Type::Int)] box $values): void
{
    $values[] = 42;
}

function update(#[StdMap(Type::String, Type::Float)] $prices): void
{
    $prices['apple'] = 2.5;
}

class User {}

function visit(#[StdOrderedMap(Type::Int, User::class)] $users): void
{
    foreach ($users as $id => $user) {
        var_dump($id, $user);
    }
}

function main(): void
{
    $values = std::vector(Type::Int);
    append($values);
    var_dump($values[0]); // int(42), the caller observes the modification

    $prices = std::map(Type::String, Type::Float);
    update($prices);
    var_dump($prices['apple']); // float(2.5)

    $users = std::orderedMap(Type::Int, User::class);
    $users[1] = new User();
    visit($users);
}
```

| Type annotation | Corresponding container | Type arguments |
|---|---|---|
| `#[StdVector(T)]` | `std::vector(T)` | One element type |
| `#[StdMap(K, V)]` | `std::map(K, V)` | Key type, value type |
| `#[StdOrderedMap(K, V)]` | `std::orderedMap(K, V)` | Key type, value type |

`T`, `K`, and `V` are placeholders in this table. Actual declarations require `Type::*` constants or `ClassName::class` supported by the std factories, not runtime variables, strings such as `"int"`, or named arguments. See [Std Containers](std-containers.md) for value types; keys only support `Type::Int` and `Type::String`.

The `StdVector` type annotation does not accept an initial size. The caller can still specify one through `std::vector(Type::Int, 100)`; it is not part of the parameter type contract. There is currently no `StdArray` parameter type annotation; use `toStdArray()` for fixed-length arrays.

## Migrating from toStd*()

Explicit type recovery remains supported:

```php
function append_explicit($source): void
{
    $values = $source->toStdVector(Type::Int);
    $values[] = 42;
}
```

With a parameter type annotation, operate on the parameter directly and remove the local type-recovery statement:

```php
function append_typed(#[StdVector(Type::Int)] $values): void
{
    $values[] = 42;
}
```

This improves the type contract and syntax; it does not introduce a different container storage model. Explicit recovery of local Box values and fixed-length arrays can still use the [toStd* keyword methods](keyword-method.md).

## Namespaces and Methods

The type annotations and `Type` live in the root namespace and follow PHP name resolution, including fully qualified names and imported aliases:

```php
namespace App;

use StdVector as VectorOf;
use Type;

interface IntConsumer
{
    public function accept(#[VectorOf(Type::Int)] $values): void;
}

class Consumer implements IntConsumer
{
    public function accept(#[VectorOf(Type::Int)] $values): void
    {
        $values[] = 1;
    }
}
```

Parameters of named functions, methods, interface methods, abstract methods, and trait methods are supported. An override or interface implementation that retains a container contract must use the same container kind, key type, and value type: it cannot replace `StdVector(Type::Int)` with `StdVector(Type::Float)`. If a child method widens the parameter to an untyped or `mixed` parameter, automatic container type recovery no longer applies to that parameter.

## Call Boundaries and Type Checking

The ABI still passes the Box handle as `php::Var`. The function entry checks the Box, container kind, and element types, then obtains a concrete C++ container reference without copying the container or converting its elements individually. These checks execute on each call, but do not require runtime Attribute scanning or reflection.

An incorrect container kind, key or value type, PHP array, `null`, or other non-Box value causes a `TypeError`. The type annotations do not automatically convert PHP arrays into std containers. Even when the value type is `Type::Any`, the argument must match that container contract.

The existing dynamic-callable rule that converts statically known std container arguments into PHP arrays is unchanged: `$callback($values)` is not equivalent to a named direct call such as `append($values)`. Dynamic calls to these functions require an actual Box value. For example, a direct call to a function without a container type contract can return a dynamically typed Box value:

```php
function box_value($value) { return $value; }

function main(): void
{
    $values = std::vector(Type::Int);
    $box = box_value($values); // Direct call preserves the Box; the return has no static container metadata
    $callback = 'append';
    $callback($box);
    var_dump($values[0]); // int(42)
}
```

## Current Limitations

- Only one type annotation is allowed per parameter or property; duplicates or combinations are rejected. The PHP type may be omitted or declared as `box`: `#[StdVector(Type::Int)] mixed $values` is a compilation error.
- Reference, variadic, defaulted, and constructor-promoted parameters are unsupported.
- Closure, arrow-function, and Generator parameters are unsupported. Generic return-value declarations are not provided.
- Containers cannot hold Native objects across this Box parameter boundary; existing [Native object escape restrictions](native-class.md) remain unchanged.
- A parameter binding cannot be replaced with another Box or ordinary value, unset with `unset($values)`, or captured by reference by a Closure. Assigning a same-type std container still copies its contents under the existing rules; element reads, writes, appends, deletions, and iteration follow the [Std container restrictions](std-containers.md).

The contract is preserved in incremental compilation's declaration cache and exported library stubs. Changing a type annotation causes callers depending on the declaration to be translated again. These type annotations are interpreted only by TypePHP; they do not give ordinary Zend PHP C++ generic containers or equivalent parameter checks.

## Property Type Annotations

```php
class State
{
    #[StdVector(Type::Int)] public box $values;
    #[StdMap(Type::Str, Type::Int)] public $counts;
    #[StdOrderedMap(Type::Int, User::class)] public box $users;
}
```

Properties may omit the PHP type or declare `box`. The compiler preserves the container contract and detects conflicting attributes. A property is not a function-entry local container binding: operating on a property currently still uses existing `toStd*()` recovery, without generating a long-lived C++ container reference. Native classes still cannot have Box container properties.

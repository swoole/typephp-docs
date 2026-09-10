# Strongly Typed References

TypePHP supports references without changing the fixed type of a local variable. The compiler uses two reference paths depending on whether the callee is known at compile time.

## Native References for Statically Resolved Calls

Fixed `int`, `float`, `bool`, `string`, and `array` locals can be passed to an exactly typed by-reference parameter. When the function or method is statically resolved, TypePHP emits a native C++ `T&`; no `zval` boxing or Zend reference allocation is needed.

```php
function increment(int &$value): void
{
    $value++;
}

function main(): void
{
    $count = 1;
    increment($count); // $count remains int
    var_dump($count);  // int(2)
}
```

The parameter type must exactly match the local's fixed type. A reference cannot be used to write a different type into the local.

Fixed `int`, `float`, `bool`, `string`, and `array` properties of a Native Class can also be passed directly to an exactly matching reference parameter:

```php
#[Native]
class Counter
{
    public int $value = 0;
}

function increment(int &$value): void
{
    $value++;
}

$counter = new Counter();
increment($counter->value);
```

This creates a call-scoped C++ `T&` to the fixed field; it does not create a PHP reference. The property cannot be bound with `=&`, wrapped in `std::ref()`, returned by reference, or retained after the call. Nullable, union, `mixed`, hooked, and readonly Native properties do not use this path. A Native property declared as `any` instead supports the ordinary dynamic PHP reference model.

By-reference variadic parameters are also supported for ordinary functions and methods whose signatures are known at compile time, including direct, named, and unpacked arguments. A dynamic closure cannot declare a by-reference variadic parameter.

## Local Reference Aliases

A fixed local may have a one-time, unconditional, function-local alias:

```php
function main(): void
{
    $name = "TypePHP";
    $alias =& $name;
    $alias .= " compiler";

    var_dump($name); // string(16) "TypePHP compiler"
}
```

The alias is a C++ reference with a permanent target. Therefore it cannot be rebound, initialized only in a conditional branch or loop, unset, captured by reference, returned by reference, or stored in an array, property, or global variable.

## Dynamic Calls: `std::ref()` and `toRef()`

For a closure, variable function, or another call whose signature cannot be resolved statically, mark a by-reference argument explicitly:

```php
function main(): void
{
    $rename = function (string &$value): void {
        $value .= " compiler";
    };

    $name = "TypePHP";
    $rename(std::ref($name));
    // Equivalent: $rename($name->toRef());
}
```

`std::ref()` and `toRef()` accept only a variable, array element, or object property, and only as a call argument. TypePHP creates a call-scoped Zend reference, writes the result back after validating that its type has not changed, and rejects dynamic code that retains the temporary reference beyond the call.

Calls with a statically known signature infer by-reference parameters automatically; do not add `std::ref()` to them.

## Supported Boundaries

Native local references are supported for `int`, `float`, `bool`, `string`, and `array`. Fixed object, resource/stream, high-precision, Native Class object handles, typed-object, Box, and Std-container locals cannot be referenced. These values already have handle-like semantics or fixed layouts for which rebinding would weaken the type system. This restriction on Native Class object handles does not prevent an exact fixed Native property from being passed directly to a matching strongly typed reference parameter.

Typed object and static properties remain reference-capable through Zend's typed-property checks, and PHP array elements remain dynamically reference-capable.

If code needs unrestricted PHP reference identity, initialize the value with `std::any()` and use the dynamic `php::Var` reference path instead of a fixed local.

## Choosing the Right Form

| Situation | Form |
|---|---|
| Known TypePHP function/method with an exact `&` parameter | Pass the variable directly |
| Fixed Native property and a matching, statically known `&` parameter | Pass the property directly |
| One local alias with a permanent target | `$alias =& $value` |
| Closure, variable function, or dynamic method call | `std::ref($value)` or `$value->toRef()` |
| Reference must escape, rebind, or follow unrestricted PHP identity | Store the value as `std::any()` |

See [Compile-time Functions](compile-time-functions.md) for the `std::ref()` call wrapper and [Keyword Methods](keyword-method.md) for `toRef()`.

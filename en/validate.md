# Validate Compile-time Attribute

`Validate` uses PHP's `filter_var()` validation rules to check the parameters of ordinary functions, methods, or anonymous functions. The check statements are **statically inserted at compile time**; at runtime it does not scan Attributes, does not use reflection, and does not perform dynamic invocation. The Attribute processing itself has no runtime overhead, while the generated `filter_var()` checks execute when the function is called. It does not support arrow functions, because an arrow function has only a single expression and no function body into which the check statements can be inserted.

## Basic Usage

```php
function register(
    #[Validate(FILTER_VALIDATE_EMAIL)]
    string $email,
): void {
}
```

When validation fails it throws:

```php
throw new \ValueError('Parameter $email is invalid');
```

`Validate` only validates parameters; it does not overwrite or convert the original parameter using the return value of `filter_var()`.

## Official PHP Reference

- [Validation filters list, available options and flags](https://www.php.net/manual/zh/filter.filters.validate.php)
- [All predefined Filter constants](https://www.php.net/manual/zh/filter.constants.php#constant.filter-validate-bool)
- [`filter_var()` function description](https://www.php.net/manual/zh/function.filter-var.php)
- [`FILTER_NULL_ON_FAILURE`](https://www.php.net/manual/zh/filter.constants.php#constant.filter-null-on-failure)

The rules usable with `Validate` are the `FILTER_VALIDATE_*` entries in the official PHP "validation filters" list. The supported `options` and flags for each rule should also follow that list.

## Constructor Parameters

```php
#[Validate(
    int $filter,
    int|array $options = 0,
    ?string $message = null,
)]
```

- `filter`: required, only `FILTER_VALIDATE_*` validation rules are allowed.
- `options`: optional, format is identical to the third argument of `filter_var()`, and may be an integer flags value or an array.
- `message`: optional, a custom `ValueError` message.

`FILTER_SANITIZE_*` is not allowed. This Attribute's responsibility is validation, not sanitizing or modifying data; passing a sanitize rule produces a compile error.

## Parameter Type Compatibility

TypePHP rejects, at compile time, filter and parameter type combinations that can be statically proven to always fail. For example:

```php
function register(
    #[Validate(FILTER_VALIDATE_EMAIL)] int $email,
): void {
}
```

No `int` value can possibly match an email format, so the code above produces a compile error. URL, IP, and MAC address validation apply the same rule to explicitly non-string scalars such as `int`, `float`, and `bool`.

PHP's `filter_var()` converts scalars to strings before filtering, so TypePHP does not simply require all parameters to be declared as `string`. For example, `FILTER_VALIDATE_INT` can be used with `int` or `string`, and `FILTER_VALIDATE_BOOLEAN` can also be used with `bool`, numeric values, or strings.

Union types are allowed to compile as long as at least one member can pass validation:

```php
function register(
    #[Validate(FILTER_VALIDATE_EMAIL)] int|string $email,
): void {
}
```

`array` parameters are by default incompatible with scalar filters; array mode is only recognized after explicitly using `FILTER_REQUIRE_ARRAY` or `FILTER_FORCE_ARRAY`. For undeclared types, `mixed`, callables, and object types whose failure cannot be proven at the current stage, the compiler preserves runtime validation rather than making a rejection judgment based on heuristics.

## Ranges and Flags

```php
function connect(
    #[Validate(
        FILTER_VALIDATE_INT,
        options: [
            'options' => [
                'min_range' => 1,
                'max_range' => 65535,
            ],
        ],
        message: 'Port must be between 1 and 65535',
    )]
    int $port,
): void {
}
```

Integer flags can also be passed directly:

```php
function connect(
    #[Validate(FILTER_VALIDATE_IP, FILTER_FLAG_IPV4)]
    string $address,
): void {
}
```

The compiler automatically merges `FILTER_NULL_ON_FAILURE` into `options` and uses a strict `=== null` check to determine validation failure. This distinguishes a legitimate `false` returned by `FILTER_VALIDATE_BOOLEAN` from an actual validation failure.

## Combining with NotNull

`Validate` can be used together with `NotNull` or `NotEmpty`. The check order on the same parameter is fixed as `NotNull`, `NotEmpty`, `Validate`, regardless of the order in which the Attributes are written:

```php
function register(
    #[Validate(FILTER_VALIDATE_EMAIL)]
    #[NotEmpty]
    #[NotNull]
    string $email,
): void {
}
```

In the example above, even if the Attributes are written in reverse order, a null check, an `empty()` check, and an email format validation are performed in that sequence.

## Namespace and library

`Validate` is in the root namespace. Inside a namespace use `#[\Validate(...)]`, or import it first via `use \Validate;`; `use ... as ...` aliases are also supported.

When using `-m lib`, the published `<target>.stub.php` preserves the `Validate` declaration; the library internally contains the actual check code, and consuming projects use the generated method or function declarations without needing to resolve the Attribute at runtime.

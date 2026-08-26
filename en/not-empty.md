# NotEmpty Compile-Time Attribute

`NotEmpty` is used on parameters of ordinary functions, methods, or anonymous functions. When `empty($parameter)` is `true`, it throws a `ValueError` at the very beginning of the function body. Arrow functions are not supported, because an arrow function has only a single expression and no function body in which to insert the check statement.

```php
function save(#[NotEmpty] string $name): void
{
}
```

The compiler statically inserts an equivalent check:

```php
if (empty($name)) {
    throw new \ValueError('Parameter $name must not be empty');
}
```

It uses PHP's `empty()` semantics, so `null`, `false`, `0`, `0.0`, the empty string, the string `'0'`, and the empty array all fail. To reject only null, use [`NotNull`](not-null.md).

The check statement is generated at compile time and does not require runtime Attribute parsing; the actual `empty()` check executes on every call. `NotEmpty` accepts no arguments and supports ordinary functions, methods, closures, and constructor-promoted parameters.

When multiple validation Attributes are combined on the same parameter, the check order is fixed as `NotNull`, `NotEmpty`, `Validate`, regardless of the order in which the Attributes are written.

In namespaces, use `#[\NotEmpty]`, or import it first via `use \NotEmpty;`.

# NotNull Compile-Time Attribute

`NotNull` is used on parameters of ordinary functions, methods, or anonymous functions. The compiler checks the parameter before the other statements of the function body; when the parameter is strictly equal to `null`, it throws a `ValueError`. Arrow functions are not supported, because an arrow function has only a single expression and no function body in which to insert the check statement.

The check code is **statically inserted at compile time**; at runtime no Attributes are parsed, and it does not depend on reflection or dynamic calls. Attribute handling itself has no runtime overhead; the inserted strict null check executes when the function is called.

```php
function save(#[NotNull] ?string $name): void
{
    // business code
}
```

Because `?string` explicitly allows `null` while `NotNull` rejects `null` at the function entry, the compiler reports a non-fatal warning about this, and compilation continues. The warning also applies to union types that explicitly contain `null`; undeclared types and `mixed` do not trigger this warning merely because their runtime value could be `null`.

This is equivalent to executing at the function entry:

```php
if ($name === null) {
    throw new \ValueError('Parameter $name must not be null');
}
```

`0`, `0.0`, `false`, the empty string, the string `'0'`, and the empty array are not null, so they do not trigger `NotNull`. To reject these empty values, use [`NotEmpty`](not-empty.md).

When multiple parameters use `NotNull`, the checks are inserted in the order the parameters are declared. It can also be used on constructor parameters corresponding to constructor-promoted properties.

To validate formats such as email, IP, URL, or integer ranges, it can be combined with [`Validate`](validate.md).

When multiple validation Attributes are combined on the same parameter, the check order is fixed as `NotNull`, `NotEmpty`, `Validate`, regardless of the order in which the Attributes are written.

`NotNull` accepts no arguments. In namespaces, use `#[\NotNull]`, or import it first via `use \NotNull;`.

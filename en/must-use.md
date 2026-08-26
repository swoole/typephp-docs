# MustUse Compile-time Attribute

`MustUse` is used to forbid callers from discarding the return value of a function or method. It suits error results, immutable objects, and computation results that must be handled explicitly.

```php
#[MustUse]
function openFile(string $file): Result
{
}

openFile('data.txt');          // compile error
$result = openFile('data.txt'); // correct
```

Methods are equally supported:

```php
class User
{
    #[MustUse]
    public function withName(string $name): static
    {
    }
}
```

Assignment, returning, passing as an argument, participating in an expression, or continuing to call a method all count as using the return value. An error is reported only when the call itself is written as a standalone expression.

`MustUse` is a purely compile-time constraint; it generates no runtime code and uses no reflection or dynamic checking. It cannot be used on functions or methods that return `void`.

Inside a namespace use `#[\MustUse]`, or import it first via `use \MustUse;`. When using `-m lib`, the published stub preserves the declaration so consuming projects can perform the same check.

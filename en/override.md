# Override Compile-time Attribute

`Override` is used to explicitly declare that a method must override a parent class method, or a method declared in an interface implemented by the class or by its parent class. TypePHP checks this requirement at compile time; when no matching method is found it produces a fatal error directly, unaffected by PHP's `error_reporting` configuration.

```php
class BaseService
{
    public function execute(): void
    {
    }
}

class UserService extends BaseService
{
    #[\Override]
    public function execute(): void
    {
    }
}
```

Implementing an interface method also counts as a valid override:

```php
interface Formatter
{
    public function format(): string;
}

class JsonFormatter implements Formatter
{
    #[\Override]
    public function format(): string
    {
        return '{}';
    }
}
```

If neither the parent class nor any interface declares a method with the same name, TypePHP stops compilation:

```php
class UserService
{
    #[\Override]
    public function execute(): void
    {
    }
}
```

```text
Fatal error: UserService::execute() has #[\Override] attribute,
but no matching parent method exists
```

## Hard Rules

- It can only be used on methods, accepts no arguments, and cannot be declared more than once on the same method.
- It matches public or protected methods of the parent class, and also matches abstract methods and interface methods.
- Parent class private methods are not override targets.
- `__construct()` is not an `Override` matching target, even if the parent class also defines a constructor.
- After a same-name target is found, the method must still pass the ordinary compatibility checks for return type, parameter types, visibility, static, and final.
- `Override` on a trait method is validated when the trait is used by a concrete class; the same trait can be used in classes that satisfy the contract, and produce a compile error in classes that have no matching parent or interface method.

## Namespace

`Override` is a built-in Attribute in PHP's root namespace. Inside a namespace use the fully qualified name:

```php
#[\Override]
public function execute(): void
{
}
```

It can also be imported or aliased according to ordinary PHP rules:

```php
namespace App;

use \Override;
use \Override as Replaces;

class ParentClass
{
    public function first(): void
    {
    }

    public function second(): void
    {
    }
}

class Child extends ParentClass
{
    #[Override]
    public function first(): void
    {
    }

    #[Replaces]
    public function second(): void
    {
    }
}
```

TypePHP recognizes it exactly according to PHP's name resolution rules; a same-name Attribute in another namespace such as `App\Override` does not have this compile-time semantics.

When using `-m lib`, the generated library stub preserves `Override`, so consuming projects continue to validate the same inheritance contract at compile time.

Reference: [PHP `Override` official documentation](https://www.php.net/manual/zh/class.override.php).

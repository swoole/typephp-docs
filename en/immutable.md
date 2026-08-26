# Immutable Compile-time Attribute

`#[Immutable]` declares that a piece of code can only read an object or parameter, not modify it. It is similar to C++ `const`: the check happens entirely during the TypePHP compilation stage, creates no read-only proxy object, writes no Zend metadata, and adds no runtime branches.

It suits query methods, value objects, read-only service interfaces, and public APIs where accidental modification should be prevented by the compiler.

For the namespace rules shared by all built-in Attributes, see [Compile-time Attributes](compile-time-attributes.md).

## Immutable Methods

Placing `#[Immutable]` on an instance method treats `$this` inside that method as an immutable object:

```php
class User
{
    private string $name = 'Rango';

    #[Immutable]
    public function name(): string
    {
        return $this->name;
    }

    public function rename(string $name): void
    {
        $this->name = $name;
    }

    #[Immutable]
    public function description(): string
    {
        return 'User: ' . $this->name(); // allowed: name() is also Immutable
    }
}
```

An immutable method can only call other immutable methods. The following call fails at compile time:

```php
class User
{
    #[Immutable]
    public function invalid(): void
    {
        $this->rename('new name');
        // Fatal error: cannot call a mutable method on immutable $this
    }
}
```

`#[Immutable]` cannot be placed on a plain function declaration, because a plain function has no `$this`. Place it on the function parameters that need protection instead.

## Immutable Parameters

`#[Immutable]` on a parameter protects both the parameter variable and the object it references:

```php
function display(#[Immutable] User $user): string
{
    return $user->name();
}

function total(#[Immutable] array $values): int
{
    return count($values);
}
```

It can be used on function, method, constructor, and Closure parameters. An immutable parameter cannot be reassigned, have its elements modified, or be passed to a parameter that may modify it:

```php
function invalid(#[Immutable] array $values, #[Immutable] User $user): void
{
    $values[] = 1;           // compile error
    sort($values);           // compile error: sort()'s parameter is a writable reference
    $user = new User();      // compile error
    $user->rename('other');  // compile error
}
```

A called function can also explicitly receive an immutable object with `#[Immutable]`:

```php
function userName(#[Immutable] User $user): string
{
    return $user->name();
}
```

An immutable reference parameter is legal, and its semantics are similar to C++ `const &`: the call still passes by reference, but the function body cannot modify the value.

```php
function inspect(#[Immutable] User &$user): string
{
    return $user->name();
}
```

## Operations Forbidden by the Compiler

For an immutable `$this`, parameter, or an object alias derived from it, the compiler forbids:

- reassignment, destructuring assignment, property writes, and array element writes;
- compound assignments such as `+=` and `.=`, as well as `++` and `--`;
- `unset()`, taking references, and `foreach (... as &$value)`;
- calling determinate object methods not marked `#[Immutable]`;
- passing the object to a determinate parameter without `#[Immutable]`;
- passing to a writable reference parameter;
- saving the immutable object into mutable properties, arrays, global or static state;
- directly `return`-ing or `yield`-ing the immutable object so it escapes as a mutable object.

A local alias does not lift the restriction:

```php
function inspect(#[Immutable] User $user): string
{
    $alias = $user;
    $alias->rename('other'); // compile error, $alias is still immutable
    return $alias->name();
}
```

`clone` creates a new object with an independent identity, so the clone result is mutable:

```php
function renamedCopy(#[Immutable] User $user): User
{
    $copy = clone $user;
    $copy->rename('copy'); // allowed
    return $copy;
}
```

For scalars and PHP copy-on-write values, ordinary reads and copies are not modifications. For example, an immutable array can be copied and `count()` can be called on it; it only fails when writing to the original value or handing it to a writable reference parameter.

## Property Hook

When reading a Property Hook through an immutable object, the getter itself must also be declared `#[Immutable]`:

```php
class Profile
{
    public string $name = 'Rango' {
        #[Immutable]
        get => strtoupper($this->name);
    }

    #[Immutable]
    public function displayName(): string
    {
        return $this->name;
    }
}
```

For the declaration and runtime requirements of Property Hooks, see the PHP 8.4 property hook rules.

## MethodsFor Extension Methods

An object extension method can act on an immutable object only when its receiver parameter is also declared `#[Immutable]`:

```php
#[MethodsFor(User::class)]
class UserExtensions
{
    public static function label(#[Immutable] User $user): string
    {
        return $user->name();
    }
}

function label(#[Immutable] User $user): string
{
    return $user->label();
}
```

Keyword extension methods for arrays, strings, and so on are distinguished in the same way by effect. Read operations such as `$values->count()` are allowed, while operations such as `$values->sort()` that modify the receiver are rejected.

## Inheritance, Traits and Closures

The Immutable contract continues to propagate with the code structure:

- A subclass can add `#[Immutable]` when overriding a method, but cannot remove the Immutable method/parameter contract already declared by the parent class or interface;
- Methods injected by a Trait retain `#[Immutable]`;
- Closure, arrow function, and Generator bodies retain the immutable state of captured variables and `$this`;
- the library stub preserves `#[Immutable]` on public APIs, and consuming projects continue to check the same contract at compile time.

## Dynamic Calls Are Explicit Escape Hatches

`#[Immutable]` is a static checking tool, not a security sandbox. When the target is hidden by dynamic syntax, the compiler does not insert a runtime read-only proxy:

```php
function escape(#[Immutable] User $user): void
{
    $method = 'rename';
    $user->$method('changed'); // dynamic call: the compiler does not check the target method's effect
}
```

Variable callables, Reflection, `eval()`, and other ZendVM dynamic code are escape hatches of the same kind. Do not treat `#[Immutable]` as a permission boundary for handling untrusted code; its goal is to discover accidental modification at zero runtime cost in fully static, ordinary TypePHP code.

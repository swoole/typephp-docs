# Getter Compile-Time Attribute

`Getter` is used to automatically generate a public read method for an object property, suitable for properties that should only be read externally without directly exposing write access.

Getter methods are **statically generated at compile time**, then complete AOT compilation together with ordinary hand-written methods. At runtime no Attributes are parsed, no reflection is used, and no dynamic method calls occur, so the annotation mechanism itself has zero runtime overhead.

See [Compile-Time Attributes](compile-time-attributes.md) for the namespace rules common to all built-in Attributes.

```php
class User
{
    #[Getter]
    private string $name = 'Alice';
}

$user = new User();
echo $user->getName(); // Alice
```

The compiler generates an equivalent method:

```php
public function getName(): string
{
    return $this->name;
}
```

`Getter` is a pure compile-time Attribute and is not written into runtime Attribute metadata.

## Property Visibility

`Getter` supports `private`, `protected`, and `public` instance properties. Regardless of the property's own visibility, the generated method is always `public`:

```php
class Profile
{
    #[Getter]
    private string $name = '';

    #[Getter]
    protected int $age = 0;

    #[Getter]
    public bool $active = true;
}
```

The properties above generate, respectively:

```php
public function getName(): string;
public function getAge(): int;
public function getActive(): bool;
```

Getter has no setter behavior. Whether external code can modify the property directly is still determined by the property's original visibility.

`Getter` can be used on `readonly` properties and on properties in readonly classes, because the generated method only reads the property and does not modify it. Properties with Property Hooks cannot use `Getter`; Property Hooks already define independent read semantics, and the compiler reports an error at compile time to avoid conflicting read interfaces.

The same property can use `Getter`, [`Setter`](setter.md), and [`With`](with.md) at the same time:

```php
#[Getter, Setter, With]
private string $name = '';
```

## Method Name and Return Type

The method name is formed by `get` plus the property name with its first letter capitalized:

| Property | Generated method |
|---|---|
| `$name` | `getName()` |
| `$userId` | `getUserId()` |
| `$_value` | `get_value()` |

When the property declares a type, the generated method uses the same return type. Properties without a declared type generate methods without a return type declaration, and the return value is handled according to TypePHP's dynamic type rules.

If a method with the same name already exists in the class, the compiler reports an error according to the ordinary duplicate-method rules and does not override the user-written method.

## Constructor Property Promotion

Constructor-promoted properties also support `Getter`:

```php
class User
{
    public function __construct(
        #[Getter]
        private int $id,
        #[Getter]
        protected string $name,
    ) {
    }
}
```

This generates public `getId()` and `getName()`.

When a single declaration contains multiple properties, a Getter is generated for each property:

```php
class Point
{
    #[Getter]
    private int $x = 0, $y = 0;
}
```

The example above generates `getX()` and `getY()`.

## Namespaces

`Getter` lives in the root namespace and follows PHP's standard name resolution rules. In other namespaces you can use the fully qualified name:

```php
namespace App\Model;

class User
{
    #[\Getter]
    private string $name = '';
}
```

You can also import it first and then use the short name or an alias:

```php
namespace App\Model;

use \Getter;
use \Getter as Readable;

class User
{
    #[Getter]
    private string $name = '';

    #[Readable]
    private int $age = 0;
}
```

Without an import, `#[Getter]` inside a namespace resolves to, for example, `App\Model\Getter`, and does not trigger TypePHP's Getter feature.

## Usage Restrictions

`Getter` can only be used on instance properties and accepts no arguments. The following usages produce compile errors:

```php
class InvalidExample
{
    #[Getter] // Error: a static property is not an object instance property
    public static int $count = 0;
}

#[Getter] // Error: cannot be used on a function
function getValue(): int
{
    return 1;
}
```

It also cannot be used on classes, ordinary methods, class constants, or function parameters. Constructor-promoted properties are object properties, so they are an allowed exception.

## TypePHP Library

When compiling with `-m lib`, Getter methods are public methods of the class:

- The `php_*` implementations of the Getter are generated and exported in the library;
- The auto-generated `<target>.stub.php` preserves the `#[Getter]` on the property;
- Consuming projects generate the Getter method declarations from the Attribute;
- The actual Getter method implementation is imported from the dynamic library.

If the entire class uses `#[NoExport]`, neither the class nor its Getters enter the published stub or are exported as part of the library ABI.

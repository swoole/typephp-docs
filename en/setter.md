# Setter Compile-Time Attribute

`Setter` is used to generate a public write method for an instance property. The property itself can be `private`, `protected`, or `public`, and the generated method is always `public`.

Setter methods are **statically generated at compile time** and participate in AOT compilation as ordinary methods. At runtime no Attributes are scanned, no reflection is used, and no dynamic calls are performed, so there is no additional runtime overhead compared to an equivalent hand-written Setter.

```php
class User
{
    #[Setter]
    private string $name = '';
}

$user = new User();
$user->setName('Zhang San');
```

The compiler generates an equivalent method:

```php
public function setName(string $name): void
{
    $this->name = $name;
}
```

The method parameter inherits the property type. When the property declares no type, the parameter declares no type either. Constructor-promoted properties and multiple properties in a single declaration are also supported.

`Setter` can be used together with `Getter` and `With`:

```php
#[Getter, Setter, With]
private int $age = 0;
```

`Setter` can only be used on ordinary, writable, non-static instance properties and accepts no arguments. The following properties cannot use `Setter`, and the compiler reports an error directly:

- properties explicitly declared as `readonly`;
- properties that are implicitly readonly in a readonly class;
- properties with `get` or `set` Property Hooks.

In namespaces, use `#[\Setter]`, or import it first via `use \Setter;`.

When using `-m lib`, the published stub preserves `#[Setter]`, and consuming projects obtain the method declaration from it while the method implementation is provided by the dynamic library.

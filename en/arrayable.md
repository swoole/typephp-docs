# Arrayable Compile-Time Attribute

`Arrayable` is used to automatically generate a public `toArray(): array` method for a class. By default it returns all non-static `public` properties in the current class and all base classes, using the property names as keys of the associative array.

```php
#[Arrayable]
class User
{
    public int $id = 1;
    public string $name = 'Zhang San';
    private string $password = '';
}

$data = (new User())->toArray();
// ['id' => 1, 'name' => 'Zhang San']
```

The generated `toArray()` is equivalent to an ordinary hand-written method:

```php
public function toArray(): array
{
    return [
        'id' => $this->id,
        'name' => $this->name,
    ];
}
```

The method is generated at **compile time** and participates directly in type analysis and AOT compilation. At runtime it does not scan Attributes, use reflection, or dynamically look up fields, so the annotation mechanism itself has zero runtime overhead. At runtime it only performs ordinary property reads and array construction.

TypePHP's `$object->toArray()` is a keyword method. The underlying array conversion helper, after detecting that the object has a zero-argument `toArray()`, calls the method generated here and verifies that the return value is actually an array. Therefore `Arrayable` is fully compatible with TypePHP's existing object-to-array conversion semantics.

## Selecting Fields

The optional `fields` parameter lets you select fields and control the order of the result:

```php
#[Arrayable(fields: ['name', 'id'])]
class User
{
    private int $id = 1;
    protected string $name = 'Zhang San';
    public string $email = 'user@example.com';
}

// ['name' => 'Zhang San', 'id' => 1]
```

The positional argument form is equivalent:

```php
#[Arrayable(['name', 'id'])]
class User {}
```

When `fields` is explicitly specified, you can select `public`, `protected`, or `private` instance properties declared in the current class, as well as `public` or `protected` instance properties in a base class that are accessible to the current subclass. Dynamic or unknown properties, static properties, and base class `private` properties cannot be selected; duplicate fields are also reported at compile time.

Property values can be of any type. `Arrayable` does not perform stringification or any other value conversion; it only reads the current value of each property and writes it directly into the result array.

- Omitting the argument entirely: select all public instance properties.
- Passing an empty array `[]`: generate a `toArray()` that always returns an empty array.
- Properties are written into the result in the order specified by `fields`.

`toArray()` only preserves the current value of each property and does not recursively call the property objects' `toArray()`. Public Property Hooks execute their `get` Hook just like ordinary property reads.

## Existing Methods and Inheritance

`Arrayable` only generates an ordinary `public function toArray(): array` method at compile time; it does not skip generation just because the current class or a base class already has a method with the same name.

After generation, the method follows exactly the same PHP class method rules as hand-written methods: a same-named method in the current class is reported as a duplicate definition; overriding a base class method checks `final`, visibility, and method signature compatibility.

`Arrayable` can only be used on named classes. In namespaces, use `#[\Arrayable]`, or import it first via `use \Arrayable;`.

When using `-m lib`, the published stub preserves `#[Arrayable]` and its field list. Consuming projects obtain the same `toArray()` declaration, while the actual method body is provided by the dynamic library.

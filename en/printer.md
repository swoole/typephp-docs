# Printer Compile-Time Attribute

`Printer` is used to automatically generate PHP's standard magic method `public function __toString(): string` for a class. By default it outputs the short name of the class together with all non-static `public` properties in the current class and its base classes.

`__toString()` is **statically generated at compile time** and participates in AOT compilation as an ordinary method. At runtime it does not iterate over Attributes, use reflection, or create methods dynamically, so the annotation mechanism itself has zero runtime overhead; at runtime it only performs the property reads and string concatenation that `__toString()` itself requires.

It is compatible with both PHP and TypePHP calling conventions: contexts such as PHP's `echo $object` and string interpolation invoke `__toString()`; TypePHP's keyword method `$object->toString()` performs object-to-string conversion under the hood and ultimately calls this `__toString()`. The compiler does not generate a second `toString()` wrapper method.

```php
#[Printer]
class User
{
    public int $id = 1;
    public string $name = 'Zhang San';
    private string $password = '';
}

echo new User();
// User(id=1, name=Zhang San)

echo (new User())->toString();
// User(id=1, name=Zhang San)
```

`private`, `protected`, and static properties do not enter the output. Properties are ordered from base classes to subclasses, and by declaration order within the class; when there are no public properties, the output looks like `User()`.

You can use the optional `fields` parameter to select and order the fields to output:

```php
#[Printer(fields: ['name', 'id'])]
class User
{
    private int $id = 1;
    protected string $name = 'Zhang San';
    public string $email = 'user@example.com';
}

echo new User();
// User(name=Zhang San, id=1)
```

Positional arguments can also be used:

```php
#[Printer(['id', 'name'])]
class User {}
```

When `fields` is explicitly specified, you can select `public`, `protected`, or `private` instance properties declared in the current class, as well as `public` or `protected` instance properties in a base class that are accessible to the current subclass. Dynamic or unknown properties, static properties, and base class `private` properties cannot be selected; duplicate fields are also reported at compile time.

Property values can be of any type. `string` properties participate in concatenation directly; all non-string types such as int, float, bool, array, object, and mixed are first converted via TypePHP's `toString()` and then concatenated. Passing an empty array `[]` produces a result like `User()`; omitting the argument entirely means selecting all public properties.

`Printer` only generates an ordinary `public function __toString(): string` method at compile time; it does not skip generation just because the current class or a base class already has a method with the same name.

After generation, the method follows exactly the same PHP class method rules as hand-written methods: a same-named method in the current class is reported as a duplicate definition; overriding a base class method checks `final`, visibility, and method signature compatibility.

`Printer` can only be used on named classes. In namespaces, use `#[\Printer]`, or import it first via `use \Printer;`.

When using `-m lib`, the published stub preserves `#[Printer]`, and consuming projects obtain the same method declaration while the actual implementation is provided by the dynamic library.

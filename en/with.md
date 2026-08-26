# With Compile-Time Attribute

`With` generates a method that does not modify the original object: it clones the current object, sets the property on the clone, and returns the clone.

With methods are **statically generated at compile time** and participate in AOT compilation as ordinary methods, without depending on runtime Attributes, reflection, or dynamic method calls. The annotation generation mechanism itself has zero runtime overhead; object cloning and property assignment are the operations this method explicitly requires.

```php
class User
{
    #[Getter, With]
    private string $name = '';
}

$original = new User();
$copy = $original->withName('Zhang San');
```

The compiler generates an equivalent method:

```php
public function withName(string $name): static
{
    $clone = clone $this;
    $clone->name = $name;
    return $clone;
}
```

The return type is `static`, so when called within an inheritance hierarchy it still returns an object of the current runtime class. Object cloning follows PHP's ordinary `clone`/`__clone()` semantics.

`With` supports `private`, `protected`, and `public` instance properties and constructor-promoted properties, and can also be used together with `Getter` and `Setter`. It does not support static properties and accepts no arguments.

`With` needs to rewrite the property on the clone object, so it cannot be used on properties explicitly declared `readonly` or on implicitly readonly properties in a readonly class. Properties with `get` or `set` Property Hooks also cannot use `With`, to avoid conflicts between the generated method and the Hook's read/write semantics; all these cases are reported at compile time.

In namespaces, use `#[\With]`, or import it first via `use \With;`.

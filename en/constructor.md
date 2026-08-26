# Constructor Compile-time Attribute

`Constructor` is placed on instance properties and generates a public constructor based on the order in which the properties are declared.

```php
class User
{
    #[Constructor]
    private int $id;

    #[Constructor]
    private string $name = 'typephp';
}
```

The compiler generates an equivalent method at compile time:

```php
public function __construct(int $id, string $name = 'typephp')
{
    $this->id = $id;
    $this->name = $name;
}
```

The property types and default values are copied to the constructor parameters. A required property without a default value cannot come after a property with a default value. `private`, `protected`, `public`, and readonly instance properties are all supported; static properties are not supported.

`Constructor` can be combined with property generation annotations:

```php
#[Constructor, Getter, With]
private int $id;
```

If the class already declares `__construct()`, the compiler reports an error; it will not merge with or override the user constructor. Trait and Enum properties also cannot use this annotation.

## Parent Constructor

The generated constructor checks the parent class constructor according to PHP's inheritance rules:

- When the parent class and its inheritance chain have no constructor, the current class constructor is generated directly;
- When the nearest inheritable parent constructor has no required parameters, the compiler automatically inserts `parent::__construct()` before all property assignments;
- When the parent constructor has required parameters, the compiler reports an error. In this case you need to remove `Constructor`, declare `__construct()` yourself, and pass the parent parameters explicitly;
- A private parent constructor is not called, and the generated current class constructor is still allowed to be used;
- A final parent constructor cannot be overridden, and the compiler reports an error according to ordinary PHP method rules.

`Constructor` provides no parent constructor parameter mapping, avoiding introducing implicit, hard-to-maintain parameter passing rules into property annotations.

The constructor is generated statically at compile time and participates in AOT compilation as an ordinary method, requiring no runtime Attribute, reflection, or dynamic invocation. When using `-m lib`, the published stub preserves the property annotations, and consuming projects get the same constructor declaration.

Inside a namespace use `#[\Constructor]`, or import it first via `use \Constructor;`.

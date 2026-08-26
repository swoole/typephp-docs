# Constant Expressions

PHP requires constant expressions for Attribute arguments, constant values, and various default values. TypePHP checks these expressions at compile time against the target PHP version, and invalid expressions produce a compile error directly.

"Constant expressions" here do not mean "literals only". They can contain arrays, constant references, and some operations; in certain positions they can also contain `new` expressions.

## Common Forms

The following forms can be used in constant expressions:

```php
const DEFAULT_LIMIT = 100;

class Config
{
    public const int LIMIT = DEFAULT_LIMIT * 2;

    public array $options = [
        'enabled' => true,
        'limit' => self::LIMIT,
    ];
}
```

Commonly usable expressions include:

- `null`, `true`, `false`, integer, float, and string literals;
- plain constants and class constants, such as `DEFAULT_LIMIT`, `Config::LIMIT`;
- `ClassName::class` and PHP magic constants;
- arrays and array unpacking;
- arithmetic, comparison, bitwise, logical, string concatenation, ternary expressions, and `??`;
- indexing into a constant array;
- enum cases allowed by PHP and reading their properties.

Plain function or method calls are not constant expressions:

```php
class Config
{
    // Compile error: a plain function call cannot be used in a class constant
    public const int LIMIT = loadLimit();

    // Compile error: a plain function call cannot be used in a property default value
    public int $timeout = loadTimeout();
}
```

If a property value must be computed by a function, do the assignment in the constructor or in a regular method.

## Where `new` Is Allowed

PHP divides constant expressions into plain constant expressions and constant expressions that allow dynamic values. TypePHP uses the same rules:

| Position | Is `new` allowed |
|---|---:|
| Attribute arguments | Allowed |
| Parameter default values of functions or methods | Allowed |
| Global `const` | Allowed |
| Static local variable default values | Allowed |
| Class, interface, or Trait constants | Not allowed |
| Object property default values | Not allowed |
| Enum case values | Not allowed |

Constructor-promoted properties are an easily confused exception. Their default value belongs to the constructor parameter default value, not to a property default value, so `new` is allowed:

```php
class Service
{
    public function __construct(
        public Client $client = new Client(),
    ) {}
}
```

For example, Attribute and parameter default values can create objects directly:

```php
#[Attribute(Attribute::TARGET_METHOD)]
class Route
{
    public function __construct(
        public string $path,
        public ?RouteOptions $options = null,
    ) {}
}

#[Route('/users', new RouteOptions(cache: true))]
function listUsers(Client $client = new Client()): array
{
    return [];
}
```

But the same expression cannot be used as a class constant or a property default value:

```php
class Service
{
    public const CLIENT = new Client(); // Compile error

    public Client $client = new Client(); // Compile error
}
```

When a property needs an object default value, use a constructor:

```php
class Service
{
    public Client $client;

    public function __construct()
    {
        $this->client = new Client();
    }
}
```

## Attribute Arguments

Plain runtime Attributes support non-empty arrays, nested arrays, constants, and `new` expressions:

```php
#[Rule([
    'groups' => ['create', 'update'],
    'normalizer' => new NameNormalizer(trim: true),
])]
class User
{
}
```

These values can be read normally through the PHP Reflection API:

```php
$attribute = (new ReflectionClass(User::class))->getAttributes(Rule::class)[0];
$arguments = $attribute->getArguments();
$instance = $attribute->newInstance();
```

Attribute arguments must still satisfy PHP's constant expression rules. The following plain function call is not allowed:

```php
#[Rule(loadRules())] // Compile error
class User
{
}
```

Anonymous classes, dynamic class names, `static` class names, and argument unpacking cannot be used in `new`:

```php
#[Rule(new class {})]       // Anonymous classes are not allowed
#[Rule(new $className())]   // Dynamic class names are not allowed
#[Rule(new static())]       // new static is not allowed
#[Rule(new Rule(...$args))] // Argument unpacking is not allowed
class User
{
}
```

## PHP 8.5 Expressions

When PHP 8.5 is selected as the target version, constant expressions also support:

- scalar type casting;
- `static` closures;
- first-class callables, such as `strlen(...)`, `Factory::create(...)`.

```php
#[Callback(strlen(...))]
#[Fallback(static function (string $value): string {
    return $value;
})]
class Handler
{
}
```

Closures in constant expressions must be `static`, and they cannot capture variables via `use (...)`. First-class callables differ from ordinary calls:

```php
strlen(...)       // PHP 8.5 first-class callable, usable
strlen('value')   // Plain function call, cannot be used in a constant expression
```

The object cast `(object) [...]` is only allowed in positions that support dynamic values, such as Attribute arguments; it cannot be used in class constants or property default values.

PHP 8.5 allows static closures and first-class callables as class constant expressions, but TypePHP's current class constant storage does not yet support these two kinds of object values. For now, use them in request-time scenarios such as Attribute arguments, not in class constants.

## Short-Circuit and Conditional Expressions

Like PHP, TypePHP performs constant folding first. Branches that are determined not to execute do not participate in constant expression checks:

```php
#[Example(true || loadValue())]
#[Example(true ? 1 : loadValue())]
class Target
{
}
```

If the condition cannot be determined at compile time, all possibly-executed branches must be valid constant expressions.

## Target PHP Version

Expression rules follow the target PHP version selected by the project, not the PHP interpreter version running the TypePHP compiler. The static closures and first-class callable constant expressions added in PHP 8.5 produce compile errors under lower target versions.

The project can select the PHP version through compile arguments or project configuration; see [Command-Line Arguments](options.md) and [`project.yml` Configuration](project-yml.md) for details.

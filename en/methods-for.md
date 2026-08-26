# MethodsFor Compile-Time Attribute

`MethodsFor` adds methods to objects, PHP base types, or arbitrary expressions without modifying the original type. It is a TypePHP compile-time feature and can only be used in TypePHP static code.

See [Compile-Time Attributes](compile-time-attributes.md) for the namespace rules common to all built-in Attributes.

A Provider is an ordinary class with `#[MethodsFor(...)]`. The `public static` methods in the class become extension methods, and the first parameter of the static method is the receiver:

```php
#[MethodsFor(Type::String)]
final class StringExtensions
{
    public static function surround(
        string $value,
        string $left = '[',
        string $right = ']'
    ): string {
        return $left . $value . $right;
    }
}

function main(): void
{
    $name = 'TypePHP';
    echo $name->surround('<', '>'); // <TypePHP>
}
```

When `$name->surround('<', '>')` is called, the first parameter `$value` is passed automatically by the compiler; the arguments provided at the call site correspond starting from the second parameter of the static method.

## 1. Base Type Extensions

Base types use the compile-time type symbols in the root namespace `Type`:

| Target | Receiver parameter | Purpose |
|---|---|---|
| `Type::Int` | `int` | Integer |
| `Type::Float` | `float` | Floating-point number |
| `Type::Bool` | `bool` | Boolean |
| `Type::String` | `string` | String |
| `Type::Array` | `array` | PHP array |
| `Type::Object` | `object` | A value whose static type is the generic `object` |
| `Type::Any` | `any` | A value whose static type is `any` |
| `Type::Stream` | `stream` | Stream resource |
| `Type::Box` | `box` | Box resource |
| `Type::BigInt` | `bigint` | High-precision integer |
| `Type::BigFloat` | `bigfloat` | High-precision floating-point number |
| `Type::Decimal` | `decimal` | High-precision decimal number |

The root-namespace `Type` here is a type-symbol class provided to user code, not the compiler-internal `TypePHP\Type`.

One Provider declares only one target, but the same target can be extended by multiple Provider classes:

```php
#[MethodsFor(Type::Int)]
final class IntFormatting
{
    public static function toBytes(int $value): string
    {
        return pack('J', $value);
    }
}

#[MethodsFor(Type::Int)]
final class IntPredicates
{
    public static function isEven(int $value): bool
    {
        return $value % 2 === 0;
    }
}

function main(): void
{
    $size = 42;
    var_dump($size->isEven());
    var_dump($size->toBytes());
}
```

Two methods with the same name for the same target that differ only in case cannot both be registered; otherwise the compiler reports a duplicate extension method.

### Type::Any Is Not a Wildcard

`Type::Any` only matches receivers whose static type is already `any`; it does not match every concrete type:

```php
#[MethodsFor(Type::Any)]
final class AnyExtensions
{
    public static function debugType(any $value): string
    {
        return get_debug_type($value);
    }
}

function inspect(any $value): void
{
    echo $value->debugType(); // The static type of $value is any
}
```

If you need to provide a method for all receivers, use the keyword extension.

## 2. Keyword Extensions

The string target `'*'` means the method can act on any receiver. The first parameter must be declared as `any`:

```php
#[MethodsFor('*')]
final class DebugExtensions
{
    public static function inspectValue(any $value, string $label = 'value'): any
    {
        echo $label, ': ';
        var_dump($value);
        return $value;
    }
}

function main(): void
{
    $number = 42;
    $text = 'hello';
    $items = [1, 2, 3];

    $number->inspectValue('number');
    $text->inspectValue('text');
    $items->inspectValue('items');
}
```

Keyword extensions cannot override the compiler's built-in keyword methods such as `toInt()`, `toString()`, `toArray()`, `toAny()`, and `toRef()`.

The difference between `Type::Any` and `'*'` is:

- `Type::Any` only matches values whose static type is `any`.
- `'*'` can match any static type.

When the receiver is `any`, the lookup order is built-in keyword methods, keyword extensions, then `Type::Any` extensions.

Keyword extension method names are globally reserved. `MethodsFor('*')` and any `MethodsFor(Type::*)` or `MethodsFor(ClassName::class)` Provider cannot define methods with the same name; otherwise the compiler reports a conflict, and it does not silently override based on file scan order.

## 3. Object Extensions

Object extensions use the target class's `ClassName::class`:

```php
namespace App\Model;

final class User
{
    public function __construct(public string $name) {}
}
```

The Provider is not required to be in the same namespace as the target class. For example, extensions can be organized in another namespace:

```php
namespace App\Extension;

use App\Model\User;

#[\MethodsFor(User::class)]
final class UserExtensions
{
    public static function displayName(User $user): string
    {
        return strtoupper($user->name);
    }

    public static function rename(User $user, string $name): User
    {
        $user->name = trim($name);
        return $user;
    }
}
```

They are still called like ordinary object methods:

```php
namespace App;

use App\Model\User;

function main(): void
{
    $user = new User('Alice');

    echo $user->displayName();
    echo $user->rename(' Bob ')->displayName();
}
```

An object Provider's target must be a class, not an interface:

```php
#[MethodsFor(SomeInterface::class)] // Compile error
final class InvalidExtensions {}
```

Object extensions are looked up by the static type of the receiver, not the actual runtime type. The order is:

1. The class corresponding to the static type.
2. Parent classes, from nearest to farthest.
3. If the static stage can determine that the receiver is definitely an object, look up `MethodsFor(Type::Object)`.

For example, `Child extends Base` can use methods from `MethodsFor(Base::class)`; if both Child and Base provide an extension with the same name, Child's Provider takes precedence. When a variable is statically declared as Base, it is only looked up starting from Base, even if a Child is stored in it at runtime.

`Type::Object` is the general fallback for object lookup, but it is not used for expressions whose static type is `any`, for nullable values where null has not yet been excluded, or for other expressions that cannot be determined to be objects. Interfaces cannot be Provider targets, but a receiver of an interface type is definitely an object, so `MethodsFor(Type::Object)` can be used when neither the interface's real methods nor the class extensions match.

`MethodsFor` lives in the root namespace and fully follows PHP's name resolution rules. In a namespaced file, use the fully qualified name:

```php
namespace App\Extension;

#[\MethodsFor(\Type::String)]
final class StringExtensions {}
```

You can also import `MethodsFor` and `Type` from the root namespace first:

```php
namespace App\Extension;

use \MethodsFor;
use \Type;

#[MethodsFor(Type::String)]
final class StringExtensions {}
```

`use ... as ...` aliases are supported as well:

```php
namespace App\Extension;

use \MethodsFor as Provider;
use \Type as TargetType;

#[Provider(TargetType::String)]
final class StringExtensions {}
```

Without the corresponding `use`, `#[MethodsFor(Type::String)]` inside a namespace resolves to `App\Extension\MethodsFor` and `App\Extension\Type` according to PHP rules, and is not treated as a root-namespace TypePHP compile-time symbol. Only an Attribute that finally resolves to the root-namespace `MethodsFor` registers extension methods.

Object targets use the same set of rules. `User::class` can be imported via `use App\Model\User;`, or written as the full class name `\App\Model\User::class`; `use ... as ...` aliases work as well.

### Object Method Priority

Object methods are looked up in the following order:

1. Real instance methods that exist in the class;
2. Object extension methods;
3. `__call()`;
4. Undefined method error.

Extension methods cannot override instance methods that already exist in the class, but take precedence over `__call()` when no real method exists.

## 4. Provider Method Rules

Registerable methods must satisfy the following rules:

- They must be declared as `public static`.
- They must have at least one parameter.
- The first parameter is the receiver, and its type must match the Provider target.
- The receiver cannot be passed by reference.
- Call arguments correspond starting from the second parameter.
- Default parameters and variadic parameters follow ordinary PHP method rules.
- The return type is used for static type inference of subsequent chained calls.
- Method names are registered exactly as declared, with no automatic snake_case, PascalCase, or camelCase conversion.
- Method name lookup is case-insensitive, consistent with ordinary PHP methods.
- `private` and `protected` methods are not registered and can serve as internal helpers of the Provider.
- Methods whose names start with a double underscore are not registered.
- `public` non-static methods produce a compile error instead of being silently ignored.

The following two names are different extensions and are not converted into each other:

```php
public static function displayName(User $user): string;
public static function display_name(User $user): string;
```

Callers must use the corresponding name:

```php
$user->displayName();  // Matches displayName
$user->display_name(); // Matches display_name
$user->DISPLAYNAME();  // Different case, still matches displayName
```

## 5. Chained Calls

Provider methods should declare accurate return types so the compiler can continue resolving chained methods:

```php
#[MethodsFor(Type::String)]
final class TextExtensions
{
    public static function quoted(string $value): string
    {
        return '"' . trim($value) . '"';
    }
}

function main(): void
{
    $user = new \App\Model\User(' Alice ');

    echo $user
        ->displayName() // object extension, returns string
        ->quoted()      // string extension, still returns string
        ->upper();      // TypePHP built-in generic String method
}
```

## 6. Scope of Use

MethodsFor only participates in TypePHP static method call analysis. The following calls support extension lookup:

```php
$user->displayName();
(new User('Alice'))->displayName();
$repository->findUser()->displayName();
```

The following calls do not use extension methods:

```php
$method = 'displayName';
$user->$method();

User::displayName();
call_user_func([$user, 'displayName']);
eval('$user->displayName();');
```

Limitations:

- Object extensions do not apply to static method calls.
- Dynamic method names and dynamic callbacks do not participate in extension lookup.
- ZendVM dynamic scripts, ordinary PHP scripts, and `eval()` cannot call TypePHP extension methods.
- Both the Provider class and the target class must be visible in the current static compilation.

The `MethodsFor` Attribute is read and removed during the compile stage and is not written into the runtime Attribute metadata of the generated program. Extension calls are compiled directly into calls to the corresponding Provider static methods, requiring no runtime reflection and no runtime extension method table lookup.

See [Extending Object Methods](object-extension-method.md) for more object extension examples, [Generic Methods](universal_method.md) for built-in generic methods, and [Keyword Methods](keyword-method.md) for built-in keyword methods.

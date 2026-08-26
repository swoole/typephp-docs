# Extended Object Methods

Extended object methods allow adding TypePHP static method calls without modifying the original class. The extension is declared via the class-level `#[MethodsFor(ClassName::class)]` attribute.

```php
<?php

namespace App;

class User
{
    public function __construct(public string $name) {}
}

#[\MethodsFor(User::class)]
final class UserExtensions
{
    public static function displayName(User $user): string
    {
        return strtoupper($user->name);
    }

    public static function greeting(
        User $user,
        string $prefix = 'Hello, '
    ): string {
        return $prefix . $user->name;
    }
}

function main(): void
{
    $user = new User('Alice');
    echo $user->displayName();       // ALICE
    echo $user->greeting('Welcome, '); // Welcome, Alice
}
```

## Definition Rules

- `MethodsFor` lives in the root namespace; within a namespace you can use `#[\MethodsFor(...)]`, or first `use \MethodsFor;` and then use `#[MethodsFor(...)]`.
- The attribute argument is the target object's `ClassName::class`.
- The target must be a class, not an interface.
- Extension methods must be `public static`.
- The first parameter must be the target object type and must not be passed by reference.
- Call arguments correspond to the static method's second and subsequent parameters.
- private/protected methods are not registered and can serve as internal helpers.
- The method name is used directly as the extension method name, without any naming style conversion.

The attribute name and the `ClassName::class` argument both follow PHP's namespace resolution rules and support aliases:

```php
namespace App\Extension;

use \MethodsFor as Provider;
use App\Model\User as ModelUser;

#[Provider(ModelUser::class)]
final class UserExtensions
{
    public static function displayName(ModelUser $user): string
    {
        return strtoupper($user->name);
    }
}
```

When not imported, `#[MethodsFor(...)]` inside a namespace points to the attribute of the same name in the current namespace and does not trigger TypePHP's extension method feature.

```php
#[\MethodsFor(User::class)]
final class UserExtensions
{
    public static function rename(User $user, string $name): User
    {
        $user->name = $name;
        return $user;
    }

    private static function normalize(string $name): string
    {
        return trim($name);
    }
}
```

## Return Types and Chained Calls

Return types participate in subsequent type inference:

```php
echo $user
    ->rename('Bob')
    ->displayName()
    ->trim()
    ->upper();
```

## Lookup Priority

Object calls are handled in the following order:

1. Built-in keyword methods and `MethodsFor('*')` keyword extensions
2. Instance methods that actually exist in the class or interface
3. Object extensions of the class corresponding to the static type
4. Parent class object extensions, from nearest to farthest
5. `MethodsFor(Type::Object)`, only when the static stage determines the receiver is definitely an object
6. `__call()`
7. Undefined method error

Extensions cannot override real instance methods. If there is no real method but a valid extension exists, the extension takes priority over `__call()`.
Lookup is based only on the static type: a variable statically declared as a parent class will not look up the runtime subclass's Provider. When a subclass and a parent class both provide an extension of the same name, the Provider closest to the static type takes priority.

## Usage Limitations

Supported:

```php
$user->displayName();
(new User('Alice'))->displayName();
$service->findUser()->displayName();
```

Not participating in extension lookup:

```php
$method = 'displayName';
$user->$method();

User::displayName();
call_user_func([$user, 'displayName']);
```

Extended object methods only take effect in TypePHP static code and do not dynamically add instance methods to ZendVM. For the complete description of universal types, keywords, and object Providers, see [Extension Method Providers](methods-for.md).

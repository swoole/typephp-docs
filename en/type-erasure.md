# Type Erasure and Recovery

TypePHP uses fixed storage when an expression has a definite type, and `php::Var` when a value must remain dynamic. Type erasure is the transition from a definite type to `php::Var`; type recovery explicitly converts or checks a dynamic value before returning to fixed storage.

```text
fixed type  -- std::any() / toAny() -->  php::Var
fixed type  <-- conversion or assertion -- php::Var
```

Recovery does not remember a value's former type. The program must name the type it now expects, and TypePHP performs the corresponding conversion or runtime check.

## Explicit Type Erasure

Use `std::any($value)` or the equivalent `toAny()` keyword method when one value must enter PHP's dynamic storage model:

```php
$count = 1;                    // fixed int
$value = std::any($count);     // php::Var containing int(1)
$same = $count->toAny();       // equivalent expression form
```

`std::any()` may be called without an argument to create dynamic storage initialized to `null`:

```php
$result = std::any();
$result = load_value();
```

After erasure, the value can change PHP type and uses Zend dynamic operators and dispatch. The compiler can no longer apply fixed-scalar or typed-object Native Call rules until the value is recovered.

Native Class objects and Native-element Std containers cannot be erased into `php::Var`, because their pointers must remain inside the compiler-managed native object graph.

## Storage That Is Naturally Dynamic

Some storage locations are designed to cross dynamic PHP boundaries and therefore expose `php::Var` values.

### Global and Static Dynamic Slots

Values obtained through `global` or `$GLOBALS[...]` are dynamic slots unless the compiler has a separately established fixed native slot. A static local or static property declared as `mixed`, left untyped, or initialized with `std::any()` is also dynamic and retains its value between accesses.

```php
class Registry
{
    public static mixed $service = null;
}

function register(Service $service): void
{
    Registry::$service = std::any($service);
}

function service(): Service
{
    return std::object(Registry::$service, Service::class);
}
```

A static scalar with a definite initializer and a typed static property may retain fixed or declared-type storage. Do not assume that the `static` keyword alone always erases a value; the declaration and initializer determine the slot.

### Ordinary PHP Array Elements

An ordinary PHP array is held by `php::Array`, while each element value uses a dynamic `php::Var` slot; keys follow PHP's integer/string key rules. Reading an element therefore commonly loses the concrete object class or fixed scalar context:

```php
$users = ['owner' => new User()];
$owner = std::object($users['owner'], User::class);
```

Use [std::list / std::dict and their type annotations](typed-arrays.md) when a PHP array needs compiler-checked key/value rules, or a [Std container](std-containers.md) when its element type and native layout must remain fixed.

Other common dynamic boundaries include untyped/`mixed` parameters and returns, dynamic function or method calls, extension APIs returning general PHP values, and untyped object properties.

## Recovering a Fixed Type

Choose the recovery operation according to the required destination type:

| Destination | Recovery form | Behavior |
|---|---|---|
| `int` | `(int) $value`, `intval($value)`, `$value->toInt()`, `std::int($value)` | Converts to fixed native int |
| `float` | `(float) $value`, `floatval($value)`, `$value->toFloat()`, `std::float($value)` | Converts to fixed native float |
| `bool` | `(bool) $value`, `boolval($value)`, `$value->toBool()`, `std::bool($value)` | Converts to fixed native bool |
| `string` | `(string) $value`, `strval($value)`, `$value->toString()` | Converts to `php::Str` |
| `array` | `(array) $value`, `$value->toArray()` | Converts to `php::Array` |
| Concrete object | `std::object($value, User::class)` or `$value->toObject(User::class)` | Checks `instanceof User` and restores class information |
| Generic object | `$value->toObject()` | Converts/checks as `php::Object` without a concrete class |
| Stream | `$value->toStream()` | Recovers a stream resource |
| High-precision number | `toBigInt()`, `toDecimal()`, `toBigFloat()` or matching `std::*` constructor | Constructs the selected fixed numeric type |
| Std container | `toStdArray()`, `toStdVector()`, `toStdMap()`, `toStdOrderedMap()` | Establishes a fixed container type on its first top-level assignment |
| Typed PHP array | `$value->toStdList(T)`, `$value->toStdDict(K, V)` | Assigns an identical contract directly; otherwise performs O(n) key and value checks, after `toArray()` for non-arrays |

Named function and method parameters can use [Type Annotations](std-container-parameters.md) to restore array, vector, map, or ordered-map types automatically, for example `#[StdArray(Type::Int, [2, 3])] $values` or `#[StdVector(Type::Int)] $values`. The PHP parameter type may be omitted or declared as `box`, never `mixed`; Box passing and dynamic-call conversion rules remain unchanged.

```php
function consume(mixed $payload): void
{
    $id = $payload['id']->toInt();
    $name = $payload['name']->toString();
    $user = std::object($payload['user'], User::class);

    $user->update($id, $name);
}
```

Scalar recovery performs conversion according to the selected operation. Concrete object recovery is an assertion plus runtime `is-a` check; an incompatible value raises a type error rather than being converted into an unrelated object. `std::object()` and `toObject(ClassName::class)` have the same TypePHP semantics. Prefer `std::object()` when the source must also run on Zend PHP through a user-provided polyfill.

## Erasure Is Not a Reference Conversion

`std::any()` changes the storage/type model, while `std::ref()` only marks a referenceable call argument. Use [Strongly Typed References](strong-references.md) for reference passing. Code that needs unrestricted PHP reference identity can deliberately store the value as `std::any()` before using the dynamic reference path.

See [Basic Type Conversion](type-convert.md) and [Keyword Methods](keyword-method.md) for the complete conversion API.

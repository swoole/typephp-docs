# ArrayDef

`#[ArrayDef]` adds compile-time key and value type constraints to PHP `array` properties. It suits scenarios where you want to keep using ordinary PHP arrays while having TypePHP check direct element writes.

Instance properties, static properties, and constructor-promoted properties of ordinary Zend Classes can use it; instance `array` properties of [`#[Native]` native classes](native-class.md) can also use it. Native Class itself does not support static properties. `ArrayDef` exists only during the compilation stage; it is not registered as a runtime Attribute and does not change the property's PHP `array` type.

## Basic Syntax

`ArrayDef` accepts one or two positional arguments:

```php
#[ArrayDef(ValueType)]
public array $list = [];

#[ArrayDef(KeyType, ValueType)]
public array $map = [];
```

- One argument declares a List, where the argument represents the element type;
- Two arguments declare a Map, where the first argument represents the key type and the second the value type;
- Map keys only allow `Type::Int` or `Type::String`;
- Arguments must be `Type::*` or `ClassName::class`; named arguments are not supported;
- The annotation can only be used on properties whose declared type is exactly `array`.

The following declarations all produce compile-time errors:

```php
#[ArrayDef]                         // missing type arguments
public array $missing = [];

#[ArrayDef(Type::Int, Type::String, Type::Bool)] // too many arguments
public array $tooMany = [];

#[ArrayDef(Type::Bool, Type::String)] // Map key cannot be bool
public array $invalidKey = [];

#[ArrayDef(Type::Int)]
public mixed $notAnArray = [];      // the property must be explicitly declared as array
```

## List

One argument declares a List, where the argument is the element type:

```php
class Article
{
    #[ArrayDef(Type::String)]
    public array $tags = [];
}

$article = new Article();
$article->tags[] = 'php';
$article->tags[count($article->tags)] = 'aot'; // in a contiguous List, equivalent to append
$article->tags[0] = 'typephp';                 // modify an existing element
```

A List uses contiguous integer indexes and only supports the following writes:

- `$object->property[] = $value`: append an element;
- `$object->property[$index] = $value`: modify an existing element, fill a hole left by a deleted element, or write to PHP's current next-append position.

Explicit index writes uniformly generate a `safeArrayIndex($index, $array)` bounds check. This check obtains the real next append index via Zend's `zend_hash_next_free_element()`, and follows the rule `zend_hash_next_index_insert()` uses for the first index of an empty array, rather than using `count($array)`. This distinction matters after `unset()` creates a hole: PHP does not rewind the automatic append index when deleting an element. Writing equal to that index appends, a smaller non-negative index modifies an element or fills a hole, and exceeding it throws a runtime error. The compiler performs no special recognition of the `count()` AST form.

```php
$index = count($article->tags);
$article->tags[$index] = 'next';     // allowed in a contiguous List: this position is exactly the next append position
$article->tags[$index + 2] = 'gap';  // runtime error: holes are not allowed
$article->tags[-1] = 'invalid';      // runtime error
```

After `unset()`, `count()` can no longer be used to derive the append index:

```php
$article->tags = [0 => 'php', 1 => 'aot', 2 => 'compiler'];
unset($article->tags[2]);

$article->tags[3] = 'next'; // allowed: PHP's next append index is still 3, while count() is 2
```

## Map

Two arguments declare a Map. The first argument is the key type and the second is the value type:

```php
class Dictionary
{
    #[ArrayDef(Type::String, Type::Int)]
    public array $counts = [];
}

$dict = new Dictionary();
$dict->counts['php'] = 1;
```

A Map must provide keys explicitly; the append syntax is not supported:

```php
$dict->counts[] = 1; // compile-time error
```

Map keys follow PHP's own array storage rules. In particular, canonical decimal integer string keys may still be converted to integer keys by PHP; `ArrayDef` checks the type of the write expression and does not change Zend Array's key conversion behavior.

## Supported Value Types

List elements and Map values support the following types:

| Declaration | Written value |
|---|---|
| `Type::Int` | `int` |
| `Type::Float` | `float` |
| `Type::Bool` | `bool` |
| `Type::String` | `string` |
| `Type::Array` | PHP `array` |
| `Type::Object` | any PHP object |
| `Type::Any` | any ordinary PHP value |
| `Type::Stream` | Stream |
| `Type::BigInt` | BigInt |
| `Type::BigFloat` | BigFloat |
| `Type::Decimal` | Decimal |
| `ClassName::class` | an object of the specified PHP class or a subclass |

Std Containers are not `Type::*` type symbols and cannot be written as ArrayDef elements. `Type::Box` is also not a legal ArrayDef type argument. BigInt, BigFloat, and Decimal are declared via their respective explicit type symbols.

## Class Element Types

List elements or Map values can use `ClassName::class`. Subclass objects also satisfy the constraint:

```php
class UserCollection
{
    #[ArrayDef(App\User::class)]
    public array $users = [];

    #[ArrayDef(Type::String, App\User::class)]
    public array $usersByName = [];
}

$collection = new UserCollection();
$collection->users[] = new App\User();
$collection->usersByName['admin'] = new App\Admin(); // Admin extends User
```

When the static class is known, compatibility is checked at compile time. When the value is `any` or a plain `object` whose concrete class cannot be determined, the compiler inserts a runtime check requiring it to be the target class or a subclass.

Native Class objects have no Zend `zval` representation and cannot be stored in PHP arrays, so they cannot be used as the Class element type of `ArrayDef`.

Map keys can still only be `Type::Int` or `Type::String`. The following declaration is invalid:

```php
#[ArrayDef(App\User::class, Type::String)] // Class cannot be a Map key
public array $invalid = [];
```

## Static and Dynamic Checks

When the static types of keys or values are already determined, matching writes directly generate ordinary array writes; a clear mismatch produces a compile-time Fatal Error:

```php
class Names
{
    #[ArrayDef(Type::Int, Type::String)]
    public array $values = [];
}

$names = new Names();
$names->values[1] = 'one';    // passed directly
$names->values['1'] = 'one';  // compile-time error: key must be int
$names->values[1] = 1;        // compile-time error: value must be string
```

When the expression type is `any`, the compiler inserts `toIntExact()`, `toStringExact()`, `toObjectExact()`, or the corresponding high-precision type check. The check is strict and does not perform PHP weak type conversion:

```php
function put(Dictionary $dict, any $key, any $value): void
{
    $dict->counts[$key] = $value;
}

put($dict, 'php', 1);   // runtime check passes
put($dict, 1, 1);       // TypeError: an int will not be converted into a string key
put($dict, 'php', '1'); // TypeError: a string will not be converted into an int value
```

Therefore, appends and Map writes with fully known static types have no extra type checks; dynamic `any` writes bear the runtime checks required by the declared type. Explicit index modifications on a List retain the aforementioned bounds check regardless of whether the value type is statically known.

## Scope and Escapes

`ArrayDef` is a compile-time static checking tool, not a runtime generic array. It currently only checks first-level direct element assignments that the compiler can clearly identify:

```php
$object->values[$key] = $value;
ClassName::$values[$key] = $value;
```

The following paths are not recursively checked for array contents:

- default property arrays and complete arrays passed in at construction;
- whole-property replacement such as `$object->values = $newArray`;
- nested element writes such as `$object->values[0]['nested'] = $value`;
- in-place operations such as `++`, `--`, `+=`, `.=`;
- taking references, reference parameters, and `foreach (... as &$value)`;
- `array_map()`, variable functions, Reflection, `eval()`, or other ZendVM dynamic paths.

These escape paths are not automatically scanned, copied, or wrapped. After writing elements that do not conform to the declaration through them, the behavior relative to the `ArrayDef` contract is undefined. `ArrayDef` also does not restrict array reads.

If the business requires all entry points to have runtime strong-type guarantees, use wrapper methods to control writes, or choose [Std strong-typed containers](std-containers.md), rather than treating `ArrayDef` as a runtime container.

## Namespace

`ArrayDef` and `Type` are both in the root namespace and follow ordinary PHP name resolution rules. In other namespaces the fully qualified names can be used:

```php
namespace App\Model;

class Article
{
    #[\ArrayDef(\Type::String)]
    public array $tags = [];
}
```

They can also be imported explicitly:

```php
namespace App\Model;

use \ArrayDef;
use \Type;

class Article
{
    #[ArrayDef(Type::String)]
    public array $tags = [];
}
```

For the full Attribute name resolution rules, see [Compile-time Attributes](compile-time-attributes.md).

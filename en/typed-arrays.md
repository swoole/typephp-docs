# Typed PHP Arrays: std::list / std::dict

`std::list(T)` and `std::dict(K, V)` use ordinary PHP arrays with key and value contracts enforced during TypePHP's static compilation. They are not Box-wrapped C++ containers. Factory creation, assignment between identical contracts, and native parameter passing do not scan the whole array; explicit conversion from an ordinary array or another value does require runtime validation.

## Declaration and Indexing

```php
$list = std::list(Type::Int);
$list[-5] = 10;
$list[100] = 20;
$list[] = 30; // PHP append semantics: key 101 here

$dict = std::dict(Type::Int, Type::Str);
$dict[-5] = 'hello';
$dict[100] = 'world';
// $dict[] = 'no'; // Compile error: dict requires an explicit key

class MyUser { public int $id = 0; }
$users = std::list(MyUser::class);
$users[] = new MyUser();
```

A list has integer keys and allows negative, sparse, and nonconsecutive indices, without vector-style bounds checks. Integer-key dicts use the same PHP indexing semantics, but have a different operation contract: only lists support `[]` append. The two kinds are not interchangeable as typed parameters.

Dict keys support only `Type::Int` and `Type::Str` (`Type::String` is an alias). Values support `Type::Int`, `Type::Float`, `Type::Bool`, `Type::Str`, `Type::Array`, `Type::Object`, `Type::Any`, and `ClassName::class`. Native objects cannot be stored.

PHP's underlying key rules are preserved: a numeric string such as `'123'` is stored as an integer key. For string-key dicts, TypePHP emits a conversion to `str` when fetching keys in `foreach`, keeping loop keys at their declared type without changing PHPX or PHP array internals. Built-ins such as `array_keys()` retain their ordinary PHP results.

## Converting Existing Values

Use the `toStdList(T)` or `toStdDict(K, V)` keyword method to convert an existing value into a typed PHP array:

```php
$raw = [4, 5];
$list = $raw->toStdList(Type::Int);
$list[] = 6;

$counts = ['alice' => 2];
$dict = $counts->toStdDict(Type::Str, Type::Int);
$dict['bob'] = 3;

$copy = $list->toStdList(Type::Int); // Identical contract: ordinary copy-on-write assignment
```

If the source is already a `StdList` or `StdDict` with exactly the requested contract, conversion is ordinary assignment and does not scan elements. An ordinary array has every key and value checked strictly at runtime; a mismatch throws `TypeError`. Other values first pass through `toArray()`, then receive the same array checks. A non-Native `ClassName::class` value type checks each object against that class or its subclasses. Conversion leaves the source array unchanged.

**Performance risk:** A conversion that requires validation traverses the entire array, taking O(n) time. Non-array sources also run `toArray()` first. Repeated conversion of large arrays, especially inside loops, can noticeably increase runtime. Use it carefully: convert once when entering a typed boundary and reuse the result where possible.

Validation uses the key types actually stored by PHP. Numeric string keys such as `'123'` become integer keys, so an ordinary array containing them fails strict `toStdDict(Type::Str, ...)` validation. A typed dict with the same contract is assigned directly without another check.

## Static Type Constraints

```php
$list = std::list(Type::Int);
// $list['1'] = 10; // Compile error: keys must be int; no implicit conversion
// $list[] = '10';  // Compile error: values must be int

$key = std::any(1);
$list[$key] = 10; // Generated internal strict check: the runtime key must be int

$textKey = std::any('1');
$list[$textKey->toInt()] = 20; // Explicitly request string-to-integer conversion
```

An `any` / `var` can directly supply a key. TypePHP inserts an internal PHPX strict check: lists and integer-key dicts require an actual runtime int; string-key dicts require an actual string. Other runtime types raise `TypeError`, without coercing numeric strings, floats, or booleans. Reads, writes, presence checks, and deletions follow the same rule. Internal Exact APIs are not user keyword methods.

A mismatched non-var key is a compile error. Strongly typed values still require matching static types, except for `Type::Any` values; dynamic values require explicit use of existing keyword methods such as `toInt()` or `toString()`. Ordinary indexed operations and native function entry perform no whole-array checks; explicit `toStdList()` / `toStdDict()` conversion is the exception.

Direct indexed reads and assignments, `isset()`, `empty()`, and element `unset()` are supported. Element references, reference `foreach`, nested array writes, and compound mutations such as `+=`, `??=`, and `++` are currently unsupported. Use ordinary element assignments with explicit types instead. With `varint_types` enabled, integer expressions that may widen to float also require explicit type recovery.

## Copies, References, and foreach

```php
$list = std::list(Type::Int);
$list[] = 1;
$copy = $list;       // Contract propagation; PHP copy-on-write
$alias = &$list;     // A static reference with the same contract
$copy[] = 2;         // Does not modify $list
$alias[] = 3;        // Modifies $list

foreach ($list as $key => $value) {
    // Both $key and $value have concrete int types
    echo $key, ':', $value, "\n";
}
```

Class-valued iteration preserves the declared class type. By-value closure captures also preserve the contract and copy-on-write behavior; by-reference captures are forbidden.

## Function and Method Parameters

The attribute declares the concrete array contract. The PHP parameter type may be omitted or declared as the compatible `array` type; explicit `mixed`, `box`, nullable types, unions, and other types are rejected. Properties follow the same compatibility rule:

```php
function inspect(#[StdList(MyUser::class)] array $users): void
{
    foreach ($users as $user) {
        echo $user->id;
    }
}

function update(#[StdDict(Type::Str, MyUser::class)] array &$users): void
{
    $users['alice'] = new MyUser();
}
```

The caller and parameter must have identical container kinds, key types, and value types. By-value parameters use PHP copy-on-write; `&` parameters can modify the caller's array. Methods use the same syntax. Inside a namespace, import the root-level `StdList` / `StdDict` attributes or use fully qualified names.

Currently, calls must resolve statically to a native TypePHP function or method. Annotated functions and methods have no Zend entry points and cannot be invoked by dynamic PHP, string callbacks, or Zend polymorphic dispatch. Closure parameter annotations, defaults, variadics, and promoted parameters are unsupported. Returning a reference to a typed array is forbidden.

## Property Type Annotations

```php
class State
{
    #[StdList(Type::Int)] public array $values = [];
    #[StdDict(Type::Str, MyUser::class)] public array $users = [];
    #[StdList(Type::Str)] public $names = []; // Omitted type is inferred as array
}

$state = new State();
$state->values[-5] = 10;
$state->values[100] = 20; // Sparse indices and holes are allowed
$state->values[] = 30;
```

Instance and static properties of ordinary classes, and instance array properties of Native classes, support `StdList` / `StdDict` type annotations. Omitted types are inferred as `array`, and explicit list index writes have no bounds checks.

Property attributes currently reuse the existing first-level direct element assignment checks, with the new list/dict key and value rules. They do not change Zend object escape behavior, scan whole property arrays, or automatically promote property reads to closed-contract local typed containers. Whole-property replacement, dynamic PHP object mutation, and reference escapes are not fully protected by these property attributes; encapsulate write entry points. The dynamic PHP restrictions below apply to local containers created by factories, passed through native parameters, and propagated by assignment.

## Dynamic PHP Boundary

```php
$list = std::list(Type::Int);
$list[] = 1;
var_dump(array_search(1, $list, true), array_keys($list)); // Allowed: read-only
// array_push($list, 2); // Compile error: dynamic reference mutation
// sort($list);         // Compile error: dynamic reference mutation
// array_walk($list, $callback); // Compile error: may modify elements
// $callback(std::ref($list));   // Compile error: reference escape
// $callback($list->toRef());    // Also forbidden
```

Read-only built-ins and corresponding TypePHP universal methods are available. Their returned arrays are ordinary arrays and do not automatically inherit the source contract. Dynamic PHP by-value calls receive value snapshots, not writable references to the original container; an unknown callback's actual reference signature can still produce a Zend warning.

Calls to TypePHP-defined functions and methods require matching `StdList` / `StdDict` parameter annotations rather than unannotated `array` / `mixed` parameters.

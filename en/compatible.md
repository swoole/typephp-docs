## Unsupported Syntax

The `TypePHP` compiler supports the vast majority of `PHP` syntax. Because `TypePHP` is a statically compiled language, the following syntax that relies on dynamic handling by `ZendVM` is not supported:

1. `$$` syntax is not supported. Local variables are compiler symbols and cannot be used at runtime.
2. The `extract` function is not supported; local variables cannot be created at runtime.
3. Function calls with a number of arguments inconsistent with the declaration are not supported. The `func_get_args()` function cannot be used to implicitly accept extra arguments; variadic parameters must be declared explicitly.
4. Taking a reference to a `Property Hook` property is not supported.
5. Automatic inference of by-reference arguments in dynamic calls is not supported; use the `refval()` function explicitly to convert a call argument to a reference.
6. By-reference parameters and by-reference returns of closures and arrow functions are not supported.
7. By-reference variadic parameters are not supported, e.g. `function foo(&...$args) {}`.
8. Dynamic code cannot `use` a statically compiled `Trait`. Traits are combined with classes at compile time, so runtime dynamic binding is not supported.
9. `Closure::call()/bind()/bindTo()` is not supported. The `$this` and class scope of a `Closure` are determined at compile time and cannot be rebound at runtime.
10. The `Lazy Object API`, such as `ReflectionClass::newLazyGhost()/newLazyProxy()`, is not supported. `PHP` forbids converting built-in classes into `Lazy Object`.

## Newly Supported Syntax

- `Property Hooks` introduced in `PHP 8.4` is supported.
- The `Pipe Operator` introduced in `PHP 8.5` is supported.

Support for new syntax in `TypePHP` does not rely on `ZendVM`; instead, it is implemented by the compiler as zero-cost abstractions.

## No Top-level Code
The compiler requires that all code be inside a `function`; top-level code is not allowed. Embedded `HTML`, i.e. `PHP` template files, is not supported. This is entirely different from scripting languages such as `PHP` and `JavaScript`, and is consistent with `C++`, `Java`, `Golang`, and `Rust`.

Therefore, template files and configuration files cannot be compiled; they must be loaded dynamically using `include/require` and executed dynamically in `ZendPHP`.

## Source File Encoding
All `.php` files must use `UTF-8` encoding. Other encodings (such as `GBK`, `Shift_JIS`, `ISO-8859-1`) are not allowed.

## Type Immutability
The `TypePHP` compiler requires that a variable's type not be changed. For example, if a variable is declared as an `Object`, it cannot be used as a string or array. This is fundamentally different from `PHP`.

```php
$str = "hello world";
$str = new StringObject("hello");
```

A variable of type `string` cannot be assigned an object.

```bash
Fatal error: Cannot re-assign variable from `php::Object` to `php::Str`
```
This behavior is completely different from `ZendPHP`, where the `$str` variable would be converted from string to object.

> At the underlying design level, `ZendPHP` is `any`-typed; the language does not store variable types.

In addition to strings, object types are also immutable.

```php
$o = new stdClass;
$o = new ArrayObject;
```

The variable `$o` is declared as type `stdClass`, then converted to `ArrayObject` during execution, which is not allowed in the TypePHP compiler. It will throw the following compilation error:

```bash
Fatal error: Cannot re-assign typed object `$o` from `TestObject` to `stdClass`
```

Object properties also follow stricter static typing rules, but the `unset`/`null` semantics differ between different property types.

### Fixed Value Type Properties

These five property types — `int`, `float`, `bool`, `string`, `array` — are optimized by the TypePHP compiler as fixed value types. A property slot always keeps its declared type; it is not allowed to be changed to an uninitialized or null state via `unset()` or by assigning `null`.

```php
class User {
    public int $id = 0;
    public array $roles = [];
}

$user = new User();
unset($user->id);   // TypePHP does not allow relying on PHP's property unset semantics
$user->roles = null; // TypePHP does not allow changing an array property to null
```

In `ZendPHP`, `unset($obj->prop)` can put a property into an uninitialized state; in the TypePHP compiler, this is equivalent to changing the type of a fixed value type property, and is therefore not allowed. If null semantics are required for business reasons, the property should be explicitly declared as a nullable type, e.g. `public ?int $id = null;`.

Fixed value type properties use fixed slots within the heap. When not explicitly initialized, TypePHP uses the zero value of the corresponding type — for example `0` for `int` and `false` for `bool`. Therefore, the behavior of uninitialized typed properties and `??` is not guaranteed to match ZendPHP's uninitialized state; this is TypePHP's fixed-layout semantics.

### Concrete Object Type Properties

Concrete class object properties can be `unset()`, but **cannot be assigned `null`** unless the property is declared as a nullable type (`?Profile`):

```php
class Profile {}

class User {
    public Profile $profile;
}

$user = new User();
$user->profile = new Profile();
$user->profile = null;  // Not allowed, compilation error: Cannot assign null
unset($user->profile);  // Allowed
```

When `null` needs to be allowed, the property should be explicitly declared as a nullable type and given a default value:

```php
class User {
    public ?Profile $profile = null;
}

$user = new User();
$user->profile = null;  // Allowed
```

For non-null object assignment, `AOT` allows type conversions that satisfy the `is-a` relationship — a subclass object can be assigned to a base-class property, consistent with `ZendPHP` behavior.

```php
class Base {}
class Child extends Base {}
class Other {}

class Holder {
    public Base $object;
}

$holder = new Holder();
$holder->object = new Base();  // Allowed
$holder->object = new Child(); // Allowed, Child is-a Base
$holder->object = new Other(); // Not allowed, Other is-not-a Profile
```

However, the reverse assignment is not allowed — a base-class object cannot be assigned to a property declared as a subclass, because a base-class instance does not satisfy the subclass's `is-a` relationship.

```php
class Holder {
    public Child $object;
}

$holder = new Holder();
$holder->object = new Child(); // Allowed
$holder->object = new Base();  // Not allowed, Base is-not-a Child
```

### Parent and Child Classes Cannot Declare a private Property with the Same Name

For `public` and `protected` properties, TypePHP follows PHP's inheritance rules: a same-name declaration in a child class describes the same inherited property slot, and must satisfy the compatibility requirements for type, visibility, and `readonly`.

However, TypePHP does not support a child class hiding a parent class's `private` property with a same-name `private` property. ZendPHP would create two separate property slots for this pattern; to keep property resolution, clone, and typed property semantics clear, the compiler rejects it at compile time.

```php
class ParentBox {
    private int $id = 1;
}

class ChildBox extends ParentBox {
    private string $id = 'child'; // Not allowed: would hide ParentBox::$id
}
```

Please use a different property name instead; if a child class needs to reuse the parent's state, design the parent property as `protected`, or access it through methods provided by the parent class.

## Using References in Dynamic Calls

`TypePHP` cannot determine the argument types of a dynamic call at compile time, so it cannot automatically infer whether an argument is a reference. Native calls or built-in function calls can automatically infer argument types and convert them to references without explicit user specification.
For example, if a `Closure` function's parameter is a reference type, it can only be determined at runtime and cannot be inferred automatically by the `TypePHP` compiler; you need to explicitly use the `refval()` function or the equivalent keyword method `toRef()` to convert it to a reference.

```php
// The function's parameters and return value can only be obtained at runtime
$fn = getClosure();
// The compiler cannot determine whether an argument should use value or reference passing, so value passing is used by default
$fn($a, $b, $c);
// $c will be explicitly passed by reference instead of by value
$fn($a, $b, refval($c));
// Equivalent: toRef() is a TypePHP-specific keyword method
$fn($a, $b, $c->toRef());
```

Closures and arrow functions also cannot currently declare by-reference parameters or return by reference. This restriction is different from `use (&$value)` reference capture; reference capture is already supported.

## Closure Rebinding Is Not Supported

TypePHP supports closures, arrow functions, `use ($value)` value capture, and `use (&$value)` reference capture, but does not support the following APIs that rely on runtime rebinding:

- `Closure::call()`;
- `Closure::bind()`;
- `Closure::bindTo()`.

```php
class Target
{
    private string $value = 'hidden';
}

$target = new Target();
$reader = function (): string {
    return $this->value;
};

$reader->call($target);          // Not supported
$reader->bindTo($target);        // Not supported
Closure::bind($reader, $target); // Not supported
```

TypePHP reports a compilation error directly when it can statically confirm the receiver is a closure. A closure's captured variables, lexical class scope, and `$this` are all determined by the AOT compiler; replacing `$this` or the class scope at runtime would break this static model, so ZendVM's rebinding behavior is not emulated.

When you need to access the target object, pass the object explicitly as a regular argument and operate through its public API:

```php
class Target
{
    public function value(): string
    {
        return 'visible';
    }
}

$reader = static function (Target $target): string {
    return $target->value();
};

echo $reader(new Target());
```

## Reserved Keyword Methods

`toInt()`, `toString()`, `toArray()`, and similar are TypePHP reserved keyword methods, and take priority over normal object method resolution. A zero-argument `toArray()` on an object can be called by conversion helpers; do not define an object method with the same name that requires arguments, because call arguments will not be handled with normal object method semantics. When you need a business serialization method, use a non-reserved name such as `serializeNode()`.

## Array null Keys

In `ZendPHP`, `$array[null] = 1` is equivalent to `$array[''] = 1`; `null` is implicitly converted to the empty string `""` as the key. But in the TypePHP compiler, `$array[null]` is treated as the `$array[]` append operation, which is incompatible with `PHP` behavior.

| Syntax              | ZendPHP behavior                           | TypePHP compiler behavior                 |
|---------------------|--------------------------------------------|-------------------------------------------|
| `$array[] = 1`      | Append an element                          | Append an element                         |
| `$array[null] = 1`  | `$array[''] = 1` (null converted to empty string) | Append an element (incompatible with PHP) |
| `$array[''] = 1`    | Empty string key                           | Empty string key                          |

Therefore, if you need to explicitly use an empty string as a key, use `$array['']` directly instead of `$array[null]`.

## Strict Mode
The TypePHP compiler does not allow manually setting the current file to non-strict mode: `declare(strict_types=0)` will cause a compilation error:

```bash
Fatal error: declare(strict_types=0) is not allowed, only strict_types=1 is supported
```

## Undefined Variables
In `ZendPHP`, `isset()` can be used to check whether a variable exists. The TypePHP compiler does not support this pattern. Local variables must be defined before use. Therefore the following code is not allowed.

```php
// The variable is undefined; isset returns false
if (!isset($var)) {
    // stmts
}
```

Must be changed to:
```php
$var = null;
if (!isset($var)) {
    // stmts
}
```
The `isset($var)` expression will always be `true` during static compilation.

If an undefined variable is used, `ZendPHP` only throws a `Warning`, but the compiler reports an error directly and does not allow the use of undefined variables.

```php
function main()
{
    var_dump($testVar);
}
```

```bash
Fatal error: Undefined variable `$testVar` in undef-var.php:4
```

The execution result in `ZendPHP` is as follows:
```bash
Warning: Undefined variable $testVar in undef-var.php on line 4
NULL
```

## Attribute Arguments

TypePHP supports PHP Attribute syntax, including non-empty arrays, nested arrays, constant expressions, and `new` expression arguments.

```php
#[MyAttribute]
#[MyAttribute(1234)]
#[MyAttribute(value: 1234)]
#[MyAttribute(MyAttribute::VALUE)]
#[MyAttribute([])]
#[MyAttribute([1, 2, 3, 'str', true])]
#[MyAttribute(new AttributeOptions(enabled: true))]
#[MyAttribute(100 + 200)]
class Thing
{
}
```

Attribute arguments still cannot contain ordinary function calls:

```php
#[MyAttribute(loadOptions())] // Compilation error
class Thing
{
}
```

### C Extension Compatibility

TypePHP guarantees that the PHP userland `ReflectionAttribute::getArguments()`, `ReflectionAttribute::newInstance()`, and `ReflectionAttribute::__toString()` can read Attribute arguments that contain request-time values such as non-empty arrays, `new`, and closures.

However, the approach where third-party C extensions read arguments directly using the Zend Attribute data structure or low-level C APIs is not supported. For example:

- Directly accessing `zend_attribute`, `zend_attribute_arg.value`;
- Directly calling `zend_get_attribute_value()`;
- Replacing the internal function handler of `ReflectionAttribute::getArguments()` or `ReflectionAttribute::newInstance()`.

TypePHP stores a request-time factory marker in the Attribute's persisted arguments and converts it into an ordinary request-time `zval` when the PHP Reflection method executes. The above C extension paths may bypass this conversion and read the internal marker; if the extension also replaces the ReflectionAttribute handler, it may override TypePHP's handler or be overridden by TypePHP. Therefore, even if an Attribute containing only simple arguments currently appears to work, do not rely on this unsupported calling approach.

A known example is [`symfony/php-ext-deepclone`](https://github.com/symfony/php-ext-deepclone): this extension includes an implementation that directly intervenes in the internal ReflectionAttribute processing flow, and cannot be guaranteed to be compatible with TypePHP's request-time Attribute argument factory.

To be compatible with TypePHP, C extensions should obtain arguments through the PHP userland ReflectionAttribute methods, or the extension should provide its own bridging implementation that explicitly adapts to TypePHP.

Other locations such as class constants and property default values have different `new` expression restrictions; see [Constant Expressions](constant-expressions.md) for the complete rules.

## Traits Are Only Visible at Compile Time

TypePHP treats Traits as compile-time AST templates. During the `convert` phase, a Trait's properties, constants, and methods are injected into the class that uses it and compiled to native code like members directly declared by that class. The Trait itself is not registered as a runtime Trait in ZendVM.

Therefore the following restrictions apply:

- `trait_exists()` returns `false` when querying a Trait defined by TypePHP;
- Dynamic PHP code loaded via `include`, `require`, or `eval()` cannot `use` a Trait defined by TypePHP;
- A TypePHP Trait entity cannot be obtained at runtime through Reflection.

The following pattern is supported, because both the Trait and the class using it are visible at AOT compile time:

```php
trait HasName
{
    public function name(): string
    {
        return self::class;
    }
}

class User
{
    use HasName;
}
```

The following dynamic code is not supported:

```php
trait HasName
{
    public function name(): string
    {
        return self::class;
    }
}

function main(): void
{
    eval(<<<'PHP'
        class DynamicUser {
            use HasName; // Trait "HasName" cannot be found at runtime
        }
        PHP);
}
```

If a class needs to use a TypePHP Trait, the class should also be included in the TypePHP project so that the compiler completes the Trait composition in the AOT phase. Classes that must be dynamically loaded should declare the required members directly, or inherit from a normal TypePHP class that has already been statically compiled with the Trait composed in.

`self`, `parent`, and `static` in Trait methods are compiled according to the composed class context; `__TRAIT__` still retains the original Trait name and can be used to identify the method's source.

## Throwing Exceptions in Destructors

In `ZendPHP`, an exception thrown by the `__destruct()` destructor is caught and ignored by the engine (`zend_try/catch` is invoked in `zend_objects_store_del`), and does not cause resource leaks.

However, in `AOT` compilation mode, the object destruction path goes through `phpx` C++ RAII mechanisms such as `Variant::unset()` / `~Variant()` / `destroy()`. An exception thrown in a destructor interrupts the Zend Engine's two-phase cleanup flow in `zend_objects_store_del`, causing some memory not to be freed (`free_obj` and GC buffer removal are skipped).

The compiler detects exceptions thrown in destructors at compile time and outputs a warning:

```bash
Warning: Throwing exception in MyClass::__destruct() may cause memory leak
```

**Scope of impact**: only exceptions thrown during destructor execution cause memory leaks. The program will not crash or produce undefined behavior, only a minor memory leak (each trigger leaks roughly the object struct and a GC buffer slot).

**Recommendations**:

- Avoid operations that may throw exceptions in `__destruct()`
- If you need to perform operations that may fail during destruction, use `try/catch` inside the method to catch and handle exceptions
- Destructors should only be used to release resources, and should not contain business logic

```php
// Not recommended: throwing an exception in a destructor
class MyClass {
    function __destruct() {
        throw new \Exception("error in destructor"); // Compile warning + runtime memory leak
    }
}

// Recommended: catch and handle internally
class MyClass {
    function __destruct() {
        try {
            // Operations that may fail
        } catch (\Throwable $e) {
            error_log("destructor error: " . $e->getMessage());
        }
    }
}
```

## Coroutine Entry Points Must Catch Exceptions

When using Swoole or Swow coroutines, **you must use `try/catch` inside each coroutine entry point's callback to catch `\Throwable`**. Exceptions cannot propagate to other coroutines, and you cannot rely on a `try/catch` outside the coroutine-creating call to handle them.

This requirement applies to all entry points that may enter a new coroutine or an independent event callback, including:

- `Swoole\Coroutine\run()`, `Swoole\Coroutine::create()`, and `go()`;
- Swoole Server event callbacks such as `onRequest`, `onMessage`, and `onTask`;
- Timer, Event, Process, and other callbacks scheduled by extensions.

The correct approach is to establish an exception boundary immediately when the coroutine callback begins executing:

```php
\Swoole\Coroutine\run(static function (): void {
    try {
        runApplication();
    } catch (\Throwable $exception) {
        error_log((string) $exception);
    }
});
```

You cannot catch only outside the code that creates the coroutine:

```php
// Wrong: the outer catch cannot catch an exception that escapes from the coroutine callback
try {
    \Swoole\Coroutine\run(static function (): void {
        runApplication();
    });
} catch (\Throwable $exception) {
    error_log((string) $exception);
}
```

Exceptions not caught inside the coroutine entry point may cross the Zend callback boundary and abort the process with a message similar to the following:

```text
terminate called after throwing an instance of '_zend_object*'
```

You must catch `\Throwable`, not only `\Exception`, because PHP `Error` also needs to be handled within the current coroutine. After catching, record the complete exception information, and let the current coroutine explicitly decide whether to end the current task, close the connection, or terminate the Worker; the exception must not be rethrown beyond the coroutine boundary.

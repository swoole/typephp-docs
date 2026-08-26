## Calling C++ Functions from PHP Code

A `C++` function can be called from `PHP` code if it meets the following conditions:

1. It must be prefixed with `php_`
2. It must use `PHP` types for parameters and return values
3. The function must be declared in a `.stub.php` file

> In `C++`, the function name must be lowercase; in `PHP` code, function names are case-insensitive.

### PHP and C++ Type Mapping Table

| PHP type   | C++ type        |
| -------- | ------------- |
| int      | php::Int      |
| bool     | php::Bool     |
| array    | php::Array    |
| mixed    | php::Var      |
| float    | php::Float    |
| string   | php::Str      |
| object   | php::Object   |
| resource | php::Resource |
| void     | void          |

For example, declare a function in the `.stub.php` file:

```php
function bar(int $a, bool $b, float $c, string $d, array $e, object $f, mixed $g): array {
   // The stub only declares; the function has no implementation code
}
```

The corresponding function in `C++` is:
```cpp
php::Array php_bar(php::Int a, php::Bool b, php::Float c, php::String d, php::Array e, php::Object f, php::Var g) {
   // Implement this function in C++ code
}
```

You can simplify it with `using namespace php`:
```cpp
using namespace php;

Array php_bar(Int a, Bool b, Float $c, String d, Array e, Object f, Var g) {
   // Implement this function in C++ code
}
```

> Note that in `C++` code the function name must have the `php_` prefix.

Call this function in other `PHP` code:
```php
$arr = bar(1, false, 3.14, "hello", [1, 2, 3,], new stdClass, null);
```

## Calling PHP Functions from C++ Code
1. The function name must have the `php_` prefix
1. The `php_func_decl.h` header file must be included
1. Only functions compiled by the TypePHP compiler can be called; dynamic functions and builtin functions cannot be called directly

For example, define a function in `PHP`:
```php
function my_func($a, $b, $c): mixed {
    var_dump($a, $b, $c);
    return [$a, $b, $c];
}
```

The way to call it from `C++` is:
```cpp
php::Array list = php_my_func("hello", 1234, php::null);
```

### Calling Builtin Functions
Use the `Facade API` provided by `PHPX`

```cpp
php::var_dump(v1);
php::file_get_contents(file);
```

## FFI

Compiled programs can use the PHP `FFI` extension, but the extension capability must be enabled in the runtime environment, for example by setting in `php.ini`:

```ini
ffi.enable=1
```

Objects such as `FFI::cdef()`, `FFI::new()`, and `FFI\CData` are still managed by the PHP FFI extension at runtime. The TypePHP compiler does not translate FFI calls directly into C/C++ static calls, so this kind of code mainly follows the runtime semantics of ZendPHP and the FFI extension itself.

Suggestions:

- FFI header files and dynamic library paths should be configured per the runtime environment, avoiding absolute paths that depend on the build machine.
- Reading `FFI\CData` array elements as rvalues should keep ordinary read semantics; writable access is triggered only when they are written as lvalues.
- For performance-sensitive interfaces with a fixed ABI, prefer the `.stub.php` + C++ implementation approach to avoid runtime FFI parsing and dynamic call overhead.

### Calling User Functions
User functions must be called dynamically through the `ZendVM`.
```cpp
php::call("my_user_func", {a, b, c});
```

## Namespaces
If a function uses a namespace, the name must be changed to `php_{namespace}__{function name}`. For multi-level namespaces, replace the backslash `\\` with double underscores. For example:

```php
Foo\\Bar\\baz();
```
The corresponding `C++` function is
```cpp
php_foo__bar__baz();
```

## Classes and Methods
In addition to functions, you can also implement `PHP` classes. Methods, static methods, properties, constants, and static properties are declared in the `stub` file, while only methods and static methods are implemented in the `C++` file.

* Namespace and class names are joined with double underscores (`__`) as the prefix

### Stub File

```php
class ClassFoo {
    protected string $prop;
    public function __construct(string $name);
    public function bar(int $a): int;
}
```

Properties, static properties, and constants only need to be defined in the `stub` file. The `C++` code only needs to implement the class's methods

```cpp
void php_classfoo____construct(php::Object &this_, php::String name) {}
php::Int php_classfoo__bar(php::Object &this_, php::String name) {}
```

* The first parameter of a class method function is `this_`, representing the current object; for a static method, the first parameter is `NULL`
* You can use `this_.attr()` to read and write object properties, and `this_.call()` to call object methods

## Default Parameters
Default parameters are allowed. For example, declare a function in the `stub` file as:

```php
function foo(string $a = "hello", int $b = 2026);
```

The calling code is:
```php
foo();
```

C++ code:
```cpp
void php_foo(String a, Int b) {
    // The value of a is "hello"
    // The value of b is 2026
}
```

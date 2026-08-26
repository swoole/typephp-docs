# Python Interop

TypePHP can call Python modules, functions, and objects directly at the language level. Both runtimes live in the same process, and arguments and return values are converted through [phpy](https://github.com/swoole/phpy) — no JSON, RPC, or Python subprocess is involved.

Python interop is an optional extension-level feature. TypePHP programs that do not use Python syntax do not depend on phpy; when using this feature, the `phpy` extension must be loaded in the program's PHP runtime environment.

## Environment Setup

First install CPython, the development headers, and phpy. phpy supports Linux, macOS, and Windows, and currently requires Python 3.10 or higher and PHP 8.1 or higher. Building from source, for example:

```bash
git clone https://github.com/swoole/phpy.git
cd phpy
phpize
./configure --enable-phpy --with-python-config=/usr/bin/python3-config
make -j
sudo make install
```

Load the extension in the `php.ini` used by your TypePHP program:

```ini
extension=phpy
```

Verify the extension and the Python runtime:

```bash
php --ri phpy
/usr/bin/python3 --version
```

Third-party Python packages must be installed into the same Python environment that phpy uses. For example:

```bash
/usr/bin/python3 -m pip install numpy
```

phpy must match the Zend ABI of TypePHP/PHP. Do not mix extensions built for different PHP versions, ZTS/NTS modes, or Debug/Release ABIs, or the program may crash at startup or during object destruction.

TypePHP does not link `libphpy.so` directly. The compiler only generates dynamic calls to the Zend class, method, and object APIs, so code that does not use Python adds no runtime dependency. If the program actually executes a Python expression but phpy is not loaded, an ordinary PHP `Error` is thrown; when a Python module does not exist or a Python call fails, a `PyError` is thrown.

## First Program

```php
<?php

use Python\math;

function main(): void
{
    $result = math\sqrt(81);
    echo $result->toValue()->toFloat(), "\n";
}
```

Compile and run:

```bash
tpc hello.php
./hello
```

Output:

```text
9
```

The return value of `math\sqrt()` is still a `PyObject` by default. `toValue()` explicitly converts it into the TypePHP/PHP type system, and `toFloat()` then yields a `float`.

## Importing Python Modules

Use `use python\module` to import a module:

```php
use python\sys;
use Python\numpy as np;
use python\numpy\linalg as linalg;
```

These correspond respectively to Python's:

```python
import sys
import numpy as np
import numpy.linalg as linalg
```

The `python` root namespace is case-insensitive, so `python`, `Python`, and `PYTHON` are all valid. Module paths, member names, method names, and keyword arguments remain strictly case-sensitive:

```php
python\len([1, 2, 3]); // Correct
Python\len([1, 2, 3]); // Correct
python\Len([1, 2, 3]); // Error: Len does not exist in Python
```

`use python\...` only establishes a compile-time module alias; the module is imported only when first actually accessed. A module that is declared but never used does not invoke Python, nor does it error if the module is not installed:

```php
use python\module_that_is_not_installed;

function main(): void
{
    echo "This program does not import the module.\n";
}
```

`use` itself fully follows PHP's namespace and alias rules; TypePHP does not rewrite PHP's name resolution for Python. In the global namespace you can write `python\math\sqrt()` directly; inside `namespace App`, `python\math\sqrt()` is resolved by PHP as `App\python\math\sqrt()`, so you must import it or use the fully qualified name:

```php
namespace App;

use python\math;

$a = math\sqrt(16);          // Imported alias
$b = \python\math\sqrt(25); // Fully qualified name, no `use` needed
```

`python` is a special root namespace of TypePHP and cannot be used to declare an ordinary PHP namespace. A module alias also cannot conflict with other `use` symbols in the same file.

You can also use PHP's native function and constant import syntax, together with `as`:

```php
namespace App;

use function python\len;
use function python\math\sqrt as pySqrt;
use const python\math\pi as pyPi;

len([1, 2, 3]);
$root = pySqrt(16);
$pi = pyPi;
```

Python symbols remain case-sensitive; PHP's alias resolution continues to follow PHP's own rules.

## Module Functions, Classes, and Variables

Callables in a Python module are invoked using PHP namespace-function syntax:

```php
use Python\numpy as np;

$array = np\array([1, 2, 3]);
$zeros = np\zeros([2, 3]);
```

`np\array()` may be a function, a Python class, or an object implementing `__call__`. TypePHP does not guess the member kind; callability is determined by Python at runtime.

Module variables are read using PHP namespace-constant syntax:

```php
use Python\math;
use Python\sys;

$pi = math\pi;
$path = sys\path;
```

Do not write `math::pi` or `math::$pi`; those belong to PHP class-member syntax. Python modules are mapped to namespaces in TypePHP, and member reads are still performed by Python at runtime.

The values of module variables are also `PyObject`s. TypePHP only provides reads; overwriting or deleting module variables through namespace syntax is not supported:

```php
$path = sys\path;      // Supported
sys\path = $newPath;  // Not supported
unset(sys\path);      // Not supported
```

When you genuinely need to modify Python module state, call Python's `setattr()` / `delattr()` explicitly; that is a Python operation the application actively performs.

Modules are cached by their full name. When the same module is used with different aliases across files, the underlying behavior still follows Python's `sys.modules` import semantics.

## Python Builtins and Object Construction

Call Python builtins via `python\name()`:

```php
$length = python\len([1, 2, 3]);
$power = python\pow(2, 10);
python\print('Hello from Python');
```

Common Python proxy objects can be constructed directly:

```php
$list = python\list([1, 2, 3]);       // PyList
$dict = python\dict(['answer' => 42]); // PyDict
$tuple = python\tuple([1, 2]);         // PyTuple
$set = python\set([1, 2]);             // PySet
$str = python\str(123);                 // PyStr
$integer = python\int('42');            // PyObject
$bytes = python\bytes("binary");        // PyObject
```

The container constructors are syntactic sugar for phpy Facades, for example:

```php
$a = python\list([1, 2, 3]);
$b = new PyList([1, 2, 3]);
```

Both have the same runtime semantics. In particular, `python\dict($phpArray)` builds a `PyDict` from the PHP array's keys/values; it does not directly execute CPython's `dict(iterable)`.

First-class callable syntax for Python builtins is not currently supported. When you need to store or pass a Python callable, obtain the corresponding `PyObject` from a module attribute or object attribute first.

## Operating on Python Objects

Python return values are represented by phpy's existing proxy classes. When the concrete type cannot be determined statically, they are uniformly typed as `PyObject`; no second `python\Object` class name is introduced.

### Attributes and Methods

```php
$name = $object->name;
$object->name = 'TypePHP';
$result = $object->greet('hello', suffix: '!');
unset($object->name);
```

These operations respectively invoke Python's attribute read, `setattr`, call, and `delattr` protocols. Python methods support named arguments, and argument names are case-sensitive.

### Subscript and `isset()`

```php
$last = $list[-1];
$list[-1] = 42;
unset($list[-2]);

if (isset($dict['name'])) {
    echo $dict['name'];
}
```

list and tuple support Python negative indexing. `isset()` keeps PHP semantics: it returns `false` when the key/index does not exist, or when the corresponding value is Python `None`. Python exceptions other than `KeyError` and `IndexError` are not swallowed.

### Iteration

```php
foreach ($pythonIterable as $index => $value) {
    echo $index, ': ', $value, "\n";
}
```

The key of a generic Python iterator is the iteration ordinal starting from `0`. `PyDict` uses the dictionary's own keys/values. Python exceptions raised during iteration continue to propagate as `PyError`.

### Calling Callable Objects

```php
$result = $pythonCallable($left, right: 42);

$args = [1, 2, 3];
$result = $pythonCallable(...$args);
```

Non-callable Python objects throw a `PyError` containing a Python `TypeError`.

## Argument Conversion

When entering a Python function, method, constructor, or operator boundary, TypePHP values are automatically converted to Python values:

| TypePHP value | Python value |
|---|---|
| `null` | `None` |
| `bool` | `bool` |
| `int` | `int` |
| `float` | `float` |
| `string` | `str`; the string must be valid UTF-8 |
| list array | `list` |
| map array | `dict` |
| empty array | `list` |
| `PyObject` and its subclasses | The original Python object is kept, contents are not copied |
| TypePHP callable | A callable proxy that Python can call synchronously |

Arrays are deep-copied recursively. Recursive arrays, cyclic containers, excessively deep nesting, and invalid UTF-8 throw a `PyError`; they will not recurse infinitely or crash the process.

All arguments are evaluated strictly left-to-right following the source order, and each is evaluated exactly once:

```php
$result = python\pow(mark(2), mark(3));
```

If the same array is passed to Python frequently inside a loop, convert it once and reuse the proxy object:

```php
$pyItems = python\list($items);

for ($i = 0; $i < 1000; $i++) {
    processor::consume($pyItems); // Only passes the Python object reference
}
```

Avoid passing `$items` directly on every call; otherwise it is deep-copied again on each boundary crossing. Use `python\bytes()` for binary data; an ordinary TypePHP `string` is converted to a Python `str` by default.

## Return Values and Explicit Conversion

The results of Python function calls, method calls, constructor calls, and operations remain `PyObject` or one of its concrete proxy subclasses by default. Even if Python returns `int`, `float`, `bool`, or `str`, TypePHP does not change the variable type based on the runtime type.

Use `PyObject::toValue()` to explicitly enter the TypePHP/PHP type system:

```php
$pyValue = python\int(42);

$plain = $pyValue->toValue();
$integer = $pyValue->toValue()->toInt();
$float = $pyValue->toValue()->toFloat();
$boolean = $pyValue->toValue()->toBool();
$string = $pyValue->toValue()->toString();
$array = python\list([1, 2, 3])->toArray();
```

The functional form `python\scalar($value)` is equivalent to phpy's `PyCore::scalar($value)` and can be used for compatibility with existing code:

```php
$integer = python\scalar($pyValue)->toInt();
```

`toValue()`, `python\scalar()`, and `PyCore::scalar()` use the same conversion rules. The converted result is an ordinary TypePHP value; subsequent operations no longer use the Python protocol.

`toArray()` specifically converts Python `list`, `set`, `tuple`, `dict`, and iterator objects, recursively deep-copying their elements; unsupported Python types return an empty array. Iterators are consumed, so repeated calls may return an empty array. phpy provides a concrete `toArray()` method on `PyObject`; it is also a TypePHP keyword method. When the receiver is a statically-known `PyObject` subclass, the compiler calls the phpy method directly; otherwise it falls back to the generic conversion path. `toValue()` is just an ordinary method of `PyObject`, not a TypePHP keyword method.

`toString()` remains a TypePHP keyword method and uses `PyObject::__toString()`; no additional same-named conversion method needs to be provided in phpy. To convert to other PHP scalars, call `toValue()` first, for example `$pyObject->toValue()->toInt()`.

## Operators

Whenever one side of an expression is statically typed as `PyObject` or a subclass, TypePHP uses Python's full operator protocol. An ordinary TypePHP value on the other side is first converted to a Python object:

```php
$seven = python\int(7);
$three = python\int(3);

$sum = $seven + $three;
$product = $seven * 10;
$reflected = 10 + $seven;
$quotient = $seven / 2; // Python true division
```

Supported protocols include:

- Arithmetic, power, and bitwise operations: `+ - * / % ** << >> & | ^`.
- Unary operations: `+ - ~`.
- Comparison: `== != < <= > >=`.
- Identity: `===` and `!==`.
- Conditional truth value, `!`, `&&`, `||`, and `xor`.
- Compound assignment on variables, object properties, and subscripts, for example `+=`, `*=`, `<<=`.

`/` uses Python true division, not floor division. TypePHP has no `//` operator; call the corresponding Python function explicitly when floor division is needed.

`==` uses Python's value-equality protocol; `===` uses Python object identity:

```php
$list = python\list([1]);
$alias = $list;

var_dump($list === $alias);          // true
var_dump($list === python\list([1])); // false
```

Compound assignment uses Python's in-place protocol and updates the left-hand side with the object returned by the protocol, so correct results are obtained for both mutable and immutable Python types.

### Unary Sign Limitations in Dynamic PHP Code

TypePHP AOT preserves the source AST, so:

```php
-$value; // Python operator.neg(value)
$value * -1; // Python operator.mul(value, -1)
```

The two invoke different Python protocols and may behave differently.

Ordinary PHP code executed dynamically through the ZendVM is an exception. Zend compiles `-$value` and `+$value` into multiplication by `-1` and `1`, so phpy's opcode handler cannot recover the original source intent. Therefore `-$value` in dynamic code keeps the behavior of `$value * -1`, and `+$value` keeps the behavior of `$value * 1`.

For ordinary Python `int` and conventional `float`, the results are usually the same; custom Python types may implement `__neg__()`, `__pos__()`, and `__mul__()` separately, in which case AOT and dynamic code can differ. TypePHP does not rewrite ordinary PHP code through a global AST hook.

## Passing TypePHP Callables to Python

TypePHP functions, closures, and callable objects can be passed as Python arguments and called back synchronously by Python within the current call chain:

```php
$values = python\list([1, 2, 3]);
$mapped = python\map(
    fn (int $value): int => $value * 2,
    $values,
);

$sum = python\sum($mapped)->toValue()->toInt();
echo $sum, "\n"; // 12
```

Keyword arguments passed by Python to the callback are bound to the TypePHP callable's parameters by name. Callbacks are synchronous and can only be used within a Python call relationship initiated by TypePHP; TypePHP does not register functions, classes, or modules with Python that can be independently imported.

## Exception Handling

A missing Python module, missing member, argument error, type error, and user-code exception are uniformly mapped to `PyError`:

```php
try {
    python\len();
} catch (PyError $error) {
    echo $error->getMessage(), "\n";

    // The original Python exception object; all are optional PyObject properties.
    $type = $error->type;
    $value = $error->value;
    $traceback = $error->traceback;
}
```

`PyError` extends PHP's `Exception` and retains the `type`, `value`, `error`, and `traceback` Python objects. Ordinary PHP errors still use the PHP exception system; for example, when phpy is not loaded, failing to resolve `PyCore` throws an `Error`.

Cross-VM exceptions are not silently converted to `null`. After handling an exception you can continue making Python calls; phpy cleans up CPython's pending error state.

## Relationship to Using phpy Directly

TypePHP does not reimplement the Python VM, nor does it create a second set of Python object classes. The following forms can be mixed:

```php
use Python\os;

$module1 = PyCore::import('os');
$name1 = $module1->name;

$name2 = os\name;
$list1 = new PyList([1, 2, 3]);
$list2 = python\list([1, 2, 3]);
```

`use python\module`, `python\name()`, and Python operators are TypePHP's compile-time syntactic sugar; CPython initialization, the GIL, reference counting, object proxies, conversion, and exceptions are all handled by phpy. TypePHP does not include phpy headers, nor does it generate direct calls to phpy C++ symbols.

## Limitations

- Python `threading` is not supported.
- `asyncio` is not supported.
- CPython subinterpreters are not supported.
- TypePHP is not compiled into a Python extension, and importing TypePHP programs from Python is not supported.
- No Python export annotations for TypePHP functions or classes are provided.
- Python source code is not compiled; Python modules are still loaded by CPython at runtime.
- `from package import *` syntax is not supported.
- First-class callable syntax for Python builtins is not supported.
- Whether a Python symbol exists can usually only be determined at runtime.
- Python interop is not currently available for the TypePHP WASM target.
- Unary sign in dynamic PHP code has the protocol difference described above.

TypePHP's goal is to let applications call Python packages efficiently and reliably, not to fully implement Python syntax or an async runtime inside PHP.

During development you can also use:

- [Generate Python IDE hints](python-ide-helper.md): generate declarations for builtins and specified modules that are indexed only by the editor.
- [Convert Python code to TypePHP](python-convert.md): mechanically convert a supported Python AST into TypePHP source code that can be further reviewed and modified.

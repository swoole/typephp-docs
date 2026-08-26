# Converting Python Code to TypePHP

`--convert-python-to-php` is a built-in Python source conversion tool in the `tpc` command entry, used during development to convert a Python script into TypePHP source code that is easier to continue migrating. It is a mechanical conversion tool; it does not guarantee that any Python program is fully equivalent to the output code, and it does not participate in compiling the final program.

## What It Can Do for You

Convert an existing Python file into PHP source code that uses TypePHP's Python namespace syntax. For example, after converting a data-processing script into a TypePHP source file, you can compile it into a native program with the TypePHP compiler, or continue maintaining it in an editor with PHP syntax.

## Prerequisites

The conversion process invokes `python3` from `PATH` to parse the Python source's syntax tree, so you only need a usable `python3` on the host — it does **not depend on the phpy extension, nor does it require the target Python modules to be installed**.

## Usage

```bash
./tpc --convert-python-to-php <file.py> > <file.php>
```

The conversion result is written to standard output and error messages to standard error; exit code `0` means success and `1` means failure. Normally you redirect the result to a file:

```bash
./tpc --convert-python-to-php script.py > script.php
```

## Example

```python
import math
print(math.sqrt(16))
```

```php
use python\math;

function main(): void
{
    python\print(math\sqrt(16));
}
```

An ordinary `import` is converted to a namespace `use`, and module functions are invoked in the form `python\module\function()`.

The generated result should undergo code review and TypePHP testing before being put into production. In particular, destructuring counts, keyword-only arguments, decorators, and Python/PHP scalar semantics have the downgrade rules explicitly listed below; the converter does not replace business-level compatibility validation.

## Syntax Support Reference

The table below lists the Python syntax currently supported by the converter (✅) and the unsupported syntax that raises an error when encountered (❌). Unsupported syntax does not produce PHP code that looks usable but has incorrect semantics; instead it directly raises an error with the source file and line number.

### Statements

| Python syntax | Status | Conversion rule / error |
|---|---|---|
| `x = expr` | ✅ | `$x = expr;`; module-level variables are automatically injected with `global` |
| `x = y = 1` (chained assignment) | ✅ | `$x = $y = 1;` (name targets only; errors on attribute/subscript targets) |
| `x += expr` and other augmented assignments | ✅ | Supports the `+ - * / % ** << >> \| ^ &` family; `//=` and `@=` are expanded into `python\operator\floordiv/matmul($x, ...)` calls |
| `x: int = expr` | ✅ | The annotation is ignored and converted to an ordinary assignment |
| `x: int` (annotation only) | ✅ | Converted to a comment `// annotation-only declaration: x`; not registered as a module global |
| `a, b = x` (destructuring) | ✅ | `[$a, $b] = $x->toArray();` (PyObject is converted to a PHP array then destructured; elements may be names/attributes/subscripts. Nested destructuring, starred destructuring `a, *b = x`, and chained destructuring are not supported. When the element count does not match, nulls are filled in per PHP semantics; Python's ValueError is not raised) |
| `def f(...)` | ✅ | A function named `main` is renamed to `main_` (to avoid conflict with the TypePHP entry point), and call sites are rewritten accordingly |
| Nested `def` | ❌ | `FunctionDef: nested functions require Python closure scope analysis` |
| `@decorator` | ✅ | See "Function Decorators" below |
| `return [expr]` | ✅ | `return [expr];` |
| `if / elif / else` | ✅ | Isomorphic conversion |
| `while` | ✅ | Isomorphic conversion; `while/else` is not supported |
| `for i in iter` | ✅ | `foreach (iter as $i)`; `for/else` and tuple targets are not supported |
| `break` / `continue` / `pass` | ✅ | `pass` → `// pass` comment |
| `global x` | ✅ | `global $x;` (may appear duplicated alongside an auto-injected global; redundant but legal, a known behavior) |
| `del x` / `del o.a` / `del d[k]` | ✅ | `unset(...)`; `del (a, b)` tuple/list targets are expanded element by element; illegal `del` targets (such as `del f()`) are rejected first by the Python parser |
| Module-level string literals (docstring) | ✅ | Converted to a `/** ... */` comment (`*/` is escaped to `* /`) |
| `import a.b` | ✅ | `use python\a;` (only the first segment becomes the alias) |
| `import a.b as x` | ✅ | `use python\a\b as x;` (the `as` is omitted when the alias equals the last segment) |
| `from m import f [as g]` | ✅ | Call sites are mapped to `python\m\f(...)` |
| `from . import m` | ❌ | `ImportFrom: relative imports are not supported yet` |
| `from m import *` | ❌ | `ImportFrom: star imports are not supported` |
| `class` | ❌ | `ClassDef` |
| `with` | ❌ | `With` |
| `raise` / `try` / `assert` | ❌ | `Raise` / `Try` / `Assert` |
| `async def` / `await` | ❌ | `AsyncFunctionDef` (`await` is unreachable; the outer construct errors first) |
| `match` | ❌ | `Match` |
| `nonlocal` | ❌ | `Nonlocal` |

### Function Signatures

| Python form | Status | TypePHP output |
|---|---|---|
| `def f(x, y=4)` | ✅ | `function f($x, $y = 4)` |
| `def f(a, *, b)` | ✅ | `function f($a, $b = null)` (keyword-only parameters without defaults are padded with `null`) |
| `def f(*args)` / `def f(**kw)` | ✅ | `function f(...$args)` |
| `def f(*a, **kw)` | ❌ | `FunctionDef: simultaneous *args and **kwargs cannot be represented by one PHP signature` |
| `lambda a, b=2: a + b` | ✅ | `fn ($a, $b = 2) => $a + $b` |

### Expressions

| Python syntax | Status | Conversion rule / error |
|---|---|---|
| Literals `int / float / str / True / False / None` | ✅ | `var_export`; `None` → `null` |
| `b'...'` bytes | ❌ | `{file}: Python bytes literals are not supported yet` (no line number) |
| `1j` complex | ❌ | `{file}: Python complex literals are not supported yet` (no line number) |
| Variable names | ✅ | `$name`; `this` is escaped to `$this_` |
| Module alias as a value | ❌ | `a Python module cannot be used as a first-class value in TypePHP namespace syntax` |
| Attribute chain `o.a.b` | ✅ | `$o->a->b`; for a module alias chain only the first segment is a module member: `sys.version_info.major` → `sys\version_info->major` |
| Module attribute assignment/deletion | ❌ | `Attribute: Python module attributes cannot be assigned or deleted` |
| Function calls | ✅ | Already-defined functions connect directly `f(...)`; builtin functions map to `python\len(...)`; `from m import f` maps to `python\m\f(...)`; other names are callable as variables `$f(...)` |
| Keyword arguments / `*args` / `**kwargs` calls | ✅ | `f(x: 1, ...$args)` |
| Container literals `[] () {} {:}` | ✅ | `python\list/tuple/set/dict([...])`, `...` unpacking supported |
| Binary operators `+ - * / % ** << >> \| ^ &` | ✅ | Isomorphic conversion |
| `//` floor division / `@` matrix multiplication | ✅ | `python\operator\floordiv(a, b)` / `python\operator\matmul(a, b)` |
| Unary operators `- + not ~` | ✅ | `- + ! ~` |
| Comparisons `== != < <= > >=` | ✅ | Isomorphic conversion |
| `is` / `is not` | ✅ | `===` / `!==` |
| `in` / `not in` | ✅ | `python\operator\contains(b, a)` (arguments swapped) / negated |
| Chained comparisons `a < b < c` | ❌ | `Compare: chained comparisons require explicit temporary variables` |
| `a and b` / `a or b` | ❌ | `BoolOp` |
| `x if c else y` | ✅ | `(c ? x : y)` |
| Subscript `a[i]` / slice `a[l:u:s]` | ✅ | `$a[$i]` / `$a[python\slice(l, u, s)]` (defaults are `null`) |
| f-string | ✅ | Concatenation + `->toString()`; precedence-sensitive expressions such as operators are fully parenthesized |
| f-string `!r` conversion / `:03d` format spec | ❌ | `FormattedValue: formatted f-string conversions are not supported yet` |
| Walrus `:=` | ✅ | Assignment inside an expression `($n = 10)` |
| Comprehensions / generator expressions | ❌ | `ListComp` / `SetComp` / `DictComp` / `GeneratorExp` |
| `yield` / `yield from` | ❌ | `Yield` / `YieldFrom` |

### Function Decorators

Decorators are rebound to the same-named module variable at the start of `main()` (before other top-level statements) **bottom-up** following Python semantics:

```python
@a
@b
def greet(): ...
```

```php
function greet() { ... }

function main(): void
{
    global $greet;
    $greet = b('greet');
    $greet = a('greet');
    ...
}
```

- A decorator can be a defined function, an imported symbol from `from m import f`, a module attribute, or a decorator factory (`@dec('x')` → `$greet = dec('x')('greet');`);
- The decorated function name is registered as a module global, and all call sites (including those inside other function bodies) invoke the decorated result indirectly via `global` + variable: `$greet()`;
- Recursive calls inside the decorated function body also resolve to the decorated variable, consistent with Python semantics.

### Error and Downgrade Principles

When semantics can be preserved strictly, the converter uses native PHP forms directly: `print()` with no arguments or with safely convertible arguments is converted to `echo` with a newline, and `sys.exit()` with an integer-literal exit code is converted to `exit`. `print()` with `sep`, `end`, `file`, or `flush` arguments, as well as `sys.exit()` with string or object forms, do not fully match PHP behavior and remain Python calls.

For syntax that is not yet supported (such as `class`, `async`, `with`, `try`, comprehensions, generators, nested functions, chained comparisons, etc.), the converter **raises an error directly and gives the source file and line number**, and does not generate PHP code that looks usable but has incorrect semantics. It is recommended to convert the core logic first, then manually complete the small number of unsupported fragments.

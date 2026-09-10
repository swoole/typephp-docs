# Debugging Guide

The TypePHP compiler translates PHP code into C++ and compiles it into a native binary, so the debugging toolchain is based on GDB/LLDB and does not support runtime debuggers such as Xdebug or Zend breakpoints.

---

## Prerequisites

Before debugging, ensure the following three configurations are in place:

1. **Disable optimization** — set the optimization level to `-O0`, otherwise functions may be inlined and variables may be optimized away
2. **Enable debug symbols** — add the `--debug` argument at compile time, equivalent to passing `-g -O0` to the C++ compiler
3. **Use a Debug build for PHPX / PHP** (recommended) — makes it easier to set breakpoints inside PHP internal functions

```bash
# PHPX Debug build
cmake . -DCMAKE_BUILD_TYPE=Debug && make -j$(nproc)

# PHP Debug build
./configure --enable-debug && make -j$(nproc)
```

---

## Compile commands

### Basic usage

```bash
# Generate a debuggable binary
./tpc app.php --debug

# Using project.yml
./tpc project.yml --debug
```

`--debug` automatically sets `-O0` and appends the `-g` compile flag, disabling all optimizations.

### Other debug-related options

| Option | Effect |
|------|------|
| `-d`, `--debug` | Disable optimization and add debug symbols |
| `-O0` | Set the optimization level to 0 alone (equivalent to the optimization part of `--debug`) |
| `--sanitize=address` | Enable AddressSanitizer to detect out-of-bounds access, use-after-free, etc. |
| `--sanitize=undefined` | Enable UndefinedBehaviorSanitizer to detect integer overflow, null pointers, etc. |
| `-j <num>` | Number of parallel compile jobs (equivalent to `make -j`), default 4 |

```bash
# ASAN + debug symbols for quickly locating memory errors
./tpc app.php --debug --sanitize=address

```

### Viewing the generated C++ code

The compiled `.cc` and `.o` files are located in the `build/` directory. You can inspect the generated C++ source to understand the compiler's translation results:

```bash
ls build/*.cc
# Sample output:
# build/main.cc
# build/src__utils.cc
# build/src__database.cc
```

---

## GDB / LLDB debugging

### Starting

```bash
gdb ./hello
```

LLDB (macOS):

```bash
lldb ./hello
```

### Function symbol naming rules

All AOT-compiled functions are exported in the binary with C linkage symbols, following these naming rules:

#### Ordinary functions

```
php_{function name}
```

If namespaces are used in the source code, the slash `/` is replaced with a double underscore `__`. **Function names are all lowercase.**

```php
// PHP source
function my_add(int $a, int $b): int { ... }
Foo\Bar\baz();

// Corresponding symbols
php_my_add
php_foo__bar__baz
```

#### Class methods

```
php_{class name}__{method name}
```

If the class has a namespace, the namespace is used as a prefix separated by `__`. **All lowercase.**

```php
// PHP source
class UserService {
    public function create(array $data): int { ... }
}

// Corresponding symbol
php_userservice__create
```

#### Built-in functions and methods

| Type | Symbol format | Example |
|------|----------|------|
| Built-in functions | `zif_{function name}` | `zif_array_merge` |
| Built-in methods | `zim_{class name}_{method name}` | `zim_arrayobject_count` |

> The symbol formats of built-in functions/methods depend on the implementation of the specific extension; the above are common naming conventions.

### Breakpoint examples

```bash
(gdb) b php_my_add         # set a breakpoint at the my_add function entry
(gdb) b php_userservice__create  # set a breakpoint at the UserService::create entry
(gdb) b main.cc:42         # set a breakpoint at a specific line in the generated C++ file
(gdb) r                    # run
```

```bash
Breakpoint 1, php_my_add (a=1, b=2) at /home/swoole/workspace/aot/build/examples/myext/test.cc:9
9       php::Int tmp_var_0 = 0;
```

### Viewing variables

#### Native C++ types

`int64_t`, `double`, `bool`, etc. can be viewed directly with `print`:

```bash
(gdb) print a
$1 = 1
(gdb) print b
$2 = 2
```

#### PHP types (php::Variant / php::Array / php::String, etc.)

Use the `.print()` method to output a PHP-style readable representation:

```bash
(gdb) call env.print()
         array(61) {
           ["SHELL"]=>
           string(9) "/bin/bash"
           ["SESSION_MANAGER"]=>
           string(71) "local/swoole-26:@/tmp/.ICE-unix/4392,unix/swoole-26:/tmp/.ICE-unix/4392"
           ...
         }
```

The `.print()` method is available in Debug builds (PHpx internally compiles under the `DEBUG` macro), and its output format is similar to PHP's `var_dump()`.

Common debug calls:

```bash
(gdb) call var.print()      # print a php::Variant value
(gdb) call arr.print()      # print php::Array contents
(gdb) call str.toCString()  # get the C string of a php::String
(gdb) call obj.print()      # print php::Object contents
```

#### Box objects (BigInt / Decimal / BigFloat, etc.)

Box objects do not support direct viewing with `print`; they must be converted via static methods:

```bash
# View a BigInt value
(gdb) call php::BigInt::toString(bi).print()

# View a Decimal value
(gdb) call php::Decimal::toString(dec).print()
```

### Common GDB command reference

| Command | Description |
|------|------|
| `b <sym>` | Set a breakpoint at a symbol |
| `b <file>:<line>` | Set a breakpoint at a file line number |
| `b <sym> if <cond>` | Conditional breakpoint: `b php_my_func if a > 10` |
| `r` / `run` | Start the program |
| `c` / `continue` | Continue execution |
| `n` / `next` | Step over (does not enter functions) |
| `s` / `step` | Step into functions |
| `finish` | Run until the current function returns |
| `p <var>` / `print` | Print a variable value |
| `info locals` | View all local variables in the current stack frame |
| `info args` | View the current function arguments |
| `bt` / `backtrace` | View the call stack |
| `frame <n>` | Switch to the nth stack frame |
| `x/10x <ptr>` | View the 10 words a pointer points to in hexadecimal |
| `watch <var>` | Watch a variable for changes |
| `disas` | Disassemble the current function |

### LLDB equivalent commands

| Operation | GDB | LLDB |
|------|-----|------|
| Breakpoint | `b php_foo` | `b php_foo` |
| Run | `r` | `r` |
| Step over | `n` | `thread step-over` |
| Step into | `s` | `thread step-in` |
| Print variable | `p var` | `frame variable var` |
| View stack | `bt` | `bt` |
| Call method | `call obj.print()` | `expression obj.print()` |

---

## Memory error detection

### AddressSanitizer (ASAN)

```bash
# Enable at compile time
./tpc app.php --sanitize=address

# Control behavior with environment variables
export ASAN_OPTIONS=detect_leaks=1:abort_on_error=1:halt_on_error=1
./app
```

Common ASAN options:

| Option | Description |
|------|------|
| `detect_leaks=1` | Detect memory leaks when the program exits |
| `abort_on_error=1` | Abort on the first error (generates a core dump) |
| `halt_on_error=1` | Exit on the first error (no core) |
| `log_path=/tmp/asan` | Report output path prefix |

ASAN can detect:

- Heap/stack/global buffer overflows
- use-after-free / use-after-return
- Double free
- Memory leaks (requires `detect_leaks=1`)

### Valgrind (Linux)

```bash
# Full leak detection
valgrind --leak-check=full \
         --show-leak-kinds=all \
         --track-origins=yes \
         ./app

# Use for a specific program only
valgrind --leak-check=full --log-file=vg.log ./app
```

> **Note**: PHP's `zend_alloc` interferes with Valgrind detection. If you suspect a leak comes from the phpx runtime layer, you can disable ZendMM at compile time, or first set `USE_ZEND_ALLOC=0`.

### Common leak causes

| Pattern | Cause |
|------|------|
| Box objects are not released | PHP variables not unset / reference cycles / GC not triggered |
| C++ `new` without `delete` | Hand-written C++ extension code does not manage lifetimes properly |
| Array element leaks | `php::Array` holds many elements with an overlong lifetime |

---

## Troubleshooting common problems

### 1. Segmentation fault

**Troubleshooting steps**:

```bash
# 1. Locate the crash point with GDB
gdb ./app
(gdb) run
# After the crash
(gdb) bt         # view the call stack
(gdb) frame 0    # locate the crashing frame
(gdb) info locals

# 2. Rebuild with ASAN enabled
./tpc app.php --sanitize=address
./app
```

**Common causes**:

- `nullptr` dereference — check whether the `.toBox<T>()` return value is null
- Array out of bounds — check the index range of `operator[]`
- use-after-free — a Box object is released early but still referenced
- Type conversion error — the class specified by `toObject()` does not match the actual type

### 2. Compilation errors

The TypePHP compiler may report errors during the translation stage (earlier than the C++ compiler). Common compile-time errors:

| Error | Cause |
|------|------|
| `Cannot re-assign variable from X to Y` | Variable types are immutable; a variable declared as `int` cannot later be assigned a `string` |
| `Undefined variable` | Variables must be defined before use; `isset()` for detecting undefined variables is not supported |
| `declare(strict_types=0) is not allowed` | Only strict mode `declare(strict_types=1)` is supported |
| `Cannot convert float to Decimal/BigInt` | Floating-point literals cannot be converted directly to high-precision types; use a string instead |
| Function call argument count mismatch | All arguments in the function declaration must be supplied |

### 3. Runtime behavior differences

Behavioral differences between AOT binaries and ZendPHP (not bugs, but inherent characteristics of AOT compilation):

- **Type errors are hard errors** — ZendPHP implicitly converts types; the TypePHP compiler reports a Fatal Error directly
- **Division behavior** — `$a / $b` in AOT behaves like C++: integer division results remain integers (unless `std::any()`)
- **String concatenation** — concatenating non-strings with strings requires explicit conversion; there is no automatic `toString()`
- **Undefined variables** — ZendPHP emits a Warning, AOT reports a compilation error directly

### 4. Box object issues

```bash
# Box objects look like "resource" types; confirm while debugging
(gdb) call box.isResource()
$1 = true

# Confirm that toBox returns non-null
(gdb) call box.toBox<BigInt>().toString(box).print()

# Check whether it is the expected Box subclass (via type information)
# Box internally has type_info / extra_info fields that can be used for identification
```

---

## Debugging the generated C++ code

When you need to deeply understand the compiler's behavior or troubleshoot code generation issues, you can directly inspect and debug the generated `.cc` files.

### Viewing the generated code

```bash
ls build/*.cc
cat -n build/main.cc | head -100
```

### Setting breakpoints in the generated code

```bash
gdb ./app
(gdb) b build/main.cc:42     # set a breakpoint at a specific line in the generated code
(gdb) r
```

### Key markers for understanding the generated code

| Generated code pattern | Corresponding PHP source |
|-------------|-------------|
| `tmp_var_N` | Compiler-generated temporary variable |
| `php_my_func()` | PHP function `my_func()` |
| `php_myclass__mymethod()` | Class `MyClass::myMethod()` |
| `php::toBigInt(expr)` | BigInt type conversion |
| `php::toDecimal(expr)` | Decimal type conversion |
| `php::BigInt::add(a, b)` | BigInt addition |
| `php::zend_call("func", args)` | Dynamic function call |

---

## Multi-threaded / coroutine debugging

The TypePHP compiler supports Swoole/Swow coroutines. Note when debugging coroutines:

```bash
# View all threads
(gdb) info threads

# Switch to a specific thread
(gdb) thread <n>

# Call stack of all threads
(gdb) thread apply all bt
```

---

## Environment variables

| Variable | Effect |
|------|------|
| `ASAN_OPTIONS` | Controls AddressSanitizer behavior |
| `USE_ZEND_ALLOC=0` | Disables PHP's memory manager (for use with Valgrind) |
| `GDBHISTFILE` | GDB command history file path |

---

## Quick diagnostic workflow

**Crash**:
1. Rebuild with `--debug`
2. Get the `bt` stack with GDB
3. Check `info locals` of the crashing frame
4. Use ASAN to assist locating

**Memory problems**:
1. Rebuild with `--sanitize=address`
2. Or use Valgrind `--leak-check=full`
3. Focus on Box object lifetimes and array references

**Unexpected behavior**:
1. Check whether the code generated in `build/*.cc` matches expectations
2. Confirm whether variable types match expectations (`info locals`)
3. Set breakpoints at key function entries and compare step by step

**Extension load failure**:
1. Check dependent libraries with `ldd`
2. Use `php -d display_startup_errors=1 -d extension=<path>` to view detailed errors
3. Confirm symbol exports with `nm -D`

---

*This document was last updated: 2026-06-03*

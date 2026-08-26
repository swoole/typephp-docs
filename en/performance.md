The TypePHP compiler treats performance as its primary goal, striving to generate the best possible executable instructions and elevate the performance of `PHP` programs to the same level as static programming languages such as `C/C++`, `Rust`, and `Golang`.

While maintaining high performance, it also guarantees absolute memory safety. Unlike `C/C++` and `Rust`, the TypePHP compiler has no `Unsafe` code. Different from `Rust`'s zero-cost abstraction design, the TypePHP compiler's design goal is to achieve high performance with low-cost abstraction and absolute safety while preserving the ease of use of the `PHP` language.

## Machine Instructions
`ZendVM` compiles `PHP` programs into `opcode` bytecode, then runs them using `ZendVM`. Each `opcode` execution requires a `function call`.

The TypePHP compiler compiles `PHP` programs into `x86_64` or `ARM64` machine instructions that can be executed by the `CPU`. Therefore the TypePHP compiler's performance is consistent with static languages such as `C/C++`, `Rust`, and `Golang`, and is more than `10` times higher than `ZendVM`.

## Native Types
The TypePHP compiler provides only `3` native types:
- `Int`: `8`-byte signed integer
- `Float`: `8`-byte signed floating-point
- `Bool`: `1` byte, only `0` or `1`

At the execution stage, all native type computations can be understood as direct operations on the `int64_t` type. For example, the following code:
```php
function add(int $a, int $b): int {
    return $a + $b;
}
```

The corresponding assembly instructions are:
```asm
add:
    mov rax, rdi    ; RAX = a
    add rax, rsi    ; RAX = RAX + b
    ret
```

With `O2/O3` optimization, using `$result = add($x, $y)` is inlined and optimized into `2` instructions:
```asm
lea rax, [x + y]   ; No call/ret instructions
mov [result], rax
```

## Object Properties

The TypePHP compiler optimizes reads and writes of object properties into efficient memory operations, with performance almost equivalent to `C Struct`.

Because object properties are optimized as memory slots with a fixed layout and fixed type, a property must always maintain its declared type throughout the object's lifetime. Do not use `unset()` or assign `null` to the fixed value type properties `int`, `float`, `bool`, `string`, or `array`, and do not write a type incompatible with the declaration; non-null assignment to a concrete class object property allows subclass objects that satisfy the `is-a` relationship. When null semantics are needed, explicitly declare a nullable type and assign `null`.

```php
class Obj {
     public int $a;
     public float $b;
}
$o = new Obj;
$obj->a = 10;
```

In actual execution, `$obj->a` is directly converted to an offset from the object pointer, yielding very high performance. This is almost identical to the way `C Struct` elements are accessed.

```c
zend_object *o;
Z_LVAL_P(o + property_offset) = 10;
```

If the object property is a native type, the TypePHP compiler generates even more efficient instructions:
```c
zend_object *o;
php::Int &property_a = Z_LVAL_P(o + property_offset);

property_a = 10;
```

This achieves peak performance in programs that compute on properties within large loops.

```php
class Obj {
     public int $a;
     public float $b;

     function foo() {
        $n = 10000000;
        while($n--) {
            $this->a += $n;
        }
    }
}
```

Equivalent to the following `C++` code
```cpp
php::Int &property_a = Z_LVAL_P(this_ + property_offset);
int64_t n = 10000000;
while (n--) {
    property_a += n;
}
```

## Function/Method Calls

### Built-in Functions/Class Methods
Built-in functions and built-in class methods provided by `ZendVM` are dynamically invoked using `ZendVM`'s `zend_call_known_function`; the TypePHP compiler fetches the `zend_function *` pointer once at runtime and stores it in a function table, reducing lookups of `EG(function_table)`.

Such functions can usually have their arguments and return values determined at compile time, and their existence can be confirmed, so they can be cached in the function table to improve performance.

If a dynamic call is used, it cannot be optimized into a `known call` and can only be invoked dynamically at runtime.

```php
$fn = "str_repeat";
// Cannot be optimized
$fn("a", 100);
```

### Dynamic Functions/Class Methods
Functions and class methods defined in user code and loaded via `Composer Autoload` need to dynamically look up `EG(function_table)` to obtain the `zend_function *` pointer, then submit it to `ZendVM` for dynamic execution.

### Native Functions/Class Methods
Functions/class methods defined by `PHP` after `AOT` compilation are invoked as native functions. Native function calls only require memory and register operations for pushing and popping the stack, offering better performance than `ZendVM`'s dynamic function calls.

> Native function calls do not produce a stack trace and cannot be obtained via `debug_backtrace`

#### Scenarios That Cannot Be Optimized into `Native Call`

The TypePHP compiler only optimizes a method call into `Native Call` when it can prove the actual type of the receiving object. In the following scenarios, the receiving object cannot be proven stable, so dynamic calls are still used:

```php
// A temporarily constructed object with an immediate method call cannot be optimized into Native Call
(new Foo())->bar();

// An object from a global variable cannot be optimized into Native Call
global $foo;
$foo->bar();
```

To obtain better method call performance in hot paths, it is recommended to use local variables or function arguments with determinable types:

```php
$foo = new Foo();
$foo->bar();
```

Native functions can be inlined and optimized by the `C++` compiler, yielding very high performance. For example:

```php
function fib(int $n): int
{
    if ($n == 1 || $n == 2) {
        return 1;
    } else {
        return fib($n - 1) + fib($n - 2);
    }
}

function main(int $argc, array $argv): void
{
    $n = $argv[2];
    $begin = microtime(true);
    echo fib($n) . "\n";
    echo "Time: " . (microtime(true) - $begin) . "\n";
}
```

In this code, the `fib` function will be tail-call optimized by the compiler, ultimately generating flat `CPU` instructions with performance hundreds of times higher than ordinary `PHP` dynamic function calls.

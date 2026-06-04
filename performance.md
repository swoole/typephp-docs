`AOT`编译器将性能作为第一目标，力求生成最佳的可执行指令，将`PHP`程序的性能提升至与`C/C++`、`Rust`、`Golang`等静态编程语言同等水平。

在保持高性能的同时，又保证绝对的内存安全。这与`C/C++`、`Rust`不同，`AOT`编译器不存在`Unsafe`代码。与`Rust`零成本抽象的设计不同，`AOT`编译器的设计目标是在保持`PHP`语言的易用性前提下，低成本抽象+绝对安全地实现高性能。

## 机器指令
`ZendVM`是将`PHP`程序编译为`opcode`字节码，然后使用`ZendVM`运行。每个`opcode`执行都需要进行一次`函数调用`。

`AOT`编译器将`PHP`程序编译为`x86_64`或`ARM64`机器指令，可被`CPU`执行。因此`AOT`编译器的性能与`C/C++`、`Rust`、`Golang`等静态语言性能一致，相比`ZendVM`要高出`10`倍以上。

## 原生类型
`AOT`编译器只提供了`3`种原生类型：
- `Int`：`8`字节有符号整数
- `Float`：`8`字节有符号浮点型
- `Bool`：`1`字节，仅`0`或`1`

在执行阶段，所有原生类型计算时可理解为对`int64_t`类型的直接操作。例如下面的代码：
```php
function add(int $a, int $b): int {
    return $a + $b;
}
```

对应的汇编指令为：
```asm
add:
    mov rax, rdi    ; RAX = a
    add rax, rsi    ; RAX = RAX + b
    ret
```

当使用`O2/O3`优化时，使用`$result = add($x, $y)`会内联被优化为`2`条指令：
```asm
lea rax, [x + y]   ; 没有call/ret指令
mov [result], rax
```

## 对象属性

`AOT`编译器对对象属性的读写，会优化为高效的内存操作，几乎可以等同于`C Struct`的性能。

```php
class Obj {
     public int $a;
     public float $b;
}
$o = new Obj;
$obj->a = 10;
```

实际执行时，`$obj->a`会直接转为对象指针的偏移，性能会非常高。这与访问`C Struct`元素的方式几乎是一致的。

```c
zend_object *o;
Z_LVAL_P(o + property_offset) = 10;
```

若对象属性为原生类型，`AOT`编译器还会生成更高效的指令：
```c
zend_object *o;
php::Int &property_a = Z_LVAL_P(o + property_offset);

property_a = 10;
```

这在大循环中对属性计算的程序中性能将达到极致。

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

等价于下面的`C++`代码
```cpp
php::Int &property_a = Z_LVAL_P(this_ + property_offset);
int64_t n = 10000000;
while (n--) {
    property_a += n;
}
```

## 函数/方法调用

### 内置函数/类方法
由`ZendVM`提供的内置函数、内置类方法，使用`ZendVM`的`zend_call_known_function`动态调用，`AOT`编译器会在运行时一次性获取`zend_function *`指针，并存储至函数表中，减少对`EG(function_table)`的查询。

此类函数通常在编译期就可以得到参数、返回值，并且可以确定函数存在，因此可以被缓存至函数表中，以提高性能。

若使用了动态调用，则无法优化为`known call`。只能在运行时动态调用。

```php
$fn = "str_repeat";
// 无法优化
$fn("a", 100);
```

### 动态函数/类方法
有用户代码定义、通过`Composer Autoload`加载的函数和类方法，需要动态查找`EG(function_table)`，获取`zend_function *`指针后提交至`ZendVM`动态执行。

### 原生函数/类方法
由`AOT`编译后`PHP`定义的函数/类方法将作为原生函数调用。原生函数调用仅需进行入栈和出栈的内存、寄存器操作，相比`ZendVM`的动态函数调用性能更好。

> 原生函数调用不会产生堆栈，无法使用`debug_backtrace`获取

原生函数可被`C++`编译器内联优化，性能会非常高。例如：

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

这段代码中`fib`函数将会被编译器进行尾递归优化，最终生成平坦的`CPU`指令，性能将比普通的`PHP`动态函数调用高出数百倍。
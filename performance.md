`AOT`编译器将性能作为第一目标，力求生成最佳的可执行指令，将`PHP`程序的性能提升至与`C/C++`、`Rust`、`Golang`等静态编程语言同等水平。

在保持高性能的同时，又保证绝对的内存安全。这与`C/C++`、`Rust`不同，`AOT`编译器不存在`Unsafe`代码。与`Rust`零成本抽象的设计不同，`AOT`编译器的目标是低成本抽象+绝对安全。

## Native Type
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
while(n--) {
    property_a += n;
}
```
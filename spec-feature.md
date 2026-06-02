`AOT`编译器除了常规的`PHP`语法之外添加了一些专有的特性。

> 文档中`any`类型表示该变量无类型，对应的`PHP`类型为`mixed`，`PHPX`类型为`php::Var`

## use native_types

要求编译器将`int`、`float`、`bool`类型转为原生的类型，以提高运算性能。

```php
use native_types;

function foo() {
    $a = 1000;
    while($a--) {

    }
}
```

使用`use native_types`后，局部变量赋值为整数时，会被声明为`php::Int`而不是`php::Var`，这在密集运算场景下会有巨大的性能提升。若不添加，则默认为`php::Var`，将使用`zval`结构体保存整数。

## use bigint_types

将当前文件中所有整数字面量自动声明为 `BigInt` 类型，无需手动使用 `std::bigInt()` 包装。

```php
declare(strict_types=1);
use bigint_types;

function main(): void {
    // 普通整数字面量自动成为 BigInt
    $a = 42;
    echo $a->toString();   // "42"

    // BigInt 之间运算
    $b = $a + 10;
    echo $b->toString();   // "52"

    // 字面量运算也是 BigInt
    $c = 100 + 200;
    echo $c->toString();   // "300"
}
```

使用 `use bigint_types` 后，所有 `Scalar_Int` 字面量在编译时会被转为 `php::newBigInt(N)` 调用。若不使用此指令，则只有 19 位及以上的超长整数字面量才会被自动识别为 BigInt（参见 [math.md §11](math.md#11-超长字面量自动识别)）。

### 与 `use native_types` 的区别

| 指令 | 整数字面量类型 | 适用场景 |
|------|--------------|---------|
| 无 | `php::Int`（原生 int64） | 普通整数运算 |
| `use native_types` | `php::Int`（原生 int64） | 高性能整数运算 |
| `use bigint_types` | `php::BigInt`（任意精度） | 需要大整数或链式调用 BigInt 方法 |

`use bigint_types` 和 `use native_types` 可以同时使用。同时使用时，非字面量的整数变量仍为 `php::Int`，但整数字面量会被提升为 `php::BigInt`。

## use decimal_types

将当前文件中所有浮点数字面量自动声明为 `Decimal` 类型，无需手动使用 `std::decimal()` 包装。

```php
declare(strict_types=1);
use decimal_types;

function main(): void {
    // 普通浮点字面量自动成为 Decimal
    $a = 3.1;
    echo $a->toString();   // "3.1"

    // Decimal 之间运算
    $b = 2.5;
    $c = $a->add($b);
    echo $c->toString();   // "5.6"

    // 混合运算：Int + Float 字面量 → Decimal
    $d = 10 + 0.5;
    echo $d->toString();   // "10.5"
}
```

使用 `use decimal_types` 后，所有 `Scalar_Float` 字面量在编译时会被转为 `php::newDecimal(...)` 调用。若不使用此指令，则只有 16 位及以上有效数字的浮点字面量才会被自动识别为 Decimal。

> **注意**：由于 PHP 解析器在解析浮点数字面量时可能已经引入了二进制浮点误差，建议在高精度要求下仍然使用字符串形式的 `std::decimal("...")` 构造。`use decimal_types` 最适合在全部使用 Decimal 运算的项目中减少样板代码。

### 与 `use bigint_types` 同时使用

```php
declare(strict_types=1);
use bigint_types;
use decimal_types;

function main(): void {
    // 整数字面量 → BigInt
    $a = 100;
    echo $a->toString();   // "100"

    // 浮点字面量 → Decimal
    $b = 2.5;
    echo $b->toString();   // "2.5"

    // BigInt + Int 字面量 → BigInt
    $c = $a + 50;
    echo $c->toString();   // "150"

    // Decimal + Float 字面量 → Decimal
    $d = $b + 1.5;
    echo $d->toString();   // "4.0"
}
```

## toObject(ClassName::class)

`toObject()` 是编译器内置的关键词方法，用于从 `any` / `mixed` 类型重建带类信息的对象类型。从数组读取元素或调用返回 `any` 类型的函数后，编译器丢失了具体的类信息，可使用此方法完成类型接续。

```php
$obj = $array['object']->toObject(App\Hello\Test::class);
$obj->foo();
```

编译器可以重新得到 `$obj` 对象的类型，将其方法调用转为 `Native Call` 而不是 `zend_call_function()` 的动态调用，性能可大幅提升。

> 不带参数的 `toObject()` 返回的是无类信息的 `php::Object`，与 `(object)` 转换等价。要重建具体类信息，必须传入 `ClassName::class`。

从数组提取元素，默认该变量的类型是`any`。对象可使用`$var->toObject(ClassName::class)`重新接续类型，其他类型则可以使用类型转换函数或者类型转换语法实现接续。方法如下：

#### 1. 使用转换语法接续类型
- 整数：`$v = (int) $array[$key]`
- 浮点：`$v = (float) $array[$key]`
- 布尔值：`$v = (bool) $array[$key]`
- 字符串：`$v = (string) $array[$key]`
- 数组：`$v = (array) $array[$key]`


请注意对象（`object`）类型在`PHP`中实际上是无类型的，它与`any`类型几乎是等价的。若使用`$v = (object) $array[$key]`，`$v`会被声明为`php::Object`而不是`php::Var`，但编译器无法获得该对象的`class`信息。因此`object`转换对编译器来说没有任何意义。同样，`callable`、`iterator`类型对编译器也没有任何帮助。

#### 2. 使用转换函数接续类型
- 整数：`$v = intval($array[$key])`
- 浮点：`$v = floatval($array[$key])`
- 布尔值：`$v = boolval($array[$key])`
- 字符串：`$v = strval($array[$key])`

请注意内置的转换函数仅此`4`种，`PHP`未提供`arrayval()`函数，因此若需要将变量声明为`array`类型，只能使用转换语法实现。

除了数组元素的类型接续之外，赋值操作若右值为`any`类型，默认左值也会被声明为`any`类型，可以使用上述方法声明更准确的类型。

```php
$a = any(3.001);
// $b 的类型将是 php::Int，值会转为 3 
$b = intval($a);
```

## any($value)
此函数的目的是将变量类型标注为 `php::Var` ，而不是原生类型。例如：

```php
$a = any(123);
$b = 123;
```

如果不使用 `any` 函数，变量会声明为 `php::Int` 类型。将无法用于非整数的赋值，丢失溢出检测等能力。

```php
$a = 10; // $a 的类型为 Int
$b = $a / 3;  // $b 的值为 3 ，类型为整型

$a = any(10); // $a 的类型为 Var
$b = $a / 3;  // $b 的值为 3.33333...，类型为浮点型
```


## objval($value, $className)

从 `mixed` / `any` 类型变量重建带类信息的对象类型，是

`$var->toObject(ClassName::class)` 的函数式写法。与 `any()` 类似，`objval()` 在 AOT 编译器中被编译期替换为表达式本身，零运行时开销。

```php
$obj = $array['object'];
$typed = objval($obj, App\Hello\Test::class);
$typed->foo();  // 编译器可生成 Native Call
```

`objval()` 第二个参数仅支持字符串字面量或 `ClassName::class` 常量。

```php
// ✅ 正确用法
$obj = objval($var, TestObjval::class);
$obj = objval($var, 'TestObjval');

// ❌ 第二个参数不能是变量
$obj = objval($var, $someClass);
```

## refval($variable)
在动态调用中将值传递修改为引用传递。**`refval()` 仅接受变量（Variable），不能传入表达式**。例如下面的代码：

```php
eval('function retval_test(&$name) { $name .= "refval test"; }');

$name = 'php ';
retval_test(refval($name));
```

`eval()` 是一个运行时执行指令的函数，它动态生成了一个`retval_test`函数。由于在静态编译阶段，根本不存在`retval_test`函数，因此编译器无法将它的参数识别为引用传递，这时就需要`refval()`函数显式地将`$name`转为引用传递。

以下用法是**错误**的：

```php
// ❌ refval() 不能传入表达式
retval_test(refval("literal string"));
retval_test(refval($a + $b));
retval_test(refval(foo()));
```

而下面的代码是不需要添加`refval()`的：
```php
class Request {
    public $data;
}

function main()
{
    $req = new Request;
    $req->data = ['get' => [],];

    parse_str("hello=world", $req->data['get']);
    var_dump($req->data['get']['hello']);
}
```

`parse_str()`是一个内置函数，在编译期就可以得到它的参数信息，第二个参数是引用类型，因此编译器会自动将参数修改为引用传递，而不需要额外添加`refval()`函数。


## toStream() 关键词方法

`toStream()` 是编译器内置的关键词方法，用于将变量重建为 Stream 类型。从数组中读取元素时编译器无法追踪其具体类型，调用 `toStream()` 可以重建 Stream 类型，从而支持链式调用 `write()`、`read()`、`close()` 等 Stream 方法。

此方法通常用于 `proc_open()` 或 `stream_socket_pair()` 返回的管道数组。

```php
// stream_socket_pair 返回两个 stream 元素组成的数组
$sockets = stream_socket_pair(
    STREAM_PF_UNIX,
    STREAM_SOCK_STREAM,
    0
);

// 编译器无法追踪数组元素的类型，使用 toStream() 重建
$client = $sockets[0]->toStream();
$server = $sockets[1]->toStream();

$client->write("hello");
echo $server->read(5);   // "hello"

$client->close();
$server->close();
```

`toStream()` 在编译期被直接替换为 `php::toStream()` 调用，无任何运行时开销。

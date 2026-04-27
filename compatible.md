## 语法兼容性

`AOT`编译器支持绝大部分`PHP`的语法。不过由于`AOT`是静态编译，某些依赖运行时确定的特性是无法支持的：

1. 不支持 `$$` 语法，局部变量为编译器符号，无法在运行时使用
2. 不支持 `extract` 函数，无法运行时创建局部变量
3. 不支持 `yield`/`generator` 生成器语法，建议使用`fiber/swoole/swow`协程，`AOT`编译器支持协程
4. 不支持多层 `break` 或者 `continue` 语法，需要改成 `goto` 或 `try/catch`，这个特性只能在虚拟机模式中实现
5. 禁止字面量字符串包含`\0`，例如`$a = "hello \0 world;"`，与`C++`不兼容
6. 不支持参数数量不匹配的函数调用，例如某个函数的参数是`3`个，但是实际运行的代码传入了`4`个，这在`PHP`动态执行阶段是允许的，但是`AOT`编译器无法支持
7. 不支持 `Property Hook` 语法
8. 不支持动态调用中使用引用，例如`Closure`闭包函数的参数是引用类型，在运行时才能确定，在`AOT`编译器中不支持，需要显式使用`refval()`函数转为引用

```php
// 运行时才能得到函数的参数和返回值
$fn = getClosure();
// 编译器无法确定参数应该使用值还是引用，默认使用值传递
$fn($a, $b, $c);
// $c 将显式地使用引用传递，而不是值
$fn($a, $b, refval($c)); 
```

## 类型不可变性
`AOT`编译器要求不得转换变量类型。例如一个变量声明为`Object`，则不允许作为字符串或数组来使用。这与 `PHP` 截然不同。

```php
$str = "hello world";
$str = new StringObject("hello");
```

无法将`string`类型的变量，赋值为对象。若文件使用了严格类型，`declare(strict_types=1)` 上述代码将出现编译错误。

```bash
Fatal error: Cannot re-assign variable from `php::Object` to `php::Str`
```

若未声明严格类型，则自动转为字符串。相当于以下代码：

```php
$str = "hello world";
$obj = new StringObject("hello");
$str = strval($obj);
```

这里的行为与`ZendPHP`完全不同，在`ZendPHP`中`$str`变量会从字符串类型转为对象。

除了字符串之外，对象的类型也是不可变的。

```php
$o = new stdClass;
$o = new ArrayObject;
```

变量`$o`被声明为了`stdClass`类型，而在运行过程中又被转为`ArrayObject`，在`AOT`编译器中是不被允许的。将抛出下列编译错误：

```bash
Fatal error: Cannot re-assign typed object `$o` from `TestObject` to `stdClass`
```
`AOT`编译器除了常规的`PHP`语法之外添加了一些专有的特性。非`AOT`编译器可以使用下列的`polyfills`垫片函数实现兼容。

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

### 垫片函数
无。在 `ZendPHP` 下无效果。

## objvar($obj, $class)
将一个变量声明为某个类的对象。此函数只是检查表达式是否为对象类型，并且是`class`的实例。
作用：从数组中读取元素，会发生类型丢失，使用此函数可以重建类型。

```php
$obj = objvar($array['object'], App\Hello\Test::class);
$obj->foo();
```
 
编译器可以重新得到 `$obj` 对象的类型。这样就有助于编译器将对`$obj`的方法调用转为`Native Call`而不是`zend_call_function()`的动态调用，性能可以得到大幅提升。

### 垫片函数
```php
function objval(mixed $obj, string $class): object
{
    if ($obj instanceof $class) {
        return $obj;
    }
    throw new Exception('Invalid object type');
}
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

### 垫片函数
```php
function any(mixed $var): mixed
{
    return $var;
}
```

## refval($value)
在动态调用中将值传递修改为引用传递。例如下面的代码：

```php
eval('function retval_test(&$name) { $name .= "refval test"; }');

$name = 'php ';
retval_test(refval($name));
```

`eval()` 是一个运行时执行指令的函数，它动态生成了一个`retval_test`函数。由于在静态编译阶段，根本不存在`retval_test`函数，因此编译器无法将它的参数识别为引用传递，这时就需要`refval()`函数显式地将`$name`转为引用传递。

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

### 垫片函数
```php
function &refval(&$var)
{
    return $var;
}
```

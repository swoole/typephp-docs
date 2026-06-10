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
9. 所有 `.php` 文件必须使用 `UTF-8` 编码，其他编码（如 `GBK`、`Shift_JIS`、`ISO-8859-1`）不允许

```php
// 运行时才能得到函数的参数和返回值
$fn = getClosure();
// 编译器无法确定参数应该使用值还是引用，默认使用值传递
$fn($a, $b, $c);
// $c 将显式地使用引用传递，而不是值
$fn($a, $b, refval($c)); 
```

## 不支持游离代码
编译器要求所有代码必须在`function`内，不得存在游离代码，不支持内嵌  `HTML`，也就是`PHP`模版文件。这与`PHP`、`JavaScript`等脚本语言完全不同，而是与`C++`、`Java`、`Golang`、`Rust`一致。

因此模版文件、配置文件不支持编译，需使用`include/require`动态加载，在`ZendPHP`中动态执行。

## 类型不可变性
`AOT`编译器要求不得转换变量类型。例如一个变量声明为`Object`，则不允许作为字符串或数组来使用。这与 `PHP` 截然不同。

```php
$str = "hello world";
$str = new StringObject("hello");
```

无法将`string`类型的变量，赋值为对象。

```bash
Fatal error: Cannot re-assign variable from `php::Object` to `php::Str`
```
这里的行为与`ZendPHP`完全不同，在`ZendPHP`中`$str`变量会从字符串类型转为对象。

> `ZendPHP`在底层设计上是`any`类型的，语言层面不会存储变量的类型

除了字符串之外，对象的类型也是不可变的。

```php
$o = new stdClass;
$o = new ArrayObject;
```

变量`$o`被声明为了`stdClass`类型，而在运行过程中又被转为`ArrayObject`，在`AOT`编译器中是不被允许的。将抛出下列编译错误：

```bash
Fatal error: Cannot re-assign typed object `$o` from `TestObject` to `stdClass`
```

对象属性同样遵循更严格的静态类型规则，但不同属性类型的 `unset`/`null` 语义不同。

### 固定值类型属性

`int`、`float`、`bool`、`string`、`array` 这 5 种属性会被 `AOT` 编译器当作固定值类型优化。属性槽位始终保持声明类型，不允许通过 `unset()` 或赋值 `null` 将其改成未初始化或空值状态。

```php
class User {
    public int $id = 0;
    public array $roles = [];
}

$user = new User();
unset($user->id);   // AOT 不允许依赖 PHP 的属性 unset 语义
$user->roles = null; // AOT 不允许把 array 属性改成 null
```

在 `ZendPHP` 中，`unset($obj->prop)` 可以让属性进入未初始化状态；在 `AOT` 编译器中，这等价于改变固定值类型属性的类型，因此不允许。若业务上需要空值语义，应显式声明为可空类型，例如 `public ?int $id = null;`。

### 具体对象类型属性

具体类对象属性可以被 `unset()`，也可以设置为 `null`：

```php
class Profile {}

class User {
    public Profile $profile;
}

$user = new User();
$user->profile = new Profile();
$user->profile = null;  // 允许
unset($user->profile);  // 允许
```

但是非空对象赋值时，`AOT` 要求对象的实际类型与属性声明的类完全一致。与 `ZendPHP` 不同，不能将子类对象赋值给基类属性。

```php
class Base {}
class Child extends Base {}

class Holder {
    public Base $object;
}

$holder = new Holder();
$holder->object = new Base();  // 允许
$holder->object = new Child(); // AOT 不允许，必须精确匹配 Base
```

这类代码在 `ZendPHP` 中是合法的，因为 PHP 的对象类型检查允许子类兼容父类；但 `AOT` 编译器为了保持对象属性布局和方法调用优化的确定性，要求具体对象属性的非空赋值必须使用声明类本身。

## 严格模式
`AOT` 编译器不允许手动设置当前文件为非严格模式：`declare(strict_types=0)`，这会导致编译错误：

```bash
Fatal error: declare(strict_types=0) is not allowed, only strict_types=1 is supported
```

## 变量作用域
由于`PHP`的设计所有局部变量`Local Var`的作用域是`function`级别的，所以即使在`if/else/for/while`代码块中声明的变量也会被当做`function`内的顶层局部变量来处理。这与`ZendPHP`行为是一致的。

```php
function foo() {
    if ($cond) {
        $a = [];
    }
    // 这是不被允许的
    // $a 虽然是在 if 语句中声明，但实际的作用域是整个 function
    $a = "str";
}
```



## 未定义变量
在`ZendPHP`可以使用`isset()`判断变量是否存在。`AOT`编译器不支持这种写法。局部变量必须是先定义再使用。因此下面的代码是不被允许的。

```php
// 变量没有定义，isset 返回 false
if (!isset($var)) {
    // stmts
}
```

必须修改为：
```php
$var = null;
if (!isset($var)) {
    // stmts
}
```
`isset($var)`表达式在静态编译时将一直是`true`。

若使用未定义变量，在`ZendPHP`中仅会抛出一条`Warnning`警告，但编译器会直接报错，不允许使用未定义的变量。

```php
function main()
{
    var_dump($testVar);
}
```

```bash
Fatal error: Undefined variable `$testVar` in undef-var.php:4
```

在 `ZendPHP` 中的执行结果如下：
```bash
Warning: Undefined variable $testVar in undef-var.php on line 4
NULL
```

## 注解语法
`AOT`编译器支持注解语法，但由于`ZendVM`自身的限制，不支持非空数组类型的注解参数。

```php
#[MyAttribute]
#[MyAttribute(1234)]
#[MyAttribute(value: 1234)]
#[MyAttribute(MyAttribute::VALUE)]
#[MyAttribute([])]
#[MyAttribute(100 + 200)]
class Thing
{
}
```

下面的注解语法暂时无法支持：
```php
#[MyAttribute([1, 2, 3, 'str', bool])]
class Thing
{
}
```

## Trait 中 self 的处理

在 `ZendPHP` 中，`trait` 的 `self::` 在编译时绑定到使用该 `trait` 的类（defining class）。而在 `AOT` 编译器中，`trait` 方法的 `self::` 会被当作 `static::`（延迟静态绑定）来处理，实际运行时会在被调用的类（called class）上解析。

这一差异会导致 `private` 可见性的常量或方法在子类中无法访问。因为 `private` 成员不会被子类继承，当 `static::` 在子类实例上解析时，无法找到父类中 `private` 的常量或方法。

**示例：**

```php
trait THello {
    private const array CONST_ARRAY = [
        'test_fn_1' => ['toInt' => 123],
    ];

    public function run(string $key1) {
        var_dump(self::CONST_ARRAY[$key1]);
    }
}

class TraitsTest {
    use THello;
}

class Test extends TraitsTest {
    function hello2() {
        $this->run('test_fn_1');  // self:: 被当作 static:: 处理
                                  // CONST_ARRAY 是 private，Test 未继承
                                  // 导致常量读取失败
    }
}
```

**解决方案：** 将 `trait` 中需要被子类继承访问的常量或方法的可见性从 `private` 修改为 `protected` 或 `public`。

```php
trait THello {
    protected const array CONST_ARRAY = [ /* ... */ ];
}
```

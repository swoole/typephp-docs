## 兼容性分类

TypePHP 与 ZendPHP 的差异分为三类：

- **TypePHP 设计规则**：为了静态类型、固定布局和可预测性能而有意采用的语义，不视为缺陷。
- **部分支持**：主要场景可用，但动态边界或少数语义与 ZendPHP 不完全一致。
- **暂未支持**：技术上可以实现，但当前编译器尚未提供。

## 不支持的语法

`TypePHP` 编译器支持绝大部分 `PHP` 语法。由于 `TypePHP` 是静态编译语言，以下依赖`ZendVM`动态处理的语法不支持：

1. 不支持 `$$` 语法，局部变量为编译器符号，无法在运行时使用
2. 不支持 `extract` 函数，无法运行时创建局部变量
3. 不支持参数数量与声明不一致的函数调用，不能使用 `func_get_args()` 函数隐式接受额外参数，需要显式声明为变长参数
4. 不支持对 `Property Hook` 属性取引用
5. 不支持动态调用中自动推断参数为引用，需要显式使用 `refval()` 函数将调用参数转为引用
6. 不支持闭包和箭头函数的引用参数及按引用返回
7. 不支持引用类型的变长参数，例如 `function foo(&...$args) {}`

## 新特性支持

- `PHP 8.4` 新增的 `Property Hooks`
- `PHP 8.5` 新增的 `Pipe Operator`

## 不支持游离代码
编译器要求所有代码必须在`function`内，不得存在游离代码，不支持内嵌  `HTML`，也就是`PHP`模版文件。这与`PHP`、`JavaScript`等脚本语言完全不同，而是与`C++`、`Java`、`Golang`、`Rust`一致。

因此模版文件、配置文件不支持编译，需使用`include/require`动态加载，在`ZendPHP`中动态执行。

## 源文件编码格式
所有 `.php` 文件必须使用 `UTF-8` 编码，其他编码（如 `GBK`、`Shift_JIS`、`ISO-8859-1`）不允许。

## 类型不可变性
TypePHP 编译器要求不得转换变量类型。例如一个变量声明为`Object`，则不允许作为字符串或数组来使用。这与 `PHP` 截然不同。

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

变量`$o`被声明为了`stdClass`类型，而在运行过程中又被转为`ArrayObject`，在TypePHP 编译器中是不被允许的。将抛出下列编译错误：

```bash
Fatal error: Cannot re-assign typed object `$o` from `TestObject` to `stdClass`
```

对象属性同样遵循更严格的静态类型规则，但不同属性类型的 `unset`/`null` 语义不同。

### 固定值类型属性

`int`、`float`、`bool`、`string`、`array` 这 5 种属性会被 TypePHP 编译器当作固定值类型优化。属性槽位始终保持声明类型，不允许通过 `unset()` 或赋值 `null` 将其改成未初始化或空值状态。

```php
class User {
    public int $id = 0;
    public array $roles = [];
}

$user = new User();
unset($user->id);   // TypePHP 不允许依赖 PHP 的属性 unset 语义
$user->roles = null; // TypePHP 不允许把 array 属性改成 null
```

在 `ZendPHP` 中，`unset($obj->prop)` 可以让属性进入未初始化状态；在 TypePHP 编译器中，这等价于改变固定值类型属性的类型，因此不允许。若业务上需要空值语义，应显式声明为可空类型，例如 `public ?int $id = null;`。

固定值类型属性使用堆内固定槽位。未显式初始化时，TypePHP 使用对应类型的零值，例如 `int` 为 `0`、`bool` 为 `false`。因此未初始化 typed property 及 `??` 的行为不保证与 ZendPHP 的 uninitialized 状态一致；这是 TypePHP 的固定布局语义。

### 具体对象类型属性

具体类对象属性可以被 `unset()`，但 **不可赋值为 `null`**，除非属性声明为可空类型（`?Profile`）：

```php
class Profile {}

class User {
    public Profile $profile;
}

$user = new User();
$user->profile = new Profile();
$user->profile = null;  // 不允许，编译错误：Cannot assign null
unset($user->profile);  // 允许
```

需要允许 `null` 时，应显式声明为可空类型并给定默认值：

```php
class User {
    public ?Profile $profile = null;
}

$user = new User();
$user->profile = null;  // 允许
```

非空对象赋值时，`AOT` 允许符合 `is-a` 关系的类型转换——子类对象可以赋值给基类属性，与 `ZendPHP` 行为一致。

```php
class Base {}
class Child extends Base {}
class Other {}

class Holder {
    public Base $object;
}

$holder = new Holder();
$holder->object = new Base();  // 允许
$holder->object = new Child(); // 允许，Child is-a Base
$holder->object = new Other(); // 不允许，Other is-not-a Profile
```

但反向赋值是不允许的——不能将基类对象赋值给声明为子类的属性，因为基类实例并不满足子类的 `is-a` 关系。

```php
class Holder {
    public Child $object;
}

$holder = new Holder();
$holder->object = new Child(); // 允许
$holder->object = new Base();  // 不允许，Base is-not-a Child
```

### 父子类不能声明同名 private 属性

对于 `public` 和 `protected` 属性，TypePHP 沿用 PHP 的继承规则：子类同名声明描述的是同一个继承 property slot，必须满足类型、可见性和 `readonly` 的兼容性要求。

但 TypePHP 不支持子类用同名 `private` 属性隐藏父类 `private` 属性。ZendPHP 会为这种写法创建两个独立属性槽位；为保持属性解析、clone 和 typed property 语义清晰，编译器会在编译阶段拒绝它。

```php
class ParentBox {
    private int $id = 1;
}

class ChildBox extends ParentBox {
    private string $id = 'child'; // 不允许：会隐藏 ParentBox::$id
}
```

请改用不同的属性名；如果子类需要复用父类状态，应将父类属性设计为 `protected`，或通过父类提供的方法访问。

## 在动态调用中使用引用

`TypePHP`无法在编译阶段确定动态调用的参数类型，因此无法自动推断参数是否为引用。原生调用或内置函数调用，可以自动推断参数类型，转为引用，无需用户显式指定。
例如 `Closure` 闭包函数的参数是引用类型，在运行时才能确定，在 `TypePHP` 编译器中无法自动判断，需要显式使用 `refval()` 函数或等价关键词方法 `toRef()` 转为引用

```php
// 运行时才能得到函数的参数和返回值
$fn = getClosure();
// 编译器无法确定参数应该使用值还是引用，默认使用值传递
$fn($a, $b, $c);
// $c 将显式地使用引用传递，而不是值
$fn($a, $b, refval($c));
// 等价写法：toRef() 是 TypePHP 专有关键词方法
$fn($a, $b, $c->toRef());
```

闭包和箭头函数目前也不能声明引用参数或按引用返回。此限制与 `use (&$value)` 引用捕获不同；引用捕获已经支持。

## 保留关键词方法

`toInt()`、`toString()`、`toArray()` 等是 TypePHP 保留关键词方法，其优先级高于普通对象方法解析。对象上的零参数 `toArray()` 可由转换 helper 调用；不要定义需要参数的同名对象方法，因为调用参数不会按普通对象方法语义处理。需要业务序列化方法时应使用 `serializeNode()` 等非保留名称。

## 数组 null 键

在 `ZendPHP` 中，`$array[null] = 1` 等价于 `$array[''] = 1`，`null` 会被隐式转换为空字符串 `""` 作为键名。但在 TypePHP 编译器中，`$array[null]` 被当作 `$array[]` 追加操作处理，这与 `PHP` 行为不兼容。

| 写法                 | ZendPHP 行为                    | TypePHP 编译器行为   |
|--------------------|-------------------------------|-----------------|
| `$array[] = 1`     | 追加元素                          | 追加元素            |
| `$array[null] = 1` | `$array[''] = 1`（null 转为空字符串） | 追加元素（与 PHP 不兼容） |
| `$array[''] = 1`   | 空字符串键                         | 空字符串键           |

因此，如果需要显式使用空字符串作为键，应直接使用 `$array['']` 而非 `$array[null]`。

## 严格模式
TypePHP 编译器不允许手动设置当前文件为非严格模式：`declare(strict_types=0)`，这会导致编译错误：

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
在`ZendPHP`可以使用`isset()`判断变量是否存在。TypePHP 编译器不支持这种写法。局部变量必须是先定义再使用。因此下面的代码是不被允许的。

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
TypePHP 编译器支持注解语法，但当前不支持非空数组和 `new` 表达式作为注解参数。

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

在 `ZendPHP` 中，`trait` 的 `self::` 在编译时绑定到使用该 `trait` 的类（defining class）。而在 TypePHP 编译器中，`trait` 方法的 `self::` 会被当作 `static::`（延迟静态绑定）来处理，实际运行时会在被调用的类（called class）上解析。

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

## 析构方法中抛出异常

在 `ZendPHP` 中，`__destruct()` 析构方法抛出异常会被引擎捕获并忽略（`zend_objects_store_del` 中调用 `zend_try/catch`），不会造成资源泄漏。

但在 `AOT` 编译模式下，对象销毁路径经过 `phpx` 的 `Variant::unset()` / `~Variant()` / `destroy()` 等 C++ RAII 机制。析构方法中抛出异常会中断 Zend Engine 的 `zend_objects_store_del` 两阶段清理流程，导致部分内存未被释放（`free_obj` 和 GC 缓冲区移除被跳过）。

编译器在编译时会检测到析构方法中抛出异常，并输出警告：

```bash
Warning: Throwing exception in MyClass::__destruct() may cause memory leak
```

**影响范围**：仅在析构方法执行抛出的异常会导致内存泄漏。程序不会崩溃或产生未定义行为，仅会造成轻微的内存泄漏（每次触发约泄漏对象结构体和 GC 缓冲区槽位）。

**建议**：

- 避免在 `__destruct()` 中执行可能抛出异常的操作
- 如需在析构时执行可能失败的操作，使用 `try/catch` 在方法内部捕获并处理异常
- 析构方法应仅用于释放资源，不应包含业务逻辑

```php
// 不推荐：析构方法中抛出异常
class MyClass {
    function __destruct() {
        throw new \Exception("error in destructor"); // 编译警告 + 运行时内存泄漏
    }
}

// 推荐：在内部捕获并处理
class MyClass {
    function __destruct() {
        try {
            // 可能失败的操作
        } catch (\Throwable $e) {
            error_log("destructor error: " . $e->getMessage());
        }
    }
}
```

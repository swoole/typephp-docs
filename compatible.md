## 不支持的语法

`TypePHP` 编译器支持绝大部分 `PHP` 语法。由于 `TypePHP` 是静态编译语言，以下依赖`ZendVM`动态处理的语法不支持：

1. 不支持 `$$` 语法，局部变量为编译器符号，无法在运行时使用
2. 不支持 `extract` 函数，无法运行时创建局部变量
3. 不支持参数数量与声明不一致的函数调用，不能使用 `func_get_args()` 函数隐式接受额外参数，需要显式声明为变长参数
4. 不支持对 `Property Hook` 属性取引用
5. 不支持动态调用中自动推断参数为引用，需要显式使用 `refval()` 函数将调用参数转为引用
6. 不支持闭包和箭头函数的引用参数及按引用返回
7. 不支持引用类型的变长参数，例如 `function foo(&...$args) {}`
8. 动态代码不能 `use` 在 静态编译的 `Trait`，`Trait` 在编译期与 `Class` 组合，不支持运行时动态绑定
9. 不支持 `Closure::call()/bind()/bindTo()`，`Closure` 的 `$this` 与类作用域在编译期确定，不允许在运行时重新绑定

## 已支持的新语法

- 已支持 `PHP 8.4` 新增的 `Property Hooks`
- 已支持 `PHP 8.5` 新增的 `Pipe Operator`

`TypePHP`对新语法的支持，并不依赖`ZendVM`，而是由编译器实现的零成本抽象。

## 不支持游离代码
编译器要求所有代码必须在`function`内，不得存在游离代码，不支持内嵌  `HTML`，也就是`PHP`模版文件。这与`PHP`、`JavaScript`等脚本语言完全不同，而是与`C++`、`Java`、`Golang`、`Rust`一致。

因此模版文件、配置文件不支持编译，需使用`include/require`动态加载，在`ZendPHP`中动态执行。

## 源文件编码格式
所有 `.php` 文件必须使用 `UTF-8` 编码，其他编码（如 `GBK`、`Shift_JIS`、`ISO-8859-1`）不允许。

## 类型不可变性
`TypePHP` 编译器要求不得转换变量类型。例如一个变量声明为`Object`，则不允许作为字符串或数组来使用。这与 `PHP` 截然不同。

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

## 不支持闭包重绑定

TypePHP 支持闭包、箭头函数、`use ($value)` 值捕获和 `use (&$value)` 引用捕获，但不支持以下依赖运行时重绑定的 API：

- `Closure::call()`；
- `Closure::bind()`；
- `Closure::bindTo()`。

```php
class Target
{
    private string $value = 'hidden';
}

$target = new Target();
$reader = function (): string {
    return $this->value;
};

$reader->call($target);          // 不支持
$reader->bindTo($target);        // 不支持
Closure::bind($reader, $target); // 不支持
```

TypePHP 会在能够静态确认接收者为闭包时直接报告编译错误。闭包的捕获变量、词法类作用域和 `$this` 均由 AOT 编译器确定；运行时替换 `$this` 或类作用域会破坏这一静态模型，因此不会模拟 ZendVM 的重绑定行为。

需要访问目标对象时，应将对象作为普通参数显式传入，并通过其公开 API 操作：

```php
class Target
{
    public function value(): string
    {
        return 'visible';
    }
}

$reader = static function (Target $target): string {
    return $target->value();
};

echo $reader(new Target());
```

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

## Attribute 参数

TypePHP 支持 PHP Attribute 语法，包括非空数组、嵌套数组、常量表达式和 `new` 表达式参数。

```php
#[MyAttribute]
#[MyAttribute(1234)]
#[MyAttribute(value: 1234)]
#[MyAttribute(MyAttribute::VALUE)]
#[MyAttribute([])]
#[MyAttribute([1, 2, 3, 'str', true])]
#[MyAttribute(new AttributeOptions(enabled: true))]
#[MyAttribute(100 + 200)]
class Thing
{
}
```

Attribute 参数仍然不能包含普通函数调用：

```php
#[MyAttribute(loadOptions())] // 编译错误
class Thing
{
}
```

### C 扩展兼容性

TypePHP 保证 PHP 用户态的 `ReflectionAttribute::getArguments()`、`ReflectionAttribute::newInstance()` 和 `ReflectionAttribute::__toString()` 可以读取包含非空数组、`new`、闭包等请求期值的 Attribute 参数。

但是，第三方 C 扩展直接使用 Zend Attribute 数据结构或底层 C API 读取参数的方式不受支持，例如：

- 直接访问 `zend_attribute`、`zend_attribute_arg.value`；
- 直接调用 `zend_get_attribute_value()`；
- 替换 `ReflectionAttribute::getArguments()` 或 `ReflectionAttribute::newInstance()` 的 internal function handler。

TypePHP 会在 Attribute 的持久化参数中保存请求期工厂标记，并在 PHP Reflection 方法执行时将其转换为普通请求期 `zval`。上述 C 扩展路径可能绕过该转换，读取到内部标记；如果扩展同时替换 ReflectionAttribute handler，还可能覆盖 TypePHP 的 handler 或被 TypePHP 覆盖。因此，即使只包含简单参数的 Attribute 当前看似可用，也不要依赖这种未受支持的调用方式。

已知示例是 [`symfony/php-ext-deepclone`](https://github.com/symfony/php-ext-deepclone)：该扩展包含直接介入 ReflectionAttribute 内部处理流程的实现，不能保证与 TypePHP 的请求期 Attribute 参数工厂兼容。

需要兼容 TypePHP 时，C 扩展应通过 PHP 用户态 ReflectionAttribute 方法取得参数，或者由扩展自行提供明确适配 TypePHP 的桥接实现。

类常量和属性默认值等其他位置具有不同的 `new` 表达式限制，完整规则参见[常量表达式](constant-expressions.md)。

## Trait 仅在编译期可见

TypePHP 将 Trait 作为编译期 AST 模板处理。在 `convert` 阶段，Trait 的属性、常量和方法会被注入使用它的类，并像该类直接声明的成员一样编译为原生代码。Trait 本身不会注册为 ZendVM 中的运行期 Trait。

因此存在以下限制：

- `trait_exists()` 查询 TypePHP 定义的 Trait 时返回 `false`；
- 由 `include`、`require` 或 `eval()` 加载的动态 PHP 代码不能 `use` TypePHP 定义的 Trait；
- 不能通过 Reflection 在运行期取得 TypePHP Trait 实体。

以下写法受支持，因为 Trait 与使用它的类都在 AOT 编译期可见：

```php
trait HasName
{
    public function name(): string
    {
        return self::class;
    }
}

class User
{
    use HasName;
}
```

以下动态代码不受支持：

```php
trait HasName
{
    public function name(): string
    {
        return self::class;
    }
}

function main(): void
{
    eval(<<<'PHP'
        class DynamicUser {
            use HasName; // 运行期找不到 Trait "HasName"
        }
        PHP);
}
```

如果类需要使用 TypePHP Trait，应将该类也纳入 TypePHP 项目，由编译器在 AOT 阶段完成 Trait 组合。必须动态加载的类则应直接声明所需成员，或继承一个已经静态编译并组合了 Trait 的普通 TypePHP 类。

Trait 方法中的 `self`、`parent` 和 `static` 按组合后的类上下文编译；`__TRAIT__` 仍保留原始 Trait 名称，可用于识别方法来源。

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

## 协程入口必须捕获异常

使用 Swoole 或 Swow 协程时，**必须在每个协程入口的回调内部使用 `try/catch` 捕获 `\Throwable`**。异常不能穿透至其他协程，也不能依赖协程创建调用外围的 `try/catch` 处理。

这项要求适用于所有可能进入新协程或独立事件回调的入口，包括：

- `Swoole\Coroutine\run()`、`Swoole\Coroutine::create()` 和 `go()`；
- Swoole Server 的 `onRequest`、`onMessage`、`onTask` 等事件回调；
- Timer、Event、Process 以及其他由扩展调度的回调。

正确写法是在协程回调开始执行时立即建立异常边界：

```php
\Swoole\Coroutine\run(static function (): void {
    try {
        runApplication();
    } catch (\Throwable $exception) {
        error_log((string) $exception);
    }
});
```

不能只在创建协程的代码外层捕获：

```php
// 错误：外层 catch 不能捕获从协程回调中穿出的异常
try {
    \Swoole\Coroutine\run(static function (): void {
        runApplication();
    });
} catch (\Throwable $exception) {
    error_log((string) $exception);
}
```

未在协程入口内部捕获的异常可能越过 Zend 回调边界，使进程以类似以下信息中止：

```text
terminate called after throwing an instance of '_zend_object*'
```

必须捕获 `\Throwable`，不能只捕获 `\Exception`，因为 PHP `Error` 同样需要在当前协程内部处理。捕获后应记录完整异常信息，并由当前协程明确决定结束本次任务、关闭连接或终止 Worker；不得将异常重新抛向协程边界之外。

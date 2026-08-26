# 常量表达式

PHP 要求 Attribute 参数、常量值和各种默认值使用常量表达式。TypePHP 在编译阶段按照目标 PHP 版本检查这些表达式，非法表达式会直接产生编译错误。

这里的“常量表达式”并不等于“只能写字面量”。它可以包含数组、常量引用和部分运算；在特定位置还可以包含 `new` 表达式。

## 常用写法

以下写法可以用于常量表达式：

```php
const DEFAULT_LIMIT = 100;

class Config
{
    public const int LIMIT = DEFAULT_LIMIT * 2;

    public array $options = [
        'enabled' => true,
        'limit' => self::LIMIT,
    ];
}
```

常用的可用表达式包括：

- `null`、`true`、`false`、整数、浮点数和字符串字面量；
- 普通常量和类常量，例如 `DEFAULT_LIMIT`、`Config::LIMIT`；
- `ClassName::class` 和 PHP 魔术常量；
- 数组以及数组展开；
- 算术、比较、位运算、逻辑运算、字符串拼接、三元表达式和 `??`；
- 对常量数组进行下标读取；
- PHP 允许的枚举 case 及其属性读取。

普通函数或方法调用不是常量表达式：

```php
class Config
{
    // 编译错误：普通函数调用不能用于类常量
    public const int LIMIT = loadLimit();

    // 编译错误：普通函数调用不能用于属性默认值
    public int $timeout = loadTimeout();
}
```

如果需要通过函数计算属性值，应在构造方法或普通方法中完成赋值。

## 哪些位置允许 new

PHP 将常量表达式分为普通常量表达式和允许动态值的常量表达式。TypePHP 使用相同规则：

| 使用位置 | 是否允许 `new` |
|---|---:|
| Attribute 参数 | 允许 |
| 函数或方法的参数默认值 | 允许 |
| 全局 `const` | 允许 |
| 静态局部变量默认值 | 允许 |
| 类、接口或 Trait 常量 | 不允许 |
| 对象属性默认值 | 不允许 |
| Enum case 值 | 不允许 |

构造器提升属性是一个容易混淆的例外。它的默认值属于构造器参数默认值，而不是属性默认值，因此允许使用 `new`：

```php
class Service
{
    public function __construct(
        public Client $client = new Client(),
    ) {}
}
```

例如 Attribute 和参数默认值可以直接创建对象：

```php
#[Attribute(Attribute::TARGET_METHOD)]
class Route
{
    public function __construct(
        public string $path,
        public ?RouteOptions $options = null,
    ) {}
}

#[Route('/users', new RouteOptions(cache: true))]
function listUsers(Client $client = new Client()): array
{
    return [];
}
```

但相同表达式不能作为类常量或属性默认值：

```php
class Service
{
    public const CLIENT = new Client(); // 编译错误

    public Client $client = new Client(); // 编译错误
}
```

属性需要对象默认值时，应使用构造方法：

```php
class Service
{
    public Client $client;

    public function __construct()
    {
        $this->client = new Client();
    }
}
```

## Attribute 参数

普通运行时 Attribute 支持非空数组、嵌套数组、常量和 `new` 表达式：

```php
#[Rule([
    'groups' => ['create', 'update'],
    'normalizer' => new NameNormalizer(trim: true),
])]
class User
{
}
```

这些值可以通过 PHP Reflection API 正常读取：

```php
$attribute = (new ReflectionClass(User::class))->getAttributes(Rule::class)[0];
$arguments = $attribute->getArguments();
$instance = $attribute->newInstance();
```

Attribute 参数仍必须满足 PHP 的常量表达式规则。下面的普通函数调用不允许使用：

```php
#[Rule(loadRules())] // 编译错误
class User
{
}
```

`new` 中不能使用匿名类、动态类名、`static` 类名或参数展开：

```php
#[Rule(new class {})]       // 不允许匿名类
#[Rule(new $className())]   // 不允许动态类名
#[Rule(new static())]       // 不允许 new static
#[Rule(new Rule(...$args))] // 不允许参数展开
class User
{
}
```

## PHP 8.5 表达式

选择 PHP 8.5 作为目标版本时，常量表达式还支持：

- 标量类型转换；
- `static` 闭包；
- 第一类 callable，例如 `strlen(...)`、`Factory::create(...)`。

```php
#[Callback(strlen(...))]
#[Fallback(static function (string $value): string {
    return $value;
})]
class Handler
{
}
```

常量表达式中的闭包必须是 `static`，并且不能通过 `use (...)` 捕获变量。第一类 callable 与普通调用不同：

```php
strlen(...)       // PHP 8.5 第一类 callable，可以使用
strlen('value')   // 普通函数调用，不能用于常量表达式
```

对象转换 `(object) [...]` 只允许出现在支持动态值的位置，例如 Attribute 参数；不能用于类常量或属性默认值。

PHP 8.5 允许把静态闭包和第一类 callable 作为类常量表达式，但 TypePHP 当前的类常量存储尚不支持这两类对象值。现阶段请将它们用于 Attribute 参数等请求期场景，不要用于类常量。

## 短路和条件表达式

TypePHP 与 PHP 一样会先进行常量折叠。确定不会执行的分支不参与常量表达式检查：

```php
#[Example(true || loadValue())]
#[Example(true ? 1 : loadValue())]
class Target
{
}
```

如果条件不能在编译期确定，所有可能执行的分支都必须是合法的常量表达式。

## 目标 PHP 版本

表达式规则以项目选择的目标 PHP 版本为准，而不是运行 TypePHP 编译器的 PHP 解释器版本。PHP 8.5 新增的静态闭包和第一类 callable 常量表达式，在较低目标版本中会产生编译错误。

项目可通过编译参数或项目配置选择 PHP 版本，具体用法参见[命令行参数](options.md)和[`project.yml` 配置](project-yml.md)。

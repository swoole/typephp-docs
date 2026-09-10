# 编译期函数

编译期函数是 TypePHP 编译器专有的语法入口。编译器在静态编译阶段识别这些函数，并直接生成对应的 C++ 代码；它们通常不会在运行时按普通 PHP 函数查找。

关键词方法是另一套语法体系，不列入本文的编译期函数清单，详见[关键词方法](keyword-method.md)。

## 函数清单

| 函数 | 作用 |
|------|------|
| `std::any([$value])` | 将表达式降级为 `mixed` / `any` / `php::Var`；省略参数时默认为 `null` |
| `std::object($value, ClassName::class)` | 检查对象并恢复具体类信息 |
| `std::ref($value)` | 显式按引用传递变量、数组元素或对象属性 |
| `std::expected($condition)` | 标记条件通常为真，帮助编译器优化常见分支 |
| `std::unexpected($condition)` | 标记条件通常为假，帮助编译器优化少见分支 |
| `std::int($value)` | 创建 native int 表达式 |
| `std::float($value)` | 创建 native float 表达式 |
| `std::bool($value)` | 创建 native bool 表达式 |
| `std::bigInt($value)` | 创建 BigInt 高精度整数 |
| `std::decimal($value)` | 创建 Decimal 高精度十进制数 |
| `std::bigFloat($value)` | 创建 BigFloat 高精度浮点数 |
| `std::array($type, $size)` | 创建定长 StdArray 容器 |
| `std::vector($type[, $size])` | 创建动态 StdVector 容器 |
| `std::map($keyType, $valueType)` | 创建哈希 StdMap 容器 |
| `std::orderedMap($keyType, $valueType)` | 创建有序 StdOrderedMap 容器 |

## 使用位置与名称写法

这些核心函数的用途不同，适用位置也不同：

| 函数 | 可使用的位置 | 说明 |
|------|--------------|------|
| `std::any()` | 任意普通值表达式位置 | 可用于赋值、函数参数、返回值、数组元素、运算式、条件表达式等。 |
| `std::object()` | 任意普通值表达式位置 | 检查运行时对象类型，并恢复编译器可识别的具体类信息。 |
| `std::ref()` | 仅限调用参数 | 用于把一个可引用的值传给需要引用的参数；它不是普通值转换函数。 |
| `std::expected()` | 布尔条件表达式 | 用于 `if`、`elseif`、循环或三元表达式等通常为真的条件。 |
| `std::unexpected()` | 布尔条件表达式 | 用于错误、越界、缓存未命中等通常为假的条件。 |

编译期函数是全局 `std` 类的静态方法。`std` 类名及其方法名遵循 PHP 规则，不区分大小写；`Type::*` 类型常量仍区分大小写。

## std::any([$value])

`std::any()` 将表达式的编译期类型降级为动态类型。降级后变量可以接收任意 PHP 值，但编译器也不再为它生成 native type、typed object 或 Native Call 优化。

参数可以省略。无参数调用 `std::any()` 会创建一个初始值为 `null` 的动态值，适合在实际值尚未确定时先建立动态存储。

`std::any()` 是普通值表达式，可以放在任何需要值的位置。例如：

```php
function identity(mixed $value): mixed {
    return std::any($value);
}

function main(): void {
    $values = [std::any(1), std::any("two")];
    var_dump(std::any(3) + 1);
    var_dump(identity(std::any($values[0])));
}
```

```php
function main(): void {
    $value = std::any(10);

    $value = "string";
    var_dump($value);
}
```

`std::any()` 常用于分支中可能产生不同类型的场景：

```php
class FileLogger {}
class NullLogger {}

function create_logger(bool $debug) {
    if ($debug) {
        $logger = std::any(new FileLogger());
    } else {
        $logger = std::any(new NullLogger());
    }

    return $logger;
}
```

它也可用于保留 PHP 动态运算语义：

```php
function main(): void {
    $a = std::any(10);
    $b = $a / 3;

    var_dump($b); // float(3.333333...)
}
```

## std::object($value, ClassName::class)

`std::object()` 是 `$value->toObject(ClassName::class)` 的函数形式。它会检查
`$value` 是否属于指定类，并为后续编译恢复具体类信息：

```php
$user = std::object($payload['user'], User::class);
echo $user->getName();
```

类名参数必须能在编译期解析，支持具体类名、类名字符串字面量、`self::class`
和 `parent::class`。子类对象可以通过父类或接口检查。

关键词方法语法是 TypePHP 独有的，而静态函数调用可以在 Zend PHP 执行路径中
提供兼容实现，从而让同一份业务代码同时运行：

```php
class std {
    public static function object(mixed $value, string $class): object {
        if (!$value instanceof $class) {
            throw new TypeError("Expected an instance of {$class}");
        }
        return $value;
    }
}
```

此后应用代码在 Zend PHP 和 TypePHP 中均可调用 `std::object()`。TypePHP 会将其
直接降低为带检查的对象转换，不产生运行时静态方法分派。

## std::ref($value)

`std::ref()` 用于显式传递引用。它主要用于动态调用、闭包调用、可变函数调用等编译期无法获知参数是否为引用的场景。

`std::ref()` 的参数必须是可作为引用的左值：变量、数组元素或对象属性。不能传入字面量、函数返回值、算术表达式等临时值。

`std::ref()` 只能作为一次函数或方法调用的参数使用：

```php
$callback(std::ref($value));
```

不要把它当作普通值表达式使用；例如以下写法不受支持：

```php
return std::ref($value);
$list = [std::ref($value)];
$sum = std::ref($value) + 1;
```

变量引用：

```php
function main(): void {
    $fn = function (&$name): void {
        $name .= " compiler";
    };

    $name = "php";
    $fn(std::ref($name));

    var_dump($name); // string(12) "php compiler"
}
```

数组元素引用：

```php
function main(): void {
    $fn = function (&$value): void {
        $value = "changed";
    };

    $data = ["name" => "origin"];
    $fn(std::ref($data["name"]));

    var_dump($data["name"]); // string(7) "changed"
}
```

对象属性引用：

```php
class Box {
    public string $value = "origin";
}

function main(): void {
    $fn = function (&$value): void {
        $value = "changed";
    };

    $box = new Box();
    $fn(std::ref($box->value));

    var_dump($box->value); // string(7) "changed"
}
```

静态函数和内置函数的参数信息在编译期明确时，编译器会自动处理引用参数，不需要额外使用 `std::ref()`：

```php
function main(): void {
    parse_str("hello=world", $result);
    var_dump($result["hello"]); // string(5) "world"
}
```

原生 `T&` 引用、局部别名、动态调用写回与引用逃逸限制详见[强类型引用](strong-references.md)。

## std::expected($condition) 和 std::unexpected($condition)

`std::expected()` 和 `std::unexpected()` 用于向编译器提供分支发生概率的提示：

- `std::expected($condition)` 表示该条件在大多数情况下为真。
- `std::unexpected($condition)` 表示该条件在大多数情况下为假。

例如，把正常处理路径标记为常见分支：

```php
function handle_request(bool $ready): int {
    if (std::expected($ready)) {
        // 绝大多数请求会进入这个分支
        return 1;
    }

    return 0;
}
```

把错误或其他少见情况标记为非常见分支：

```php
function normalize_id(int $id): int {
    if (std::unexpected($id < 0)) {
        return 0;
    }

    return $id;
}
```

也可以在循环条件中使用：

```php
function countdown(int $remaining): void {
    while (std::expected($remaining > 0)) {
        echo $remaining, "\n";
        $remaining--;
    }
}
```

两个函数都只接受一个非展开参数，并返回 `bool`。它们不会改变条件的真假结果、求值次数或业务逻辑，只提供优化提示。

分支预测提示应当基于实际运行情况使用。如果无法确定某个分支是否明显更常见，直接使用普通条件表达式即可：

```php
if ($condition) {
    // 不需要强行添加 std::expected() 或 std::unexpected()
}
```

## std::int($value)

`std::int()` 显式创建 native int 表达式。适合需要明确进入 native 整数运算路径的热点代码。

```php
function main(): void {
    $sum = std::int(0);

    for ($i = std::int(0); $i < 1000; $i++) {
        $sum += $i;
    }

    var_dump($sum);
}
```

从动态值转换：

```php
function main(): void {
    $value = std::any("123");
    $id = std::int($value);

    var_dump($id + 1);
}
```

## std::float($value)

`std::float()` 显式创建 native float 表达式。

```php
function main(): void {
    $x = std::float(1.5);
    $y = std::float(2);

    var_dump($x * $y);
}
```

从动态值转换：

```php
function main(): void {
    $value = std::any("3.14");
    $pi = std::float($value);

    var_dump($pi);
}
```

## std::bool($value)

`std::bool()` 显式创建 native bool 表达式。

```php
function main(): void {
    $enabled = std::bool(1);

    if ($enabled) {
        echo "enabled\n";
    }
}
```

从动态值转换：

```php
function main(): void {
    $value = std::any("");
    $ok = std::bool($value);

    var_dump($ok); // bool(false)
}
```

## std::bigInt($value)

`std::bigInt()` 创建 BigInt 高精度整数。参数可使用整数或字符串；超长整数建议使用字符串，避免 PHP 解析阶段丢失精度。

```php
function main(): void {
    $a = std::bigInt(100);
    $b = std::bigInt("999999999999999999999999999999");

    var_dump(($a + $b)->toString());
}
```

BigInt 不允许从 float 构造，应改用字符串或整数：

```php
function main(): void {
    $value = "3";
    $big = std::bigInt($value);

    var_dump($big->toString());
}
```

## std::decimal($value)

`std::decimal()` 创建 Decimal 高精度十进制数。金融、金额、精确小数计算建议优先使用字符串参数。

```php
function main(): void {
    $price = std::decimal("19.99");
    $tax = std::decimal("0.08");
    $total = $price + ($price * $tax);

    var_dump($total->toString());
}
```

浮点字面量可直接传入，编译器会尽量使用源码字面量文本构造 Decimal：

```php
function main(): void {
    $rate = std::decimal(0.125);

    var_dump($rate->toString());
}
```

不建议从 float 变量构造 Decimal，因为变量已经丢失源码字面量文本：

```php
function main(): void {
    $raw = "0.1";
    $value = std::decimal($raw);

    var_dump($value->toString());
}
```

## std::bigFloat($value)

`std::bigFloat()` 创建 BigFloat 高精度浮点数，适合科学计算或需要更大指数范围的场景。

```php
function main(): void {
    $pi = std::bigFloat("3.141592653589793238462643383279502884197");
    $radius = std::bigFloat(10);
    $area = $pi * $radius * $radius;

    var_dump($area->toString());
}
```

从整数或浮点值构造：

```php
function main(): void {
    $a = std::bigFloat(42);
    $b = std::bigFloat(1.5);

    var_dump(($a + $b)->toString());
}
```

## std::array($type, $size)

`std::array()` 创建定长 StdArray 容器。大小必须是整数字面量，容器只能在函数顶层作用域的变量首次赋值时创建，不能对已有变量重新赋值为新的 StdArray。

```php
function main(): void {
    $items = std::array(Type::Int, 3);

    $items[0] = 10;
    $items[1] = 20;
    $items[2] = 30;

    var_dump($items[1]);
}
```

嵌套定长数组：

```php
function main(): void {
    $matrix = std::array(std::array(Type::Int, 3), 2);

    $matrix[0][0] = 1;
    $matrix[1][2] = 9;

    var_dump($matrix[1][2]);
}
```

对象类型元素：

```php
class User {
    public function __construct(public string $name) {}
}

function main(): void {
    $users = std::array(User::class, 2);

    $users[0] = new User("rango");
    $users[1] = new User("swoole");

    var_dump($users[0]->name);
}
```

## std::vector($type[, $size])

`std::vector()` 创建动态 StdVector 容器。第一个参数是元素类型，第二个可选参数是初始大小，必须是整数字面量。

```php
function main(): void {
    $numbers = std::vector(Type::Int);

    $numbers[] = 10;
    $numbers[] = 20;

    var_dump(count($numbers));
}
```

指定初始大小：

```php
function main(): void {
    $numbers = std::vector(Type::Int, 3);

    $numbers[0] = 1;
    $numbers[1] = 2;
    $numbers[2] = 3;

    var_dump($numbers[2]);
}
```

对象或接口类型元素：

```php
interface Task {
    public function id(): int;
}

class Job implements Task {
    public function __construct(private int $id) {}

    public function id(): int {
        return $this->id;
    }
}

function main(): void {
    $tasks = std::vector(Task::class);

    $tasks[] = new Job(1);
    $tasks[] = new Job(2);

    var_dump($tasks[0]->id());
}
```

## std::map($keyType, $valueType)

`std::map()` 创建哈希 StdMap 容器，底层对应 C++ `std::unordered_map`。键类型仅支持 `Type::Int`、`Type::String` 或 `Type::String`。

```php
function main(): void {
    $scores = std::map(Type::String, Type::Int);

    $scores["alice"] = 90;
    $scores["bob"] = 80;

    var_dump($scores["alice"]);
}
```

整数键：

```php
function main(): void {
    $users = std::map(Type::Int, Type::String);

    $users[1001] = "alice";
    $users[1002] = "bob";

    var_dump($users[1002]);
}
```

对象类型值：

```php
class Connection {
    public function __construct(public string $name) {}
}

function main(): void {
    $pool = std::map(Type::String, Connection::class);

    $pool["main"] = new Connection("main");

    var_dump($pool["main"]->name);
}
```

## std::orderedMap($keyType, $valueType)

`std::orderedMap()` 创建有序 StdOrderedMap 容器，底层对应 C++ `std::map`。键类型限制与 `std::map()` 相同。

```php
function main(): void {
    $items = std::orderedMap(Type::String, Type::Int);

    $items["b"] = 2;
    $items["a"] = 1;

    foreach ($items as $key => $value) {
        echo $key . ":" . $value . "\n";
    }
}
```

高精度数值作为 value：

```php
function main(): void {
    $balances = std::orderedMap(Type::Int, Type::Decimal);

    $balances[1] = std::decimal("19.99");
    $balances[2] = std::decimal("100.50");

    var_dump($balances[2]->toString());
}
```

## 类型描述参数

Std 容器的 `$type`、`$keyType`、`$valueType` 不是普通运行时变量，而是编译期类型描述。常用取值如下：

| 类型描述 | 含义 |
|----------|------|
| `Type::Int` | native int |
| `Type::Float` | native float |
| `Type::Bool` | native bool |
| `Type::BigInt` | BigInt |
| `Type::Decimal` | Decimal |
| `Type::BigFloat` | BigFloat |
| `Type::String` | string |
| `Type::Array` | array |
| `Type::Object` | object |
| `Type::Any` | any / mixed |
| `Type::Stream` | stream |
| `ClassName::class` | 指定类、抽象类或接口 |

## 使用限制

`std::array()`、`std::vector()`、`std::map()`、`std::orderedMap()` 是容器构造入口，只能用于变量首次赋值，并且要求位于函数顶层作用域：

```php
function main(): void {
    $numbers = std::vector(Type::Int); // 正确

    // 错误：不能对已有变量重新创建容器
    // $numbers = std::vector(Type::Int);
}
```

错误示例：

```php
function main(bool $flag): void {
    if ($flag) {
        // 错误：不能在 if/while/for 等嵌套语句块中创建 std 容器
        $numbers = std::vector(Type::Int);
    }
}
```

若需要从 `mixed` / `any` 值恢复 Std 容器类型，应使用关键词方法 `toStdArray()`、`toStdVector()`、`toStdMap()`、`toStdOrderedMap()`，它们不属于本文列出的编译期函数。

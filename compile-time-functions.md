# 编译期函数

编译期函数是 AOT 编译器专有的语法入口。编译器在静态编译阶段识别这些函数，并直接生成对应的 C++ 代码；它们通常不会在运行时按普通 PHP 函数查找。

关键词方法是另一套语法体系，例如 `toAny()`、`toRef()`、`toObject()`、`toStdVector()` 等，不列入本文的编译期函数清单。关键词方法详见 [关键词方法](keyword-method.md)。

## 函数清单

| 函数 | 作用 |
|------|------|
| `any($value)` | 将表达式降级为 `mixed` / `any` / `php::Var` |
| `refval($value)` | 显式按引用传递变量、数组元素或对象属性 |
| `objval($value, $className)` | 从 `mixed` / `any` 恢复对象声明类型 |
| `std::int($value)` | 创建 native int 表达式 |
| `std::float($value)` | 创建 native float 表达式 |
| `std::bool($value)` | 创建 native bool 表达式 |
| `std::bigInt($value)` | 创建 BigInt 高精度整数 |
| `std::decimal($value)` | 创建 Decimal 高精度十进制数 |
| `std::bigFloat($value)` | 创建 BigFloat 高精度浮点数 |
| `std::array($type, $size)` | 创建定长 StdArray 容器 |
| `std::vector($type[, $size])` | 创建动态 StdVector 容器 |
| `std::map($keyType, $valueType)` | 创建哈希 StdMap 容器 |
| `std::ordered_map($keyType, $valueType)` | 创建有序 StdOrderedMap 容器 |

## any($value)

`any()` 将表达式的编译期类型降级为动态类型。降级后变量可以接收任意 PHP 值，但编译器也不再为它生成 native type、typed object 或 Native Call 优化。

```php
function main(): void {
    $value = any(10);

    $value = "string";
    var_dump($value);
}
```

`any()` 常用于分支中可能产生不同类型的场景：

```php
class FileLogger {}
class NullLogger {}

function create_logger(bool $debug) {
    if ($debug) {
        $logger = any(new FileLogger());
    } else {
        $logger = any(new NullLogger());
    }

    return $logger;
}
```

它也可用于保留 PHP 动态运算语义：

```php
function main(): void {
    $a = any(10);
    $b = $a / 3;

    var_dump($b); // float(3.333333...)
}
```

## refval($value)

`refval()` 用于显式传递引用。它主要用于动态调用、闭包调用、可变函数调用等编译期无法获知参数是否为引用的场景。

`refval()` 的参数必须是可作为引用的左值：变量、数组元素或对象属性。不能传入字面量、函数返回值、算术表达式等临时值。

变量引用：

```php
function main(): void {
    $fn = function (&$name): void {
        $name .= " compiler";
    };

    $name = "php";
    $fn(refval($name));

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
    $fn(refval($data["name"]));

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
    $fn(refval($box->value));

    var_dump($box->value); // string(7) "changed"
}
```

静态函数和内置函数的参数信息在编译期明确时，编译器会自动处理引用参数，不需要额外使用 `refval()`：

```php
function main(): void {
    parse_str("hello=world", $result);
    var_dump($result["hello"]); // string(5) "world"
}
```

## objval($value, $className)

`objval()` 从 `mixed` / `any` 值恢复对象声明类型。它会让编译器知道后续表达式按指定类、父类、抽象类或接口处理，并在运行时插入对象类型检查。

第二个参数只支持字符串字面量或 `ClassName::class`。不能使用变量动态传入类名。

```php
class User {
    public function name(): string {
        return "rango";
    }
}

function main(): void {
    $data = ["user" => new User()];

    $user = objval($data["user"], User::class);
    var_dump($user->name());
}
```

使用字符串类名：

```php
class Service {
    public function run(): string {
        return "ok";
    }
}

function main(): void {
    $value = any(new Service());

    $service = objval($value, "Service");
    var_dump($service->run());
}
```

`objval()` 使用 `is-a` 关系检查对象类型，因此子类对象可以作为父类、抽象类或接口使用：

```php
interface Logger {
    public function write(string $message): void;
}

class FileLogger implements Logger {
    public function write(string $message): void {
        echo $message;
    }
}

function main(): void {
    $value = any(new FileLogger());
    $logger = objval($value, Logger::class);

    $logger->write("hello");
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
    $value = any("123");
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
    $value = any("3.14");
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
    $value = any("");
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
    $items = std::array(native_types::type_int, 3);

    $items[0] = 10;
    $items[1] = 20;
    $items[2] = 30;

    var_dump($items[1]);
}
```

嵌套定长数组：

```php
function main(): void {
    $matrix = std::array(std::array(native_types::type_int, 3), 2);

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
    $numbers = std::vector(native_types::type_int);

    $numbers[] = 10;
    $numbers[] = 20;

    var_dump(count($numbers));
}
```

指定初始大小：

```php
function main(): void {
    $numbers = std::vector(native_types::type_int, 3);

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

`std::map()` 创建哈希 StdMap 容器，底层对应 C++ `std::unordered_map`。键类型仅支持 `native_types::type_int`、`complex_types::type_string` 或 `complex_types::type_str`。

```php
function main(): void {
    $scores = std::map(complex_types::type_string, native_types::type_int);

    $scores["alice"] = 90;
    $scores["bob"] = 80;

    var_dump($scores["alice"]);
}
```

整数键：

```php
function main(): void {
    $users = std::map(native_types::type_int, complex_types::type_string);

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
    $pool = std::map(complex_types::type_string, Connection::class);

    $pool["main"] = new Connection("main");

    var_dump($pool["main"]->name);
}
```

## std::ordered_map($keyType, $valueType)

`std::ordered_map()` 创建有序 StdOrderedMap 容器，底层对应 C++ `std::map`。键类型限制与 `std::map()` 相同。

```php
function main(): void {
    $items = std::ordered_map(complex_types::type_string, native_types::type_int);

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
    $balances = std::ordered_map(native_types::type_int, native_types::type_decimal);

    $balances[1] = std::decimal("19.99");
    $balances[2] = std::decimal("100.50");

    var_dump($balances[2]->toString());
}
```

## 类型描述参数

Std 容器的 `$type`、`$keyType`、`$valueType` 不是普通运行时变量，而是编译期类型描述。常用取值如下：

| 类型描述 | 含义 |
|----------|------|
| `native_types::type_int` | native int |
| `native_types::type_float` | native float |
| `native_types::type_bool` | native bool |
| `native_types::type_bigint` | BigInt |
| `native_types::type_decimal` | Decimal |
| `native_types::type_bigfloat` | BigFloat |
| `complex_types::type_string` / `complex_types::type_str` | string |
| `complex_types::type_array` | array |
| `complex_types::type_object` | object |
| `complex_types::type_any` / `complex_types::type_var` | any / mixed |
| `complex_types::type_stream` | stream |
| `ClassName::class` | 指定类、抽象类或接口 |

## 使用限制

`std::array()`、`std::vector()`、`std::map()`、`std::ordered_map()` 是容器构造入口，只能用于变量首次赋值，并且要求位于函数顶层作用域：

```php
function main(): void {
    $numbers = std::vector(native_types::type_int); // 正确

    // 错误：不能对已有变量重新创建容器
    // $numbers = std::vector(native_types::type_int);
}
```

错误示例：

```php
function main(bool $flag): void {
    if ($flag) {
        // 错误：不能在 if/while/for 等嵌套语句块中创建 std 容器
        $numbers = std::vector(native_types::type_int);
    }
}
```

若需要从 `mixed` / `any` 值恢复 Std 容器类型，应使用关键词方法 `toStdArray()`、`toStdVector()`、`toStdMap()`、`toStdOrderedMap()`，它们不属于本文列出的编译期函数。

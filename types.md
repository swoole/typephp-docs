# 类型系统

TypePHP 编译器在标准 `PHP` 类型之外扩展了高精度数值类型和强类型容器，并在编译期进行类型推断和检查。本文档介绍编译器支持的所有类型、类型转换方法及使用限制。

对象类型还可以通过同命名空间中的普通函数补充仅在静态编译阶段生效的实例方法，详见[扩展对象方法](object-extension-method.md)。

对对象布局和调用性能有极致要求、且不需要进入 ZendVM 的内部数据结构，可以显式声明为 [`#[Native]` 原生类](native-class.md)。原生类采用固定 C++ 字段布局和独立 tracing GC，具有比普通 PHP 对象更严格的静态类型与互操作边界。

## ZendPHP 的类型系统边界

ZendPHP 本质上仍是动态、弱类型语言。PHP 提供的参数和返回值类型声明（通常也称为 type hints）主要约束函数调用边界，而不是为函数中的变量建立不可改变的静态类型。

参数类型在进入函数时检查，但普通参数进入函数体后只是一个 `zval`，可以被重新赋值为任意其他类型：

```php
declare(strict_types=1);

function change(int $value): void
{
    $value = 'text';
    $value = ['php', 'typephp'];
}

change(1); // 合法
```

即使参数使用引用传递，参数声明本身也只负责入口检查：

```php
function changeRef(int &$value): void
{
    $value = 'text';
}

$value = 1;
changeRef($value);
var_dump($value); // string(4) "text"
```

返回类型同样是在函数退出、返回值离开函数时检查。它不会限制函数体内临时变量的类型变化。局部变量也没有独立的类型声明，其类型可以随每次赋值改变。

`declare(strict_types=1)` 只关闭函数调用边界上的部分标量隐式转换，使不匹配的参数或返回值抛出 `TypeError`；它不会让局部变量或参数在函数体内变成固定类型，因此也不会把 ZendPHP 转换成静态强类型语言。

在 PHP 的可变存储中，真正持续携带类型约束的是声明了类型的对象属性，包括静态属性。属性值可以修改，但属性声明的类型不能在运行期间改变；每次写入都会由 ZendVM 检查：

```php
class User
{
    public int $id = 0;
}

$user = new User();
$user->id = 42;     // 合法
$user->id = 'text'; // TypeError
```

这个约束附着在属性槽上。即使把属性作为引用传给函数，也不能通过引用绕过：

```php
$user = new User();
changeRef($user->id); // TypeError：不能向 int 属性写入 string
```

无显式默认值时，ZendPHP 的 typed property 不是类型零值，也不是 `null`，而是尚未初始化的 `IS_UNDEF` 状态。TypePHP 为了生成固定类型的 C++ 属性槽，对常见固定值类型采用确定的零值初始化，因此这里存在明确的兼容性差异：

| 属性声明 | ZendPHP 无显式默认值 | TypePHP 无显式默认值 |
|---|---|---|
| `public int $value;` | 未初始化；读取抛出 `Error` | `0` |
| `public float $value;` | 未初始化；读取抛出 `Error` | `0.0` |
| `public bool $value;` | 未初始化；读取抛出 `Error` | `false` |
| `public string $value;` | 未初始化；读取抛出 `Error` | 空字符串 `''` |
| `public array $value;` | 未初始化；读取抛出 `Error` | 空数组 `[]` |
| `public ?int $value;` | 未初始化；不会自动成为 `null` | 可空动态存储；需要显式默认值时应写 `= null` |
| `public mixed $value;` | 未初始化 | 动态值，默认 `null` |
| `public $value;` | `null` | `null` |

如果源码显式声明默认值，例如 `public int $value = 10` 或 `public ?int $value = null`，则使用声明值，不使用表中的隐式初始状态。

这项差异会影响直接读取、`isset()`、`??` 和 Reflection 初始化状态。例如 ZendPHP 中未初始化的 `public int $value` 会让 `isset($object->value)` 返回 `false`，`$object->value ?? 10` 返回 `10`；TypePHP 的固定槽已经包含 `0`，不能依赖 ZendPHP 的 uninitialized 行为。需要表达“尚未设置”时，应显式使用可空类型及 `= null`，而不要依赖无默认值的 typed property。更多限制参见[兼容性说明](compatible.md#固定值类型属性)。

PHP 8.4 也支持 typed class constant，但类常量本身不可写，不属于可变变量的存储模型。

普通 PHP 数组同样没有键和值的元素类型。一个数组可以同时保存不同类型的键和值，也可以在运行时任意改变结构：

```php
$values = [];
$values[] = 1;
$values[] = 'text';
$values['user'] = new User();
$values[10] = ['nested' => true];
```

`array` 参数或属性类型只表示这个值本身必须是 PHP 数组，不会约束数组内部元素。PHPDoc 中的 `list<int>`、`array<string, User>` 等写法仅供 IDE 和静态分析工具使用，ZendVM 不会执行这些约束。

因此，ZendPHP 并没有覆盖变量、参数、局部值和容器元素的完整强类型系统。TypePHP 在兼容 PHP 语法的基础上增加编译期类型推断、固定原生类型、[`#[ArrayDef]`](array-def.md)、[Std 强类型容器](std-containers.md)、[`#[Immutable]`](immutable.md) 和 [`#[Native]`](native-class.md) 等约束；这些能力才会在编译阶段限制类型变化，或生成具有固定 C++ 存储类型的代码。

## 1. 类型总览

TypePHP 编译器支持以下 C++ 存储类型（`php::*`）：

| 类型常量 | C++ 类型 | 对应 PHP 类型 | 说明 |
|---------|----------|--------------|------|
| `TYPE_VAR` | `php::Var` | `mixed` | 动态类型，使用 zval 存储 |
| `TYPE_INT` | `php::Int` | `int` | 原生 64 位整数 |
| `TYPE_FLOAT` | `php::Float` | `float` | 原生双精度浮点 |
| `TYPE_BOOL` | `php::Bool` | `bool` | 原生布尔值 |
| `TYPE_STR` | `php::Str` | `string` | 原生字符串 |
| `TYPE_ARRAY` | `php::Array` | `array` | 原生数组 |
| `TYPE_OBJECT` | `php::Object` | `object` | 通用对象（无具体类信息） |
| `TYPE_STREAM` | `php::Stream` | `resource` | 流资源类型 |
| `TYPE_RESOURCE` | `php::Resource` | `resource` | 通用资源类型 |
| `TYPE_BIGINT` | `php::BigInt` | — | 任意精度整数 |
| `TYPE_DECIMAL` | `php::Decimal` | — | 十进制高精度小数 |
| `TYPE_BIGFLOAT` | `php::BigFloat` | — | 二进制高精度浮点 |
| `TYPE_STD_VECTOR` | `php::StdVector` | — | C++ `std::vector` |
| `TYPE_STD_ARRAY` | `php::StdArray` | — | C++ `std::array`（定长） |
| `TYPE_STD_MAP` | `php::StdMap` | — | C++ `std::unordered_map` |
| `TYPE_STD_ORDERED_MAP` | `php::StdOrderedMap` | — | C++ `std::map`（有序） |
| `TYPE_ARGS` | `php::Args` | — | 可变参数列表 |
| `TYPE_REF` | `php::Ref` | — | 引用类型 |
| `TYPE_VOID` | `void` | `void` | 无返回值 |

## 2. 原生类型

在 `use native_types` 声明下，编译器将 `int`、`float`、`bool` 映射为原生 C++ 类型，消除 `zval` 包装开销。

```php
declare(strict_types=1);
use native_types;

function sum(int $n): int {
    $total = 0;        // php::Int
    $pi = 3.14159;     // php::Float
    $flag = true;      // php::Bool
    $name = "hello";   // php::Str
    $items = [1, 2];   // php::Array
    return $total + $n;
}
```

不使用 `use native_types` 时，所有变量默认为 `php::Var`（动态类型）。

### 2.1 原生类型与 PHP 类型声明映射

| PHP 类型声明 | AOT 类型 | 说明 |
|-------------|----------|------|
| `int` | `php::Int` | |
| `float` / `double` | `php::Float` | |
| `bool` / `false` / `true` | `php::Bool` | |
| `string` | `php::Str` | |
| `array` | `php::Array` | |
| `object` | `php::Object` | 无具体类信息 |
| `void` / `never` | `void` | |
| `mixed` | `php::Var` | |
| `null` | `php::Var` | 退化为动态类型 |
| `callable` | `php::Var` | 编译器无法追踪 |
| `iterable` | `php::Var` | 编译器无法追踪 |
| `stream` | `php::Stream` | TypePHP 专有 |

> **注意**：`null`、`callable`、`iterable` 类型声明在编译阶段会退化为 `php::Var`，无法享受原生类型的性能优势。

## 3. 高精度数值类型

TypePHP 编译器提供三种高精度数值类型，详见 [math.md](math.md)。

### 3.1 构造方式

```php
declare(strict_types=1);
use native_types;

// 通过 std:: 工厂函数构造
$a = std::bigInt("12345678901234567890");
$b = std::decimal("0.1");
$c = std::bigFloat("1.2345e100");

// 字面量自动识别（超长整数 / 高精度浮点）
$d = 12345678901234567890;    // 19 位以上自动识别为 BigInt
$e = 0.1234567890123456;      // 16 位以上有效数字自动识别为 Decimal
```

### 3.2 声明指令

- **`use bigint_types`**：文件中所有整数字面量自动成为 `BigInt`
- **`use decimal_types`**：文件中所有浮点字面量自动成为 `Decimal`

```php
declare(strict_types=1);
use bigint_types;
use decimal_types;

$a = 42;      // php::BigInt（不是 php::Int）
$b = 3.14;    // php::Decimal（不是 php::Float）
```

### 3.3 不可变性

Big* 类型是**不可变的**（immutable）——每次运算返回新值，不修改原变量。复合赋值运算（`+=`、`-=` 等）在编译时展开为 `$a = BigInt::add($a, ...)`。

## 4. Std 强类型容器

TypePHP 编译器直接映射 C++ 标准库容器，提供零开销的类型安全存储。键类型仅支持 `Type::Int` 和 `Type::String`。

如果数据必须保持普通 PHP `array`，但希望编译器检查属性上的直接元素写入，可以使用 [`#[ArrayDef]`](array-def.md)。它不会把 PHP 数组转换成 Std Container；两者的存储模型和动态边界不同。

```php
declare(strict_types=1);
use native_types;

// std::vector — 动态数组
$v = std::vector(Type::Int);
$v[] = 10;
$v[] = 20;

// std::array — 定长数组（编译期边界检查）
$a = std::array(Type::Float, 5);
$a[0] = 3.14;

// std::ordered_map — 有序映射
$m = std::ordered_map(Type::String, Type::Int);
$m["key"] = 100;

// std::map — 哈希映射
$u = std::map(Type::Int, User::class);
$u[1] = new User(1);
```

### 4.1 类型符号

根命名空间 `Type` 提供带有 IDE 补全和拼写检查的编译期类型符号。它不同于编译器内部使用的 `TypePHP\Type`。

| 类型符号 | 用途 |
|---------|------|
| `Type::Int` | 标注整型元素 |
| `Type::Float` | 标注浮点元素 |
| `Type::Bool` | 标注布尔元素 |
| `Type::BigInt` | 标注 BigInt 元素 |
| `Type::BigFloat` | 标注 BigFloat 元素 |
| `Type::Decimal` | 标注 Decimal 元素 |
| `Type::String` | 标注字符串元素 |
| `Type::Array` | 标注数组元素 |
| `Type::Object` | 标注对象元素 |
| `Type::Any` | 标注动态类型元素 |
| `Type::Stream` | 标注 Stream 元素 |

容器值类型也可以是任意 PHP 类名（如 `User::class`），编译器会为每个具体类型生成独立的 C++ 模板实例。

## 5. 类型推断

编译器在编译期通过 `detectTypeOfExpr()` 对表达式进行类型推断：

### 5.1 字面量

| 表达式 | 推断类型 |
|--------|---------|
| 整数字面量 `123` | `php::Int`（启用 `bigint_types` 时为 `php::BigInt`） |
| 浮点字面量 `3.14` | `php::Float`（启用 `decimal_types` 时为 `php::Decimal`） |
| 超长整数（≥19 位） | 自动识别为 `php::BigInt` |
| 高精度浮点（≥16 位有效数字） | 自动识别为 `php::Decimal` |
| 布尔字面量 `true` / `false` | `php::Bool` |
| 字符串字面量 `"hello"` | `php::Str`（启用 `native_types` 时） |
| 数组字面量 `[1, 2]` | `php::Array` |

### 5.2 类型转换表达式

| 表达式 | 推断类型 |
|--------|---------|
| `(int) $x` | `php::Int` |
| `(float) $x` | `php::Float` |
| `(bool) $x` | `php::Bool` |
| `(string) $x` | `php::Str` |
| `(array) $x` | `php::Array` |
| `(object) $x` | `php::Object`（**无类信息**） |

### 5.3 二元运算

编译器根据左右操作数类型按优先级确定结果类型：

1. 任一操作数为 `BigFloat` → 结果 `BigFloat`
2. 任一操作数为 `Decimal` → 结果 `Decimal`
3. 任一操作数为 `BigInt` → 结果 `BigInt`
4. 任一操作数为 `Float` → 结果 `Float`
5. 任一操作数为 `Int` → 结果 `Int`
6. 否则退化为 `Var`

> 除法例外：两个 `Int` 相除结果仍为 `Int`（整数除法），若除不尽则退化为 `Var`。

### 5.4 函数返回值

- 内置函数（如 `fopen`、`stream_socket_client`）的返回类型由编译器内置规则确定
- 用户定义函数的返回类型从 `declare` 声明推断
- 动态调用（`call_user_func`、`eval` 定义的函数）退化为 `Var`

## 6. 类型转换

> `to*` 关键词方法是实现类型转换的主要途径，详细说明见 [类型转换](type-convert.md) 文档。

### 6.1 自动类型提升

Big* 类型与普通 Int/Float 混合运算时，编译器自动进行类型提升：

| 源类型 | 目标类型 | 提升方式 |
|--------|---------|---------|
| `Int` | `BigInt` | `php::newBigInt($n)` |
| `Float` | `Decimal` (字面量) | `php::newDecimal("...")` |
| `Float` | `Decimal` (变量) | **报错**——必须使用字符串构造 |
| `Float` | `BigInt` | **报错**——禁止转换 |
| `Float` | `BigFloat` | `php::newBigFloat($f)` |
| `Int` | `Decimal` | `php::newDecimal(php::toString($n))` |
| `Int` | `BigFloat` | `php::newBigFloat($n)` |
| `BigInt` | `Decimal` | `php::newDecimal(php::BigInt::toString($n))` |
| `BigInt` | `BigFloat` | `php::BigFloat::newInstance(php::BigInt::toString($n))` |
| `Decimal` | `BigFloat` | `php::BigFloat::newInstance(php::Decimal::toString($n))` |

### 6.2 手动类型转换

编译器提供类型转换函数，也可在 ZendPHP 中通过 polyfills 兼容：

```php
// 通过 std:: 工厂构造 Big* 类型（推荐）
$a = std::bigInt("1234567890");      // string → BigInt
$b = std::decimal("0.01");           // string → Decimal
$c = std::bigFloat("1.23e100");      // string → BigFloat

// Big* → 普通类型的方法
$d = $a->toInt();                    // BigInt → Int（可能溢出）
$e = $b->toFloat();                  // Decimal → Float
$f = $c->toString();                 // BigFloat → String

// 跨 Big* 类型转换
$g = std::bigInt($a->toString());    // Decimal → String → BigInt
$h = std::decimal($a->toString());   // BigInt → String → Decimal
```

### 6.3 类型接续（从 Var 恢复类型）

从数组取出元素或调用返回 `any` 类型的函数后，编译器丢失类型信息。可使用以下方式完成类型接续：

```php
// 对象类型接续
$user = $array['user']->toObject(User::class);
echo $user->greet();  // 编译器可生成 Native Call

// Stream 类型接续
$sockets = stream_socket_pair(STREAM_PF_UNIX, STREAM_SOCK_STREAM, 0);
$client = $sockets[0]->toStream();
$client->write("hello");

// 基础类型转换语法
$v = (int) $array['count'];       // → php::Int
$v = (float) $array['price'];     // → php::Float
$v = (bool) $array['active'];     // → php::Bool
$v = (string) $array['name'];     // → php::Str
$v = (array) $array['items'];     // → php::Array

// 基础类型转换函数
$v = intval($array['count']);     // → php::Int
$v = floatval($array['price']);   // → php::Float
$v = boolval($array['active']);   // → php::Bool
$v = strval($array['name']);      // → php::Str

// 显式降级为动态类型
$v = $array['mixed']->toAny();    // → php::Var，等价于 any($array['mixed'])

// 显式转为引用
$ref = $array['value']->toRef();  // → php::Ref，等价于 refval($array['value'])
```

> **注意**：`(object)` 转换得到的是 `php::Object`，**不包含具体的类信息**，编译器无法对其方法进行 Native Call 优化。请始终使用 `toObject(ClassName::class)` 重建对象类型。

### 6.4 类型丢弃

某些场景下，一个变量在不同条件分支中需要持有不同类型的值（如不同类的对象），此时编译器的静态类型推断会成为障碍——编译器根据第一次赋值推断类型，后续赋值不同类型会报错。使用 `any()` 或等价关键词方法 `toAny()` 可以将类型标注为 `php::Var`，放弃编译期类型跟踪，交由运行时动态处理。

```php
class Foo1 {
    public function run() {
        var_dump(__METHOD__);
    }
}

class Foo2 {
    public function run() {
        var_dump(__METHOD__);
    }
}

function main() {
    $rand = random_int(0, 10000);
    if ($rand % 2) {
        $o = any(new Foo1());
    } else {
        $o = any(new Foo2());
    }
    if (method_exists($o, 'run')) {
        $o->run();
    }
}
```

在此例中：
- 去掉 `any()`，编译器会将 `$o` 的类型锁定为 `Foo1`，`else` 分支中赋值 `Foo2` 对象会报类型冲突错误
- 加上 `any()` 后，`$o` 被标注为 `php::Var`，可接收任意类型的值，方法调用走动态分发

`any()` 与 `toAny()` 只影响编译期类型推断，不会生成额外的运行时类型检查。需要注意：降级为 `php::Var` 后，编译器也不会再为该变量生成 typed object 的 Native Call 优化。


## 7. 类型限制

### 7.1 跨 Big* 类型禁止隐式混合

编译器阻止可能导致精度损失的跨类型隐式混合：

```php
$a = std::bigInt("100");
$b = std::decimal("2.5");
$c = $a + $b;  // ❌ 编译错误：BigInt 和 Decimal 不能直接运算
```

解决方式：显式转换为同类型后再运算。

### 7.2 Float 转 Decimal 仅限字面量

```php
$a = std::decimal("0.1");    // ✅ 推荐
$b = std::decimal(0.1);      // ✅ 字面量 —— 编译器提取原始文本 "0.1" 作为字符串处理
$pi = 3.14159;
$c = std::decimal($pi);      // ❌ 编译错误：无法从变量转换
```

> **特殊逻辑**：当 `std::decimal()` 的参数为浮点字面量时，编译器直接从 AST 提取 `rawValue` 作为字符串传递给构造函数，避免二进制浮点误差。但变量不保留原始文本，因此无法安全转换。

### 7.3 Float 禁止转 BigInt

```php
$a = 3.14;
$b = std::bigInt($a);       // ❌ 编译错误：Cannot convert float to BigInt
$b = std::bigInt("3");      // ✅ 使用字符串
```

### 7.4 Big* 类型不支持自增/自减

Big* 类型是不可变的，`++` / `--` 语义不匹配，编译器会报错：

```php
$a = std::bigInt("100");
$a++;  // ❌ 编译错误
$a--;  // ❌ 编译错误
```

### 7.5 联合类型、交叉类型、Nullable 类型退化为 Var

PHP 的联合类型（`int|float`、`int|string` 等）、交叉类型（`A&B`）和 Nullable 类型（`?A`）在编译时退化为 `php::Var`，无法享受原生类型的性能优势。编译器仍会保留运行时 type check，避免静态阶段无法处理时绕过 PHP 类型约束。

### 7.6 object 类型不保留类信息

`(object)` 转换和 `object` 类型声明只产生 `php::Object`，编译器不知道具体的类名，无法优化方法调用。必须使用 `toObject(ClassName::class)` 重建。

### 7.7 Std 容器键类型限制

`std::ordered_map` 和 `std::map` 的键类型仅支持：
- `Type::Int` — 整数键
- `Type::String` / `Type::String` — 字符串键

其他键类型会导致编译错误。

### 7.8 Std 容器不能在 foreach 中删除元素

当 std 容器处于 `foreach` 循环中且启用锁定时，不能对其元素执行 `unset` 操作。

### 7.9 原生类型变量不能 unset

原生类型变量（`php::Int`、`php::Float` 等）不能使用 `unset()`，因为它们在 C++ 中是栈上值类型。

### 7.10 对象属性类型固定，固定值类型不能 unset 或改成 null

TypePHP 编译器要求对象属性始终保持声明时的类型，不能在运行过程中改成其他类型。

```php
declare(strict_types=1);
use native_types;

class User {
    public int $id = 0;
    public Profile $profile;
}

$user = new User();
unset($user->id); // ❌ 不允许依赖 PHP 的属性 unset 语义
$user->id = null; // ❌ 固定值类型属性不能改成 null

unset($user->profile); // ✅ 对象属性可进入 null/unset 状态
$user->profile = null; // ✅ 对象属性可显式设置为 null
```

在 PHP 解释器中，`unset($obj->prop)` 可以让属性进入未初始化状态；对固定值类型属性赋值 `null` 也会把属性改成空值。从 AOT 的类型系统看，这等价于把属性从声明的 `int`、`float`、`bool`、`string`、`array` 改成 `null`/未初始化状态。AOT 不允许这些固定值类型属性改变类型，因此属性永远是声明时的类型。

具体类对象属性使用静态对象类型规则：非空赋值必须满足 `is-a` 关系，因此允许把子类对象赋给基类属性，不允许把无继承关系的对象或基类对象赋给子类属性。非 nullable 属性可以 `unset()`，但不能赋值为 `null`。

如果业务上需要“没有值”的状态，应显式使用可空类型并赋值：

```php
class User {
    public ?int $id = null;
}

$user->id = null; // ✅ 类型声明中允许 null
```

### 7.11 关闭原生类型以启用特定行为

需要溢出检测、动态类型赋值等动态特性时，不应使用 `use native_types`，或使用 `any()` 将变量标注为 `php::Var`：

```php
$a = any(10);        // $a 的类型为 php::Var，保留溢出检测能力
$b = $a / 3;         // 浮点除法，结果为 3.333...
```

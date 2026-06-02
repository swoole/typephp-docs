# 类型系统

`AOT` 编译器在标准 `PHP` 类型之外扩展了高精度数值类型和强类型容器，并在编译期进行类型推断和检查。本文档介绍编译器支持的所有类型、类型转换方法及使用限制。

## 1. 类型总览

AOT 编译器支持以下 C++ 存储类型（`php::*`）：

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
| `TYPE_STD_MAP` | `php::StdMap` | — | C++ `std::map`（有序） |
| `TYPE_STD_UNORDERED_MAP` | `php::StdUnorderedMap` | — | C++ `std::unordered_map` |
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
| `stream` | `php::Stream` | AOT 专有 |

> **注意**：`null`、`callable`、`iterable` 类型声明在编译阶段会退化为 `php::Var`，无法享受原生类型的性能优势。

## 3. 高精度数值类型

AOT 编译器提供三种高精度数值类型，详见 [math.md](math.md)。

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

AOT 编译器直接映射 C++ 标准库容器，提供零开销的类型安全存储。键类型仅支持 `native_types::type_int` 和 `complex_types::type_string`。

```php
declare(strict_types=1);
use native_types;

// std::vector — 动态数组
$v = std::vector(native_types::type_int);
$v[] = 10;
$v[] = 20;

// std::array — 定长数组（编译期边界检查）
$a = std::array(native_types::type_float, 5);
$a[0] = 3.14;

// std::map — 有序映射
$m = std::map(complex_types::type_string, native_types::type_int);
$m["key"] = 100;

// std::unordered_map — 哈希映射
$u = std::unordered_map(native_types::type_int, User::class);
$u[1] = new User(1);
```

### 4.1 类型辅助类

| 辅助类 | 常量 | 值 | 用途 |
|--------|------|-----|------|
| `native_types` | `type_int` | `'int'` | 标注整型元素 |
| `native_types` | `type_float` | `'float'` | 标注浮点元素 |
| `native_types` | `type_bool` | `'bool'` | 标注布尔元素 |
| `native_types` | `type_bigint` | `'bigint'` | 标注 BigInt 元素 |
| `native_types` | `type_bigfloat` | `'bigfloat'` | 标注 BigFloat 元素 |
| `native_types` | `type_decimal` | `'decimal'` | 标注 Decimal 元素 |
| `complex_types` | `type_string` / `type_str` | `'string'` | 标注字符串元素 |
| `complex_types` | `type_array` | `'array'` | 标注数组元素 |
| `complex_types` | `type_object` | `'object'` | 标注对象元素 |
| `complex_types` | `type_any` / `type_var` | `'any'` | 标注动态类型元素 |
| `complex_types` | `type_stream` | `'stream'` | 标注 Stream 元素 |

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
```

> **注意**：`(object)` 转换得到的是 `php::Object`，**不包含具体的类信息**，编译器无法对其方法进行 Native Call 优化。请始终使用 `toObject(ClassName::class)` 重建对象类型。

### 6.4 类型丢弃

某些场景下，一个变量在不同条件分支中需要持有不同类型的值（如不同类的对象），此时编译器的静态类型推断会成为障碍——编译器根据第一次赋值推断类型，后续赋值不同类型会报错。使用 `any()` 可以将类型标注为 `php::Var`，放弃编译期类型跟踪，交由运行时动态处理。

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

`any()` 在编译期被直接替换为表达式本身，**零运行时开销**。

### 垫片函数
```php
function any(mixed $var): mixed
{
    return $var;
}
```

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

### 7.5 联合类型退化为 Var

PHP 的联合类型（`int|float`、`int|string` 等）在编译时退化为 `php::Var`，无法享受原生类型的性能优势。

### 7.6 object 类型不保留类信息

`(object)` 转换和 `object` 类型声明只产生 `php::Object`，编译器不知道具体的类名，无法优化方法调用。必须使用 `toObject(ClassName::class)` 重建。

### 7.7 Std 容器键类型限制

`std::map` 和 `std::unordered_map` 的键类型仅支持：
- `native_types::type_int` — 整数键
- `complex_types::type_string` / `complex_types::type_str` — 字符串键

其他键类型会导致编译错误。

### 7.8 Std 容器不能在 foreach 中删除元素

当 std 容器处于 `foreach` 循环中且启用锁定时，不能对其元素执行 `unset` 操作。

### 7.9 原生类型变量不能 unset

原生类型变量（`php::Int`、`php::Float` 等）不能使用 `unset()`，因为它们在 C++ 中是栈上值类型。

### 7.10 关闭原生类型以启用特定行为

需要溢出检测、动态类型赋值等动态特性时，不应使用 `use native_types`，或使用 `any()` 将变量标注为 `php::Var`：

```php
$a = any(10);        // $a 的类型为 php::Var，保留溢出检测能力
$b = $a / 3;         // 浮点除法，结果为 3.333...
```

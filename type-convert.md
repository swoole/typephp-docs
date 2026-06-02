# 类型转换

`to*` 是 AOT 编译器提供的关键词方法（keyword method），用于将表达式或变量的值显式转换为目标类型。与普通通用方法不同，`to*` 方法在编译器中具有一等公民地位——无论 receiver 为何种类型，编译器都按照内置规则进行转换，跳过类型方法表查找。

所有 `to*` 方法调用在编译时完全消解为 C++ 函数调用，**零运行时开销**。

---

## 1. 基础类型转换

在任意表达式上调用，将结果转换为对应原生类型：

```php
declare(strict_types=1);
use native_types;

function convert_basic(mixed $input): void
{
    $i = $input->toInt();          // → php::Int    等价于 (int) $input
    $f = $input->toFloat();        // → php::Float  等价于 (float) $input
    $s = $input->toString();       // → php::Str    等价于 (string) $input
    $b = $input->toBool();         // → php::Bool   等价于 (bool) $input
    $a = $input->toArray();        // → php::Array  等价于 (array) $input
}
```

| 方法 | 目标类型 | 生成代码 | 说明 |
|------|---------|---------|------|
| `toInt()` | `php::Int` | `php::toInt($expr)` | 可能溢出 |
| `toFloat()` | `php::Float` | `php::toFloat($expr)` | |
| `toString()` | `php::Str` | `php::toString($expr)` | |
| `toBool()` | `php::Bool` | `php::toBool($expr)` | |
| `toArray()` | `php::Array` | `php::toArray($expr)` | 见下方"对象转数组"说明 |

### `toArray()` 对象转数组

当 receiver 是**对象**时，`php::toArray()` 会优先检查对象的类是否定义了 `toArray()` 方法：

- **有 `toArray()` 方法**：调用该方法，返回值必须是数组，否则抛出异常
- **无 `toArray()` 方法**：沿用原有逻辑，将对象的属性表转为数组

```php
class User {
    public int $id;
    public string $name;

    public function toArray(): array {
        return [
            'uid' => $this->id,
            'display_name' => $this->name,
        ];
    }
}

function main(): void {
    $user = new User(1, 'admin');
    $arr = $user->toArray();  // → php::toArray($user)
    // 调用 User::toArray()，返回自定义数组结构
    // $arr = ['uid' => 1, 'display_name' => 'admin']

    // 没有 toArray() 方法的对象 → 读取属性表
    $plain = (array) $other;  // → php::toArray($other)
    // 返回属性表：['id' => ..., 'name' => ...]
}
```

> **注意**：`toArray()` 方法的查找基于 PHP 的函数表（大小写不敏感），因此 `toArray` 和 `toarray` 都可匹配。如果 `toArray()` 返回非数组类型，编译器会抛出 `"toArray() method must return an array, got ..."` 异常。

### 与 PHP 强制转换的区别

PHP 的 `(int)` / `(float)` 等强制转换语法虽然也支持，但 `to*` 方法有以下优势：

- **链式调用**：`$result->toString()->length()` 一步完成转换和操作，无需中间变量
- **类型推断**：编译器精确推断 `to*` 返回类型，后续方法调用可享受针对性优化
- **通用性**：`to*` 在 `mixed` 类型和 Big* 类型上均可使用

---

## 2. Stream 类型转换

`toStream()` 将表达式的值转换为 `php::Stream`，之后可直接调用 Stream 通用方法（如 `write`、`read`、`close`）。

```php
declare(strict_types=1);
use native_types;

function stream_example(): void
{
    // 从数组取出元素后接续 Stream 类型
    $pair = stream_socket_pair(AF_UNIX, SOCK_STREAM, 0);
    $pair[0]->toStream()->write("hello");   // → fwrite($pair[0], "hello")
    $data = $pair[1]->toStream()->read(5);  // → fread($pair[1], 5)
    var_dump($data);                        // string(5) "hello"
}
```

> **注意**：`toStream()` 不会自动关闭连接。使用完毕后应调用 `->close()` 释放资源。

---

## 3. 高精度数值类型转换

Big* 类型（BigInt / Decimal / BigFloat）定义了更精确的转换路径，编译器针对每种源类型使用专用的转换函数，避免精度损失。

### 3.1 BigInt 上的转换

```php
declare(strict_types=1);
use native_types;

function bigint_convert(): void
{
    $b = std::bigInt("12345678901234567890");

    $i = $b->toInt();            // php::BigInt::toInt($b) — 可能溢出
    $f = $b->toFloat();          // php::BigInt::toFloat($b)
    $s = $b->toString();         // php::BigInt::toString($b)
    $d = $b->toDecimal();        // php::newDecimal(php::BigInt::toString($b))
    $bf = $b->toBigFloat();      // php::BigFloat::newInstance(php::BigInt::toString($b))
}
```

### 3.2 Decimal 上的转换

```php
$d = std::decimal("123.456");

$i = $d->toInt();               // php::Decimal::toInt($d)
$f = $d->toFloat();             // php::Decimal::toFloat($d)
$s = $d->toString();            // php::Decimal::toString($d)
$b = $d->toBigInt();            // php::newBigInt(php::Decimal::toString($d))
$bf = $d->toBigFloat();         // php::BigFloat::newInstance(php::Decimal::toString($d))
```

### 3.3 BigFloat 上的转换

```php
$bf = std::bigFloat("1.23e100");

$i = $bf->toInt();              // php::BigFloat::toInt($bf)
$f = $bf->toFloat();            // php::BigFloat::toFloat($bf)
$s = $bf->toString();           // php::BigFloat::toString($bf)
$b = $bf->toBigInt();           // php::newBigInt(php::BigFloat::toString($bf))
$d = $bf->toDecimal();          // php::newDecimal(php::BigFloat::toString($bf))
```

### 3.4 跨 Big* 类型转换规则

Big* → String 始终通过各类型的 `toString()` 静态方法，避免二进制浮点误差：

| 源类型 | 目标类型 | 生成代码 |
|--------|---------|---------|
| BigInt | Decimal | `php::newDecimal(php::BigInt::toString($b))` |
| BigInt | BigFloat | `php::BigFloat::newInstance(php::BigInt::toString($b))` |
| Decimal | BigInt | `php::newBigInt(php::Decimal::toString($d))` |
| Decimal | BigFloat | `php::BigFloat::newInstance(php::Decimal::toString($d))` |
| BigFloat | BigInt | `php::newBigInt(php::BigFloat::toString($bf))` |
| BigFloat | Decimal | `php::newDecimal(php::BigFloat::toString($bf))` |

> 所有跨 Big* 转换都经过字符串中间态，确保十进制精度不丢失。

---

## 4. 对象类型转换

`toObject()` 将表达式的值转换为 `php::Object`。无参数时得到泛型对象（无具体类信息），传 `ClassName::class` 可重建带类信息的类型。

```php
declare(strict_types=1);
use native_types;

function object_convert(mixed $input): void
{
    // 无参数：泛型对象（无类信息）
    $obj = $input->toObject();

    // 带类名：编译器可优化后续方法调用
    $user = $input->toObject(User::class);
    echo $user->getName();   // Native Call
}
```

| 用法 | 生成代码 | 类型信息 |
|------|---------|---------|
| `$x->toObject()` | `php::toObject($x)` | 无（`php::Object`） |
| `$x->toObject(User::class)` | `php::toObject($x, ce_User, true)` | 有（编译器知道具体类） |

> **提示**：`toObject(ClassName::class)` 替代了已移除的 `objval()` 函数，建议在需要链式调用时优先使用。

---

## 5. Std 容器类型转换

`toStd*` 方法将 `php::Var` 类型的变量转换为指定的 C++ 标准库容器类型。这类方法**必须在顶层作用域**调用，且**不能重复赋值**已声明的变量。

```php
declare(strict_types=1);
use native_types;

function std_convert(): void
{
    $data = get_data();  // 返回 mixed，内部持有 StdContainerBox

    // 从 mixed 恢复为 std::vector<int>
    $v = $data->toStdVector(native_types::type_int);
    $v[] = 42;
    echo $v[0];  // 42

    // 从 mixed 恢复为 std::map<string, int>
    $m = $data->toStdMap(complex_types::type_string, native_types::type_int);
    $m["key"] = 100;
}
```

| 方法 | 目标类型 | 键类型限制 | 说明 |
|------|---------|-----------|------|
| `toStdArray(type, size)` | `php::StdArray<T, N>` | 索引为 `int` | 定长数组，大小在编译期确定 |
| `toStdVector(type)` | `php::StdVector<T>` | 索引为 `int` | 动态数组 |
| `toStdMap(ktype, vtype)` | `php::StdMap<K, T>` | `int` 或 `string` | 有序映射 |
| `toStdUnorderedMap(ktype, vtype)` | `php::StdUnorderedMap<K, T>` | `int` 或 `string` | 哈希映射 |

### 使用限制

- **必须在顶层作用域**：`toStd*` 只能在函数体的最外层作用域调用，不能在 `if` / `for` 等嵌套块中调用
- **不能重复赋值**：被 `toStd*` 赋值的变量不能再被赋值为其他类型
- **源变量必须已存在**：`toStd*` 必须作用在一个已定义且有值的变量上

---

## 6. 类型推断

编译器对 `to*` 方法的返回类型有精确的内置推断规则。无论 receiver 是何种类型，检测到 `to*` 方法时直接返回对应目标类型：

```php
$val = get_any_value();
// 编译器推断：$val->toString() 返回 php::Str，length() 是 Str 上的通用方法
echo $val->toString()->length();
```

类型推断覆盖以下场景：
- **链式调用**：`$a->toBigInt()->mul(3)->toString()` — 编译器逐级推断 BigInt → BigInt → Str
- **条件分支**：`if` 两个分支中 `to*` 的返回类型在不同分支中独立推断
- **函数返回值**：`return $x->toString()` → 函数返回类型为 `php::Str`

---

## 7. void 类型限制

返回类型为 `void` 的表达式不能调用任何方法（包括 `to*`），编译器会在编译期报错：

```php
function bar(): void
{
    var_dump(__FUNCTION__);
}

// ❌ 编译错误：Cannot call method on void
bar()->toString();
bar()->toInt();
```

> 这是一项编译期安全检查，防止在无意义的 void 表达式上链式调用方法。

---

## 8. 与通用方法的关系

`to*` 方法是**语言关键词**，而非普通的通用方法：

| 特征 | `to*` 关键词方法 | 普通通用方法 |
|------|-----------------|-------------|
| 方法查找 | 直接匹配内置表 `KEYWORD_METHOD_MAP` | 按类型查找 `UNIVERSAL_METHODS` 表 |
| receiver 类型 | 任意类型均可（包括 `mixed`） | 必须匹配已注册的类型 |
| 参数校验 | 编译期特殊处理 | 通用 `min_args` / `max_args` 校验 |
| 代码生成 | `genToConvertCall()` 专用逻辑 | 按 handler 类型分派 |

这种设计使 `to*` 方法在 `mixed` / `any` 类型上也能无歧义地工作，是实现类型接续（从动态类型恢复到静态类型）的核心机制。

---

## 9. 综合示例

```php
declare(strict_types=1);
use native_types;

function comprehensive_convert(): void
{
    // 链式转换 + 高精度运算
    $a = std::bigInt("100");
    $result = $a->toDecimal()
                ->mul(std::decimal("0.05"))
                ->toBigFloat()
                ->add(std::bigFloat("1.0"))
                ->toString();
    var_dump($result);  // 精确的十进制结果字符串

    // Stream 链式操作
    $pair = stream_socket_pair(AF_UNIX, SOCK_STREAM, 0);
    $pair[0]->toStream()->write("ping");
    $response = $pair[1]->toStream()->read(4);
    var_dump($response);  // string(4) "ping"

    // Std 容器提取
    $raw = get_container();                 // 返回 mixed
    $vec = $raw->toStdVector(native_types::type_int);
    $vec[] = 10;
    $vec[] = 20;
    echo $vec->count();                     // 2
}
```

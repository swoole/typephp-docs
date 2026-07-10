`to*` 是 `TypePHP` 编译器提供的关键词方法（`Keyword Method`），用于将表达式或变量的值显式转换为目标类型。与普通通用方法不同，`to*` 方法在编译器中具有一等公民地位——无论 receiver 为何种类型，编译器都按照内置规则进行转换，跳过类型方法表查找。

所有 `to*` 方法调用都会在编译期消解为明确的 C++ 函数调用。基础标量类型转换通常没有额外调度开销；对象类型转换如果携带类名，会在运行时执行对象类型检查。

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
| `toAny()` | `php::Var` | `php::Var($expr)` | 降级为动态类型，等价于 `any($expr)` |

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

## 2. 动态类型与引用转换

`toAny()` 和 `toRef()` 是 TypePHP 专有关键词方法，用于替代函数式写法 `any()` 和 `refval()`。二者完全等价，但方法形式更适合链式表达式。

### 2.1 `toAny()`

`toAny()` 将表达式降级为 `php::Var` / `mixed` / `any` 动态类型，等价于 `any($expr)`。它不会恢复对象类信息，也不会生成对象类型检查。

```php
declare(strict_types=1);
use native_types;

function any_example(object $value): void
{
    $a = any($value);
    $b = $value->toAny();  // 与 any($value) 等价
}
```

典型使用场景：

- 希望从 typed object 降级为动态值，避免后续按静态对象类型生成 native call。
- 需要让运算回到 ZendVM/PHP 动态语义，例如整数除法、混合类型运算等。
- 参数类型需要 `mixed` / `any`，但当前表达式是原生类型或 typed object。

`toAny()` 不接受任何参数：

```php
$value->toAny();       // ✅
$value->toAny($type);  // ❌ 编译错误
```

### 2.2 `toRef()`

`toRef()` 将表达式显式转换为引用，等价于 `refval($expr)`。它主要用于动态调用、闭包调用等编译期无法获知参数是否按引用传递的场景。

```php
function append_text(&$value, string $suffix): void
{
    $value .= $suffix;
}

function ref_example(): void
{
    $name = 'AOT';

    append_text(refval($name), ' compiler');
    append_text($name->toRef(), ' runtime'); // 与 refval($name) 等价
}
```

`toRef()` 只能用于编译器可定位的左值，例如变量、数组元素、对象属性：

```php
$value->toRef();        // ✅ 变量
$array['key']->toRef(); // ✅ 数组元素
$object->prop->toRef(); // ✅ 对象属性

(1 + 2)->toRef();       // ❌ 常量/临时表达式不能转引用
foo()->toRef();         // ❌ 调用结果不能转引用
```

与 `refval()` 一样，`toRef()` 不接受任何参数：

```php
$value->toRef();     // ✅
$value->toRef(true); // ❌ 编译错误
```

> **提示**：静态函数和内置函数的参数信息明确时，编译器可以自动处理引用参数。只有动态调用、闭包调用、可变函数调用等无法在编译期获得参数签名的场景，才需要显式使用 `toRef()` / `refval()`。

---

## 3. Stream 类型转换

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

## 4. 高精度数值类型转换

Big* 类型（BigInt / Decimal / BigFloat）定义了更精确的转换路径，编译器针对每种源类型使用专用的转换函数，避免精度损失。

### 4.1 BigInt 上的转换

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

### 4.2 Decimal 上的转换

```php
$d = std::decimal("123.456");

$i = $d->toInt();               // php::Decimal::toInt($d)
$f = $d->toFloat();             // php::Decimal::toFloat($d)
$s = $d->toString();            // php::Decimal::toString($d)
$b = $d->toBigInt();            // php::newBigInt(php::Decimal::toString($d))
$bf = $d->toBigFloat();         // php::BigFloat::newInstance(php::Decimal::toString($d))
```

### 4.3 BigFloat 上的转换

```php
$bf = std::bigFloat("1.23e100");

$i = $bf->toInt();              // php::BigFloat::toInt($bf)
$f = $bf->toFloat();            // php::BigFloat::toFloat($bf)
$s = $bf->toString();           // php::BigFloat::toString($bf)
$b = $bf->toBigInt();           // php::newBigInt(php::BigFloat::toString($bf))
$d = $bf->toDecimal();          // php::newDecimal(php::BigFloat::toString($bf))
```

### 4.4 跨 Big* 类型转换规则

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

## 5. 对象类型转换

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
| `$x->toObject(User::class)` | `php::toObject($x, ce_User)` | 有（编译器知道目标类） |

> **提示**：`toObject(ClassName::class)` 和 `objval($var, ClassName::class)` 均可用于对象类型接续，前者支持链式调用。带类名的对象转换使用 PHP `instanceof` / `is-a` 关系进行运行时检查，详见 [对象类型转换](object-type-conversion.md)。

---

## 6. Std 容器类型转换

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
    $m = $data->toStdOrderedMap(complex_types::type_string, native_types::type_int);
    $m["key"] = 100;
}
```

| 方法 | 目标类型 | 键类型限制 | 说明 |
|------|---------|-----------|------|
| `toStdArray(type, size)` | `php::StdArray<T, N>` | 索引为 `int` | 定长数组，大小在编译期确定 |
| `toStdVector(type)` | `php::StdVector<T>` | 索引为 `int` | 动态数组 |
| `toStdOrderedMap(ktype, vtype)` | `php::StdOrderedMap<K, T>` | `int` 或 `string` | 有序映射 |
| `toStdMap(ktype, vtype)` | `php::StdMap<K, T>` | `int` 或 `string` | 哈希映射 |

### 使用限制

- **必须在顶层作用域**：`toStd*` 只能在函数体的最外层作用域调用，不能在 `if` / `for` 等嵌套块中调用
- **不能重复赋值**：被 `toStd*` 赋值的变量不能再被赋值为其他类型
- **源变量必须已存在**：`toStd*` 必须作用在一个已定义且有值的变量上

---

## 7. 类型推断

编译器对 `to*` 方法的返回类型有精确的内置推断规则。无论 receiver 是何种类型，检测到 `to*` 方法时直接返回对应目标类型：

```php
$val = get_any_value();
// 编译器推断：$val->toString() 返回 php::Str，length() 是 Str 上的通用方法
echo $val->toString()->length();
```

类型推断覆盖以下场景：
- **链式调用**：`$a->toBigInt()->mul(3)->toString()` — 编译器逐级推断 BigInt → BigInt → Str
- **动态降级**：`$obj->toAny()` — 编译器将后续值视为 `php::Var`
- **显式引用**：`$value->toRef()` — 编译器将参数按引用传递
- **条件分支**：`if` 两个分支中 `to*` 的返回类型在不同分支中独立推断
- **函数返回值**：`return $x->toString()` → 函数返回类型为 `php::Str`

---

## 8. void 类型行为

返回类型为 `void` / `never` 的表达式作为值使用时会按 `null` 处理。对这类表达式继续调用 `to*` 方法没有实际意义，结果等价于对 `null` 做对应转换。

```php
function bar(): void
{
    var_dump(__FUNCTION__);
}

$value = bar()->toAny();     // 等价于 any(null)
$text = bar()->toString();   // 等价于 php::toString(null)
```

---

## 9. 与通用方法的关系

`to*` 方法是**语言关键词**，而非普通的通用方法：

| 特征 | `to*` 关键词方法 | 普通通用方法 |
|------|-----------------|-------------|
| 方法查找 | 直接匹配内置表 `KEYWORD_METHOD_MAP` | 按类型查找 `UNIVERSAL_METHODS` 表 |
| receiver 类型 | 任意类型均可（包括 `mixed`） | 必须匹配已注册的类型 |
| 参数校验 | 编译期特殊处理 | 通用 `min_args` / `max_args` 校验 |
| 代码生成 | `genToConvertCall()` 专用逻辑 | 按 handler 类型分派 |

这种设计使 `to*` 方法在 `mixed` / `any` 类型上也能无歧义地工作，是实现类型接续（从动态类型恢复到静态类型）的核心机制。

---

## 10. 综合示例

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

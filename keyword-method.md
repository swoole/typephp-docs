# 关键词方法

关键词方法（keyword method）是编译器内置的一等公民方法，调用时跳过类型方法表查找，直接由编译器匹配内置规则生成 C++ 代码。无论 receiver 是何种类型（包括 `mixed` / `any` / `var`），关键词方法都能无歧义地工作。

所有关键词方法调用在编译时由编译器直接识别，不会产生动态方法分派开销。部分关键词方法仍会执行必要的运行时转换或检查，例如 `toObject(ClassName::class)` 会校验对象是否满足目标类型。


## 关键词方法示例

```php
$var->toString(); 
$data[0]->toInt();
$obj->attr->toStream();
echo $array['object']->toObject(User::class)->getName();
```

---

## 1. 内置关键词方法表

### 1.1 类型转换方法

| 方法 | 目标类型 | 生成代码 | 说明 |
|------|---------|---------|------|
| `toInt()` | `php::Int` | `php::toInt($expr)` | 转为整数 |
| `toFloat()` | `php::Float` | `php::toFloat($expr)` | 转为浮点数 |
| `toString()` | `php::Str` | `php::toString($expr)` | 转为字符串 |
| `toBool()` | `php::Bool` | `php::toBool($expr)` | 转为布尔值 |
| `toArray()` | `php::Array` | `php::toArray($expr)` | 转为数组；若 receiver 为对象且有 `toArray()` 方法，优先调用该方法 |
| `toAny()` | `php::Var` | `php::Var($expr)` | 降级为动态类型；等价于 `any($expr)` |
| `toRef()` | `php::Ref` | `$expr.toReference()` / `php::toReference(...)` | 显式转为引用；等价于 `refval($expr)` |
| `toStream()` | `php::Stream` | `php::toStream($expr)` | 转为流类型 |
| `toBigInt()` | `php::BigInt` | `php::toBigInt($expr)` | 转为高精度整数 |
| `toBigFloat()` | `php::BigFloat` | `php::toBigFloat($expr)` | 转为高精度浮点数 |
| `toDecimal()` | `php::Decimal` | `php::toDecimal($expr)` | 转为高精度十进制数 |
| `toObject()` | `php::Object` | `php::toObject($expr)` | 转为对象；支持 `toObject(ClassName::class)` 传类名重建类型信息 |

这些方法在任何类型的表达式上均可调用。`toInt()` / `toFloat()` / `toString()` 等基础转换等价于 PHP 的类型强制转换语法，但支持链式调用和精确的编译期类型推断；`toAny()` 等价于 `any()`；`toRef()` 等价于 `refval()`。详见 [类型转换](type-convert.md)。

`toAny()` 和 `toRef()` 均不接受参数：

```php
$value->toAny();  // ✅ any($value)
$value->toRef();  // ✅ refval($value)
```

`toRef()` 只能作用于变量、数组元素、对象属性等可定位左值。动态调用、闭包调用等无法在编译期获知参数是否为引用时，可使用 `toRef()` 显式传引用。

### 1.2 Std 容器转换方法

| 方法 | 目标类型 | 说明 |
|------|---------|------|
| `toStdArray(type, size)` | `php::StdArray<T, N>` | 从 Var 恢复定长数组引用，编译期校验类型 |
| `toStdVector(type)` | `php::StdVector<T>` | 从 Var 恢复动态数组引用 |
| `toStdOrderedMap(keyType, valueType)` | `php::StdOrderedMap<K, V>` | 从 Var 恢复有序映射引用 |
| `toStdMap(keyType, valueType)` | `php::StdMap<K, V>` | 从 Var 恢复哈希映射引用 |

`toStd*` 方法必须在顶层作用域调用，且目标变量不能重复赋值。这些方法内部调用 `php::toStdContainer<T>()`，提取 Box 资源中的容器引用并做运行时类型 ID 校验，返回零拷贝引用。

```php
// 被调方：从 Var 参数恢复容器引用
function process($data): void {
    $v = $data->toStdVector(native_types::type_int);
    $v[] = 42;  // 直接修改调用方的原容器
}
```

详见 [Std 容器](std-containers.md)。

### 1.3 方法查找优先级

当 receiver 类型为 `mixed` / `any` / `var` 时，方法查找顺序为：

1. **内置关键词方法** — 上表中的 `to*` 和 `toStd*` 方法
2. **扩展关键词方法** — 根命名空间 `__` 开头的函数（见下文第 2 节）
3. **类型扩展方法** — 按 String → Array → Int → Float → Bool → Stream → BigInt → Decimal → BigFloat 顺序查找
4. **动态调用** — 退化为 ZendVM 方法调用

---

## 2. 扩展关键词方法

扩展关键词方法允许用户通过定义符合约定的 PHP 函数，在**任意类型**（`mixed` / `any` / `var`）上添加可链式调用的方法。

### 2.1 命名约定

在**根命名空间**中定义 `__` 开头的函数，方法调用时**驼峰命名自动转为下划线命名**：

```
方法调用:  $receiver->camelCaseName()
    ↓ 驼峰转下划线
函数查找:  __camel_case_name($receiver, ...)
```

```php
// 定义
function __var_dump(mixed $var): void {
    var_dump($var);
}

// 调用
$str = "hello";
$str->varDump();  // 输出 string(5) "hello"

$arr = [1, 2, 3];
$arr->varDump();  // 输出 array(3) { [0]=> int(1) ... }
```

### 2.2 函数签名要求

必须同时满足以下三个条件，编译器才会将其注册为关键词扩展方法：

1. **根命名空间**：函数必须定义在 `namespace` 为空的作用域中
2. **`__` 前缀**：函数名必须以双下划线 `__` 开头
3. **第一个参数为 `mixed` / `any`**：第一个参数的类型必须是 `mixed` 或 `any`（对应 `php::Var`）

```php
// ✅ 合法
function __debug(mixed $var): void {
    print_r($var);
}

// ✅ 合法 — any 与 mixed 等效
function __to_json(any $var): string {
    return json_encode($var, JSON_UNESCAPED_UNICODE);
}

// ❌ 有命名空间 — 不会被注册
namespace Utils;
function __debug(mixed $var): void { ... }

// ❌ 第一个参数不是 mixed/any — 不会被注册
function __to_upper(string $var): string { ... }

// ❌ 没有 __ 前缀 — 不会被注册
function debug(mixed $var): void { ... }
```

### 2.3 参数传递规则

方法调用的参数从**第二个参数开始传递**——第一个参数自动作为 receiver 传入：

```php
declare(strict_types=1);
use native_types;

// 定义：2 个参数（第 1 个是 receiver）
function __assert_type(mixed $var, string $expected): bool {
    return get_debug_type($var) === $expected;
}

// 定义：3 个参数（第 1 个是 receiver）
function __between(mixed $var, int $min, int $max): bool {
    return $var >= $min && $var <= $max;
}

function main(): void {
    $val = 42;

    // 调用时传 1 个参数 → 对应 $expected
    var_dump($val->assertType("int"));  // bool(true)

    // 调用时传 2 个参数 → 对应 $min, $max
    var_dump($val->between(0, 100));   // bool(true)
}
```

### 2.4 返回类型推断

返回类型直接从函数声明推断，支持所有编译器内置类型：

```php
function __to_json(mixed $var): string {
    return json_encode($var, JSON_UNESCAPED_UNICODE);
}

function __length(mixed $var): int {
    if (is_string($var)) return strlen($var);
    if (is_array($var)) return count($var);
    return 0;
}

// 编译器精确推断返回类型
$json = $obj->toJson();     // → php::Str，可继续调用 String 通用方法
echo $json->length();       // 编译器精确推断 length() 返回 Int
```

### 2.5 可变参数

如果函数使用 `...` 可变参数，方法调用时参数数量无上限：

```php
function __log_all(mixed ...$vars): void {
    foreach ($vars as $v) {
        error_log((string) $v);
    }
}

$logger->logAll("msg1", "msg2", "msg3");  // → __log_all($logger, "msg1", "msg2", "msg3")
```

### 2.6 与内置关键词方法的优先级

内置关键词方法**优先于**扩展关键词方法。如果定义了与内置方法同名的扩展方法，内置方法优先生效：

```php
// 内置 toArray 是关键词方法，优先级最高
// 即使定义了 __to_array，也不会覆盖内置行为
function __to_array(mixed $var): array {
    return [$var];
}

function main(): void {
    $obj = new User();
    $obj->toArray();     // → php::toArray($obj)（内置，不是 __to_array）
}
```

> **建议**：避免定义与内置关键词方法驼峰名冲突的扩展方法（如 `varDump` 不会冲突，但 `toArray` / `toString` 等会与内置方法形成后缀冲突）。

### 2.7 完整示例

```php
<?php
declare(strict_types=1);
use native_types;

// 调试输出
function __var_dump(mixed $var): void {
    var_dump($var);
}

// JSON 序列化
function __to_json(mixed $var, int $flags = 0): string {
    return json_encode($var, JSON_UNESCAPED_UNICODE | $flags);
}

// 类型检查
function __is_of_type(mixed $var, string $type): bool {
    return get_debug_type($var) === $type;
}

function main(): void {
    $str = "hello world";
    $str->varDump();       // 调试输出

    $arr = ["name" => "test", "value" => 42];
    echo $arr->toJson();   // {"name":"test","value":42}

    $num = 123;
    var_dump($num->isOfType("int"));  // bool(true)
}
?>
```

---

## 3. 与通用方法的区别

| 特征 | 关键词方法 | 通用方法 |
|------|-----------|---------|
| 方法查找 | 直接匹配 `KEYWORD_METHOD_MAP` 或 `__` 函数表 | 按类型查找 `UNIVERSAL_METHODS` 表 |
| receiver 类型 | **任意类型**（包括 `mixed` / `any`） | 必须匹配已注册的具体类型 |
| 注册方式 | 内置（`KEYWORD_METHOD_MAP`）或根命名空间 `__` 函数 | 按类型前缀注册 |
| 返回类型推断 | 内置表直接确定 / 函数声明推断 | 由方法 handler 定义 |
| 典型用途 | 类型转换、Std 容器恢复、跨类型工具方法 | 专属于特定类型的方法 |

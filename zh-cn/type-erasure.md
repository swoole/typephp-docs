# 类型擦除与恢复

TypePHP 在表达式类型确定时使用固定存储，需要保留动态性时使用 `php::Var`。类型擦除是从确定类型进入 `php::Var` 的过程；类型恢复则是在动态值重新进入固定存储前，显式执行转换或类型检查。

```text
固定类型  -- std::any() / toAny() -->  php::Var
固定类型  <-- 显式转换或断言 ---------- php::Var
```

恢复过程不会记忆一个值曾经是什么类型。程序必须明确指出当前期望的类型，TypePHP 再执行对应的转换或运行时检查。

## 显式类型擦除

单个值需要进入 PHP 动态存储模型时，使用 `std::any($value)` 或等价的 `toAny()` 关键词方法：

```php
$count = 1;                    // 固定 int
$value = std::any($count);     // 保存 int(1) 的 php::Var
$same = $count->toAny();       // 等价的表达式写法
```

`std::any()` 可以省略参数，此时创建一个初始值为 `null` 的动态存储：

```php
$result = std::any();
$result = load_value();
```

擦除后，值可以改变 PHP 类型，运算与调用走 Zend 动态语义。恢复类型之前，编译器不能再为它应用固定标量或 typed object 的 Native Call 规则。

Native Class 对象和保存 Native 元素的 Std 容器不能擦除为 `php::Var`，因为其中的指针必须留在编译器管理的原生对象图内。

## 天然具有动态类型的存储

部分存储位置用于跨越 PHP 动态边界，因此会暴露 `php::Var` 值。

### global 与 static 动态槽位

通过 `global` 或 `$GLOBALS[...]` 取得的值通常是动态槽位，除非编译器已为它建立独立的固定原生槽位。声明为 `mixed`、未标注类型或使用 `std::any()` 初始化的 static 局部变量、静态属性同样是动态值，并会在多次访问之间保留状态。

```php
class Registry
{
    public static mixed $service = null;
}

function register(Service $service): void
{
    Registry::$service = std::any($service);
}

function service(): Service
{
    return std::object(Registry::$service, Service::class);
}
```

初始值类型确定的 static 标量、声明了明确类型的静态属性可以保留固定或声明类型存储。不能仅因为出现 `static` 关键词就认定值一定被擦除，实际槽位由声明和初始值决定。

### 普通 PHP 数组元素

普通 PHP 数组本身由 `php::Array` 保存，但每个元素值都使用动态 `php::Var` 槽位；键遵循 PHP 的整数/字符串键规则。从数组读取元素时，通常会丢失具体对象类信息或固定标量上下文：

```php
$users = ['owner' => new User()];
$owner = std::object($users['owner'], User::class);
```

普通 PHP 数组需要编译期键值约束时使用 [std::list / std::dict 及其类型注解](typed-arrays.md)；需要元素类型和原生布局始终固定时使用 [Std 容器](std-containers.md)。

其他常见动态边界还包括未标注或 `mixed` 参数与返回值、动态函数或方法调用、返回通用 PHP 值的扩展 API，以及未标注类型的对象属性。

## 恢复固定类型

根据目标类型选择恢复操作：

| 目标类型 | 恢复写法 | 行为 |
|---|---|---|
| `int` | `(int) $value`、`intval($value)`、`$value->toInt()`、`std::int($value)` | 转换为固定原生 int |
| `float` | `(float) $value`、`floatval($value)`、`$value->toFloat()`、`std::float($value)` | 转换为固定原生 float |
| `bool` | `(bool) $value`、`boolval($value)`、`$value->toBool()`、`std::bool($value)` | 转换为固定原生 bool |
| `string` | `(string) $value`、`strval($value)`、`$value->toString()` | 转换为 `php::Str` |
| `array` | `(array) $value`、`$value->toArray()` | 转换为 `php::Array` |
| 具体对象 | `std::object($value, User::class)` 或 `$value->toObject(User::class)` | 检查 `instanceof User` 并恢复类信息 |
| 通用对象 | `$value->toObject()` | 转换/检查为无具体类信息的 `php::Object` |
| Stream | `$value->toStream()` | 恢复流资源 |
| 高精度数值 | `toBigInt()`、`toDecimal()`、`toBigFloat()` 或对应 `std::*` 构造入口 | 构造指定的固定数值类型 |
| Std 容器 | `toStdArray()`、`toStdVector()`、`toStdMap()`、`toStdOrderedMap()` | 在首次顶层赋值时建立固定容器类型 |

具名函数和方法参数可通过 [类型注解](std-container-parameters.md) 自动恢复 vector、map 或 ordered map 类型，例如 `#[StdVector(Type::Int)] $values`。PHP 参数类型可省略或为 `box`，不允许 `mixed`；Box 传递及动态调用转换规则保持不变。

```php
function consume(mixed $payload): void
{
    $id = $payload['id']->toInt();
    $name = $payload['name']->toString();
    $user = std::object($payload['user'], User::class);

    $user->update($id, $name);
}
```

标量恢复会按照所选操作执行转换。具体对象恢复是断言加运行时 `is-a` 检查；值不兼容时会抛出类型错误，而不会把它转换成无关对象。`std::object()` 与 `toObject(ClassName::class)` 在 TypePHP 中语义相同；源码还需要通过用户提供的 polyfill 运行于 Zend PHP 时，应优先使用 `std::object()`。

## 类型擦除不是引用转换

`std::any()` 改变存储与类型模型，`std::ref()` 只标记一个可引用的调用参数。引用传递详见[强类型引用](strong-references.md)。需要完整 PHP 引用身份的代码，可以先用 `std::any()` 明确建立动态存储，再使用动态引用路径。

完整转换 API 详见[基础类型转换](type-convert.md)与[关键词方法](keyword-method.md)。

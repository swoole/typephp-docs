# 原生类

原生类（Native Class）是 TypePHP 为极致性能场景提供的独立对象模型。使用 `#[Native]` 后，类会被编译为具有固定字段布局的 C++ 对象；属性直接访问字段，确定的方法调用直接进入 TypePHP 生成的 `php_*` 函数。

原生类不注册到 ZendVM，不创建 `zend_class_entry`、Zend Object、动态属性表或 Reflection 元数据。它不是普通 PHP class 的自动优化模式，而是开发者主动选择、功能边界更严格的静态类型。

## 第一个原生类

```php
#[Native]
class Point
{
    public int $x = 0;
    public int $y = 0;

    public function __construct(int $x, int $y)
    {
        $this->x = $x;
        $this->y = $y;
    }

    public function sum(): int
    {
        return $this->x + $this->y;
    }
}

function move(Point $point, int $x, int $y): void
{
    $point->x = $x;
    $point->y = $y;
}

function main(): void
{
    $point = new Point(10, 20);
    move($point, 30, 40);
    echo $point->sum(); // 70
}
```

`#[Native]` 位于根命名空间，并遵循[编译期注解](compile-time-attributes.md)的名称解析规则。该 Attribute 只能用于具名 class，不能用于匿名类、Interface、Trait 或 Enum。

## 性能模型

原生类的热路径具有确定成本：

- 对象变量是一个原生指针；
- `$a = $b` 只复制指针，不复制对象；
- 属性读写是固定偏移的 C++ 字段访问，不查属性名称；
- 方法继续使用 TypePHP 的函数式 ABI，第一个参数是确定的 `this` 对象；
- 确定方法调用不经过 `zend_call_function()`；
- visibility、类型和 Interface 契约尽可能在编译期检查；
- 没有动态装箱、Reflection 和隐藏的 ZendVM fallback。

原生类适合大量小对象、对象图、计算模型和高频属性访问。需要 Reflection、动态属性、序列化或把对象交给普通 PHP 扩展时，应继续使用普通 class。

## 属性声明与默认值

每个实例属性都必须显式声明类型：

```php
#[Native]
final class RequestContext
{
    public bool $ready = false;
    public int $status = 0;
    public float $elapsed = 0.0;
    public string $method = '';
    public array $headers = [];
    public object $request;
    public Stream $body;
    public mixed $metadata = null;
}
```

未写默认值的属性使用确定零值，而不是 PHP typed property 的未初始化状态：

| 属性类型 | 初始值 |
|---|---|
| `bool` | `false` |
| `int` / `float` | `0` / `0.0` |
| `string` | 空字符串 |
| `array` | 空数组 |
| `mixed` / `any` | `null` |
| Zend Object、Native Object | 空指针 |
| `Stream` | 空 resource 状态 |

支持 bool、int、float、string、array、object、确定的普通 class、其他 Native Class、Stream、mixed/any、合法的非 Native union/nullable 类型，以及 BigInt、BigFloat、Decimal。

以下类型不能作为原生类属性：

- `Box`；
- `std::array`、`std::vector`、`std::map`、`std::ordered_map` 等 Std 容器；
- PHP 属性声明本身不允许的 `void`、`never`、`callable` 等类型；
- `readonly` 属性和 readonly class。

Std 容器可以在函数局部保存具体 Native Class 的对象，但不能嵌入原生类属性。普通 PHP array 不能保存 Native Object。

### 属性引用

只有显式声明为 `any` 的属性允许取引用：

```php
#[Native]
class Holder
{
    public any $dynamic = null;
    public string $name = '';
}

$ref =& $holder->dynamic; // 允许
$ref =& $holder->name;    // 编译错误
```

即使 string、array、object 或 mixed 在底层具有 PHPX 包装层，也不能取引用；动态代码可能通过引用写入另一种类型，破坏固定字段约束。Native Object 变量本身同样不能取引用，因为对象变量已经具有共享身份语义。

## 对象身份、null 与 clone

Native Object 保留 PHP 对象常用的身份语义：

```php
$a = new Point(1, 2);
$b = $a;
$b->x = 10;
echo $a->x; // 10

$copy = clone $a;
$copy->x = 99;
echo $a->x; // 10
```

`$a = $b` 不复制实体；`clone` 才执行字段浅复制，并在存在时调用 `__clone()`。

`unset($object)` 和 `$object = null` 只清空当前变量槽，不会清空真实对象，也不影响其他别名。对空指针调用方法会抛出 `Error`。`?NativeClass` 使用空指针表示 null，并支持纯 Native 链上的 nullsafe `?->`。

条件表达式只判断对象指针是否非空，不调用 `toBool()`：

```php
if ($point) {
    // 只表示 $point 不是 null
}
```

`===` 和 `!==` 比较对象身份，也支持与 `null` 比较。`==`、`!=`、大小比较、算术和位运算没有可靠的 Zend 对象语义，因此会产生编译错误；需要值相等时应声明明确的普通方法。

## 参数和返回值

Native Object 只能通过显式的 Native Class 类型传递：

```php
function required(Point $point): Point
{
    return $point;
}

function optional(?Point $point): ?Point
{
    return $point;
}
```

`Point $point` 在函数入口保证不是 null；`?Point $point` 才允许 null。参数按指针值传递，修改对象对调用方可见，而重新给局部参数赋值不会替换调用方的变量槽。

以下签名不支持：

- Native Object 参数或返回值使用 `&`；
- Native Object variadic 参数；
- 包含 Native Class 的 union 或 intersection；
- 用 `mixed`、`object` 或 Interface 承载 Native Object。

Native Object 只能传给 TypePHP 能静态解析、且参数明确声明为相同 Native Class 或其 Native 基类的函数/方法。它不能传给普通 PHP 函数、扩展函数、匿名函数或动态 callable。

## 继承、Trait 与 Interface

原生类支持单继承、abstract class、abstract method、final 和方法覆盖。父类和子类都必须显式声明 `#[Native]`，原生类与普通 ZendVM class 不能互相继承：

```php
#[Native]
abstract class Shape
{
    abstract public function area(): float;
}

#[Native]
final class Rectangle extends Shape
{
    public float $width = 0.0;
    public float $height = 0.0;

    public function area(): float
    {
        return $this->width * $this->height;
    }
}
```

继承关系需要动态分派时，TypePHP 生成有限的 C++ virtual thunk；方法主体仍是普通 `php_*` 函数。PHP 不支持同一 class 中按参数签名声明多个同名方法，原生类也不增加 C++ 风格的方法重载。

Trait 会在编译期把属性、常量、方法和 Property Hook AST 注入原生类，随后像类自身成员一样编译。

Interface 本身仍是普通 PHP Interface，并注册到 ZendVM。原生类可以 `implements` Interface，编译器会检查方法、签名和 Property Hook 契约，但这个关系只在编译期存在：

```php
#[Native]
class NativeCounter implements Countable
{
    public function count(): int
    {
        return 1;
    }
}

$counter = new NativeCounter();
echo count($counter); // 直接调用 NativeCounter::count()
```

Native Object 不能赋值或传递为 Interface 类型，ZendVM 的 Reflection 也看不到该实现关系。需要多态传参时，应使用共同的 Native 基类；需要复用实现时可使用 Trait。

## Iterator 与 ArrayAccess

实现 `Iterator` 或 `IteratorAggregate` 后可以直接用于 `foreach`。TypePHP 在编译期把循环转换为对应的方法调用：

```php
#[Native]
class RangeIterator implements Iterator
{
    public int $position = 0;
    public int $limit = 0;

    public function __construct(int $limit) { $this->limit = $limit; }
    public function rewind(): void { $this->position = 0; }
    public function valid(): bool { return $this->position < $this->limit; }
    public function current(): int { return $this->position; }
    public function key(): int { return $this->position; }
    public function next(): void { $this->position++; }
}

foreach (new RangeIterator(3) as $value) {
    echo $value;
}
```

没有实现 Iterator/IteratorAggregate 的 Native Object 不能 `foreach`，TypePHP 不会像普通 PHP 对象那样枚举 public 属性。引用遍历也不支持。

实现 `ArrayAccess` 后，读取、赋值、追加、`isset()`、`empty()`、`??` 和 `unset()` 会直接映射为 `offsetGet/offsetSet/offsetExists/offsetUnset` 调用。为了避免引用和临时值语义不明确，不支持元素的间接修改，例如：

```php
$bag['count']++;          // 不支持
$bag['count'] += 1;       // 不支持
$bag['nested']['x'] = 1;  // 不支持
$ref =& $bag['value'];    // 不支持
```

## Property Hook 与生成器注解

Native Class 支持 PHP 8.4 Property Hook 的直接 get/set。Hook 在编译期变成确定的方法调用，不经过 Zend property handler。间接写入、复合写入、取引用以及对 Hook 使用 `isset()`/`empty()` 不支持。

Getter、Setter、Constructor、Printer、Arrayable 等只依赖编译期 AST 生成的 Attribute 可以作用于 Native Class；生成结果与手写方法一样进入 Native Call。具体组合限制仍以各 Attribute 页面为准。

`readonly` 不支持。PHP readonly 依赖 Zend 属性初始化状态和运行时 property handler，与原生类的固定字段模型不同。

## 类型转换与语言集成

原生类不会调用 PHPX 的动态关键词转换 helper。类必须实际声明相应零参数方法，而且返回类型必须完全一致：

```php
#[Native]
class Result
{
    public int $value = 0;

    public function toArray(): array { return ['value' => $this->value]; }
    public function toInt(): int { return $this->value; }
    public function toBool(): bool { return $this->value !== 0; }
    public function toString(): string { return (string) $this->value; }
}

$result = new Result();
echo (string) $result;
$json = json_encode($result->toArray());
```

支持 `toArray()`、`toString()`、`toInt()`、`toFloat()` 和 `toBool()` 等关键词方法。`__toString(): string` 可以作为 `toString()` 的别名；字符串强转、`strval()`、拼接和 `echo` 使用同一规则。缺少对应方法或返回类型不匹配时，编译器直接报错。

实现 `Countable` 后，单参数 `count($object)` 映射为确定的 `count()` 方法调用。`__invoke()` 也支持确定调用。

`json_encode()` 不接受 Native Object。先显式调用 `toArray()` 可以清楚展示对象图转换和内存分配边界。

## 生命周期与 GC

Native Object 不使用 Zend 引用计数或 `std::shared_ptr`。TypePHP/PHPX 使用独立的 tracing GC 跟踪 Native Object 字段、局部变量、TypePHP global/static slot 和受支持的局部 Std 容器，因此对象之间可以循环引用。

完整的对象布局、root frame、回收阈值、对象复活、Fiber 和 request shutdown 流程参见[GC 机制：原生类对象的 tracing GC](gc.md#6-原生类对象的-tracing-gc)。

GC 自动运行，不提供必须由业务代码调用的收集接口。对象不可达或 request shutdown 时会执行 `__destruct()`，每个对象最多一次；继承链按最派生类到基类执行。与 PHP 引用计数对象不同，最后一个变量离开作用域时不保证立即执行析构，也不保证多个不可达对象之间的析构顺序。

因此不要依赖 `unset($object)` 立即关闭文件或提交事务。需要确定时机的资源应提供显式 `close()`、`commit()` 等方法，`__destruct()` 只作为最终清理保障。

TypePHP global 和 static local 可以保存 Native Object；ZTS 下使用 request/thread 隔离的原生 slot，并在 request shutdown 清理。字面量或编译期可求值字符串形式的 `$GLOBALS['name']` 可映射到同一 slot，动态 `$GLOBALS[$name]` 不支持 Native Object。

## ZendVM 边界与不支持功能

Native Object 没有 zval 表示。编译器会拒绝它跨越 ZendVM 边界，而不会静默装箱或退化为普通对象。不能：

- 放入普通 PHP array、普通对象属性、mixed/object 变量或 Box；
- 传给未知 PHP/扩展函数、动态方法、动态 callable 或 `call_user_func()`；
- 捕获进 Zend Closure，或作为 TypePHP Generator 的参数、局部变量、返回值、`this` 或 `yield` 值；
- 使用 Reflection、WeakReference、serialize 或直接传给 `json_encode()`；
- 在 `eval()` 或动态 PHP 代码中访问；
- 使用变量方法名 `$object->$method()`、动态属性或动态 class 创建；
- 使用 `get_class()`、`get_parent_class()`、`get_called_class()` 等运行时 class introspection。

以下动态魔术方法也不支持：

- `__call()`、`__callStatic()`；
- `__get()`、`__set()`、`__isset()`、`__unset()`；
- `__sleep()`、`__wakeup()`、`__serialize()`、`__unserialize()`；
- `__set_state()`、`__debugInfo()`。

所有这些限制都会在编译期报告 FatalError。若业务需要上述能力，请使用普通 class；不要把原生类当作普通 PHP 对象的无条件替代品。

## 选择建议

优先使用普通 class，只有在性能分析确认对象布局、属性查找或动态分派是热点时，再把边界清晰的内部数据结构改为 `#[Native]`。

适合原生类的代码通常具备以下特征：

- 对象只在 TypePHP 静态代码内流转；
- 类和调用目标都可以在编译期确定；
- 属性读写或方法调用频率很高；
- 不依赖 Reflection、序列化、动态调用和 PHP 扩展对象接口；
- 可以接受 tracing GC 的析构时机。

只要对象需要进入 PHP 框架、扩展函数或动态 API，普通 class 通常是更可靠的选择。

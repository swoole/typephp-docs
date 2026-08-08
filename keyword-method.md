# 关键词方法

关键词方法（Keyword Method）是 TypePHP 编译器内置的方法语法。它们不依赖对象中是否真正声明了同名方法，而是由编译器直接识别并生成对应代码。

关键词方法主要用于类型转换和类型接续：

```php
$id = $value->toInt();
$name = $value->toString();
$user = $value->toObject(User::class);
$stream = $value->toStream();
```

关键词方法可以作用于变量、数组元素、属性和其他表达式：

```php
$data['id']->toInt();
$response->body->toString();
$repository->find($id)->toObject(User::class);
```

## 1. 内置关键词方法

| 方法 | 返回类型 | 参数 | 说明 |
|------|----------|------|------|
| `toInt()` | `int` | 无 | 转换为整数 |
| `toFloat()` | `float` | 无 | 转换为浮点数 |
| `toString()` | `string` | 无 | 转换为字符串 |
| `toBool()` | `bool` | 无 | 转换为布尔值 |
| `toArray()` | `array` | 无 | 转换为数组 |
| `toAny()` | `mixed` | 无 | 将静态类型降级为动态类型 |
| `toRef()` | 引用 | 无 | 显式取得可引用表达式的引用 |
| `toStream()` | `stream` | 无 | 转换为 Stream 类型 |
| `toObject()` | `object` | 可选类名 | 转换为对象或恢复具体对象类型 |
| `toBigInt()` | `BigInt` | 无 | 转换为高精度整数 |
| `toBigFloat()` | `BigFloat` | 无 | 转换为高精度浮点数 |
| `toDecimal()` | `Decimal` | 无 | 转换为高精度十进制数 |
| `toStdArray()` | `StdArray` | 类型、长度 | 恢复定长 Std 容器 |
| `toStdVector()` | `StdVector` | 元素类型 | 恢复动态 Std 容器 |
| `toStdMap()` | `StdMap` | 键和值类型 | 恢复哈希映射 |
| `toStdOrderedMap()` | `StdOrderedMap` | 键和值类型 | 恢复有序映射 |

内置关键词方法优先级最高，用户代码不能通过对象扩展或关键词扩展覆盖它们。

## 2. 基础类型转换

### 2.1 toInt

```php
$id = '123'->toInt();
var_dump($id); // int(123)

$enabled = true->toInt();
var_dump($enabled); // int(1)
```

等价于将表达式转换为 TypePHP 的原生整数类型。返回值可继续调用 Int 通用方法：

```php
$result = $value->toInt()->add(10);
```

### 2.2 toFloat

```php
$price = '12.50'->toFloat();
var_dump($price); // float(12.5)
```

返回 TypePHP 原生浮点类型：

```php
$rounded = $value->toFloat()->round(2);
```

### 2.3 toString

```php
$text = $value->toString();
echo $text->trim()->upper();
```

`toString()` 常用于从 `mixed` 恢复字符串类型，使后续字符串通用方法能够被静态解析。

### 2.4 toBool

```php
$enabled = $config['enabled']->toBool();

if ($enabled) {
    echo 'enabled';
}
```

### 2.5 toArray

```php
$items = $value->toArray();
echo $items->count();
```

对于普通值，`toArray()` 执行数组转换。对于具有零参数 `toArray()` 方法的对象，底层转换 helper 可以调用对象方法取得数组结果。

## 3. 动态类型转换

### 3.1 toAny

`toAny()` 将当前表达式降级为 `mixed` / `any` 动态类型：

```php
$dynamic = $typedValue->toAny();
```

它适用于需要回到 Zend 动态语义的场景，例如：

- 参数明确要求 `mixed`
- 希望放弃当前静态类型信息
- 后续操作必须在运行时决定
- 需要避免继续使用原生数值运算规则

`toAny()` 不接受参数：

```php
$value->toAny();       // 正确
$value->toAny('arg');  // 编译错误
```

类型一旦降级，编译器不会自动恢复原来的具体对象类型。需要时应使用 `toObject(ClassName::class)` 重新接续。

### 3.2 toRef

`toRef()` 显式取得引用，等价于 TypePHP 的 `refval()`：

```php
$callback($value->toRef());
```

它主要用于编译器无法从动态调用中获知参数是否按引用传递的场景：

```php
$callable($data->toRef());
```

`toRef()` 只能用于能够定位的左值，例如变量、数组元素和对象属性：

```php
$value->toRef();
$array['key']->toRef();
$object->property->toRef();
```

临时计算结果通常不能取得引用：

```php
($a + $b)->toRef(); // 不支持
```

`toRef()` 不接受参数。

## 4. 对象类型接续

### 4.1 通用对象

不传类名时，`toObject()` 返回没有具体类信息的通用对象：

```php
$object = $value->toObject();
```

这种对象可以进行动态属性和方法访问，但编译器无法把调用解析为某个具体类的原生方法。

### 4.2 恢复具体对象类型

传入 `ClassName::class` 可以恢复具体类信息：

```php
$user = $value->toObject(User::class);

echo $user->getName();
```

常见场景是从数组、动态函数或 `mixed` 返回值中取得对象：

```php
$user = $container['user']->toObject(User::class);
$user->save();
```

也可以直接链式调用：

```php
echo $container['user']
    ->toObject(User::class)
    ->getName()
    ->upper();
```

带类名的对象转换会检查实际对象是否符合目标类或其继承关系。类型不匹配时抛出类型错误。

对象类型接续的详细说明见[对象类型转换](object-type-conversion.md)。

## 5. Stream 类型转换

`toStream()` 将动态资源恢复为 TypePHP Stream 类型：

```php
$stream = $value->toStream();
$stream->write('hello');
$stream->seek(0);
echo $stream->getContents();
```

从数组或动态函数取得流资源时尤其有用：

```php
$streams[0]
    ->toStream()
    ->write($payload);
```

如果实际值不是有效流资源，转换或后续调用会产生错误。

## 6. 高精度类型转换

### 6.1 toBigInt

```php
$number = $value->toBigInt();
$result = $number->mul(100)->add(1);
```

### 6.2 toDecimal

```php
$price = $value->toDecimal();
$total = $price->mul($quantity);
```

### 6.3 toBigFloat

```php
$number = $value->toBigFloat();
$result = $number->sqrt();
```

高精度类型之间的转换规则、精度和限制见[高精度运算](math.md)与[基础类型转换](type-convert.md)。

## 7. Std 容器类型接续

Std 容器通过 Box 资源跨越动态函数边界后，会丢失具体模板类型。`toStd*` 方法用于恢复容器类型，并且不会复制底层容器。

### 7.1 StdVector

```php
$vector = $value->toStdVector(Type::Int);
$vector[] = 42;
```

### 7.2 StdArray

```php
$array = $value->toStdArray(Type::Float, 4);
$array[0] = 3.14;
```

### 7.3 StdMap

```php
$map = $value->toStdMap(
    Type::Int,
    Type::String
);
```

### 7.4 StdOrderedMap

```php
$map = $value->toStdOrderedMap(
    Type::String,
    User::class
);
```

`toStd*` 方法必须用于变量顶层赋值，目标变量不能再被赋值为其他类型。完整规则见 [Std 容器](std-containers.md)。

## 8. 类型推断与链式调用

关键词方法的返回类型由编译器直接确定：

```php
$value->toInt();               // int
$value->toString();            // string
$value->toArray();             // array
$value->toObject(User::class); // User
$value->toStream();            // stream
```

因此可以安全地继续调用对应类型的方法：

```php
$result = $value
    ->toString()
    ->trim()
    ->lower()
    ->length();
```

对象、Stream 和高精度类型也支持类型接续：

```php
$name = $value
    ->toObject(User::class)
    ->getName()
    ->trim();
```

## 9. 参数限制

除特别说明外，基础转换方法均不接受参数：

```php
$value->toInt();
$value->toString();
$value->toAny();
$value->toRef();
```

`toObject()` 可以接受一个类名：

```php
$value->toObject(User::class);
```

`toStd*` 根据容器种类接受元素类型、键类型和值类型等编译期参数。

参数数量错误会在编译阶段报告，不会延迟到运行时。

## 10. 查找优先级

TypePHP 遇到具名方法调用时，内置关键词方法优先处理。例如：

```php
$value->toString();
```

无论 receiver 当前是 `mixed`、整数、字符串还是对象，编译器都会优先将其识别为内置转换方法。

内置关键词方法不能被以下机制覆盖：

- 对象真实方法
- 对象扩展方法
- 通用类型扩展方法
- 自定义关键词扩展方法
- `__call()`

这种优先级保证类型转换语义始终明确。

## 11. 与通用方法的区别

| 对比项 | 关键词方法 | 通用方法 |
|--------|------------|----------|
| 主要用途 | 类型转换、类型接续 | 对某种类型执行操作 |
| receiver 要求 | 可作用于任意类型 | 必须匹配具体类型 |
| 查找方式 | 编译器直接识别 | 根据 receiver 类型查找 |
| 示例 | `toInt()`、`toObject()` | `trim()`、`count()`、`write()` |
| 返回类型 | 编译器内置确定 | 由方法定义确定 |

例如：

```php
$text = $value->toString(); // 关键词方法：恢复 string 类型
$text = $text->trim();      // String 通用方法：处理字符串
```

## 12. 扩展关键词方法

除内置关键词方法外，可以通过类级 `MethodsFor` 为任意值增加项目自定义方法。这是关键词方法体系的扩展能力，不属于内置关键词方法本身。

```php
#[MethodsFor('*')]
final class KeywordExtensions
{
    public static function inspect(mixed $value, string $label): mixed
    {
        echo $label, ': ';
        var_dump($value);
        return $value;
    }
}

$value->inspect('input');
```

扩展关键词方法必须是 `public static`，第一个参数必须为 `mixed`。其完整声明、校验和调用规则见[扩展方法提供者](methods-for.md)。

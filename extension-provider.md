# 扩展方法提供者

`ExtensionProvider` 可以在不修改原类型的情况下，为对象、PHP 基础类型或任意表达式增加方法。它是 TypePHP 的编译期功能，只能在 TypePHP 静态代码中使用。

所有内建 Attribute 的共同命名空间规则参见[编译期注解](compile-time-attributes.md)。

Provider 是一个带有 `#[ExtensionProvider(...)]` 的普通类。类中的 `public static` 方法会成为扩展方法，静态方法的第一个参数是接收者：

```php
#[ExtensionProvider(Type::String)]
final class StringExtensions
{
    public static function surround(
        string $value,
        string $left = '[',
        string $right = ']'
    ): string {
        return $left . $value . $right;
    }
}

function main(): void
{
    $name = 'TypePHP';
    echo $name->surround('<', '>'); // <TypePHP>
}
```

调用 `$name->surround('<', '>')` 时，第一个参数 `$value` 由编译器自动传入；调用处提供的参数从静态方法的第二个参数开始对应。

## 1. 基础类型扩展

基础类型使用根命名空间 `Type` 中的编译期类型符号：

| 目标 | Receiver 参数 | 用途 |
|---|---|---|
| `Type::Int` | `int` | 整数 |
| `Type::Float` | `float` | 浮点数 |
| `Type::Bool` | `bool` | 布尔值 |
| `Type::String` | `string` | 字符串 |
| `Type::Array` | `array` | PHP 数组 |
| `Type::Object` | `object` | 静态类型为通用 `object` 的值 |
| `Type::Any` | `any` | 静态类型为 `any` 的值 |
| `Type::Stream` | `stream` | Stream 资源 |
| `Type::Box` | `box` | Box 资源 |
| `Type::BigInt` | `bigint` | 高精度整数 |
| `Type::BigFloat` | `bigfloat` | 高精度浮点数 |
| `Type::Decimal` | `decimal` | 高精度十进制数 |

这里的根命名空间 `Type` 是提供给用户代码的类型符号类，不是编译器内部的 `TypePHP\Type`。

一个 Provider 只声明一个目标，但同一目标可以由多个 Provider 类共同扩展：

```php
#[ExtensionProvider(Type::Int)]
final class IntFormatting
{
    public static function toBytes(int $value): string
    {
        return pack('J', $value);
    }
}

#[ExtensionProvider(Type::Int)]
final class IntPredicates
{
    public static function isEven(int $value): bool
    {
        return $value % 2 === 0;
    }
}

function main(): void
{
    $size = 42;
    var_dump($size->isEven());
    var_dump($size->toBytes());
}
```

同一目标不能注册两个仅大小写不同的同名方法，否则编译器会报告重复扩展方法。

### Type::Any 不是通配符

`Type::Any` 只匹配静态类型已经是 `any` 的接收者，不会匹配所有具体类型：

```php
#[ExtensionProvider(Type::Any)]
final class AnyExtensions
{
    public static function debugType(any $value): string
    {
        return get_debug_type($value);
    }
}

function inspect(any $value): void
{
    echo $value->debugType(); // $value 的静态类型为 any
}
```

如果需要为所有接收者提供方法，应使用关键词扩展。

## 2. 关键词扩展

字符串目标 `'*'` 表示该方法可以作用于任何接收者。第一个参数必须声明为 `any`：

```php
#[ExtensionProvider('*')]
final class DebugExtensions
{
    public static function inspectValue(any $value, string $label = 'value'): any
    {
        echo $label, ': ';
        var_dump($value);
        return $value;
    }
}

function main(): void
{
    $number = 42;
    $text = 'hello';
    $items = [1, 2, 3];

    $number->inspectValue('number');
    $text->inspectValue('text');
    $items->inspectValue('items');
}
```

关键词扩展不能覆盖 `toInt()`、`toString()`、`toArray()`、`toAny()`、`toRef()` 等编译器内置关键词方法。

`Type::Any` 与 `'*'` 的区别是：

- `Type::Any` 只匹配静态类型为 `any` 的值。
- `'*'` 可以匹配任意静态类型。

当接收者是 `any` 时，查找顺序为内置关键词方法、`Type::Any` 扩展、关键词扩展。

## 3. 对象扩展

对象扩展使用目标类的 `ClassName::class`：

```php
namespace App\Model;

final class User
{
    public function __construct(public string $name) {}
}
```

Provider 不要求与目标类位于相同命名空间。例如可以在另一个命名空间集中组织扩展：

```php
namespace App\Extension;

use App\Model\User;

#[\ExtensionProvider(User::class)]
final class UserExtensions
{
    public static function displayName(User $user): string
    {
        return strtoupper($user->name);
    }

    public static function rename(User $user, string $name): User
    {
        $user->name = trim($name);
        return $user;
    }
}
```

使用时仍然像调用普通对象方法：

```php
namespace App;

use App\Model\User;

function main(): void
{
    $user = new User('Alice');

    echo $user->displayName();
    echo $user->rename(' Bob ')->displayName();
}
```

`ExtensionProvider` 位于根命名空间，并完全遵循 PHP 的名称解析规则。在带命名空间的文件中，可以使用完全限定名：

```php
namespace App\Extension;

#[\ExtensionProvider(\Type::String)]
final class StringExtensions {}
```

也可以先导入根命名空间中的 `ExtensionProvider` 和 `Type`：

```php
namespace App\Extension;

use \ExtensionProvider;
use \Type;

#[ExtensionProvider(Type::String)]
final class StringExtensions {}
```

`use ... as ...` 别名同样支持：

```php
namespace App\Extension;

use \ExtensionProvider as Provider;
use \Type as TargetType;

#[Provider(TargetType::String)]
final class StringExtensions {}
```

如果没有相应的 `use`，命名空间中的 `#[ExtensionProvider(Type::String)]` 会按照 PHP 规则解析为 `App\Extension\ExtensionProvider` 和 `App\Extension\Type`，不会被当作根命名空间中的 TypePHP 编译期符号。只有最终解析结果为根命名空间 `ExtensionProvider` 的 Attribute 才会注册扩展方法。

对象目标也使用同一套规则。`User::class` 可以通过 `use App\Model\User;` 导入，也可以写成完整类名 `\App\Model\User::class`；`use ... as ...` 别名同样有效。

### 对象方法优先级

对象方法按以下顺序查找：

1. 类中真实存在的实例方法；
2. 对象扩展方法；
3. `__call()`；
4. 未定义方法错误。

扩展方法不能覆盖类中已经存在的实例方法，但在不存在真实方法时优先于 `__call()`。

## 4. Provider 方法规则

可注册的方法必须满足以下规则：

- 必须声明为 `public static`。
- 必须至少有一个参数。
- 第一个参数是 receiver，类型必须与 Provider 目标一致。
- Receiver 不能按引用传递。
- 调用参数从第二个参数开始对应。
- 默认参数和可变参数遵循普通 PHP 方法规则。
- 返回类型会用于后续链式调用的静态类型推断。
- 方法名按声明原样注册，不进行 snake_case、PascalCase 或 camelCase 自动转换。
- 查找方法名时不区分大小写，这一点与 PHP 普通方法一致。
- `private` 和 `protected` 方法不会注册，可以作为 Provider 内部 helper。
- 以双下划线开头的方法不会注册。
- `public` 非静态方法会产生编译错误，而不是被静默忽略。

下面的两个名称是不同扩展，不会互相转换：

```php
public static function displayName(User $user): string;
public static function display_name(User $user): string;
```

调用方必须使用对应名称：

```php
$user->displayName();  // 匹配 displayName
$user->display_name(); // 匹配 display_name
$user->DISPLAYNAME();  // 大小写不同，仍匹配 displayName
```

## 5. 链式调用

Provider 方法应尽量声明准确的返回类型，以便编译器继续解析链式方法：

```php
#[ExtensionProvider(Type::String)]
final class TextExtensions
{
    public static function quoted(string $value): string
    {
        return '"' . trim($value) . '"';
    }
}

function main(): void
{
    $user = new \App\Model\User(' Alice ');

    echo $user
        ->displayName() // object 扩展，返回 string
        ->quoted()      // string 扩展，仍返回 string
        ->upper();      // TypePHP 内置 String 通用方法
}
```

## 6. 使用范围

ExtensionProvider 只参与 TypePHP 静态方法调用分析。以下调用支持扩展查找：

```php
$user->displayName();
(new User('Alice'))->displayName();
$repository->findUser()->displayName();
```

以下调用不会使用扩展方法：

```php
$method = 'displayName';
$user->$method();

User::displayName();
call_user_func([$user, 'displayName']);
eval('$user->displayName();');
```

限制说明：

- 对象扩展不适用于静态方法调用。
- 动态方法名和动态 callback 不参与扩展查找。
- ZendVM 动态脚本、普通 PHP 脚本和 `eval()` 无法调用 TypePHP 扩展方法。
- Provider 类与目标类都必须在本次静态编译中可见。

`ExtensionProvider` Attribute 会在编译阶段被读取并移除，不写入生成程序的运行时 Attribute 元数据。扩展调用会直接编译为对应 Provider 静态方法调用，不需要运行时反射，也不存在运行时扩展方法表查询。

对象扩展的更多示例见[扩展对象方法](object-extension-method.md)，内置通用方法见[通用方法](universal_method.md)，内置关键词方法见[关键词方法](keyword-method.md)。

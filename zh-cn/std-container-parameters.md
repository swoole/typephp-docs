# 类型注解

`StdVector`、`StdMap`、`StdOrderedMap`、`StdList`、`StdDict` 统称为“类型注解”，使用 PHP Attribute 语法声明参数或属性的容器种类及键、值类型。前三种用于 Box 包装的 C++ 容器，编译器自动检查传入的 Box 并恢复容器引用，不再需要在函数体内手写 `toStd*()`。`StdList` / `StdDict` 用于[强类型 PHP 数组](typed-arrays.md)。

类型注解声明容器的具体类型，PHP 参数或属性类型可以省略，也可以声明兼容类型：

| 类型注解 | 兼容的 PHP 类型 |
|---|---|
| `StdVector` / `StdMap` / `StdOrderedMap` | `box` |
| `StdList` / `StdDict` | `array` |

显式 `mixed`、`any`、可空类型、联合类型及其他不兼容类型均不允许。以下 Box 容器的参数规则适用于前三种类型注解；list/dict 的引用参数等规则见[强类型 PHP 数组](typed-arrays.md)。

## 基本用法

```php
function append(#[StdVector(Type::Int)] box $values): void
{
    $values[] = 42;
}

function update(#[StdMap(Type::String, Type::Float)] $prices): void
{
    $prices['apple'] = 2.5;
}

class User {}

function visit(#[StdOrderedMap(Type::Int, User::class)] $users): void
{
    foreach ($users as $id => $user) {
        var_dump($id, $user);
    }
}

function main(): void
{
    $values = std::vector(Type::Int);
    append($values);
    var_dump($values[0]); // int(42)，调用方可观察到修改

    $prices = std::map(Type::String, Type::Float);
    update($prices);
    var_dump($prices['apple']); // float(2.5)

    $users = std::orderedMap(Type::Int, User::class);
    $users[1] = new User();
    visit($users);
}
```

| 注解 | 对应容器 | 类型实参 |
|---|---|---|
| `#[StdVector(T)]` | `std::vector(T)` | 一个元素类型 |
| `#[StdMap(K, V)]` | `std::map(K, V)` | 键类型、值类型 |
| `#[StdOrderedMap(K, V)]` | `std::orderedMap(K, V)` | 键类型、值类型 |

`T`、`K`、`V` 是上表中的占位符，实际声明须使用 std 工厂支持的 `Type::*` 常量或 `ClassName::class`，不能使用运行时变量、`"int"` 等字符串或命名实参。值类型参见 [Std 容器](std-containers.md)；键仅支持 `Type::Int` 和 `Type::String`。

`StdVector` 注解不接受初始大小。初始大小仍由调用方的 `std::vector(Type::Int, 100)` 指定，不属于参数类型契约。目前没有 `StdArray` 参数注解，定长数组仍使用 `toStdArray()`。

## 从 toStd*() 迁移

原来的显式类型恢复方式仍然可用：

```php
function append_explicit($source): void
{
    $values = $source->toStdVector(Type::Int);
    $values[] = 42;
}
```

改用参数注解后，可以直接操作参数，删除局部类型恢复语句：

```php
function append_typed(#[StdVector(Type::Int)] $values): void
{
    $values[] = 42;
}
```

这是类型契约和写法的改进，不是另一种容器存储模型。局部 Box 值的显式类型恢复以及定长数组仍可使用 [toStd* 关键词方法](keyword-method.md)。

## 命名空间与方法

注解和 `Type` 位于根命名空间，遵循 PHP 名称解析规则，支持全限定名称和别名导入：

```php
namespace App;

use StdVector as VectorOf;
use Type;

interface IntConsumer
{
    public function accept(#[VectorOf(Type::Int)] $values): void;
}

class Consumer implements IntConsumer
{
    public function accept(#[VectorOf(Type::Int)] $values): void
    {
        $values[] = 1;
    }
}
```

具名函数、方法、接口方法、抽象方法和 trait 方法的参数均可声明。覆盖父类方法或实现接口时，保留容器契约的参数必须使用相同容器种类及键、值类型；不能把 `StdVector(Type::Int)` 改为 `StdVector(Type::Float)`。若子方法将参数放宽为无类型或 `mixed`，则该参数不再自动恢复容器类型。

## 调用边界与类型检查

底层 ABI 仍使用 `php::Var` 传递 Box 句柄。函数入口校验 Box、容器种类和元素类型，并取得具体 C++ 容器引用；不复制容器，也不逐个转换元素。检查代码在每次调用时执行，但不依赖运行时 Attribute 扫描或反射。

传入错误的容器种类、错误的键或值类型、PHP 数组、`null` 或其他非 Box 值，会抛出 `TypeError`。注解不会把 PHP 数组自动转换为 std 容器；即使 `Type::Any` 用作值类型，参数也必须匹配该容器契约。

现有动态 callable 调用将编译器已知的 std 容器实参转换为 PHP 数组的规则不变，因此 `$callback($values)` 不等价于具名直接调用 `append($values)`。动态调用此类函数需要实际的 Box 值。例如，可通过直接调用一个无容器类型契约的函数取得动态 Box 值：

```php
function box_value($value) { return $value; }

function main(): void
{
    $values = std::vector(Type::Int);
    $box = box_value($values); // 直接调用保留 Box，返回值没有静态容器元数据
    $callback = 'append';
    $callback($box);
    var_dump($values[0]); // int(42)
}
```

## 当前限制

- 一个参数或属性只能有一个容器类型注解，不允许重复或混用；PHP 类型只能省略或为 `box`，例如 `#[StdVector(Type::Int)] mixed $values` 会产生编译错误。
- 不支持引用参数、可变参数、带默认值的参数和构造器属性提升参数。
- 不支持 Closure、箭头函数或 Generator 的参数，也不提供返回值的泛型类型声明。
- 容器不能保存 Native 对象并通过此 Box 参数边界传递；原有 [Native 对象逃逸限制](native-class.md) 不变。
- 参数绑定不能替换为其他 Box 或普通值，不能 `unset($values)`，也不能被 Closure 按引用捕获。同类型 std 容器赋值仍按现有规则复制容器内容；元素读写、追加、删除和遍历遵循 [Std 容器限制](std-containers.md)。

参数契约保存在增量编译的声明缓存和导出的 library stub 中；修改注解时，依赖该声明的调用方会重新转换。注解仅由 TypePHP 解释，不会让普通 Zend PHP 获得 C++ 泛型容器或等价参数检查。

## 属性类型注解

```php
class State
{
    #[StdVector(Type::Int)] public box $values;
    #[StdMap(Type::Str, Type::Int)] public $counts;
    #[StdOrderedMap(Type::Int, User::class)] public box $users;
}
```

属性允许省略 PHP 类型或声明 `box`，编译器保存容器契约并检测注解冲突。属性本身不是函数入口的局部容器绑定：目前读取属性后操作容器仍使用已有 `toStd*()` 类型恢复；不会生成长期存活的 C++ 容器引用。Native 类仍不允许 Box 容器属性。

# ArrayDef

`#[ArrayDef]` 为 PHP `array` 属性增加编译期键和值类型约束。它适合希望继续使用普通 PHP 数组，同时让 TypePHP 检查直接元素写入的场景。

普通 Zend Class 的实例属性、静态属性和构造器提升属性均可使用；[`#[Native]` 原生类](native-class.md)的实例 `array` 属性也可以使用。Native Class 本身不支持静态属性。`ArrayDef` 只存在于编译阶段，不注册为运行时 Attribute，也不会改变属性的 PHP `array` 类型。

## 基本语法

`ArrayDef` 接受一个或两个位置参数：

```php
#[ArrayDef(ValueType)]
public array $list = [];

#[ArrayDef(KeyType, ValueType)]
public array $map = [];
```

- 一个参数声明 List，参数表示元素类型；
- 两个参数声明 Map，第一个参数表示键类型，第二个参数表示值类型；
- Map 键只允许 `Type::Int` 或 `Type::String`；
- 参数必须是 `Type::*` 或 `ClassName::class`，不支持命名参数；
- 注解只能用于类型声明恰好为 `array` 的属性。

下面这些声明都会在编译期报错：

```php
#[ArrayDef]                         // 缺少类型参数
public array $missing = [];

#[ArrayDef(Type::Int, Type::String, Type::Bool)] // 参数过多
public array $tooMany = [];

#[ArrayDef(Type::Bool, Type::String)] // Map key 不能是 bool
public array $invalidKey = [];

#[ArrayDef(Type::Int)]
public mixed $notAnArray = [];      // 属性必须明确声明为 array
```

## List

一个参数表示 List，参数是元素类型：

```php
class Article
{
    #[ArrayDef(Type::String)]
    public array $tags = [];
}

$article = new Article();
$article->tags[] = 'php';
$article->tags[count($article->tags)] = 'aot'; // 连续 List 中等价于追加
$article->tags[0] = 'typephp';                 // 修改已有元素
```

List 使用连续整数下标，只支持以下写入：

- `$object->property[] = $value`：追加元素；
- `$object->property[$index] = $value`：修改已有元素、填补已删除的空洞，或者写入 PHP 当前的下一追加位置。

显式下标写入会统一生成 `safeArrayIndex($index, $array)` 边界检查。该检查通过 Zend 的 `zend_hash_next_free_element()` 获取真实的下一追加下标，并遵循 `zend_hash_next_index_insert()` 对空数组首个下标的规则，而不是使用 `count($array)`。这一区别在 `unset()` 产生空洞后很重要：PHP 删除元素时不会回退自动追加下标。等于该下标时可追加，更小的非负下标可修改元素或填补空洞，超过它则会在运行时抛出错误。编译器不对 `count()` AST 形式进行特殊识别。

```php
$index = count($article->tags);
$article->tags[$index] = 'next';     // 连续 List 中允许：该位置就是下一追加位置
$article->tags[$index + 2] = 'gap';  // 运行时错误：不允许产生空洞
$article->tags[-1] = 'invalid';      // 运行时错误
```

`unset()` 后不能再使用 `count()` 推导追加下标：

```php
$article->tags = [0 => 'php', 1 => 'aot', 2 => 'compiler'];
unset($article->tags[2]);

$article->tags[3] = 'next'; // 允许：PHP 下一追加下标仍为 3，而 count() 为 2
```

## Map

两个参数表示 Map。第一个参数是键类型，第二个参数是值类型：

```php
class Dictionary
{
    #[ArrayDef(Type::String, Type::Int)]
    public array $counts = [];
}

$dict = new Dictionary();
$dict->counts['php'] = 1;
```

Map 必须显式提供键，不支持追加语法：

```php
$dict->counts[] = 1; // 编译期错误
```

Map 的键遵循 PHP 数组自身的存储规则。特别是规范的十进制整数字符串键仍可能被 PHP 转换为整数键；`ArrayDef` 检查的是写入表达式的类型，不会改变 Zend Array 的键转换行为。

## 支持的值类型

List 的元素和 Map 的 value 支持以下类型：

| 声明 | 写入值 |
|---|---|
| `Type::Int` | `int` |
| `Type::Float` | `float` |
| `Type::Bool` | `bool` |
| `Type::String` | `string` |
| `Type::Array` | PHP `array` |
| `Type::Object` | 任意 PHP 对象 |
| `Type::Any` | 任意普通 PHP 值 |
| `Type::Stream` | Stream |
| `Type::BigInt` | BigInt |
| `Type::BigFloat` | BigFloat |
| `Type::Decimal` | Decimal |
| `ClassName::class` | 指定 PHP 类或其子类的对象 |

Std Container 不是 `Type::*` 类型符号，并且不能作为 ArrayDef 元素写入。`Type::Box` 也不是合法的 ArrayDef 类型参数。BigInt、BigFloat 和 Decimal 通过各自明确的类型符号声明。

## Class 元素类型

List 元素或 Map 的 value 可以使用 `ClassName::class`。子类对象同样符合约束：

```php
class UserCollection
{
    #[ArrayDef(App\User::class)]
    public array $users = [];

    #[ArrayDef(Type::String, App\User::class)]
    public array $usersByName = [];
}

$collection = new UserCollection();
$collection->users[] = new App\User();
$collection->usersByName['admin'] = new App\Admin(); // Admin extends User
```

静态类明确时，兼容性在编译期检查。值是 `any` 或无法确定具体 class 的普通 `object` 时，编译器插入运行时检查，要求它是目标 class 或其子类。

Native Class 对象没有 Zend `zval` 表示，不能存入 PHP 数组，因此不能作为 `ArrayDef` 的 Class 元素类型。

Map key 仍只能是 `Type::Int` 或 `Type::String`。下面的声明不合法：

```php
#[ArrayDef(App\User::class, Type::String)] // Class 不能作为 Map key
public array $invalid = [];
```

## 静态检查与动态检查

键或值的静态类型已经确定时，匹配的写入直接生成普通数组写入；明确不匹配时产生编译期 Fatal Error：

```php
class Names
{
    #[ArrayDef(Type::Int, Type::String)]
    public array $values = [];
}

$names = new Names();
$names->values[1] = 'one';    // 直接放行
$names->values['1'] = 'one';  // 编译期错误：key 必须是 int
$names->values[1] = 1;        // 编译期错误：value 必须是 string
```

表达式类型为 `any` 时，编译器会插入 `toIntExact()`、`toStringExact()`、`toObjectExact()` 或对应高精度类型检查。检查是严格的，不进行 PHP 弱类型转换：

```php
function put(Dictionary $dict, any $key, any $value): void
{
    $dict->counts[$key] = $value;
}

put($dict, 'php', 1);   // 运行时检查通过
put($dict, 1, 1);       // TypeError：不会把 int 转成 string key
put($dict, 'php', '1'); // TypeError：不会把 string 转成 int value
```

因此，静态类型完全明确的追加和 Map 写入没有额外类型检查；动态 `any` 写入承担所声明类型所需的运行时检查。List 的显式下标修改无论值类型是否静态明确，都保留前述边界检查。

## 作用范围与逃逸

`ArrayDef` 是编译期静态检查工具，不是运行时泛型数组。当前只检查编译器能够明确识别的第一层直接元素赋值：

```php
$object->values[$key] = $value;
ClassName::$values[$key] = $value;
```

下列路径不会递归检查数组内容：

- 属性默认数组和构造时传入的完整数组；
- `$object->values = $newArray` 这种整属性替换；
- `$object->values[0]['nested'] = $value` 这种嵌套元素写入；
- `++`、`--`、`+=`、`.=` 等原地操作；
- 取引用、引用参数以及 `foreach (... as &$value)`；
- `array_map()`、变量函数、Reflection、`eval()` 或其他 ZendVM 动态路径。

这些逃逸路径不会自动扫描、复制或包装数组。通过它们写入不符合声明的元素后，行为相对于 `ArrayDef` 契约属于未定义行为。`ArrayDef` 也不限制数组读取。

如果业务需要所有入口都具备运行时强类型保证，应使用封装方法控制写入，或选择 [Std 强类型容器](std-containers.md)，而不是把 `ArrayDef` 当作运行时容器。

## 命名空间

`ArrayDef` 和 `Type` 都位于根命名空间，并遵循普通 PHP 名称解析规则。在其他命名空间中可以使用完全限定名：

```php
namespace App\Model;

class Article
{
    #[\ArrayDef(\Type::String)]
    public array $tags = [];
}
```

也可以显式导入：

```php
namespace App\Model;

use \ArrayDef;
use \Type;

class Article
{
    #[ArrayDef(Type::String)]
    public array $tags = [];
}
```

完整的 Attribute 名称解析规则参见[编译期注解](compile-time-attributes.md)。

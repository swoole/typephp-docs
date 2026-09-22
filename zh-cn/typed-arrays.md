# 强类型 PHP 数组：std::list / std::dict

`std::list(T)` 和 `std::dict(K, V)` 使用普通 PHP 数组存储，在 TypePHP 静态编译阶段约束键和值的类型。它们不是 Box 包装的 C++ 容器。工厂创建、同类型赋值和原生参数传递不会扫描整个数组；从普通数组或其他值显式转换时则需要运行时校验。

## 声明和索引

```php
$list = std::list(Type::Int);
$list[-5] = 10;
$list[100] = 20;
$list[] = 30; // PHP 追加规则，此处键为 101

$dict = std::dict(Type::Int, Type::Str);
$dict[-5] = 'hello';
$dict[100] = 'world';
// $dict[] = 'no'; // 编译错误：dict 必须显式提供键

class MyUser { public int $id = 0; }
$users = std::list(MyUser::class);
$users[] = new MyUser();
```

list 的键固定为整数，允许负数、不连续和稀疏索引，不进行 vector 式边界检查。整数键 dict 使用相同的 PHP 索引行为，但操作契约不同：只有 list 允许 `[]` 追加，两者不是同一种参数类型。

dict 的键仅支持 `Type::Int` 或 `Type::Str`（`Type::String` 是同义写法）。值支持 `Type::Int`、`Type::Float`、`Type::Bool`、`Type::Str`、`Type::Array`、`Type::Object`、`Type::Any` 和 `ClassName::class`；不能存储 Native 对象。

保留 PHP 数组的底层键规则，例如字符串 `'123'` 会存为整数键。TypePHP 在字符串键 dict 的 `foreach` 取键时生成 `str` 转换，保证循环中的 key 为声明的字符串类型，不修改 phpx 或 PHP 数组机制。`array_keys()` 等内置函数仍返回普通 PHP 结果。

## 从现有值转换

使用关键词方法 `toStdList(T)` 或 `toStdDict(K, V)` 将现有值转换为强类型 PHP 数组：

```php
$raw = [4, 5];
$list = $raw->toStdList(Type::Int);
$list[] = 6;

$counts = ['alice' => 2];
$dict = $counts->toStdDict(Type::Str, Type::Int);
$dict['bob'] = 3;

$copy = $list->toStdList(Type::Int); // 同契约：普通数组赋值，保留写时复制
```

若来源已经是完全相同契约的 `StdList` 或 `StdDict`，转换等同于赋值，不遍历元素。普通数组会在运行时逐项检查 key 和 value，要求与声明类型严格匹配；不匹配时抛出 `TypeError`。其他值先执行 `toArray()`，再按普通数组校验。值类型也可以使用非 Native 的 `ClassName::class`，此时逐项检查对象是否为该类或其子类。来源数组不会被修改。

**性能风险：** 需要校验的转换会遍历整个数组，时间复杂度为 O(n)。非数组来源还要先执行 `toArray()`。大数组或循环内反复转换可能明显增加耗时，应谨慎使用；尽量在进入强类型边界时转换一次，并复用结果。

校验依据 PHP 数组实际保存的 key 类型。数字字符串键（例如 `'123'`）会被 PHP 规范化为整数键，因此含有这种键的普通数组无法通过 `toStdDict(Type::Str, ...)` 的严格校验；已是同契约的强类型 dict 则直接赋值，不重新校验。

## 静态类型约束

```php
$list = std::list(Type::Int);
// $list['1'] = 10; // 编译错误：键不是 int，不会隐式转换
// $list[] = '10';  // 编译错误：值不是 int

$key = std::any(1);
$list[$key] = 10; // 自动生成内部严格检查：运行时 key 必须为 int

$textKey = std::any('1');
$list[$textKey->toInt()] = 20; // 用户显式要求将字符串转换为整数
```

`any` / `var` 可以直接作为 key。TypePHP 自动调用 phpx 内部严格类型检查：list / 整数键 dict 要求运行时值确实为 int，字符串键 dict 要求确实为 string，否则抛出 `TypeError`。不会隐式转换数字字符串、float、bool 等不匹配的运行时类型；读写、存在性检查和删除均遵守同样规则。内部 Exact API 不是用户关键词方法。

非 var 的 key 类型不匹配时直接编译报错。强类型 value 仍要求静态类型匹配（`Type::Any` 值除外）；动态值需要用户显式使用现有的 `toInt()`、`toString()` 等关键词方法。普通下标操作及原生函数入口不会重新检查整个数组；显式 `toStdList()` / `toStdDict()` 转换除外。

直接下标读写、`isset()`、`empty()` 和元素 `unset()` 均支持。当前不支持元素引用、引用 `foreach`、嵌套数组写入及 `+=`、`??=`、`++` 等复合修改；请使用类型明确的普通元素赋值。启用 `varint_types` 时，可能溢出并变为 float 的整数表达式也需要显式类型恢复。

## 复制、引用与 foreach

```php
$list = std::list(Type::Int);
$list[] = 1;
$copy = $list;       // 传导类型；PHP 写时复制
$alias = &$list;     // 相同类型的静态引用
$copy[] = 2;         // 不修改 $list
$alias[] = 3;        // 修改 $list

foreach ($list as $key => $value) {
    // $key 和 $value 均为明确的 int 类型
    echo $key, ':', $value, "\n";
}
```

类元素的 `foreach` 值保持声明的类类型。闭包按值捕获也保留类型和写时复制行为，不允许按引用捕获。

## 函数和方法参数

注解声明具体的数组契约，PHP 参数类型可以省略，也可以声明兼容的 `array`；显式 `mixed`、`box`、可空类型、联合类型及其他类型均不允许。属性使用相同的类型兼容性规则：

```php
function inspect(#[StdList(MyUser::class)] array $users): void
{
    foreach ($users as $user) {
        echo $user->id;
    }
}

function update(#[StdDict(Type::Str, MyUser::class)] array &$users): void
{
    $users['alice'] = new MyUser();
}
```

调用方和参数的容器种类、键类型、值类型必须完全一致。按值参数使用 PHP 写时复制；`&` 参数可修改调用方的数组。方法参数使用相同语法；命名空间中需导入根命名空间的 `StdList` / `StdDict`，也可使用全限定名。

当前仅支持静态解析的 TypePHP 原生函数、方法调用。带这些参数的函数或方法不生成 Zend 入口，不能由动态 PHP、字符串回调或 Zend 多态分派调用。暂不支持闭包参数注解、默认值、可变参数及提升属性参数。禁止通过引用返回将数组暴露给未约束的代码。

## 属性类型注解

```php
class State
{
    #[StdList(Type::Int)] public array $values = [];
    #[StdDict(Type::Str, MyUser::class)] public array $users = [];
    #[StdList(Type::Str)] public $names = []; // 省略类型时推导为 array
}

$state = new State();
$state->values[-5] = 10;
$state->values[100] = 20; // 允许稀疏索引与空洞
$state->values[] = 30;
```

普通类的实例、静态属性及 Native 类的实例数组属性均支持 `StdList` / `StdDict` 类型注解；省略类型时推导为 `array`，list 显式索引写入不做边界检查。

当前属性注解沿用原有属性的第一层直接元素赋值检查，键和值遵循新 list/dict 类型规则；它不改变 Zend 对象的逃逸机制，不会扫描整个属性数组，也不会将属性读取自动升级为受封闭契约保护的局部强类型容器。整属性替换、动态 PHP 修改对象属性、引用逃逸等入口不能依靠属性注解获得完整保证，需要封装写入入口。以下动态 PHP 边界约束针对工厂创建、原生参数传递和类型传导的局部容器。

## 动态 PHP 边界

```php
$list = std::list(Type::Int);
$list[] = 1;
var_dump(array_search(1, $list, true), array_keys($list)); // 允许：只读
// array_push($list, 2); // 编译错误：动态引用修改
// sort($list);         // 编译错误：动态引用修改
// array_walk($list, $callback); // 编译错误：可能修改元素
// $callback(std::ref($list));   // 编译错误：引用逃逸
// $callback($list->toRef());    // 同样禁止
```

只读内置函数及对应 TypePHP 通用方法可使用。它们返回的新数组仍是普通数组，不自动携带原容器的类型契约。动态 PHP 按值调用只能接收值快照，不能通过写回改变原容器；未知回调的实际引用签名仍可能产生 Zend 警告。

向 TypePHP 定义的函数或方法传递时必须使用一致的 `StdList` / `StdDict` 参数注解，不能退化成未注解的 `array` / `mixed` 参数。

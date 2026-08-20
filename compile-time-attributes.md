# 编译期注解

TypePHP 使用 PHP 原生 Attribute 语法提供编译期注解。编译器在分析源码时读取这些 Attribute，并将其转换为对应的编译行为；除非具体页面另有说明，它们不会写入运行时 Attribute 元数据。

Attribute 参数支持字面量、常量、非空数组、嵌套数组和 `new` 表达式，但仍须符合 PHP 的常量表达式规则。普通函数调用不能作为 Attribute 参数，完整范围参见[常量表达式](constant-expressions.md)。

## 编译期生成，零动态开销

`Getter`、`Setter`、`With`、`Printer` 和 `Arrayable` 生成的方法全部在**编译期**完成展开，并作为普通方法参与 TypePHP 的类型分析和 AOT 编译。程序运行时不会扫描 Attribute、不会使用反射，也不会根据 Attribute 动态生成或查找方法。

因此，这些生成类 Attribute 的机制本身是**零运行时开销**：调用生成的方法与调用开发者手写的等价方法没有区别，也不是动态调用。

`NotNull`、`NotEmpty` 和 `Validate` 同样在编译期展开，不存在运行时 Attribute 解析开销；但它们生成的参数检查和异常分支属于明确要求的业务检查，会在每次调用时执行。

`Override`、`MustUse` 和 `Immutable` 只执行编译期静态验证，不生成运行时代码。`ArrayDef` 对静态类型明确的追加和 Map 写入不增加类型检查；`any` 键或值会插入严格类型检查，List 的显式下标修改还会执行边界检查。`Native` 在编译期选择独立的原生对象模型。`Hot` 和 `Cold` 在编译期转换为 C++ 编译器的优化提示，也不会执行运行时 Attribute 检查。`Constructor` 在编译期生成普通构造方法，与手写等价方法没有额外运行时开销。

## 注解列表

| Attribute | 可用目标 | 用途 |
|---|---|---|
| [`#[NoExport]`](no-export.md) | 函数、类、方法 | 从 TypePHP library 的公开 API 和导出符号中排除声明 |
| [`#[MethodsFor(...)]`](methods-for.md) | 类 | 声明基础类型、对象类型或通配目标的扩展方法提供者 |
| [`#[Getter]`](getter.md) | 实例属性 | 自动生成返回属性值的 public Getter 方法 |
| [`#[Setter]`](setter.md) | 实例属性 | 自动生成修改属性值的 public Setter 方法 |
| [`#[With]`](with.md) | 实例属性 | 克隆对象、修改指定属性并返回新对象 |
| [`#[Printer]`](printer.md) | 类 | 根据 public 属性自动生成 `__toString()` |
| [`#[Arrayable]`](arrayable.md) | 类 | 根据 public 属性自动生成 `toArray()` |
| [`#[NotNull]`](not-null.md) | 函数或方法参数 | 拒绝严格等于 `null` 的参数值 |
| [`#[NotEmpty]`](not-empty.md) | 函数或方法参数 | 拒绝 `empty()` 为 true 的参数值 |
| [`#[Validate(...)]`](validate.md) | 函数或方法参数 | 使用 `filter_var()` 验证参数，不修改参数值 |
| [`#[Override]`](override.md) | 方法 | 强制要求方法覆盖父类方法或实现接口方法 |
| [`#[MustUse]`](must-use.md) | 函数、方法 | 禁止丢弃调用返回值 |
| [`#[Immutable]`](immutable.md) | 方法、Property Hook、参数 | 对方法中的 `$this` 或参数执行编译期不可变检查 |
| [`#[Native]`](native-class.md) | 具名类 | 将类编译为不注册到 ZendVM 的原生 C++ 对象 |
| [`#[Hot]`](hot.md) | 函数、方法 | 提示编译器优先优化高频执行路径 |
| [`#[Cold]`](cold.md) | 函数、方法 | 提示编译器按低频路径优化函数 |
| [`#[Constructor]`](constructor.md) | 实例属性 | 根据属性生成 public 构造方法 |
| [`#[ArrayDef(...)]`](array-def.md) | `array` 属性 | 声明 List 元素类型或 Map 键、值类型，并检查直接元素写入 |
| [`#[WasmExport]`](wasm-export.md) | 函数 | 将静态类型函数导出为 WASI 0.2 Component 的 WIT 接口 |

## 命名空间规则

编译期 Attribute 位于根命名空间，并完全遵循 PHP 的类名解析规则。在全局命名空间中可以直接使用短名称：

```php
#[NoExport]
function internalHelper(): void {}
```

在其他命名空间中，可以使用完全限定名：

```php
namespace App\Model;

class User
{
    #[\Getter]
    private string $name = '';
}
```

也可以通过 `use` 导入短名称：

```php
namespace App\Model;

use \Getter;
use \NoExport;

#[NoExport]
class InternalUser
{
    #[Getter]
    private string $name = '';
}
```

`use ... as ...` 别名同样有效：

```php
namespace App\Extension;

use \MethodsFor as Provider;
use \Type as TargetType;

#[Provider(TargetType::String)]
final class StringExtensions {}
```

只有最终解析到根命名空间内建 Attribute 的名称才有编译期语义。例如下面的 `Getter` 会解析为 `App\Model\Getter`，不会自动生成方法：

```php
namespace App\Model;

class Example
{
    #[Getter]
    private string $name = '';
}
```

限定名称和 `namespace\...` 相对名称也遵循普通 PHP 规则，不会因为末尾名称相同而被编译器误识别。

## 重复和互斥

TypePHP 的内建编译期 Attribute 默认均不可重复。同一个声明上无论使用短名称、完全限定名称、多个 Attribute 组还是 `use ... as ...` 别名，只要最终解析到同一个根命名空间 Attribute，就视为重复并产生编译错误：

```php
use \Getter as ReadProperty;

class User
{
    #[Getter]
    #[ReadProperty] // 编译错误：Getter 重复
    private string $name = '';
}
```

互斥关系同样在编译期统一检查。例如 `Hot` 与 `Cold` 不能同时用于同一个函数或方法。具体注解只有在显式声明为可重复时才允许重复；当前内建编译期 Attribute 均未声明为可重复。

## 统一编译诊断

编译期 Attribute 的错误使用采用统一诊断格式，包含：

- Attribute 名称；
- 目标声明，例如类、方法、属性或参数；
- Attribute 所在文件和行号；
- 重复、互斥或生成声明冲突时的另一处来源。

例如同一参数重复声明 `Validate`：

```text
Validate cannot be repeated on the same declaration
[compile-time attribute: #[Validate]; target: parameter $value;
 source: src/User.php:12;
 conflict source: #[Validate] at src/User.php:13]
```

`Getter`、`Setter`、`With`、`Printer`、`Arrayable` 或 `Constructor` 生成的方法与已有方法冲突时，诊断中的 target 指向原始 Attribute 声明，conflict source 指向已有方法，而不是只显示内部生成方法的位置。

## 使用原则

- Attribute 的参数、允许目标和具体行为以对应的二级页面为准。
- 编译器会拒绝把内建 Attribute 用在不支持的声明上。
- 用户项目可以定义同名但位于其他命名空间的 Attribute，它们不会触发 TypePHP 编译期功能。
- 编译期注解可以与普通 PHP Attribute 同时放在一个声明上。

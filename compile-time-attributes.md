# 编译期注解

TypePHP 使用 PHP 原生 Attribute 语法提供编译期注解。编译器在分析源码时读取这些 Attribute，并将其转换为对应的编译行为；除非具体页面另有说明，它们不会写入运行时 Attribute 元数据。

## 编译期生成，零动态开销

`Getter`、`Setter`、`With` 和 `Printer` 生成的方法全部在**编译期**完成展开，并作为普通方法参与 TypePHP 的类型分析和 AOT 编译。程序运行时不会扫描 Attribute、不会使用反射，也不会根据 Attribute 动态生成或查找方法。

因此，这些生成类 Attribute 的机制本身是**零运行时开销**：调用生成的方法与调用开发者手写的等价方法没有区别，也不是动态调用。

`NotNull` 同样在编译期展开，不存在运行时 Attribute 解析开销；但它生成的 `empty()` 参数检查和异常分支属于明确要求的业务检查，会在每次调用时执行。

## 注解列表

| Attribute | 可用目标 | 用途 |
|---|---|---|
| [`#[NoExport]`](no-export.md) | 函数、类、方法 | 从 TypePHP library 的公开 API 和导出符号中排除声明 |
| [`#[ExtensionProvider(...)]`](extension-provider.md) | 类 | 声明基础类型、对象类型或通配目标的扩展方法提供者 |
| [`#[Getter]`](getter.md) | 实例属性 | 自动生成返回属性值的 public Getter 方法 |
| [`#[Setter]`](setter.md) | 实例属性 | 自动生成修改属性值的 public Setter 方法 |
| [`#[With]`](with.md) | 实例属性 | 克隆对象、修改指定属性并返回新对象 |
| [`#[Printer]`](printer.md) | 类 | 根据所有 public 属性自动生成 `toString()` |
| [`#[NotNull]`](not-null.md) | 函数或方法参数 | 在函数入口拒绝 `empty()` 的参数值 |

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

use \ExtensionProvider as Provider;
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

## 使用原则

- Attribute 的参数、允许目标和具体行为以对应的二级页面为准。
- 编译器会拒绝把内建 Attribute 用在不支持的声明上。
- 用户项目可以定义同名但位于其他命名空间的 Attribute，它们不会触发 TypePHP 编译期功能。
- 编译期注解可以与普通 PHP Attribute 同时放在一个声明上。

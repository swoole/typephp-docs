# NoExport 编译期 Attribute

`NoExport` 用于将函数、类或方法保留为 library 内部实现，不把它们发布给其他 TypePHP 项目。

所有内建 Attribute 的共同命名空间规则参见[编译期注解](compile-time-attributes.md)。

```php
#[NoExport]
function normalizeInternalData(array $data): array
{
    return $data;
}

#[NoExport]
class InternalCache
{
}
```

被标记的声明仍会正常编译，当前 library 内部可以继续调用。

## 对导出结果的影响

使用 `-m lib` 编译时，被标记的声明：

- 不写入自动生成的 `<target>.stub.php`；
- 对应的 `php_*` 函数不添加 library export 修饰；
- Windows 下不会标记为 `dllexport`；
- Linux 等平台配合隐藏可见性选项保持为内部符号。

`NoExport` 不会删除实现，也不会改变普通非 library 项目中的执行行为。

## 标记函数

```php
#[NoExport]
function internalHelper(): int
{
    return 42;
}
```

PHP 实现的函数和本地 `.stub.php` 中声明的 C++ 实现函数都支持 `NoExport`。

## 标记类

```php
#[NoExport]
class InternalService
{
    public function execute(): void
    {
    }
}
```

标记类会排除整个类，并使该类的全部 `php_*` 方法保持为 library 内部符号。类的属性、常量和方法都不会进入发布 stub。

公开 API 不应引用被隐藏的类。例如公开函数的参数或返回类型不应声明为 `InternalService`。

## 标记单个方法

```php
class PublicService
{
    public function execute(): void
    {
    }

    #[NoExport]
    public function resetInternalState(): void
    {
    }
}
```

上例中的类和 `execute()` 仍然公开，只有 `resetInternalState()` 不进入发布 stub，也不导出对应的 `php_*` 方法。

## 命名空间

`NoExport` 位于根命名空间。namespace 中可以使用完全限定名：

```php
namespace App\Internal;

#[\NoExport]
class Cache {}
```

也可以导入短名称或别名：

```php
namespace App\Internal;

use \NoExport;
use \NoExport as Internal;

#[NoExport]
function helper(): void {}

#[Internal]
class Cache {}
```

如果没有 `use`，`namespace App\Internal` 中的 `#[NoExport]` 会解析为 `App\Internal\NoExport`，不具备 TypePHP 编译期语义。完整规则参见[编译期注解](compile-time-attributes.md)。

## 与其他编译期 Attribute 配合

`NoExport` 可以和其他 Attribute 一起使用。例如整个类被隐藏后，由 [`Getter`](getter.md) 生成的方法也会保持为内部方法：

```php
#[NoExport]
class InternalUser
{
    #[Getter]
    private string $name = '';
}
```

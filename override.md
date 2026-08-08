# Override 编译期 Attribute

`Override` 用于明确声明一个方法必须覆盖父类方法，或者实现类、父类所声明接口中的方法。TypePHP 会在编译期检查这一要求；找不到匹配方法时直接产生 fatal error，不受 PHP `error_reporting` 配置影响。

```php
class BaseService
{
    public function execute(): void
    {
    }
}

class UserService extends BaseService
{
    #[\Override]
    public function execute(): void
    {
    }
}
```

实现接口方法也属于有效覆盖：

```php
interface Formatter
{
    public function format(): string;
}

class JsonFormatter implements Formatter
{
    #[\Override]
    public function format(): string
    {
        return '{}';
    }
}
```

如果父类和接口都没有同名方法，TypePHP 将停止编译：

```php
class UserService
{
    #[\Override]
    public function execute(): void
    {
    }
}
```

```text
Fatal error: UserService::execute() has #[\Override] attribute,
but no matching parent method exists
```

## 硬性规则

- 只能用于方法，不接受任何参数，也不能在同一个方法上重复声明。
- 匹配父类的 public 或 protected 方法，也匹配抽象方法和接口方法。
- 父类 private 方法不属于覆盖目标。
- `__construct()` 不属于 `Override` 的匹配目标，即使父类也定义了构造方法。
- 找到同名目标后，方法仍必须通过普通的返回类型、参数类型、可见性、static 和 final 等兼容性检查。
- trait 方法上的 `Override` 会在 trait 被具体类使用时验证；同一个 trait 可以在满足契约的类中使用，而在没有匹配父方法或接口方法的类中产生编译错误。

## 命名空间

`Override` 是 PHP 根命名空间的内置 Attribute。在命名空间中应使用完全限定名：

```php
#[\Override]
public function execute(): void
{
}
```

也可以按照普通 PHP 规则导入或设置别名：

```php
namespace App;

use \Override;
use \Override as Replaces;

class ParentClass
{
    public function first(): void
    {
    }

    public function second(): void
    {
    }
}

class Child extends ParentClass
{
    #[Override]
    public function first(): void
    {
    }

    #[Replaces]
    public function second(): void
    {
    }
}
```

TypePHP 完全按照 PHP 名称解析规则识别它；`App\Override` 等其他命名空间中的同名 Attribute 不具备这项编译期语义。

使用 `-m lib` 时，生成的 library stub 会保留 `Override`，使消费项目编译时继续验证相同的继承契约。

参考：[PHP `Override` 官方文档](https://www.php.net/manual/zh/class.override.php)。

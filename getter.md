# Getter 编译期 Attribute

`Getter` 用于为对象属性自动生成公开的读取方法，适合只允许外部读取、不希望直接公开写入权限的属性。

Getter 方法在**编译期静态生成**，随后与普通手写方法一起完成 AOT 编译。运行时不解析 Attribute、不使用反射，也不是动态方法调用，因此注解机制本身是零运行时开销。

所有内建 Attribute 的共同命名空间规则参见[编译期注解](compile-time-attributes.md)。

```php
class User
{
    #[Getter]
    private string $name = 'Alice';
}

$user = new User();
echo $user->getName(); // Alice
```

编译器会生成等价方法：

```php
public function getName(): string
{
    return $this->name;
}
```

`Getter` 是纯编译期 Attribute，不会写入运行时 Attribute 元数据。

## 属性可见性

`Getter` 支持 `private`、`protected` 和 `public` 实例属性。无论属性本身是什么可见性，生成的方法始终为 `public`：

```php
class Profile
{
    #[Getter]
    private string $name = '';

    #[Getter]
    protected int $age = 0;

    #[Getter]
    public bool $active = true;
}
```

以上属性分别生成：

```php
public function getName(): string;
public function getAge(): int;
public function getActive(): bool;
```

Getter 没有 setter 行为。外部代码能否直接修改属性，仍由属性原本的可见性决定。

同一属性可以同时使用 `Getter`、[`Setter`](setter.md) 和 [`With`](with.md)：

```php
#[Getter, Setter, With]
private string $name = '';
```

## 方法名称和返回类型

方法名称由 `get` 加属性名首字母大写组成：

| 属性 | 生成方法 |
|---|---|
| `$name` | `getName()` |
| `$userId` | `getUserId()` |
| `$_value` | `get_value()` |

属性声明了类型时，生成方法使用相同的返回类型。未声明类型的属性生成无返回类型声明的方法，返回值按照 TypePHP 的动态类型规则处理。

如果类中已经存在同名方法，编译器会按普通重复方法规则报错，不会覆盖用户编写的方法。

## 构造器属性提升

构造器提升属性同样支持 `Getter`：

```php
class User
{
    public function __construct(
        #[Getter]
        private int $id,
        #[Getter]
        protected string $name,
    ) {
    }
}
```

这会生成公开的 `getId()` 和 `getName()`。

一个声明中包含多个属性时，每个属性都会生成 Getter：

```php
class Point
{
    #[Getter]
    private int $x = 0, $y = 0;
}
```

上例生成 `getX()` 和 `getY()`。

## 命名空间

`Getter` 位于根命名空间，并遵循 PHP 的标准名称解析规则。在其他命名空间中可以使用完全限定名：

```php
namespace App\Model;

class User
{
    #[\Getter]
    private string $name = '';
}
```

也可以先导入再使用短名称或别名：

```php
namespace App\Model;

use \Getter;
use \Getter as Readable;

class User
{
    #[Getter]
    private string $name = '';

    #[Readable]
    private int $age = 0;
}
```

未导入时，命名空间中的 `#[Getter]` 会解析为例如 `App\Model\Getter`，不会触发 TypePHP 的 Getter 功能。

## 使用限制

`Getter` 只能用于实例属性，不接受参数。以下写法会产生编译错误：

```php
class InvalidExample
{
    #[Getter] // 错误：静态属性不是对象实例属性
    public static int $count = 0;
}

#[Getter] // 错误：不能用于函数
function getValue(): int
{
    return 1;
}
```

也不能用于类、普通方法、类常量或函数参数。构造器提升属性是对象属性，因此属于允许的例外。

## TypePHP library

使用 `-m lib` 编译时，Getter 方法属于类的公开方法：

- 库内生成 Getter 的 `php_*` 实现并导出；
- 自动生成的 `<target>.stub.php` 保留属性上的 `#[Getter]`；
- 消费项目根据 Attribute 生成 Getter 方法声明；
- Getter 的实际方法实现从动态库导入。

如果整个类使用了 `#[NoExport]`，该类及其 Getter 都不会进入发布 stub，也不会作为 library ABI 导出。

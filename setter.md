# Setter 编译期 Attribute

`Setter` 用于为实例属性生成公开的写入方法。属性自身可以是 `private`、`protected` 或 `public`，生成的方法始终是 `public`。

Setter 方法在**编译期静态生成**并作为普通方法参与 AOT 编译。运行时不扫描 Attribute、不使用反射，也不通过动态调用执行，因此与手写等价 Setter 相比没有额外运行时开销。

```php
class User
{
    #[Setter]
    private string $name = '';
}

$user = new User();
$user->setName('张三');
```

编译器会生成等价方法：

```php
public function setName(string $name): void
{
    $this->name = $name;
}
```

方法参数沿用属性类型。未声明属性类型时，参数也不声明类型。构造器提升属性和一个声明中的多个属性同样受支持。

`Setter` 可以与 `Getter`、`With` 同时使用：

```php
#[Getter, Setter, With]
private int $age = 0;
```

`Setter` 只能用于非静态实例属性，且不接受参数。在命名空间中请使用 `#[\Setter]`，或先通过 `use \Setter;` 导入。

使用 `-m lib` 时，发布 stub 保留 `#[Setter]`，消费项目据此获得方法声明，方法实现由动态库提供。

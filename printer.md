# Printer 编译期 Attribute

`Printer` 用于给类自动生成公开的 `toString(): string` 方法。输出包含类的短名称，以及当前类和基类中所有非静态 `public` 属性。

`toString()` 在**编译期静态生成**并作为普通方法参与 AOT 编译。运行时不会遍历 Attribute、使用反射或动态创建方法，因此注解机制本身是零运行时开销；运行时仅执行 `toString()` 本身所需的属性读取和字符串拼接。

```php
#[Printer]
class User
{
    public int $id = 1;
    public string $name = '张三';
    private string $password = '';
}

echo (new User())->toString();
// User(id=1, name=张三)
```

`private`、`protected` 和静态属性不会进入输出。属性按基类到子类、类内声明顺序排列；没有公开属性时输出如 `User()`。

如果当前类已经声明了 `toString()`，或者任意基类已经提供了具体或抽象的 `toString()`，`Printer` 会被忽略，编译器不会覆盖或重复声明该方法。

`Printer` 只能用于具名类且不接受参数。在命名空间中请使用 `#[\Printer]`，或先通过 `use \Printer;` 导入。

使用 `-m lib` 时，发布 stub 保留 `#[Printer]`，消费项目会得到相同的方法声明，实际实现由动态库提供。

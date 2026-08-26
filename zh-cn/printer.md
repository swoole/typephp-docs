# Printer 编译期 Attribute

`Printer` 用于给类自动生成 PHP 标准魔术方法 `public function __toString(): string`。默认输出类的短名称，以及当前类和基类中所有非静态 `public` 属性。

`__toString()` 在**编译期静态生成**并作为普通方法参与 AOT 编译。运行时不会遍历 Attribute、使用反射或动态创建方法，因此注解机制本身是零运行时开销；运行时仅执行 `__toString()` 本身所需的属性读取和字符串拼接。

它同时兼容 PHP 和 TypePHP 的调用方式：PHP 的 `echo $object`、字符串插值等上下文会调用 `__toString()`；TypePHP 的关键词方法 `$object->toString()` 底层同样执行对象字符串转换，并最终调用这个 `__toString()`。编译器不会再生成第二个 `toString()` 包装方法。

```php
#[Printer]
class User
{
    public int $id = 1;
    public string $name = '张三';
    private string $password = '';
}

echo new User();
// User(id=1, name=张三)

echo (new User())->toString();
// User(id=1, name=张三)
```

`private`、`protected` 和静态属性不会进入输出。属性按基类到子类、类内声明顺序排列；没有公开属性时输出如 `User()`。

可以通过可选的 `fields` 参数选择并排列需要输出的字段：

```php
#[Printer(fields: ['name', 'id'])]
class User
{
    private int $id = 1;
    protected string $name = '张三';
    public string $email = 'user@example.com';
}

echo new User();
// User(name=张三, id=1)
```

也可以使用位置参数：

```php
#[Printer(['id', 'name'])]
class User {}
```

显式指定 `fields` 时，可以选择当前类中声明的 `public`、`protected` 或 `private` 实例属性，也可以选择基类中当前子类可访问的 `public` 或 `protected` 实例属性。不能选择动态或未知属性、静态属性以及基类的 `private` 属性；重复字段同样会在编译期报错。

属性值可以是任意类型。`string` 属性直接参与拼接；int、float、bool、array、object、mixed 等所有非 string 类型统一执行 TypePHP 的 `toString()` 转换后再拼接。传入空数组 `[]` 会生成形如 `User()` 的结果；完全省略参数才表示选择全部 public 属性。

`Printer` 只负责在编译期生成一个普通的 `public function __toString(): string` 方法，不会因为当前类或基类已经存在同名方法而忽略生成。

生成后，该方法与手写方法遵循完全相同的 PHP 类方法规则：当前类中出现同名方法会按重复定义报错；覆盖基类方法时会检查 `final`、可见性和方法签名兼容性。

`Printer` 只能用于具名类。在命名空间中请使用 `#[\Printer]`，或先通过 `use \Printer;` 导入。

使用 `-m lib` 时，发布 stub 保留 `#[Printer]`，消费项目会得到相同的方法声明，实际实现由动态库提供。

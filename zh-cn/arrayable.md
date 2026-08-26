# Arrayable 编译期 Attribute

`Arrayable` 用于给类自动生成公开的 `toArray(): array` 方法。默认返回当前类和所有基类中的非静态 `public` 属性，并以属性名称作为关联数组的键。

```php
#[Arrayable]
class User
{
    public int $id = 1;
    public string $name = '张三';
    private string $password = '';
}

$data = (new User())->toArray();
// ['id' => 1, 'name' => '张三']
```

生成的 `toArray()` 等价于普通的手写方法：

```php
public function toArray(): array
{
    return [
        'id' => $this->id,
        'name' => $this->name,
    ];
}
```

该方法在**编译期**生成并直接参与类型分析和 AOT 编译。运行时不会扫描 Attribute、使用反射或动态查找字段，因此注解机制本身是零运行时开销。运行时只执行普通的属性读取和数组构造。

TypePHP 的 `$object->toArray()` 是关键词方法。底层数组转换 helper 检测到对象具有零参数 `toArray()` 后，会调用这里生成的方法并检查返回值确实为数组。因此 `Arrayable` 与 TypePHP 原有的对象数组转换语义完全兼容。

## 选择字段

通过可选的 `fields` 参数可以选择字段并控制结果顺序：

```php
#[Arrayable(fields: ['name', 'id'])]
class User
{
    private int $id = 1;
    protected string $name = '张三';
    public string $email = 'user@example.com';
}

// ['name' => '张三', 'id' => 1]
```

位置参数写法与之等价：

```php
#[Arrayable(['name', 'id'])]
class User {}
```

显式指定 `fields` 时，可以选择当前类中声明的 `public`、`protected` 或 `private` 实例属性，也可以选择基类中当前子类可访问的 `public` 或 `protected` 实例属性。不能选择动态或未知属性、静态属性以及基类的 `private` 属性；重复字段同样会在编译期报错。

属性值可以是任意类型。`Arrayable` 不执行字符串化或其他值转换，只读取属性当前值并直接写入结果数组。

- 完全省略参数：选择全部 public 实例属性。
- 传入空数组 `[]`：生成始终返回空数组的 `toArray()`。
- 属性按 `fields` 中指定的顺序写入结果。

`toArray()` 只保留属性当前的值，不会递归调用属性对象的 `toArray()`。public 属性 Hook 会像普通属性读取一样执行其 `get` Hook。

## 已有方法和继承

`Arrayable` 只负责在编译期生成一个普通的 `public function toArray(): array` 方法，不会因为当前类或基类已经存在同名方法而忽略生成。

生成后，该方法与手写方法遵循完全相同的 PHP 类方法规则：当前类中出现同名方法会按重复定义报错；覆盖基类方法时会检查 `final`、可见性和方法签名兼容性。

`Arrayable` 只能用于具名类。在命名空间中请使用 `#[\Arrayable]`，或先通过 `use \Arrayable;` 导入。

使用 `-m lib` 时，发布 stub 会保留 `#[Arrayable]` 及其字段列表。消费项目能够得到相同的 `toArray()` 声明，实际方法实体由动态库提供。

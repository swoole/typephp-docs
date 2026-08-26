# Constructor 编译期 Attribute

`Constructor` 设置在实例属性上，根据属性声明顺序生成一个 public 构造方法。

```php
class User
{
    #[Constructor]
    private int $id;

    #[Constructor]
    private string $name = 'typephp';
}
```

编译器在编译期生成等价方法：

```php
public function __construct(int $id, string $name = 'typephp')
{
    $this->id = $id;
    $this->name = $name;
}
```

属性类型和默认值会复制到构造参数。没有默认值的必填属性不能位于带默认值属性之后。`private`、`protected`、`public` 和 readonly 实例属性均受支持，静态属性不受支持。

`Constructor` 可以与属性生成注解组合：

```php
#[Constructor, Getter, With]
private int $id;
```

如果类已经声明 `__construct()`，编译器会报错，不会合并或覆盖用户构造方法。Trait 和 Enum 属性也不能使用该注解。

## 父类构造方法

生成的构造方法会按照 PHP 的继承规则检查父类构造方法：

- 父类及其继承链没有构造方法时，直接生成当前类构造方法；
- 最近的可继承父构造方法没有必填参数时，编译器会在所有属性赋值之前自动插入 `parent::__construct()`；
- 父构造方法存在必填参数时，编译器会报错。此时需要移除 `Constructor`，自行声明 `__construct()` 并显式传递父类参数；
- private 父构造方法不会被调用，生成的当前类构造方法仍然允许使用；
- final 父构造方法不能被覆盖，编译器会按普通 PHP 方法规则报错。

`Constructor` 不提供父构造参数映射，避免在属性注解中引入隐式、难以维护的参数传递规则。

构造方法在编译期静态生成并作为普通方法参与 AOT 编译，不需要运行时 Attribute、反射或动态调用。使用 `-m lib` 时，发布 stub 保留属性注解，消费项目获得相同构造方法声明。

在命名空间中请使用 `#[\Constructor]`，或先通过 `use \Constructor;` 导入。

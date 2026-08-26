# With 编译期 Attribute

`With` 生成一个不会修改原对象的方法：它克隆当前对象，在克隆对象上设置属性，然后返回克隆对象。

With 方法在**编译期静态生成**并作为普通方法参与 AOT 编译，不依赖运行时 Attribute、反射或动态方法调用。注解生成机制本身是零运行时开销；对象克隆和属性赋值则是该方法明确要求执行的操作。

```php
class User
{
    #[Getter, With]
    private string $name = '';
}

$original = new User();
$copy = $original->withName('张三');
```

编译器会生成等价方法：

```php
public function withName(string $name): static
{
    $clone = clone $this;
    $clone->name = $name;
    return $clone;
}
```

返回类型为 `static`，因此在继承层次中调用时仍返回当前运行时类的对象。对象克隆遵循 PHP 的常规 `clone`/`__clone()` 语义。

`With` 支持 `private`、`protected`、`public` 实例属性和构造器提升属性，也可以与 `Getter`、`Setter` 同时设置。不支持静态属性，不接受参数。

`With` 需要在 clone 对象上重新写入属性，因此不能用于显式 `readonly` 属性或 readonly class 中隐式 readonly 的属性。带有 `get` 或 `set` 属性 Hook 的属性也不能使用 `With`，避免生成方法与 Hook 的读写语义发生冲突；这些情况都会在编译期报错。

在命名空间中请使用 `#[\With]`，或先通过 `use \With;` 导入。

# MustUse 编译期 Attribute

`MustUse` 用于禁止调用方丢弃函数或方法的返回值，适合错误结果、不可变对象和必须显式处理的计算结果。

```php
#[MustUse]
function openFile(string $file): Result
{
}

openFile('data.txt');          // 编译错误
$result = openFile('data.txt'); // 正确
```

方法同样受支持：

```php
class User
{
    #[MustUse]
    public function withName(string $name): static
    {
    }
}
```

赋值、返回、作为参数传递、参与表达式或继续调用方法都视为使用了返回值。只有把调用本身写成独立表达式时才会报错。

`MustUse` 是纯编译期约束，不生成任何运行时代码，也不使用反射或动态检查。它不能用于返回 `void` 的函数或方法。

在命名空间中请使用 `#[\MustUse]`，或先通过 `use \MustUse;` 导入。使用 `-m lib` 时，发布 stub 会保留该声明，使消费项目也能执行相同检查。

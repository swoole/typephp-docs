# NotEmpty 编译期 Attribute

`NotEmpty` 用于普通函数、方法或匿名函数的参数。当 `empty($参数)` 为 `true` 时，在函数体最前面抛出 `ValueError`。不支持箭头函数，因为箭头函数只有一个表达式，没有可供插入检查语句的函数体。

```php
function save(#[NotEmpty] string $name): void
{
}
```

编译器静态插入等价检查：

```php
if (empty($name)) {
    throw new \ValueError('Parameter $name must not be empty');
}
```

它使用 PHP `empty()` 语义，因此 `null`、`false`、`0`、`0.0`、空字符串、字符串 `'0'` 和空数组都会失败。如果只希望拒绝 null，请使用 [`NotNull`](not-null.md)。

检查语句在编译期生成，不需要运行时解析 Attribute；实际的 `empty()` 判断会在每次调用时执行。`NotEmpty` 不接受参数，支持普通函数、方法、闭包和构造器提升参数。

同一参数组合多个校验 Attribute 时，检查顺序固定为 `NotNull`、`NotEmpty`、`Validate`，与 Attribute 的书写顺序无关。

在命名空间中请使用 `#[\NotEmpty]`，或先通过 `use \NotEmpty;` 导入。

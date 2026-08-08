# NotNull 编译期 Attribute

`NotNull` 用于普通函数、方法或匿名函数的参数。编译器会在函数体的其他语句之前检查参数；当参数严格等于 `null` 时抛出 `ValueError`。不支持箭头函数，因为箭头函数只有一个表达式，没有可供插入检查语句的函数体。

检查代码在**编译期静态插入**，运行时不会解析 Attribute，也不依赖反射或动态调用。Attribute 处理本身没有运行时开销；插入的严格 null 判断会在函数调用时执行。

```php
function save(#[NotNull] ?string $name): void
{
    // 业务代码
}
```

由于 `?string` 明确允许 `null`，而 `NotNull` 又在函数入口拒绝 `null`，编译器会对此报告一条非致命 warning，编译仍会继续。该提示也适用于显式包含 `null` 的联合类型；未声明类型和 `mixed` 不会仅因其运行时可能为 `null` 而触发此提示。

等价于在函数入口执行：

```php
if ($name === null) {
    throw new \ValueError('Parameter $name must not be null');
}
```

`0`、`0.0`、`false`、空字符串、字符串 `'0'` 和空数组都不是 null，因此不会触发 `NotNull`。需要拒绝这些空值时请使用 [`NotEmpty`](not-empty.md)。

多个参数使用 `NotNull` 时，检查按照参数声明顺序插入。它也可用于构造器提升属性对应的构造器参数。

需要验证邮箱、IP、URL、整数范围等格式时，可以与 [`Validate`](validate.md) 组合使用。

同一参数组合多个校验 Attribute 时，检查顺序固定为 `NotNull`、`NotEmpty`、`Validate`，与 Attribute 的书写顺序无关。

`NotNull` 不接受参数。在命名空间中请使用 `#[\NotNull]`，或先通过 `use \NotNull;` 导入。

# NotNull 编译期 Attribute

`NotNull` 用于函数或方法的参数。编译器会在函数体的其他语句之前检查参数；当 `empty($参数)` 为 `true` 时抛出 `ValueError`。

检查代码在**编译期静态插入**，运行时不会解析 Attribute，也不依赖反射或动态调用。Attribute 处理本身没有运行时开销；插入的 `empty()` 判断是 `NotNull` 所要求的实际参数校验，会在函数调用时执行。

```php
function save(#[NotNull] string $name): void
{
    // 业务代码
}
```

等价于在函数入口执行：

```php
if (empty($name)) {
    throw new \ValueError('Parameter $name must not be empty');
}
```

请注意这是 `empty()` 语义，不只是排除 `null`。例如 `null`、`false`、`0`、`0.0`、`''`、`'0'` 和空数组都会触发异常。若只希望排除 `null`，不要使用此 Attribute。

多个参数使用 `NotNull` 时，检查按照参数声明顺序插入。它也可用于构造器提升属性对应的构造器参数。

`NotNull` 不接受参数。在命名空间中请使用 `#[\NotNull]`，或先通过 `use \NotNull;` 导入。

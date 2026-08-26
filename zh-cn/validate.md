# Validate 编译期 Attribute

`Validate` 使用 PHP `filter_var()` 的验证规则检查普通函数、方法或匿名函数参数。检查语句在**编译期静态插入**；运行时不扫描 Attribute、不使用反射，也不进行动态调用。Attribute 处理本身没有运行时开销，生成的 `filter_var()` 检查则会在函数调用时执行。它不支持箭头函数，因为箭头函数只有一个表达式，没有可供插入检查语句的函数体。

## 基本用法

```php
function register(
    #[Validate(FILTER_VALIDATE_EMAIL)]
    string $email,
): void {
}
```

验证失败时会抛出：

```php
throw new \ValueError('Parameter $email is invalid');
```

`Validate` 只验证参数，不会使用 `filter_var()` 的返回值覆盖或转换原参数。

## PHP 官方参考

- [验证过滤器列表、可用 options 和 flags](https://www.php.net/manual/zh/filter.filters.validate.php)
- [所有 Filter 预定义常量](https://www.php.net/manual/zh/filter.constants.php#constant.filter-validate-bool)
- [`filter_var()` 函数说明](https://www.php.net/manual/zh/function.filter-var.php)
- [`FILTER_NULL_ON_FAILURE`](https://www.php.net/manual/zh/filter.constants.php#constant.filter-null-on-failure)

`Validate` 可使用的规则以 PHP 官方“验证过滤器”列表中的 `FILTER_VALIDATE_*` 为准。不同规则支持的 `options` 和 flags 也应以该列表为准。

## 构造参数

```php
#[Validate(
    int $filter,
    int|array $options = 0,
    ?string $message = null,
)]
```

- `filter`：必填，只允许 `FILTER_VALIDATE_*` 验证规则。
- `options`：可选，格式与 `filter_var()` 第三个参数一致，可以是整数 flags 或数组。
- `message`：可选，自定义 `ValueError` 信息。

不允许使用 `FILTER_SANITIZE_*`。该 Attribute 的职责是验证，而不是清洗或修改数据；传入清洗规则会产生编译错误。

## 参数类型兼容性

TypePHP 会在编译期拒绝能够静态证明为必然失败的 filter 与参数类型组合。例如：

```php
function register(
    #[Validate(FILTER_VALIDATE_EMAIL)] int $email,
): void {
}
```

任何 `int` 值都不可能符合邮箱格式，因此上述代码会产生编译错误。URL、IP 和 MAC 地址验证对 `int`、`float`、`bool` 等明确的非字符串标量采用相同规则。

PHP 的 `filter_var()` 会在过滤前将标量转换为字符串，因此 TypePHP 不会简单要求所有参数必须声明为 `string`。例如 `FILTER_VALIDATE_INT` 可以用于 `int` 或 `string`，`FILTER_VALIDATE_BOOLEAN` 也可以用于 `bool`、数值或字符串。

联合类型只要至少有一个成员可能通过验证就允许编译：

```php
function register(
    #[Validate(FILTER_VALIDATE_EMAIL)] int|string $email,
): void {
}
```

`array` 参数默认与标量 filter 不兼容；显式使用 `FILTER_REQUIRE_ARRAY` 或 `FILTER_FORCE_ARRAY` 后才视为数组模式。对于未声明类型、`mixed`、可调用对象和无法在当前阶段证明必然失败的对象类型，编译器保留运行时验证，不根据经验作出拒绝判断。

## 范围和 flags

```php
function connect(
    #[Validate(
        FILTER_VALIDATE_INT,
        options: [
            'options' => [
                'min_range' => 1,
                'max_range' => 65535,
            ],
        ],
        message: '端口号必须在 1 到 65535 之间',
    )]
    int $port,
): void {
}
```

整数 flags 也可以直接传入：

```php
function connect(
    #[Validate(FILTER_VALIDATE_IP, FILTER_FLAG_IPV4)]
    string $address,
): void {
}
```

编译器会自动把 `FILTER_NULL_ON_FAILURE` 合并到 `options` 中，并使用严格的 `=== null` 判断验证失败。这一点可以区分 `FILTER_VALIDATE_BOOLEAN` 返回的合法 `false` 和真正的验证失败。

## 与 NotNull 组合

`Validate` 可以与 `NotNull` 或 `NotEmpty` 同时使用。同一参数上的检查顺序固定为 `NotNull`、`NotEmpty`、`Validate`，与 Attribute 的书写顺序无关：

```php
function register(
    #[Validate(FILTER_VALIDATE_EMAIL)]
    #[NotEmpty]
    #[NotNull]
    string $email,
): void {
}
```

上例即使反向书写 Attribute，也会依次执行 null 检查、`empty()` 检查和邮箱格式验证。

## 命名空间和 library

`Validate` 位于根命名空间。在命名空间中请使用 `#[\Validate(...)]`，或先通过 `use \Validate;` 导入，也支持 `use ... as ...` 别名。

使用 `-m lib` 时，发布的 `<target>.stub.php` 会保留 `Validate` 声明；库内部包含实际检查代码，消费项目使用生成的方法或函数声明，不需要运行时解析 Attribute。

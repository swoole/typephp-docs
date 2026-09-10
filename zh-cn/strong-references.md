# 强类型引用

TypePHP 支持在不改变局部变量固定类型的前提下使用引用。编译器会根据被调用目标能否在编译期确定，选择两条不同的引用路径。

## 静态调用的原生引用

固定类型的 `int`、`float`、`bool`、`string`、`array` 局部变量可以传给类型完全匹配的引用参数。函数或方法能被静态解析时，TypePHP 直接生成 C++ `T&`，不需要把值装箱为 `zval`，也不会分配 Zend 引用。

```php
function increment(int &$value): void
{
    $value++;
}

function main(): void
{
    $count = 1;
    increment($count); // $count 始终是 int
    var_dump($count);  // int(2)
}
```

参数类型必须与局部变量的固定类型完全匹配，不能通过引用向局部变量写入其他类型。

Native Class 中固定类型为 `int`、`float`、`bool`、`string`、`array` 的属性，也可以直接传给类型完全匹配的引用参数：

```php
#[Native]
class Counter
{
    public int $value = 0;
}

function increment(int &$value): void
{
    $value++;
}

$counter = new Counter();
increment($counter->value);
```

这里生成的是仅在本次调用期间有效、直接指向固定字段的 C++ `T&`，并没有创建 PHP 引用。因此不能使用 `=&` 为该属性建立别名，不能将它包装为 `std::ref()`，也不能按引用返回或让引用在调用后继续存在。可空、联合、`mixed`、带 hook 或 readonly 的 Native 属性不走此路径；显式声明为 `any` 的 Native 属性则使用普通的 PHP 动态引用模型。

签名在编译期已知的普通函数和方法也支持引用变长参数，包括直接参数、命名参数与参数展开；动态闭包不能声明引用变长参数。

## 局部引用别名

固定类型局部变量可以建立一次、无条件且仅限当前函数的引用别名：

```php
function main(): void
{
    $name = "TypePHP";
    $alias =& $name;
    $alias .= " compiler";

    var_dump($name); // string(16) "TypePHP compiler"
}
```

该别名是目标永久固定的 C++ 引用，因此不能重新绑定，不能只在条件分支或循环中首次绑定，也不能被 `unset()`、按引用捕获、按引用返回，或保存到数组、属性、全局变量中。

## 动态调用：`std::ref()` 与 `toRef()`

调用闭包、可变函数或其他无法在编译期解析签名的目标时，需要显式标记引用参数：

```php
function main(): void
{
    $rename = function (string &$value): void {
        $value .= " compiler";
    };

    $name = "TypePHP";
    $rename(std::ref($name));
    // 等价写法：$rename($name->toRef());
}
```

`std::ref()` 与 `toRef()` 只能接受变量、数组元素或对象属性，并且只能作为调用参数使用。TypePHP 会创建一个仅在本次调用期间有效的 Zend 引用，调用后验证类型再写回；如果动态代码试图让这个临时引用逃逸到调用之外，运行时会报错。

签名在编译期已知的调用会自动识别引用参数，不要额外添加 `std::ref()`。

## 支持边界

原生局部引用支持 `int`、`float`、`bool`、`string`、`array`。固定 object、resource/stream、高精度类型、Native Class 对象句柄、typed object、Box 与 Std 容器局部变量不能建立引用；这些值已经具有句柄语义或固定布局，允许重新绑定会破坏类型系统。Native Class 对象句柄不能引用，并不妨碍固定 Native 属性直接传给类型完全匹配的强引用参数。

typed object 属性和静态属性仍可通过 Zend 的 typed-property 检查建立引用；PHP 数组元素也仍是支持动态引用的存储位置。

如果代码需要引用逃逸、重新绑定等完整 PHP 引用身份，应使用 `std::any()` 初始化动态值，走 `php::Var` 引用路径，而不是固定类型局部变量。

## 如何选择

| 场景 | 写法 |
|---|---|
| 已知 TypePHP 函数/方法，且参数是完全匹配的 `&` 类型 | 直接传入变量 |
| 固定 Native 属性，且目标是静态已知的同类型 `&` 参数 | 直接传入属性 |
| 建立一个目标永久固定的局部别名 | `$alias =& $value` |
| 闭包、可变函数或动态方法调用 | `std::ref($value)` 或 `$value->toRef()` |
| 引用需要逃逸、重新绑定或完整 PHP 引用身份 | 使用 `std::any()` 存储值 |

`std::ref()` 调用包装器详见[编译期函数](compile-time-functions.md)，`toRef()` 详见[关键词方法](keyword-method.md)。

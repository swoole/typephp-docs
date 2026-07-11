# 扩展对象方法

扩展对象方法可以在不修改原有类的情况下，为对象增加便于调用的新方法。

例如，`App\User` 类只有一个 `name` 属性，我们可以在类的外部定义一个 `User_displayName()` 函数，然后像调用普通对象方法一样调用 `$user->displayName()`。

```php
namespace App;

class User
{
    public function __construct(public string $name)
    {
    }
}

function User_displayName(User $user): string
{
    return strtoupper($user->name);
}

function main(): void
{
    $user = new User('Alice');

    echo $user->displayName(); // ALICE
}
```

`User_displayName()` 是普通函数，TypePHP 允许将它写成更自然的对象方法调用形式：

```php
$user->displayName();
```

## 1. 定义扩展方法

定义扩展对象方法时，需要遵守三个基本规则：

1. 扩展函数必须与类处于相同的命名空间。
2. 函数名以类的短名称开头，格式为 `{类名}_{方法名}`；类名和方法名均不做命名风格转换。
3. 函数的第一个参数必须声明为要扩展的类。

下面为 `App\User` 增加一个 `greeting()` 方法：

```php
<?php

namespace App;

class User
{
    public function __construct(public string $name)
    {
    }
}

function User_greeting(User $user): string
{
    return 'Hello, ' . $user->name;
}

function main(): void
{
    $user = new User('Tom');

    echo $user->greeting(); // Hello, Tom
}
```

调用 `$user->greeting()` 时，不需要手动传入第一个 `$user` 参数。TypePHP 会自动把当前对象传给扩展函数。

## 2. 添加方法参数

扩展函数从第二个参数开始，对应方法调用中写入的参数。

```php
<?php

namespace App;

use native_types;

class User
{
    public function __construct(public string $name)
    {
    }
}

function User_greeting(User $user, string $prefix, string $suffix): string
{
    return $prefix . $user->name . $suffix;
}

function main(): void
{
    $user = new User('Alice');

    echo $user->greeting('Hello, ', '!'); // Hello, Alice!
}
```

参数对应关系如下：

```php
// 扩展函数
User_greeting($user, 'Hello, ', '!');

// 对象方法写法
$user->greeting('Hello, ', '!');
```

## 3. 默认参数

扩展方法可以像普通函数一样使用默认参数：

```php
<?php

namespace App;

use native_types;

class User
{
    public function __construct(public string $name)
    {
    }
}

function User_greeting(
    User $user,
    string $prefix = 'Hello, ',
    string $suffix = '!'
): string {
    return $prefix . $user->name . $suffix;
}

function main(): void
{
    $user = new User('Alice');

    echo $user->greeting();                  // Hello, Alice!
    echo $user->greeting('Welcome, ');       // Welcome, Alice!
    echo $user->greeting('Hi, ', ' :)');     // Hi, Alice :)
}
```

## 4. 可变参数

需要接收任意数量的参数时，可以使用 PHP 的可变参数语法：

```php
<?php

namespace App;

use native_types;

class User
{
    public function __construct(public string $name)
    {
    }
}

function User_has_any_role(User $user, string ...$roles): bool
{
    foreach ($roles as $role) {
        if ($role === 'admin' && $user->name === 'Alice') {
            return true;
        }
    }

    return false;
}

function main(): void
{
    $user = new User('Alice');

    var_dump($user->hasAnyRole('guest', 'admin', 'editor')); // bool(true)
}
```

## 5. 名称必须一致

TypePHP 不会在 camelCase、PascalCase 和 snake_case 之间自动转换。类名部分和方法名部分都必须与调用保持一致。

例如，`UserService` 的 `displayName()` 扩展方法必须定义为：

```php
function UserService_displayName(UserService $service): string
{
    return $service->name;
}

$service->displayName();
```

下面的函数不会匹配 `$service->displayName()`：

```php
function UserService_display_name(UserService $service): string {}
function user_service_displayName(UserService $service): string {}
function user_service_display_name(UserService $service): string {}
```

如果希望使用 snake_case 方法名，函数和调用必须同时使用 snake_case：

```php
function UserService_display_name(UserService $service): string
{
    return $service->name;
}

$service->display_name();
```

名称匹配不区分字母大小写，这一点与 PHP 函数名规则一致。例如 `UserService_displayName()` 可以匹配 `$service->DISPLAYNAME()`。但是下划线的位置必须完全一致。

## 6. 命名空间要求

扩展函数必须与目标类处于同一个命名空间。

```php
<?php

namespace App\Model {
    class User
    {
        public function __construct(public string $name)
        {
        }
    }

    // 正确：函数和 User 都位于 App\Model
    function User_display_name(User $user): string
    {
        return $user->name;
    }
}

namespace App\Support {
    use App\Model\User;

    // 错误：该函数位于 App\Support，不会成为 App\Model\User 的扩展方法
    function User_short_name(User $user): string
    {
        return substr($user->name, 0, 1);
    }
}

namespace {
    function main(): void
    {
        $user = new \App\Model\User('Alice');

        echo $user->displayName(); // Alice
        // $user->shortName();     // 不会找到 App\Support\User_short_name()
    }
}
```

类位于根命名空间时，扩展函数也必须定义在根命名空间：

```php
class User {}

function User_display_name(User $user): string
{
    return 'User';
}
```

## 7. 第一个参数必须是目标类

函数名正确但第一个参数类型不正确时，该函数不会成为扩展方法。

```php
namespace App;

class User {}

// 正确
function User_label(User $user): string
{
    return 'user';
}

// 错误：第一个参数不是 User
function User_identifier(int $user): string
{
    return (string) $user;
}

// 错误：第一个参数不能按引用传递
function User_reset(User &$user): void
{
}
```

第一个参数应直接声明为目标类。不要使用 `mixed`、父类或接口代替目标类。

## 8. 返回类型和链式调用

建议为扩展函数声明明确的返回类型。TypePHP 可以据此检查后续代码，并支持链式调用。

```php
<?php

namespace App;

use native_types;

class User
{
    public function __construct(public string $name)
    {
    }
}

function User_display_name(User $user): string
{
    return '  ' . $user->name . '  ';
}

function User_rename(User $user, string $name): User
{
    $user->name = $name;
    return $user;
}

function main(): void
{
    $user = new User('Alice');

    // displayName() 返回 string，可以继续调用字符串通用方法
    echo $user->displayName()->trim()->upper(); // ALICE

    // rename() 返回 User，可以继续调用 User 的扩展方法
    echo $user->rename('Bob')->displayName();   // Bob
}
```

## 9. 与类方法和 __call() 的关系

类中真正定义的方法优先级最高。扩展方法不能覆盖已有的类方法。

```php
<?php

namespace App;

class User
{
    public function name(): string
    {
        return 'class method';
    }
}

function User_name(User $user): string
{
    return 'extension method';
}

function main(): void
{
    $user = new User();

    echo $user->name(); // class method
}
```

如果类中没有对应方法，TypePHP 会查找扩展方法；找不到扩展方法时，才调用 `__call()`：

```php
<?php

namespace App;

use native_types;

class User
{
    public function __call(string $method, array $args): string
    {
        return 'magic: ' . $method;
    }
}

function User_greeting(User $user): string
{
    return 'extension greeting';
}

function main(): void
{
    $user = new User();

    echo $user->greeting(); // extension greeting
    echo $user->missing();  // magic: missing
}
```

查找顺序可以简单理解为：

1. 类中定义的方法
2. 扩展对象方法
3. `__call()`

## 10. 为第三方类添加便捷方法

扩展对象方法适合为不能直接修改的类添加项目内的便捷操作。例如，可以为数据对象增加格式化方法：

```php
<?php

namespace App\Dto;

use native_types;

class Order
{
    public function __construct(
        public int $id,
        public int $amount,
    ) {
    }
}

function Order_format_amount(Order $order, string $currency = 'CNY'): string
{
    return $currency . ' ' . number_format($order->amount / 100, 2);
}

function Order_summary(Order $order): string
{
    return '#' . $order->id . ' - ' . $order->formatAmount();
}

function main(): void
{
    $order = new Order(1001, 129900);

    echo $order->formatAmount();       // CNY 1,299.00
    echo $order->formatAmount('USD');  // USD 1,299.00
    echo $order->summary();            // #1001 - CNY 1,299.00
}
```

扩展函数内部也可以继续调用该类的普通方法或其他扩展方法。

## 11. 使用限制

扩展对象方法只在 TypePHP 编译的静态代码中有效，不会改变 PHP 类本身。

以下写法支持扩展方法：

```php
$user->displayName();
(new User('Alice'))->displayName();
$service->findUser()->displayName();
```

以下写法不会查找扩展对象方法：

```php
// 动态方法名
$method = 'displayName';
$user->$method();

// 静态方法调用
User::displayName();

// 动态回调
call_user_func([$user, 'displayName']);
```

PHP 动态脚本、`eval()` 中的代码以及未经 TypePHP 静态编译的 PHP 代码也不能使用扩展方法调用语法。

如果变量是 `mixed`，TypePHP 不知道它具体属于哪个类。可以先恢复对象类型，再调用扩展方法：

```php
$user = $value->toObject(User::class);
echo $user->displayName();
```

## 12. 完整示例

下面的示例集中展示了普通参数、默认参数、可变参数、链式调用和 `__call()` 优先级：

```php
<?php

namespace App;

use native_types;

class User
{
    public function __construct(
        public string $name,
        public int $age,
    ) {
    }

    public function __call(string $method, array $args): string
    {
        return 'undefined: ' . $method;
    }
}

function User_display_name(User $user, string $prefix = ''): string
{
    return $prefix . $user->name;
}

function User_is_adult(User $user, int $minimumAge = 18): bool
{
    return $user->age >= $minimumAge;
}

function User_with_name(User $user, string $name): User
{
    $user->name = $name;
    return $user;
}

function User_join_labels(User $user, string ...$labels): string
{
    return $user->name . ': ' . implode(', ', $labels);
}

function main(): void
{
    $user = new User('Alice', 20);

    echo $user->displayName('User: ');              // User: Alice
    var_dump($user->isAdult());                     // bool(true)
    var_dump($user->isAdult(21));                   // bool(false)
    echo $user->joinLabels('admin', 'active');      // Alice: admin, active
    echo $user->withName('Bob')->displayName();     // Bob
    echo $user->notDefined();                       // undefined: notDefined
}
```

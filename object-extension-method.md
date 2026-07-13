# 扩展对象方法

扩展对象方法允许在不修改原有类的情况下增加 TypePHP 静态方法调用。扩展通过类级 `#[ExtensionProvider(ClassName::class)]` 声明。

```php
<?php

namespace App;

class User
{
    public function __construct(public string $name) {}
}

#[\ExtensionProvider(User::class)]
final class UserExtensions
{
    public static function displayName(User $user): string
    {
        return strtoupper($user->name);
    }

    public static function greeting(
        User $user,
        string $prefix = 'Hello, '
    ): string {
        return $prefix . $user->name;
    }
}

function main(): void
{
    $user = new User('Alice');
    echo $user->displayName();       // ALICE
    echo $user->greeting('Welcome, '); // Welcome, Alice
}
```

## 定义规则

- `ExtensionProvider` 位于根命名空间；在命名空间内使用 `#[\ExtensionProvider(...)]`。
- Attribute 参数是目标对象的 `ClassName::class`。
- 扩展方法必须是 `public static`。
- 第一个参数必须是目标对象类型，并且不能按引用传递。
- 调用参数对应静态方法的第二个及后续参数。
- private/protected 方法不会注册，可以作为内部 helper。
- 方法名直接作为扩展方法名，不进行命名风格转换。

```php
#[\ExtensionProvider(User::class)]
final class UserExtensions
{
    public static function rename(User $user, string $name): User
    {
        $user->name = $name;
        return $user;
    }

    private static function normalize(string $name): string
    {
        return trim($name);
    }
}
```

## 返回类型与链式调用

返回类型会参与后续类型推断：

```php
echo $user
    ->rename('Bob')
    ->displayName()
    ->trim()
    ->upper();
```

## 查找优先级

对象调用按以下顺序处理：

1. 类中真实存在的实例方法
2. 对象扩展方法
3. `__call()`
4. 未定义方法错误

扩展不能覆盖真实实例方法。如果没有真实方法但存在有效扩展，扩展优先于 `__call()`。

## 使用限制

支持：

```php
$user->displayName();
(new User('Alice'))->displayName();
$service->findUser()->displayName();
```

不参与扩展查找：

```php
$method = 'displayName';
$user->$method();

User::displayName();
call_user_func([$user, 'displayName']);
```

扩展对象方法仅在 TypePHP 静态代码中生效，不会向 ZendVM 动态添加实例方法。完整的通用类型、关键词和对象 Provider 说明见[扩展方法提供者](extension-provider.md)。


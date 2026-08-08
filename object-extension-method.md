# 扩展对象方法

扩展对象方法允许在不修改原有类的情况下增加 TypePHP 静态方法调用。扩展通过类级 `#[MethodsFor(ClassName::class)]` 声明。

```php
<?php

namespace App;

class User
{
    public function __construct(public string $name) {}
}

#[\MethodsFor(User::class)]
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

- `MethodsFor` 位于根命名空间；在命名空间内可以使用 `#[\MethodsFor(...)]`，也可以先执行 `use \MethodsFor;` 后使用 `#[MethodsFor(...)]`。
- Attribute 参数是目标对象的 `ClassName::class`。
- 目标必须是类，不能是接口。
- 扩展方法必须是 `public static`。
- 第一个参数必须是目标对象类型，并且不能按引用传递。
- 调用参数对应静态方法的第二个及后续参数。
- private/protected 方法不会注册，可以作为内部 helper。
- 方法名直接作为扩展方法名，不进行命名风格转换。

Attribute 名称和 `ClassName::class` 参数都遵循 PHP 的命名空间解析规则，也支持别名：

```php
namespace App\Extension;

use \MethodsFor as Provider;
use App\Model\User as ModelUser;

#[Provider(ModelUser::class)]
final class UserExtensions
{
    public static function displayName(ModelUser $user): string
    {
        return strtoupper($user->name);
    }
}
```

未导入时，命名空间中的 `#[MethodsFor(...)]` 指向当前命名空间下的同名 Attribute，不会触发 TypePHP 的扩展方法功能。

```php
#[\MethodsFor(User::class)]
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

1. 内置关键词方法和 `MethodsFor('*')` 关键词扩展
2. 类或接口中真实存在的实例方法
3. 静态类型对应类的对象扩展
4. 父类对象扩展，由近到远
5. `MethodsFor(Type::Object)`，仅限静态阶段确定 receiver 一定是对象
6. `__call()`
7. 未定义方法错误

扩展不能覆盖真实实例方法。如果没有真实方法但存在有效扩展，扩展优先于 `__call()`。
查找只依据静态类型：静态声明为父类的变量不会查找运行时子类的 Provider。子类和父类都提供同名扩展时，距离静态类型最近的 Provider 优先。

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

扩展对象方法仅在 TypePHP 静态代码中生效，不会向 ZendVM 动态添加实例方法。完整的通用类型、关键词和对象 Provider 说明见[扩展方法提供者](methods-for.md)。

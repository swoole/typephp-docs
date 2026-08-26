# 命名限制

## 符号唯一性

`TypePHP` 会将命名空间、类名和函数（或方法）名组合为 `C++` 函数符号。每个名称单独看都可能是合法的 `PHP` 名称，但组合后的名称仍然必须在整个编译项目中保持唯一，不能与其他函数或类方法产生重复定义。

尤其需要注意：命名空间层级与“类名 + 方法名”的边界在生成 `C++` 符号时使用相同的分隔方式。因此，下面两个 `PHP` 声明会产生冲突：

```php
namespace App {
    class Model
    {
        public function validate(): void
        {
        }
    }
}

namespace App\Model {
    function validate(): void
    {
    }
}
```

虽然 `App\Model::validate()` 是类方法，`App\Model\validate()` 是命名空间函数，但它们都会映射到同一个 C++ 符号：

```text
php_app__model__validate
```

TypePHP 不允许这种组合后的重复定义。编译器检测到冲突时会抛出 `Fatal error`，并列出发生冲突的两个 PHP 名称及对应的 C++ 符号。需要修改命名空间、类名、函数名或方法名中的至少一个，使组合后的完整名称不再冲突，例如将命名空间函数改名为 `validate_data()`。

连续下划线也可能与编译器使用的分隔符发生冲突。例如，下面的命名空间函数和类方法同样不能同时存在：

```php
namespace App;

function user__test(): void
{
}

class User
{
    public function test(): void
    {
    }
}
```

`App\user__test()` 和 `App\User::test()` 都会映射到 `php_app__user__test`。因此，不能只检查单独的命名空间、类名或函数名是否重复；命名空间、类名、函数名及方法名组合后生成的完整编译符号也不得重复。

此限制针对整个静态编译范围，包括项目源码和被纳入编译的第三方库；即使两个声明位于不同文件中，也不能产生相同的编译符号。

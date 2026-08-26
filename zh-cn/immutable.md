# Immutable 编译期 Attribute

`#[Immutable]` 用于声明一段代码只能读取对象或参数，不能修改它。它类似 C++ 的 `const`：检查完全发生在 TypePHP 编译阶段，不创建只读代理对象，不写入 Zend 元数据，也不增加运行时分支。

它适合查询方法、值对象、只读服务接口，以及希望由编译器阻止意外修改的公共 API。

所有内建 Attribute 的共同命名空间规则参见[编译期注解](compile-time-attributes.md)。

## 不可变方法

把 `#[Immutable]` 放在实例方法上，该方法中的 `$this` 会被视为不可变对象：

```php
class User
{
    private string $name = 'Rango';

    #[Immutable]
    public function name(): string
    {
        return $this->name;
    }

    public function rename(string $name): void
    {
        $this->name = $name;
    }

    #[Immutable]
    public function description(): string
    {
        return 'User: ' . $this->name(); // 允许：name() 也是 Immutable
    }
}
```

不可变方法只能调用其他不可变方法。下面的调用会在编译期失败：

```php
class User
{
    #[Immutable]
    public function invalid(): void
    {
        $this->rename('new name');
        // Fatal error: 不能在 immutable $this 上调用 mutable 方法
    }
}
```

`#[Immutable]` 不能放在普通函数声明上，因为普通函数没有 `$this`。应把它放在需要保护的函数参数上。

## 不可变参数

参数上的 `#[Immutable]` 同时保护参数变量和它引用的对象：

```php
function display(#[Immutable] User $user): string
{
    return $user->name();
}

function total(#[Immutable] array $values): int
{
    return count($values);
}
```

在函数、方法、构造方法和 Closure 参数上都可以使用。不可变参数不能被重新赋值、修改元素或传入可能修改它的参数：

```php
function invalid(#[Immutable] array $values, #[Immutable] User $user): void
{
    $values[] = 1;           // 编译错误
    sort($values);           // 编译错误：sort() 的参数是可写引用
    $user = new User();      // 编译错误
    $user->rename('other');  // 编译错误
}
```

被调用函数也可以用 `#[Immutable]` 明确接收不可变对象：

```php
function userName(#[Immutable] User $user): string
{
    return $user->name();
}
```

不可变引用参数是合法的，其语义类似 C++ `const &`：调用仍按引用传递，但函数体不能修改该值。

```php
function inspect(#[Immutable] User &$user): string
{
    return $user->name();
}
```

## 编译器禁止的操作

对不可变的 `$this`、参数或由它建立的对象别名，编译器会禁止：

- 重新赋值、解构赋值、属性写入和数组元素写入；
- `+=`、`.=` 等复合赋值，以及 `++`、`--`；
- `unset()`、取引用和 `foreach (... as &$value)`；
- 调用未标记为 `#[Immutable]` 的确定对象方法；
- 把对象传给没有 `#[Immutable]` 的确定参数；
- 传给可写的引用参数；
- 把不可变对象保存到可变属性、数组、global 或 static 状态；
- 将不可变对象直接 `return` 或 `yield`，使其作为可变对象逃逸。

局部别名不会解除限制：

```php
function inspect(#[Immutable] User $user): string
{
    $alias = $user;
    $alias->rename('other'); // 编译错误，$alias 仍然不可变
    return $alias->name();
}
```

`clone` 会创建具有独立身份的新对象，因此 clone 的结果是可变的：

```php
function renamedCopy(#[Immutable] User $user): User
{
    $copy = clone $user;
    $copy->rename('copy'); // 允许
    return $copy;
}
```

对于标量和 PHP copy-on-write 值，正常读取和复制不属于修改。例如可以复制不可变数组，也可以调用 `count()`；只有写入原值或把它交给可写引用参数时才会失败。

## Property Hook

通过不可变对象读取 Property Hook 时，getter 本身也必须声明为 `#[Immutable]`：

```php
class Profile
{
    public string $name = 'Rango' {
        #[Immutable]
        get => strtoupper($this->name);
    }

    #[Immutable]
    public function displayName(): string
    {
        return $this->name;
    }
}
```

Property Hook 的声明和运行环境要求参见 PHP 8.4 属性 Hook 相关规则。

## MethodsFor 扩展方法

对象扩展方法只有在 receiver 参数也声明为 `#[Immutable]` 时，才能作用于不可变对象：

```php
#[MethodsFor(User::class)]
class UserExtensions
{
    public static function label(#[Immutable] User $user): string
    {
        return $user->name();
    }
}

function label(#[Immutable] User $user): string
{
    return $user->label();
}
```

数组、字符串等关键词扩展方法同样按效果区分。`$values->count()` 这类读取操作可用，`$values->sort()` 这类会修改 receiver 的操作会被拒绝。

## 继承、Trait 和闭包

Immutable 契约会随代码结构继续传播：

- 子类覆盖方法时可以新增 `#[Immutable]`，但不能移除父类或 Interface 已声明的 Immutable 方法/参数契约；
- Trait 注入的方法保留 `#[Immutable]`；
- Closure、箭头函数和 Generator 函数体会保留捕获变量及 `$this` 的不可变状态；
- library stub 会保留公开 API 上的 `#[Immutable]`，消费方编译时继续检查同一契约。

## 动态调用是显式逃逸口

`#[Immutable]` 是静态检查工具，不是安全沙箱。目标被动态语法隐藏后，编译器不会插入运行时只读代理：

```php
function escape(#[Immutable] User $user): void
{
    $method = 'rename';
    $user->$method('changed'); // 动态调用：编译器不检查目标方法效果
}
```

变量 callable、Reflection、`eval()` 和其他 ZendVM 动态代码也属于同类逃逸口。不要把 `#[Immutable]` 当作处理不可信代码的权限边界；它的目标是在完全静态的常规 TypePHP 代码中，以零运行时成本发现意外修改。

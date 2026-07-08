# 对象类型转换

对象类型转换用于在编译期恢复或声明对象的类信息，使编译器可以生成更高效、更安全的 C++ 代码。它主要涉及三类场景：

- `objval($value, ClassName::class)`
- `$value->toObject(ClassName::class)`
- typed object 变量赋值、函数返回值、属性写入等类型兜底检查

这些场景都与“对象是否属于某个类”有关，但并不都只发生在编译期。

## 1. 类型关系

对象类型判断使用 PHP 的 `is-a` 语义，也就是 `instanceof` 关系：

- `Child` 对象可以作为 `Base` 对象使用。
- `Base` 对象不能作为 `Child` 对象使用。
- 接口和抽象类按 PHP 的实现/继承关系判断。

运行时底层使用 `php::toObject($value, ce)` 检查对象类型。该函数要求 `$value` 必须是对象，并且 `value instanceof ce` 成立，否则抛出异常。

## 2. 编译期行为

以下信息会在编译期处理，不依赖运行时判断。

### 2.1 类型声明和类型推断

当编译器可以从语法直接得到类名时，会记录变量的 typed object 信息：

```php
$user = new User();
```

此时 `$user` 会被记录为 `User` 类型。后续调用 `$user->method()` 时，编译器可以直接解析到 `User::method()`，生成 native call。

### 2.2 静态可证明安全的赋值

如果右值类型在编译期可以证明是左值类型本身或子类，赋值直接通过，不插入运行时检查：

```php
$base = new Base();
$base = new Child(); // Child is-a Base，编译期安全
```

### 2.3 静态可证明错误的赋值

如果编译期可以确定对象类型不兼容，会直接报编译期错误：

```php
$child = new Child();
$child = new Base(); // Base 不是 Child，编译期错误
```

无关类之间的赋值也会在编译期报错。

### 2.4 `objval()` 和 `toObject(Class::class)` 的类型接续

`objval()` 和带类名参数的 `toObject()` 会在编译期告诉编译器“这个表达式后续按指定类处理”：

```php
$user = objval($data['user'], User::class);
$user->getName(); // 编译器按 User 类型解析

$user = $data['user']->toObject(User::class);
$user->getName(); // 同上
```

这里的类名参数必须能在编译期解析，例如字符串字面量、`ClassName::class`、`self::class`、`parent::class`。

## 3. 运行时行为

以下情况必须在运行时检查，因为编译期无法完全确定真实对象类型。

### 3.1 `objval()` 和 `toObject(Class::class)` 的对象检查

虽然 `objval()` / `toObject(Class::class)` 会在编译期接续类型信息，但它们仍会生成运行时检查：

```php
$user = objval($value, User::class);
```

生成的核心逻辑等价于：

```cpp
php::toObject(value, ce_User)
```

运行时会检查：

- `$value` 必须是对象。
- `$value instanceof User` 必须成立。

因此，`AdminUser extends User` 的对象可以通过 `User::class` 检查；普通 `stdClass` 或其他无关类对象不能通过。

### 3.2 typed object 赋值兜底检查

当右值是 `mixed` / `any`，或右值的声明类型不够精确时，编译器会插入运行时检查：

```php
$child = new Child();
$child = any($value);
```

编译器不能在静态阶段知道 `$value` 的真实类，因此会在运行时检查 `$value instanceof Child`。

函数或方法的返回类型如果是父类、接口或抽象类，也属于不够精确的情况：

```php
function make(): Base {
    return new Child();
}

$child = new Child();
$child = make(); // 运行时检查返回值是否 instanceof Child
```

如果 `make()` 实际返回 `Child`，检查通过；如果返回 `Base`，检查失败。

### 3.3 Std 容器对象元素转换

Std 容器中保存对象类型时，从容器中取出元素并恢复为对象类型也会走运行时对象检查：

```php
$objects = std::vector(User::class);
$user = $objects[0];
```

当容器元素类型是具体类名时，编译器知道目标类，但运行时仍需要确保取出的值是该类或其子类对象。

## 4. 不做精确类比较

对象类型转换不使用“精确类相等”规则，不要求 `get_class($value) === ClassName::class`。

```php
class Base {}
class Child extends Base {}

$base = objval(new Child(), Base::class);      // 合法
$base = (new Child())->toObject(Base::class);  // 合法
```

如果确实需要精确类比较，应在业务代码中显式使用 `get_class()` 或 `$value::class` 判断。对象类型转换本身遵循 PHP 类型系统的 `is-a` 规则。

## 5. 选择建议

- 已经有准确静态类型时，不需要使用 `objval()` 或 `toObject(Class::class)`。
- 从数组、动态返回值、`mixed` / `any` 值中取出对象后，需要恢复类信息时，使用 `objval()` 或 `toObject(Class::class)`。
- 希望保留动态类型时，使用 `toAny()` 或 `any()`，不要恢复为 typed object。
- 需要强制引用时，使用 `toRef()` 或 `refval()`，这与对象类型转换无关。


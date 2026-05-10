## 在 PHP 代码中调用 C++ 函数

`C++`函数需满足以下条件，就可以在`PHP`代码调用：

1. 必须以 `php_` 为前缀
2. 必须以 `PHP` 类型作为参数和返回值
3. 必须在 `.stub.php` 文件中声明该函数

> 在`C++`中函数名称必须为小写，在`PHP`代码中函数名称是大小写不敏感的

### PHP 与 C++ 类型映射表

| PHP 类型   | C++ 类型        |
| -------- | ------------- |
| int      | php::Int      |
| bool     | php::Bool     |
| array    | php::Array    |
| mixed    | php::Var      |
| float    | php::Float    |
| string   | php::Str      |
| object   | php::Object   |
| resource | php::Resource |
| void     | void          |

例如在`.stub.php`文件中声明一个函数：

```php
function bar(int $a, bool $b, float $c, string $d, array $e, object $f, mixed $g): array {
   // stub 仅声明，函数没有实现代码
}
```

则在 `C++` 中对应的函数为:
```cpp
php::Array php_bar(php::Int a, php::Bool b, php::Float c, php::String d, php::Array e, php::Object f, php::Var g) {
   // 在 C++ 代码中实现此函数
}
```

可使用`using namespace php` 简化为：
```cpp
using namespace php;

Array php_bar(Int a, Bool b, Float $c, String d, Array e, Object f, Var g) {
   // 在 C++ 代码中实现此函数
}
```

> 注意在`C++`代码中函数名称必须添加`php_`前缀

在其他 `PHP` 代码中调用此函数：
```php
$arr = bar(1, false, 3.14, "hello", [1, 2, 3,], new stdClass, null);
```

## 在 C++ 代码中调用 PHP 函数
1. 函数名称必须添加 `php_` 前缀
1. 必须包含 `php_func_decl.h` 头文件
1. 仅限于被`AOT`编译器编译后的函数，若是动态函数、内置函数，无法直接调用

例如在`PHP`中定义了一个函数：
```php
function my_func($a, $b, $c): mixed {
    var_dump($a, $b, $c);
    return [$a, $b, $c];
}
```

`C++`中调用的方式为：
```cpp
php::Array list = php_my_func("hello", 1234, php::null);
```

### 调用内置函数
使用`PHPX`提供的`Facade API`

```cpp
php::var_dump(v1);
php::file_get_contents(file);
```

### 调用用户函数
用户函数必须通过`ZendVM`动态调用。
```cpp
php::call("my_user_func", {a, b, c});
```

## 命名空间
若函数使用了命名空间，则需要修改为 `php_{命名空间}__{函数名称}` ，命名空间为多层，则需要将斜杠`\\`替换为双下划线。例如：

```php
Foo\\Bar\\baz();
```
对应的`C++`函数为
```cpp
php_foo__bar__baz();
```

## 类和方法
除了函数之外，也可以实现`PHP`类，方法、静态方法、属性、常量、静态属性在`stub`文件中声明，在`C++`文件中只实现方法和静态方法。

* 命名空间和类名使用双下划线（`__`）分割作为前缀

### stub 文件

```php
class ClassFoo {
    protected string $prop;
    public function __construct(string $name);
    public function bar(int $a): int;
}
```

属性、静态属性、常量在`stub`文件中定义即可。`C++`代码仅需实现类的方法

```cpp
void php_classfoo____construct(php::Object &this_, php::String name) {}
php::Int php_classfoo__bar(php::Object &this_, php::String name) {}
```

* 类方法函数的第一个参数是`this_`，表示当前对象，若是静态方法，则第一个参数为`NULL`
* 可使用 `this_.attr()` 读写对象属性，使用 `this_.call()` 调用对象方法

## 默认参数
允许使用默认参数，例如在`stub`文件中函数声明为：

```php
function foo(string $a = "hello", int $b = 2026);
```

调用的代码为：
```php
foo();
```

C++ 代码：
```cpp
void php_foo(String a, Int b) {
    // a 的值是 "hello"
    // b 的值是 2026
}
```
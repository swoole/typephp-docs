# Python 互调用

TypePHP 可以在语言层面直接调用 Python 模块、函数和对象。两个运行时位于同一个进程中，参数和返回值通过 [phpy](https://github.com/swoole/phpy) 转换，不需要 JSON、RPC 或 Python 子进程。

Python 互调用是可选的扩展级特性。不使用 Python 语法的 TypePHP 程序不依赖 phpy；使用该特性时，需要在程序的 PHP 运行环境中加载 `phpy` 扩展。

## 环境准备

首先安装 CPython、开发头文件和 phpy。phpy 支持 Linux、macOS 和 Windows，要求 PHP 8.1 或更高版本。以源码构建为例：

```bash
git clone https://github.com/swoole/phpy.git
cd phpy
phpize
./configure --enable-phpy --with-python-config=/usr/bin/python3-config
make -j
sudo make install
```

在 TypePHP 程序使用的 `php.ini` 中加载扩展：

```ini
extension=phpy
```

验证扩展和 Python 运行时：

```bash
php --ri phpy
/usr/bin/python3 --version
```

第三方 Python 包必须安装到 phpy 所使用的同一个 Python 环境中。例如：

```bash
/usr/bin/python3 -m pip install numpy
```

phpy 与 TypePHP/PHP 的 Zend ABI 必须匹配。不要混用针对不同 PHP 版本、ZTS/NTS 模式或 Debug/Release ABI 构建的扩展，否则可能在程序启动或对象析构时崩溃。

TypePHP 不直接链接 `libphpy.so`。编译器只生成对 Zend class、method 和 object API 的动态调用，因此没有使用 Python 的代码不会增加运行时依赖。若程序实际执行了 Python 表达式但 phpy 未加载，会抛出普通 PHP `Error`；Python 模块不存在或 Python 调用失败时，则抛出 `PyError`。

## 第一个程序

```php
<?php

use Python\math;

function main(): void
{
    $result = math\sqrt(81);
    echo $result->toValue()->toFloat(), "\n";
}
```

编译并运行：

```bash
tpc hello.php
./hello
```

输出：

```text
9
```

`math\sqrt()` 的返回值默认仍是 `PyObject`。`toValue()` 明确地将它转换到 TypePHP/PHP 类型系统，再由 `toFloat()` 得到 `float`。

## 导入 Python 模块

使用 `use python\module` 导入模块：

```php
use python\sys;
use Python\numpy as np;
use python\numpy\linalg as linalg;
```

这分别对应 Python 的：

```python
import sys
import numpy as np
import numpy.linalg as linalg
```

`python` 根命名空间不区分大小写，因此 `python`、`Python` 和 `PYTHON` 都合法。模块路径、成员名、方法名和关键字参数仍严格区分大小写：

```php
python\len([1, 2, 3]); // 正确
Python\len([1, 2, 3]); // 正确
python\Len([1, 2, 3]); // 错误：Python 中不存在 Len
```

`use python\...` 只建立编译期模块别名，模块在第一次实际访问时才导入。仅声明但从未使用的模块不会调用 Python，也不会因为模块未安装而报错：

```php
use python\module_that_is_not_installed;

function main(): void
{
    echo "This program does not import the module.\n";
}
```

`python` 是 TypePHP 的特殊根命名空间，不能用来声明普通 PHP 命名空间。模块别名也不能与当前文件中的其他 `use` 符号冲突。

## 模块函数、类和变量

Python 模块中的 callable 使用 PHP 命名空间函数语法调用：

```php
use Python\numpy as np;

$array = np\array([1, 2, 3]);
$zeros = np\zeros([2, 3]);
```

`np\array()` 可能是函数、Python class 或实现了 `__call__` 的对象。TypePHP 不猜测成员种类，可调用性由 Python 在运行时判断。

模块变量使用 PHP 命名空间常量语法读取：

```php
use Python\math;
use Python\sys;

$pi = math\pi;
$path = sys\path;
```

不要写成 `math::pi` 或 `math::$pi`，它们属于 PHP class member 语法。Python 模块在 TypePHP 中映射为命名空间，成员读取仍由 Python 在运行时完成。

模块按完整名称缓存。同一个模块在多个文件中使用不同别名时，底层仍遵循 Python 的 `sys.modules` 导入语义。

## Python 内置函数和对象构造

通过 `python\name()` 调用 Python builtin：

```php
$length = python\len([1, 2, 3]);
$power = python\pow(2, 10);
python\print('Hello from Python');
```

常用 Python 代理对象可以直接构造：

```php
$list = python\list([1, 2, 3]);       // PyList
$dict = python\dict(['answer' => 42]); // PyDict
$tuple = python\tuple([1, 2]);         // PyTuple
$set = python\set([1, 2]);             // PySet
$str = python\str(123);                 // PyStr
$integer = python\int('42');            // PyObject
$bytes = python\bytes("binary");        // PyObject
```

其中容器构造是 phpy Facade 的语法糖，例如：

```php
$a = python\list([1, 2, 3]);
$b = new PyList([1, 2, 3]);
```

两者具有相同的运行时语义。特别是 `python\dict($phpArray)` 按 PHP 数组的 key/value 构造 `PyDict`，并不是直接执行 CPython 的 `dict(iterable)`。

Python builtin 的一级可调用语法目前不受支持。需要将 Python callable 保存或传递时，可以先从模块属性或对象属性取得对应的 `PyObject`。

## 操作 Python 对象

Python 返回值使用 phpy 已有的代理类表示。无法静态确定具体类型时统一为 `PyObject`，不会引入另一套 `python\Object` 类名。

### 属性和方法

```php
$name = $object->name;
$object->name = 'TypePHP';
$result = $object->greet('hello', suffix: '!');
unset($object->name);
```

这些操作分别调用 Python 的属性读取、`setattr`、call 和 `delattr` 协议。Python 方法支持命名参数，参数名称区分大小写。

### 下标和 `isset()`

```php
$last = $list[-1];
$list[-1] = 42;
unset($list[-2]);

if (isset($dict['name'])) {
    echo $dict['name'];
}
```

list 和 tuple 支持 Python 负索引。`isset()` 保持 PHP 语义：key/index 不存在，或者对应值为 Python `None` 时返回 `false`。除 `KeyError` 和 `IndexError` 之外的 Python 异常不会被吞掉。

### 迭代

```php
foreach ($pythonIterable as $index => $value) {
    echo $index, ': ', $value, "\n";
}
```

通用 Python iterator 的 key 是从 `0` 开始的迭代序号。`PyDict` 使用字典自身的 key/value。迭代期间发生的 Python 异常会作为 `PyError` 继续传播。

### 调用 callable 对象

```php
$result = $pythonCallable($left, right: 42);

$args = [1, 2, 3];
$result = $pythonCallable(...$args);
```

不可调用的 Python 对象会抛出包含 Python `TypeError` 的 `PyError`。

## 参数转换

进入 Python 函数、方法、构造器或运算符边界时，TypePHP 值自动转换为 Python 值：

| TypePHP 值 | Python 值 |
|---|---|
| `null` | `None` |
| `bool` | `bool` |
| `int` | `int` |
| `float` | `float` |
| `string` | `str`，字符串必须是合法 UTF-8 |
| list array | `list` |
| map array | `dict` |
| 空数组 | `list` |
| `PyObject` 及其子类 | 保留原 Python 对象，不复制内容 |
| TypePHP callable | 可由 Python 同步调用的 callable proxy |

数组会递归深拷贝。递归数组、循环容器、过深嵌套和无效 UTF-8 会抛出 `PyError`，不会无限递归或导致进程崩溃。

所有参数严格按照源码从左到右求值，并且只求值一次：

```php
$result = python\pow(mark(2), mark(3));
```

如果同一个数组需要在循环中频繁传给 Python，建议提前转换并复用代理对象：

```php
$pyItems = python\list($items);

for ($i = 0; $i < 1000; $i++) {
    processor::consume($pyItems); // 只传递 Python 对象引用
}
```

避免每次调用都直接传 `$items`，否则每次跨越边界都会重新深拷贝。二进制数据应使用 `python\bytes()`；普通 TypePHP `string` 默认转换为 Python `str`。

## 返回值与显式转换

Python 函数、方法、构造调用和运算结果默认保持为 `PyObject` 或其具体代理子类。即使 Python 返回 `int`、`float`、`bool` 或 `str`，TypePHP 也不会根据运行时类型自动改变变量类型。

使用 `PyObject::toValue()` 显式进入 TypePHP/PHP 类型系统：

```php
$pyValue = python\int(42);

$plain = $pyValue->toValue();
$integer = $pyValue->toValue()->toInt();
$float = $pyValue->toValue()->toFloat();
$boolean = $pyValue->toValue()->toBool();
$string = $pyValue->toValue()->toString();
$array = python\list([1, 2, 3])->toArray();
```

函数式写法 `python\scalar($value)` 与 phpy 的 `PyCore::scalar($value)` 等价，可用于兼容已有代码：

```php
$integer = python\scalar($pyValue)->toInt();
```

`toValue()` 与 `python\scalar()`、`PyCore::scalar()` 使用同一套转换规则。转换后的结果是普通 TypePHP 值，后续运算不再使用 Python protocol。

`toArray()` 专门转换 Python `list`、`set`、`tuple`、`dict` 和迭代器对象，并递归深拷贝其中的元素；不支持的 Python 类型返回空数组。迭代器会被消费，重复调用可能得到空数组。phpy 在 `PyObject` 上提供具体的 `toArray()` 方法；它同时也是 TypePHP 关键词方法。receiver 是静态可知的 `PyObject` 子类时，编译器会直接调用 phpy 方法，否则回退到通用转换路径。`toValue()` 只是 `PyObject` 的普通方法，不是 TypePHP 关键词方法。

`toString()` 仍是 TypePHP 关键词方法，会使用 `PyObject::__toString()`；无需在 phpy 中额外提供同名转换方法。若要转换为其他 PHP 标量，应先调用 `toValue()`，例如 `$pyObject->toValue()->toInt()`。

## 运算符

只要表达式的一侧静态类型是 `PyObject` 或其子类，TypePHP 就使用 Python 的完整运算符协议。另一侧的普通 TypePHP 值会先转换为 Python 对象：

```php
$seven = python\int(7);
$three = python\int(3);

$sum = $seven + $three;
$product = $seven * 10;
$reflected = 10 + $seven;
$quotient = $seven / 2; // Python true division
```

支持的协议包括：

- 算术、幂和位运算：`+ - * / % ** << >> & | ^`。
- 一元运算：`+ - ~`。
- 比较：`== != < <= > >=`。
- identity：`===` 和 `!==`。
- 条件真假值、`!`、`&&`、`||` 和 `xor`。
- 对变量、对象属性和下标执行复合赋值，例如 `+=`、`*=`、`<<=`。

`/` 使用 Python true division，不是 floor division。TypePHP 没有 Python 的 `//` 运算符；需要 floor division 时显式调用相应 Python 函数。

`==` 使用 Python 值相等协议；`===` 使用 Python object identity：

```php
$list = python\list([1]);
$alias = $list;

var_dump($list === $alias);          // true
var_dump($list === python\list([1])); // false
```

复合赋值使用 Python in-place protocol，并用协议返回的对象更新左值，因此对可变和不可变 Python 类型都能得到正确结果。

### 动态 PHP 代码的一元正负号限制

TypePHP AOT 保留源码 AST，因此：

```php
-$value; // Python operator.neg(value)
$value * -1; // Python operator.mul(value, -1)
```

两者会调用不同的 Python protocol，行为可以不同。

通过 ZendVM 动态执行的普通 PHP 代码是一个例外。Zend 会把 `-$value` 和 `+$value` 编译成乘以 `-1` 和 `1`，phpy 的 opcode handler 无法恢复原始源码意图。因此动态代码中的 `-$value` 保持 `$value * -1` 的行为，`+$value` 保持 `$value * 1` 的行为。

对于普通 Python `int` 和常规 `float`，结果通常相同；自定义 Python 类型可以分别实现 `__neg__()`、`__pos__()` 和 `__mul__()`，此时 AOT 与动态代码可能不同。TypePHP 不通过全局 AST hook 改写普通 PHP 代码。

## TypePHP callable 传给 Python

TypePHP 函数、闭包和可调用对象可以作为 Python 参数，并由 Python 在当前调用链中同步回调：

```php
$values = python\list([1, 2, 3]);
$mapped = python\map(
    fn (int $value): int => $value * 2,
    $values,
);

$sum = python\sum($mapped)->toValue()->toInt();
echo $sum, "\n"; // 12
```

Python 传给回调的关键字参数会按名称绑定到 TypePHP callable 的参数。回调是同步的，只能在 TypePHP 主动发起的 Python 调用关系中使用；TypePHP 不会向 Python 注册可独立 import 的函数、类或 module。

## 异常处理

Python module 不存在、成员不存在、参数错误、类型错误和用户代码异常统一映射为 `PyError`：

```php
try {
    python\len();
} catch (PyError $error) {
    echo $error->getMessage(), "\n";

    // 原始 Python 异常对象，均为可选 PyObject 属性。
    $type = $error->type;
    $value = $error->value;
    $traceback = $error->traceback;
}
```

`PyError` 继承自 PHP `Exception`，并保留 `type`、`value`、`error` 和 `traceback` 等 Python 对象。普通 PHP 错误仍使用 PHP 异常体系，例如未加载 phpy 时解析不到 `PyCore` 会抛出 `Error`。

跨 VM 异常不会被静默转换为 `null`。处理异常后可以继续进行 Python 调用，phpy 会清理 CPython 的 pending error state。

## 与直接使用 phpy 的关系

TypePHP 没有重新实现 Python VM，也没有创建第二套 Python 对象类。下列写法可以混用：

```php
use Python\os;

$module1 = PyCore::import('os');
$name1 = $module1->name;

$name2 = os::$name;
$list1 = new PyList([1, 2, 3]);
$list2 = python\list([1, 2, 3]);
```

`use python\module`、`python\name()` 和 Python 运算符是 TypePHP 的编译期语法糖；CPython 初始化、GIL、引用计数、对象代理、转换和异常全部由 phpy 负责。TypePHP 不 include phpy 头文件，也不生成对 phpy C++ 符号的直接调用。

## 限制

- 不支持 Python `threading`。
- 不支持 `asyncio`。
- 不支持 CPython subinterpreter。
- 不把 TypePHP 编译成 Python extension，也不支持从 Python 主动 import TypePHP 程序。
- 不提供 TypePHP 函数或类的 Python 导出注解。
- 不编译 Python 源码；Python 模块仍由 CPython 在运行时加载。
- 不支持 `from package import *` 语法。
- 不支持 Python builtin 的一级可调用语法。
- Python 符号是否存在通常只能在运行时确定。
- Python 互调用目前不适用于 TypePHP WASM 目标。
- 动态 PHP 代码的一元正负号存在前述 protocol 差异。

TypePHP 的目标是让应用高效、可靠地调用 Python 包，而不是在 PHP 中完整实现 Python 语法或异步运行时。

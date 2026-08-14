# Python 代码转 TypePHP

`--convert-python-to-php` 是 `tpc` 命令入口中内置的 Python 源码转换工具，用于开发阶段把一个 Python 脚本转换成便于继续迁移的 TypePHP 源码。它是机械转换工具，不承诺任意 Python 程序与输出代码完全等价，也不参与最终程序的编译。

## 它能帮你做什么

把一个已有的 Python 文件转换为使用 TypePHP Python 命名空间语法的 PHP 源码。例如把一段数据处理脚本转成 TypePHP 源文件后，就可以用 TypePHP 编译器把它编译成原生程序，或在编辑器里继续以 PHP 语法维护。

## 前置条件

转换过程调用 `PATH` 中的 `python3` 解析 Python 源码的语法树，因此只需要主机上有可用的 `python3`，**不依赖 phpy 扩展，也不需要目标 Python 模块已安装**。

## 用法

```bash
./tpc --convert-python-to-php <file.py> > <file.php>
```

转换结果输出到标准输出，错误信息输出到标准错误，退出码 `0` 表示成功、`1` 表示失败。通常用重定向把结果保存到文件：

```bash
./tpc --convert-python-to-php script.py > script.php
```

## 示例

```python
import math
print(math.sqrt(16))
```

```php
use python\math;

function main(): void
{
    python\print(math\sqrt(16));
}
```

普通 `import` 会转换为命名空间 `use`，模块函数以 `python\模块\函数()` 的形式调用。

生成结果应经过代码审查和 TypePHP 测试后再投入生产。特别是解构数量、仅关键字参数、装饰器和 Python/PHP 标量语义存在下文明确列出的降级规则；转换器不会替代业务级兼容性验证。

## 语法支持对照

下表列出转换器当前支持的 Python 语法（✅）与暂不支持、遇到会报错的语法（❌）。不支持的语法不会生成看似可用但语义不符的 PHP 代码，而是直接抛出带源文件与行号的错误。

### 语句

| Python 语法 | 状态 | 转换规则 / 报错 |
|---|---|---|
| `x = expr` | ✅ | `$x = expr;`，模块级变量自动注入 `global` |
| `x = y = 1`（链式赋值） | ✅ | `$x = $y = 1;`（仅限名称目标；含属性/下标目标时报错） |
| `x += expr` 等增强赋值 | ✅ | 支持 `+ - * / % ** << >> \| ^ &` 系列；`//=` `@=` 展开为 `python\operator\floordiv/matmul($x, ...)` 调用 |
| `x: int = expr` | ✅ | 忽略注解，转换为普通赋值 |
| `x: int`（纯注解） | ✅ | 转为注释 `// annotation-only declaration: x`，不登记为模块全局 |
| `a, b = x`（解构） | ✅ | `[$a, $b] = $x->toArray();`（PyObject 转 PHP 数组后解构；元素允许名称/属性/下标。嵌套解构、星号解构 `a, *b = x`、链式解构不支持。元素个数不匹配时按 PHP 语义补 null，不报 Python 的 ValueError） |
| `def f(...)` | ✅ | 名为 `main` 的函数重命名为 `main_`（避免与 TypePHP 入口冲突），调用点同步改写 |
| 嵌套 `def` | ❌ | `FunctionDef: nested functions require Python closure scope analysis` |
| `@decorator` | ✅ | 见下方「函数装饰器」 |
| `return [expr]` | ✅ | `return [expr];` |
| `if / elif / else` | ✅ | 同构转换 |
| `while` | ✅ | 同构转换；`while/else` 不支持 |
| `for i in iter` | ✅ | `foreach (iter as $i)`；`for/else`、元组目标不支持 |
| `break` / `continue` / `pass` | ✅ | `pass` → `// pass` 注释 |
| `global x` | ✅ | `global $x;`（与自动注入的 global 并存时会重复出现，冗余但合法，属已知行为） |
| `del x` / `del o.a` / `del d[k]` | ✅ | `unset(...)`；`del (a, b)` 元组/列表目标逐项展开；非法 del 目标（如 `del f()`）由 Python 解析器先行拒绝 |
| 模块级字符串字面量（docstring） | ✅ | 转为 `/** ... */` 注释（`*/` 转义为 `* /`） |
| `import a.b` | ✅ | `use python\a;`（仅首段作为别名） |
| `import a.b as x` | ✅ | `use python\a\b as x;`（别名等于末段时省略 `as`） |
| `from m import f [as g]` | ✅ | 调用点映射为 `python\m\f(...)` |
| `from . import m` | ❌ | `ImportFrom: relative imports are not supported yet` |
| `from m import *` | ❌ | `ImportFrom: star imports are not supported` |
| `class` | ❌ | `ClassDef` |
| `with` | ❌ | `With` |
| `raise` / `try` / `assert` | ❌ | `Raise` / `Try` / `Assert` |
| `async def` / `await` | ❌ | `AsyncFunctionDef`（`await` 不可达，外层先报错） |
| `match` | ❌ | `Match` |
| `nonlocal` | ❌ | `Nonlocal` |

### 函数签名

| Python 形态 | 状态 | TypePHP 输出 |
|---|---|---|
| `def f(x, y=4)` | ✅ | `function f($x, $y = 4)` |
| `def f(a, *, b)` | ✅ | `function f($a, $b = null)`（无默认值的仅关键字参数补 `null`） |
| `def f(*args)` / `def f(**kw)` | ✅ | `function f(...$args)` |
| `def f(*a, **kw)` | ❌ | `FunctionDef: simultaneous *args and **kwargs cannot be represented by one PHP signature` |
| `lambda a, b=2: a + b` | ✅ | `fn ($a, $b = 2) => $a + $b` |

### 表达式

| Python 语法 | 状态 | 转换规则 / 报错 |
|---|---|---|
| 字面量 `int / float / str / True / False / None` | ✅ | `var_export`；`None` → `null` |
| `b'...'` bytes | ❌ | `{file}: Python bytes literals are not supported yet`（无行号） |
| `1j` complex | ❌ | `{file}: Python complex literals are not supported yet`（无行号） |
| 变量名 | ✅ | `$name`；`this` 转义为 `$this_` |
| 模块别名作为值 | ❌ | `a Python module cannot be used as a first-class value in TypePHP namespace syntax` |
| 属性链 `o.a.b` | ✅ | `$o->a->b`；模块别名链仅首段为模块成员：`sys.version_info.major` → `sys\version_info->major` |
| 模块属性赋值/删除 | ❌ | `Attribute: Python module attributes cannot be assigned or deleted` |
| 函数调用 | ✅ | 已定义函数直连 `f(...)`；内置函数映射 `python\len(...)`；`from m import f` 映射 `python\m\f(...)`；其他名字按变量可调用 `$f(...)` |
| 关键字参数 / `*args` / `**kwargs` 调用 | ✅ | `f(x: 1, ...$args)` |
| 容器字面量 `[] () {} {:}` | ✅ | `python\list/tuple/set/dict([...])`，支持 `...` 解包 |
| 二元运算 `+ - * / % ** << >> \| ^ &` | ✅ | 同构转换 |
| `//` 整除 / `@` 矩阵乘 | ✅ | `python\operator\floordiv(a, b)` / `python\operator\matmul(a, b)` |
| 一元运算 `- + not ~` | ✅ | `- + ! ~` |
| 比较 `== != < <= > >=` | ✅ | 同构转换 |
| `is` / `is not` | ✅ | `===` / `!==` |
| `in` / `not in` | ✅ | `python\operator\contains(b, a)`（参数交换）/ 取反 |
| 链式比较 `a < b < c` | ❌ | `Compare: chained comparisons require explicit temporary variables` |
| `a and b` / `a or b` | ❌ | `BoolOp` |
| `x if c else y` | ✅ | `(c ? x : y)` |
| 下标 `a[i]` / 切片 `a[l:u:s]` | ✅ | `$a[$i]` / `$a[python\slice(l, u, s)]`（缺省为 `null`） |
| f-string | ✅ | 拼接 + `->toString()`；运算符等优先级敏感表达式整体加括号 |
| f-string 的 `!r` 转换 / `:03d` 格式说明 | ❌ | `FormattedValue: formatted f-string conversions are not supported yet` |
| 海象 `:=` | ✅ | 表达式内赋值 `($n = 10)` |
| 推导式 / 生成器表达式 | ❌ | `ListComp` / `SetComp` / `DictComp` / `GeneratorExp` |
| `yield` / `yield from` | ❌ | `Yield` / `YieldFrom` |

### 函数装饰器

装饰器在 `main()` 起始处（其他顶层语句之前）按 Python 语义**自底向上**重绑定到同名模块变量：

```python
@a
@b
def greet(): ...
```

```php
function greet() { ... }

function main(): void
{
    global $greet;
    $greet = b('greet');
    $greet = a('greet');
    ...
}
```

- 装饰器可以是已定义函数、`from m import f` 导入符号、模块属性或装饰器工厂（`@dec('x')` → `$greet = dec('x')('greet');`）；
- 被装饰函数名登记为模块全局，所有调用点（包括其他函数体内）经 `global` + 变量间接调用装饰结果：`$greet()`；
- 被装饰函数体内的递归调用同样解析到装饰后的变量，与 Python 语义一致。

### 报错与降级原则

当语义可以严格保持时，转换器会直接使用 PHP 原生写法：无参数或可安全转换的 `print()` 转换为带换行的 `echo`，`sys.exit()` 与整数字面量退出码转换为 `exit`。带有 `sep`、`end`、`file`、`flush` 参数的 `print()`，以及字符串或对象形式的 `sys.exit()` 与 PHP 行为不完全一致，仍保留为 Python 调用。

对于尚未支持的语法（如 `class`、`async`、`with`、`try`、推导式、生成器、嵌套函数、链式比较等），转换器会**直接报错并给出源文件与行号**，不会生成看似可用但语义不符的 PHP 代码。建议先转换核心逻辑，再手动补齐少量暂不支持的片段。

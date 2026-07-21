# 使用 TypePHP 动态库

TypePHP 可以使用 `-m lib` 将 PHP 函数、类和 C++ 实现的 `php_*` 函数编译为动态库，再由其他 TypePHP 项目直接调用。本页通过一个完整示例介绍如何创建、发布和使用 TypePHP 库。

## 你需要发布什么

一个供其他 TypePHP 项目使用的库通常包含：

| 平台 | 必需文件 |
|------|----------|
| Linux | `.stub.php`、`lib<库名>.so` |
| Windows | `.stub.php`、`<库名>.dll`、`<库名>.lib` |

`-m lib` 会自动生成 `<库名>.stub.php`，描述库提供的函数和类。其他项目编译时读取它来检查接口，并找到需要链接的库。

TypePHP 在构建目录中生成的 `php_<target>_func_decl.h` 和 `php_<target>_data_decl.h` 是内部文件，不需要随库发布。

如果库还提供了额外的 C ABI 或 C++ ABI 接口，应由库作者自行编写并发布对应的 `.h` 文件。`.stub.php` 只描述供 TypePHP PHP 代码调用的函数。

## 第一步：创建库项目

下面创建一个名为 `prime2` 的简单向量库：

```text
prime2/
├── project.yml
└── src/
    ├── math.php
    ├── vector.stub.php
    └── vector.cc
```

### 编写 PHP 函数和类

创建 `src/math.php`：

```php
<?php

class Counter
{
    public const int STEP = 2;
    public int $value = 1;

    public function add(int $amount = self::STEP): int
    {
        $this->value += $amount;
        return $this->value;
    }
}

function twice(int $value): int
{
    return $value * 2;
}
```

PHP 实现不需要再手写一份 stub。编译器会自动提取函数、类、属性、常量和方法声明。

### 声明库函数

创建 `src/vector.stub.php`：

```php
<?php
function vector_new(int $size, bool $initialValue = false): mixed {}
function vector_get(mixed $vector, int $offset): bool {}
function vector_set(mixed $vector, int $offset, bool $value): void {}
```

注意：

- `.stub.php` 中的函数体必须为空，具体实现放在 C++ 文件中。
- 参数类型、返回类型和默认值属于库接口，修改后需要重新发布 stub 和动态库。
- 库项目内的本地 stub 不要添加 `@import-library`；它的函数由当前项目的 C++ 代码实现。

### 编写 C++ 实现

创建 `src/vector.cc`：

```cpp
#include <phpx.h>

using namespace php;

class VectorBox : public Box {
  public:
    std::vector<bool> values;

    VectorBox(size_t size, bool initialValue) : values(size, initialValue) {}
};

Var php_vector_new(Int size, Bool initialValue) {
    return {new VectorBox(size, initialValue)};
}

Bool php_vector_get(Var vector, Int offset) {
    auto box = vector.toBox<VectorBox>();
    return box->values.at(offset);
}

void php_vector_set(Var vector, Int offset, Bool value) {
    auto box = vector.toBox<VectorBox>();
    box->values.at(offset) = value;
}
```

C++ 函数名需要在 PHP 函数名前添加 `php_`。参数和返回值的常用对应关系如下：

| PHP | C++ |
|-----|-----|
| `int` | `php::Int` |
| `bool` | `php::Bool` |
| `float` | `php::Float` |
| `string` | `php::Str` |
| `array` | `php::Array` |
| `mixed` | `php::Var` |
| `object` | `php::Object` |
| `void` | `void` |

### 配置库项目

创建 `project.yml`：

```yaml
name: prime2
mode: lib
build-dir: build

sources:
  - src
```

`name: prime2` 决定自动生成的 `prime2.stub.php` 文件名和最终链接时使用的库名。

## 第二步：编译动态库

在库项目目录执行：

```bash
tpc project.yml -O2
```

也可以在命令行指定构建模式：

```bash
tpc project.yml -m lib -O2
```

Linux 默认生成：

```text
libprime2.so
prime2.stub.php
```

Windows 默认生成：

```text
prime2.dll
prime2.lib
prime2.stub.php
```

其中 `.dll` 是运行时动态库，`.lib` 是其他 Windows 项目链接时使用的导入库。

## 第三步：整理发布文件

Linux 可以整理为：

```text
prime2-package/
├── stubs/
│   └── prime2.stub.php
└── lib/
    └── libprime2.so
```

Windows 可以整理为：

```text
prime2-package/
├── stubs/
│   └── prime2.stub.php
└── lib/
    ├── prime2.dll
    └── prime2.lib
```

自动生成的 `prime2.stub.php` 顶部带有 `@import-library`。它表示文件中所有函数和类方法均由外部 TypePHP library 提供。库名由 stub 文件名推导，因此不要重命名该文件。

消费项目编译该 stub 时，会在当前项目中生成类注册、属性和常量实体，但不生成方法的 `php_*` 实现；方法实现始终从 `prime2` 动态库导入。

Property hook 也可以随类接口发布。生成的 stub 会保留 `get`/`set` 的声明并移除具体代码；消费项目创建属性实体，getter/setter 的实现由动态库提供。

### 隐藏库的内部接口

只供库内部使用的函数、类或方法可以添加 `#[NoExport]`：

```php
#[\NoExport]
function normalizeInternalData(array $data): array
{
    // 仅供当前库内部使用
}

#[\NoExport]
class InternalCache {}
```

声明仍参与当前库编译并可供库内代码调用，但不会进入 `<target>.stub.php`，对应 `php_*` 符号也不会导出。完整的函数、类、方法和命名空间用法参见 [`NoExport` 编译期 Attribute](no-export.md)。

## 第四步：在其他 TypePHP 项目中使用

假设应用项目结构如下：

```text
my-app/
├── project.yml
├── src/
│   └── main.php
└── deps/
    └── prime2/
        ├── stubs/
        │   └── prime2.stub.php
        └── lib/
            ├── libprime2.so      # Linux
            ├── prime2.dll        # Windows
            └── prime2.lib        # Windows
```

应用只需把 `.stub.php` 加入 `sources`，并通过 `link-paths` 指定库目录：

```yaml
name: my-app
mode: bin
build-dir: build

sources:
  - src
  - deps/prime2/stubs/prime2.stub.php

link-paths:
  - deps/prime2/lib
```

不需要再配置 `link-libs: [prime2]`。编译器读取 `prime2.stub.php` 中的 `@import-library` 后，会从文件名得到 `prime2` 并自动链接该库。

在 `src/main.php` 中可以像调用普通 PHP 函数一样使用它：

```php
<?php

function main(): void
{
    $vector = vector_new(4);
    vector_set($vector, 2, true);

    var_dump(vector_get($vector, 2));

    $counter = new Counter();
    var_dump($counter->add());
    var_dump(twice(6));
}
```

编译应用：

```bash
tpc project.yml -O2
```

## 第五步：让程序在运行时找到动态库

链接成功只表示编译器找到了库；运行应用时，操作系统也必须能够找到动态库。

### Linux

开发阶段可以临时设置：

```bash
LD_LIBRARY_PATH="$PWD/deps/prime2/lib" ./my_app
```

发布应用时，也可以把 `libprime2.so` 安装到系统动态库目录，或者为应用配置合适的 rpath。不要只复制可执行文件而遗漏 `.so`。

### Windows

最简单的方式是把 `prime2.dll` 复制到应用 `.exe` 所在目录：

```text
release/
├── my_app.exe
└── prime2.dll
```

`prime2.lib` 只在编译和链接应用时需要，应用运行时需要的是 `prime2.dll`。

## 更新库接口

修改函数参数、返回类型或默认值时：

1. PHP 实现直接修改源码；C++ 实现同时更新本地 `.stub.php` 和 C++ 代码。
2. 重新编译动态库。
3. 将新的 stub 和动态库一起发布。
4. 重新编译所有使用该库的 TypePHP 项目。

不要把新版 `.stub.php` 与旧版 `.so`、`.dll` 或 `.lib` 混用。

## 兼容性注意事项

库和使用它的项目应保持以下环境兼容：

- 操作系统和 CPU 架构一致，例如均为 Linux x86-64。
- PHP 主版本、线程安全模式（ZTS/NTS）和 PHPX 环境兼容。
- TypePHP 版本兼容。
- 使用 C++ ABI 时，编译器及其运行库 ABI 兼容。

不同操作系统和 CPU 架构需要分别编译和发布动态库。

## 常见问题

### 链接时提示找不到 `prime2`

检查：

- `link-paths` 是否指向实际的库目录。
- Linux 文件名是否为 `libprime2.so`。
- Windows 库目录中是否存在 `prime2.lib`。
- 是否使用编译器生成的 `prime2.stub.php`，且文件顶部保留 `@import-library`。

### 链接时提示找不到 `php_vector_new` 等符号

检查 C++ 实现：

- 函数名是否带有 `php_` 前缀。
- 参数和返回类型是否与 stub 一致。
- 发布 stub 文件名是否与库 target 名一致。
- 修改 stub 后是否重新编译了动态库。

### 程序编译成功，但启动时找不到动态库

这是运行时库搜索路径问题：

- Linux：配置 `LD_LIBRARY_PATH`、rpath 或系统动态库目录。
- Windows：把 `.dll` 放到 `.exe` 同目录，或者加入 `PATH`。

### 是否需要发布生成的 `func_decl.h` 和 `data_decl.h`

不需要。它们是 TypePHP 构建过程的内部文件，不是库的公共接口。TypePHP 函数接口通过 `.stub.php` 发布；额外的 C/C++ 接口才需要由库作者提供手写 `.h` 文件。

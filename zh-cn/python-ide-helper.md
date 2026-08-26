# 生成 IDE 自动提示

`--gen-python-helper` 是 `tpc` 命令入口中内置的 Python IDE 辅助文件生成工具，用于开发阶段为某个 Python 模块生成编辑器辅助文件，让 `python\math\sqrt()` 这类调用获得自动补全。它本身不参与最终程序的编译。

## 它能帮你做什么

当你用 `python\math\sqrt()`、`numpy\zeros()` 这样的语法调用 Python 模块时，编辑器默认并不知道这些符号的位置、参数和返回类型。`--gen-python-helper` 会读取目标 Python 模块的运行时信息，生成一份**仅供编辑器索引**的 PHP 辅助文件，从而让自动补全、参数提示和跳转定义正常工作。

## 前置条件

生成过程会真正导入指定的 Python 模块来采集符号，因此你的开发机上需要满足：

- 运行 `tpc` 的 PHP 已安装并启用 **phpy 扩展**；生成器通过 phpy 在当前进程中反射 Python 模块；
- 目标 **Python 模块已经安装**到运行 `tpc` 的同一个 Python 环境中（例如要生成 `numpy` 的辅助文件，先执行 `python3 -m pip install numpy`）；
- 存在可用的 `python3`。

## 用法

```bash
./tpc --gen-python-helper <Python 模块名> [--output-dir <目录>]
```

- `<Python 模块名>`：点号分隔的模块路径，例如 `math`、`numpy.linalg`；
- `--output-dir <目录>`（可选）：自定义输出根目录，支持相对路径或绝对路径，默认输出到当前目录下的 `ide-helper`。

示例：

```bash
./tpc --gen-python-helper math
./tpc --gen-python-helper numpy.linalg
./tpc --gen-python-helper numpy --output-dir .ide-helper
```

## 生成了哪些文件

默认在 `ide-helper/` 下生成：

```text
ide-helper/python/math.php
ide-helper/python/numpy/linalg.php
ide-helper/python.php
ide-helper/PyObject.php
```

- `python.php`：Python 内置符号（如 `python\len()`、`python\tuple()`）的补全，随当前 Python 环境生成；
- `PyObject.php`：所有 Python 模块辅助文件共享的公共类型提示，首次生成时写入，之后不会覆盖。

## 注意事项

- 辅助文件**只交给编辑器索引**，不要 `include` 它们，也不要把它们加入 TypePHP 项目的 `sources` 或编译输入；
- 生成器会导入目标模块，模块的顶层初始化代码和导入副作用会真实发生；不要对不可信模块运行此命令；
- 每次生成都会更新 `python.php` 和目标模块文件；已有的 `PyObject.php` 会保留，方便项目维护自定义 IDE 声明；
- 辅助文件名、函数名要符合 PHP 命名规则，因此个别 Python 保留字（如 `print`、`list`、`int`、`float`）无法生成无语法错误的符号声明，生成器会以注释标注，不会擅自改名。

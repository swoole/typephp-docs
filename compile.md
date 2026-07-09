编译器支持三种使用方式

## 单文件

```bash
./swoole_compiler your_path/file.php
```
## 目录

```bash
./swoole_compiler my_project/
```

编译器将递归遍历整个目录，查找`.php`文件，并逐个编译为`.o`目标文件，最终连接在一起生成可执行文件。

## YAML 配置

配置文件名称不固定，由命令行参数传入：

```bash
./bin/compiler.php myproject.yml
./swoole_compiler myproject.yml
```

```yaml
# 只设置产物名称，不改变输出目录
name: prime
version: 0.0.1
cxxflags: |
  -std=c++17
  -I/usr/include/python3.10
  -Wall
ldflags: |
  -L/usr/lib/x86_64-linux-gnu/
  -lssl
sources:
  - php-src
  - ./cpp-src
  - main.php
ignore:
  - vendor/autoload.php
```

使用 `YAML` 配置文件。可以配置多个源码目录，文件忽略以及编译参数。复杂的工程建议使用`yaml`配置方式。

- `sources`和`ignore`可配置多个路径，支持文件或者目录的方式，若为目录则会遍历获取所有`.php`文件
- 路径支持绝对路径和相对路径，相对路径计算时`cwd`当前目录为`yaml`文件所在的目录
- `build-dir` 同样支持绝对路径和相对路径，相对路径按 `yaml` 文件所在目录解析
- `name` 只设置产物文件名，输出仍在执行编译命令的当前目录
- `output` 可设置完整输出路径，等价于 `-o` / `--output`
- 当`yaml`配置项与执行编译命令的参数冲突时，优先使用命令行参数
- 条件加载源码、链接参数等完整配置说明见 [project.yml 配置](project-yml.md)

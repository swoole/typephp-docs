AOT 编译器仅支持`gdb`调试，不支持`Xdebug`等工具。若要使用调试功能，请确保：

- 优化等级设置为`-O0`，关闭优化，否则函数调用可能会被内联
- 开启调试符号，编译参数添加`--debug-info`
- `PHPX`和`PHP`建议编译为`debug`版本


## PHPX
```bash
cmake . -D CMAKE_BUILD_TYPE=Debug
```

## PHP
```bash
./configure --enable-debug
```

# GDB 调试


```bash
gdb ./hello
```

## 断点设置
所有函数均以`php_`为前缀，例如下面的代码

```php
function my_add(int $a, int $b): int
{
    $env = $_ENV;
    var_dump(count($env));
    return $a + $b;
}

function main()
{
    var_dump(my_add(1, 2));
}
```

`b php_my_add` 表示为 `my_add` 添加断点

```bash
b php_my_add
r
Breakpoint 1, php_my_add (a=1, b=2) at /home/swoole/workspace/aot/build/examples/myext/test.cc:9
9	    php::Int tmp_var_0 = 0;
```

如果是原生类型，则可以直接使用 `gdb` 的 `print` 指令输出其值。若为`PHP`类型，则可以使用`call var.print()`输出。

```bash
(gdb) call env.print()
         array(61) {
           ["SHELL"]=>
           string(9) "/bin/bash"
           ["SESSION_MANAGER"]=>
           string(71) "local/swoole-26:@/tmp/.ICE-unix/4392,unix/swoole-26:/tmp/.ICE-unix/4392"
           ["QT_ACCESSIBILITY"]=>
           string(1) "1"
           ["COLORTERM"]=>
           string(9) "truecolor"
           ["XDG_CONFIG_DIRS"]=>
           string(28) "/etc/xdg/xdg-ubuntu:/etc/xdg"
           ["SSH_AGENT_LAUNCHER"]=>
           string(13) "gnome-keyring"
           ["XDG_MENU_PREFIX"]=>
           string(6) "gnome-"
           ["GNOME_DESKTOP_SESSION_ID"]=>
           string(18) "this-is-deprecated"
...
```

## 函数符号

- 函数：`php_{命名空间}__{函数名}`，若没有命名空间则为`php_{函数名}`，名称必须全部转为小写
- 类方法：`php_{命名空间}__{类名}__{方法名}`，若命名空间为多层，则需要使用双下划线分割，名称必须全部转为小写
- 内置函数：`zif_{函数名}`
- 内置类方法：`zim_{类名}_{方法名}`，需要查看对应扩展的实现方式

在符号上添加断点，在函数调用发生时，对其进行调试。
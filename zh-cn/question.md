## TypePHP 编译器与 3.2 加密器是否兼容

兼容。可同时使用 TypePHP 编译器和`3.2`加密器。支持静态编译的文件生成二进制原生指令，存在不支持语法的文件依然可使用`ZendVM`动态加载，使用`3.2`加密器对`PHP`代码加密。

## TypePHP 编译器是开源免费的吗

是的。TypePHP 编译器是开源软件，遵循 **GPL** 开源协议发布，完全免费，可自由使用、修改和再分发，包括商业用途。无论是个人开发者，还是企业、政府、事业单位、学校、公益组织，均可免费使用 TypePHP 编译器，无需购买任何授权。

## `Swoole Compiler` 与 `TypePHP` 的差异

Swoole Compiler 是商业软件，用户需购买后使用。`Swoole Compiler` 专注于保护 `PHP` 项目源代码。更适合传统的`PHP-FPM`或
`Swoole`/`Swow`/`Workerman`等`PHP-CLI`模式的`PHP`程序，用户无需修改已有`PHP`代码。

TypePHP 是开源项目。所有用户均可免费使用，是全新的`PHP`编译技术，与传统`PHP`的解释执行不同， TypePHP 是将`PHP`源代码直接编译为二进制可执行程序。
已有`PHP`代码无法直接运行于 TypePHP 之下，需要根据编译器的信息修改源代码后使用。
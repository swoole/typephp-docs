## 1. 框架和类库是否编译

`vendor` 目录中的框架和类库，建议仍然使用 `Composer Autoload` 加载不编译。或者使用白名单配置的方式，单独对部分`vendor`子目录进行编译。

若`vendor`子目录中的个别`PHP`文件不支持静态编译，还可以配置`ignore`忽略部分文件。

```yaml
name: thinkphp
type: ext
version: 0.0.1
cxxflags: |
  -std=c++14
  -Wall
sources:
  - ./src/think
  - ./vendor/topthink/think-helper/src
  - ./vendor/topthink/think-orm/src
  - ./vendor/topthink/think-container/src
ignore:
  - ./vendor/topthink/think-helper/src/functions.php
```

## 2. 如何打包分发
`AOT`编译器与`Golang`这样的静态编译方式不同，为了减少文件尺寸，采用了动态链接的方式，因此需要依赖操作系统的`.so`库。因此编译生成的二进制文件部署分发需要遵循下列规则：

1. 操作系统必须一致，安装的基础库列表需要保持一致，可使用`apt/yum/dnf`等操作系统包管理工具自动完成
2. 在编译机构建`PHP`和`PHPX`，产生`libphp.so`和`libphpx.so`，保存至包中
3. 所有`PHP`扩展均使用静态编译的方式，直接编译到`PHP`中，而不是动态加载

最终分发的软件包中，应包含下列内容：

- 编译生成的二进制可执行文件
- `libphp.so`和`libphpx.so`
- `vendor/` 目录中的开源框架和类库文件

> 若动态加载了`PHP`扩展，需要将其`.so`复制到软件包中

## 3. 是否可使用 `__FILE__` 和 `__DIR__`

由于`AOT`编译器是在编译阶段确定 `__FILE__` 和 `__DIR__` 魔术常量的值，因此最终的目录是编译时`.php`文件的路径，而不是运行时。因此虽然可以使用魔术常量，但它的结果与预期可能不一致。建议使用 `getcwd()` 或者其他配置文件方式确定最终运行时的目录。
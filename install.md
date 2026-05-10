# 安装软件
下载软件包后，解压至当前目录后执行下列步骤完成安装。

`AOT`编译器支持`Windows`、`macOS`、`Linux`三种操作系统，支持`x86-64`或`ARM64`架构。编译生成的文件是对应操作系统下的原生可执行文件，而不是字节码或者其他中间指令码。

> 注意目前预览版仅支持 `PHP 8.4 +zts +embed`，请确保已安装

```shell
php-config
```
检查是否为`8.4 ZTS`，且`sapis`中包含`embed`。

```shel
php-config
Usage: /home/swoole/bin/php-config [OPTION]
Options:
  --prefix            [/opt/php-8.4]
  --includes          [-I/opt/php-8.4/include/php -I/opt/php-8.4/include/php/main -I/opt/php-8.4/include/php/TSRM -I/opt/php-8.4/include/php/Zend -I/opt/php-8.4/include/php/ext -I/opt/php-8.4/include/php/ext/date/lib]
  --ldflags           []
  --libs              [  -lm  -lpthread -lxml2 -lssl -lcrypto -lsqlite3 -lz -lcurl -lxml2 -lz -lpng16 -lz -lwebp -ljpeg -lonig -lsqlite3 -lxml2 -lxml2 -lxml2 -lxml2 -lxslt -lxml2 -lexslt -lxslt -lxml2 -lz -lssl -lcrypto]
  --extension-dir     [/opt/php-8.4/lib/php/extensions/debug-zts-20240924]
  --include-dir       [/opt/php-8.4/include/php]
  --lib-dir           [/opt/php-8.4/lib]
  --lib-embed         [libphp.so]
  --man-dir           [/opt/php-8.4/php/man]
  --php-binary        [/opt/php-8.4/bin/php]
  --php-sapis         [ cli embed phpdbg cgi]
  --ini-path          [/opt/php-8.4/lib]
  --ini-dir           []
  --configure-options [--enable-zts --with-pdo-mysql --with-pear --with-xsl --with-jpeg --with-webp --enable-embed --enable-debug --enable-gd --with-openssl --with-curl --enable-mbstring --enable-mysqlnd --enable-sockets --with-zlib --prefix=/opt/php-8.4/]
  --version           [8.4.14]
  --vernum            [80414]
```

## 1. Composer 包

使用 `composer install` 安装依赖的 `Composer` 包




```shell
cd swoole_compiler_v4/
composer install
```

## 2. 编译 `PHPX` 

需要 `GCC 9` 或更高版本，建议使用 `Ubuntu 22.04` 或更高版本的操作系统。

```shell
cd vendor/swoole/phpx
cmake .
make -j 16 
```

## 3. 配置动态链接库
需要将`PHP`和`PHPX`的`lib`目录加入到系统的`ldconfig`目录中。

```shell
sudo vim /etc/ld.so.conf.d/swoole.conf
```

加入：
```
/data/swoole_compiler_v4/vendor/swoole/phpx/lib
/opt/php-8.4/lib/
```

确认`.so`路径正确
```bash
ldconfig -p|grep php
	libphpx.so (libc6,x86-64) => /data/swoole_compiler_v4/vendor/swoole/phpx/lib/libphpx.so
	libphp.so (libc6,x86-64) => /opt/php-8.4/lib/libphp.so
```

## 4. 查看编译器
```bash
./swoole_compiler -h
Swoole-Compiler (AOT) v0.1.0

USAGE:
	./swoole_compiler <file/dir/project.yml> [options]

ARGUMENTS:
	<file>    Input PHP file/directory/project.yml to compile

OPTIONS:
	-O <level>           Optimization level (0-3, default: 0)
	-p, --profile        Enable performance profiling
	-d, --debug-info     Enable debug info
	-o, --output <file>  Output binary name (default: input basename)
	-v, --version        Show version
	-h, --help           Show this help message
	-f, --force          Force compile even if cache exists
	-m, --mode <mode>    Compilation mode, -m bin(binary) or -m ext(extension), default: bin
	-j, --job <num>      Number of parallel compilation jobs (default: 4)
	--no-literal-strings Disable literal strings optimization

EXAMPLES:
	./swoole_compiler hello.php
	./swoole_compiler bench.php -O2
	./swoole_compiler project/config.yml -O2
	./swoole_compiler my-ext/ -O2 -o myapp -m ext
	./swoole_compiler app.php -O3 -o myapp -v
```


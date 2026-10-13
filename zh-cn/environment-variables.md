# 环境变量

本页列出当前 TypePHP 编译器、TypePHP 随附的构建脚本，以及生成程序启动时会用到的环境变量。设置变量的时间很重要：构建时的 `PHP_HOME` 用于寻找 PHP SDK；运行时的 `PHPRC` 才用于寻找 `php.ini`。应用自己调用 `getenv()` 读取的任意业务变量由应用定义，不属于 TypePHP 的固定配置。

## 常用设置

Linux/macOS，使用宿主 PHP Embed 库编译并运行：

```bash
export PHP_HOME=/opt/php-8.5
export PHPX_HOME=/opt/phpx
export PATH="$PHP_HOME/bin:$PATH"
export LD_LIBRARY_PATH="$PHP_HOME/lib:$PHPX_HOME/lib${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}" # Linux
./tpc project.yml

# 在包含 php.ini 的目录下运行完整 PHP 运行时产物
PHPRC="$PWD/config/php" ./myapp
```

macOS 可按需使用 `DYLD_LIBRARY_PATH`，但发布 `.app` 时应将动态库放入包内并正确设置其加载路径；从 Finder 启动的程序不会继承当前 shell 临时 `export` 的变量。Windows PowerShell 示例：

```powershell
$env:PHP_HOME = 'C:\php'
$env:PHPX_HOME = 'C:\phpx'
$env:PATH = "C:\php;C:\phpx\lib;$env:PATH"
.\tpc.exe project.yml

$env:PHPRC = 'C:\myapp\config\php'
$env:PHP_INI_SCAN_DIR = 'C:\myapp\config\php.d'
.\myapp.exe
```

以上路径只是示例。`PHP_HOME`、`PHPX_HOME` 是编译器定位 SDK 的变量；它们不会自动加入系统动态库搜索路径，也不会设置生成程序的 `php.ini` 路径。

## 编译器和工具链

| 变量 | 使用阶段 | 作用 |
|---|---|---|
| `PHP_HOME` | 编译 | 指向匹配的 PHP 安装前缀，供 `php-config`、头文件、Embed 库和 ABI 检测使用。Linux/macOS 不设置时会尝试从当前 PHP 和 `PATH` 寻找；Windows 优先使用它。`php-builder` 编译期间会为子过程设置私有前缀。 |
| `PHPX_HOME` | 编译 | 指向 PHPX 源码或安装目录；未设置时尝试 Composer 安装的 `swoole/phpx`。Nano 和交叉编译也用它查找 PHPX 源码及目标 SDK。 |
| `HOME` | 编译，Linux/macOS | 私有 PHP 源码、php-builder 缓存和自动安装目录通常在 `~/.typephp`。 |
| `PATH` | 编译及运行 | 查找 `php-config`、C/C++ 编译器、`curl`、WASI 工具等；Windows 也用它寻找需要的 DLL。 |
| `PATHEXT` | 编译，Windows | 在 `PATH` 中查找未带扩展名的工具时使用；未设置时按 `.COM;.EXE;.BAT;.CMD` 查找。 |
| `LLVM_HOME` | 编译，Windows Clang | 当 `lld-link` 不在 `PATH` 中或编译器旁时，尝试 `$LLVM_HOME/bin/lld-link.exe` 和 `$LLVM_HOME/x64/bin/lld-link.exe`；不会回退到 MSVC `link.exe`。 |
| `INCLUDE`, `LIB`, `LIBPATH` | 编译，Windows | Microsoft ABI 构建所需的 MSVC/Windows SDK 头文件与库搜索路径；使用 LLVM 时可直接设置 `INCLUDE`、`LIB`，无需运行 `vcvars64.bat`。配置示例见 [Windows 环境](windows.md)。 |
| `TYPEPHP_STRICT_CACHE=1` | 编译 | 对 PCH 依赖进行严格检查；Windows 已默认启用。可能增加依赖检查时间。 |
| `CC`、`CXX`、`AR` | 构建 PHP/SDK | 供底层 php-src 等构建流程选择 C 编译器、C++ 编译器和归档器。TypePHP 应用的 C++ 编译器请优先用 `--compiler` 或项目配置指定；不要假定设置 `CXX` 就会覆盖它。 |
| `PKG_CONFIG_PATH` | 构建 PHP/原生依赖 | 供 `pkg-config` 查找非标准前缀的 `.pc` 文件；不是 TypePHP 专属变量。 |

下载代理可以用 `./tpc --proxy=<URL>` 指定。TypePHP 会把它传给下载器，并为子进程设置 `HTTP_PROXY`、`HTTPS_PROXY`、`ALL_PROXY` 及对应的小写变量。只设置这些通用环境变量时，`curl`、包管理器等可能使用它们，但 TypePHP 的 PHP stream 下载路径不保证读取它们；需要稳定控制所有下载时使用 `--proxy`。

## `php.ini`：`PHPRC` 与 `PHP_INI_SCAN_DIR`

普通 Embed 程序和 `php-builder` 生成的 Embed/CLI/FPM 程序使用完整 PHP 运行时，启动时可读取 PHP 配置。运行编译器的 PHP 进程也会读取自己的配置；**编译器进程加载的 `php.ini` 不会自动嵌入生成的程序**。

`PHPRC` 同样影响 `tpc` 编译器本身。自定义配置必须保留编译器必需的扩展；例如 Windows 发布包需要加载 `mbstring`，只包含 `opcache.enable=0` 的精简 ini 会导致编译器启动失败。建议复制发布包自带的 `php.ini` 后调整，保留正确的 `extension_dir` 和 `extension=mbstring`。TypePHP 会尊重用户设置的 `PHPRC`，不会自动清除它。若专用配置只用于运行产物，应在编译完成后再为产物进程设置。

| 变量 | 作用 |
|---|---|
| `PHPRC` | 指定主配置文件的位置，可为目录或配置文件路径。指向目录时，PHP 会先考虑相应 SAPI 的配置文件，再查找 `php.ini`；例如 Embed SAPI 会先查找 `php-embed.ini`。指向可读取文件时直接加载该文件。CLI/FPM 自身的配置路径选项可能具有更高优先级。 |
| `PHP_INI_SCAN_DIR` | 覆盖附加 `.ini` 扫描目录；文件按名称顺序加载。Linux/macOS 用 `:` 分隔多个目录，Windows 用 `;`。值中的空路径项表示同时扫描 PHP 编译时指定的默认目录；空字符串会关闭附加目录扫描。 |

例如将主配置与扩展配置放在不同目录：

```bash
PHPRC=/opt/myapp/conf \
PHP_INI_SCAN_DIR=/opt/myapp/conf.d \
./myapp
```

`/opt/myapp/conf` 可包含 `php.ini`；如果还存在 `php-embed.ini`，Embed SAPI 会优先加载后者。要同时保留 PHP 编译时的默认附加扫描目录，可在 Linux/macOS 使用 `PHP_INI_SCAN_DIR="/opt/myapp/conf.d:"`，在 Windows 使用末尾的 `;`。

可在普通模式程序的 `main()` 中确认实际加载情况：

```php
function main(): void
{
    var_dump(php_ini_loaded_file());
    var_dump(php_ini_scanned_files());
    var_dump(ini_get('memory_limit'));
}
```

`php --ini` 只报告**该 PHP CLI 进程**的配置，不能证明 TypePHP 生成程序加载了同一份配置。`--nano` 使用独立的 Nano 启动流程，不读取 `php.ini`；因此 `PHPRC` 和 `PHP_INI_SCAN_DIR` 不会配置 Nano 产物。Nano 仍有部分内建 INI 注册项，但这不等于加载配置文件。PHP 搜索顺序和附加目录规则见 [PHP 官方配置文件说明](https://www.php.net/manual/en/configuration.file.php)。

为 php-builder 的私有运行时准备专用配置目录时，不要直接复用宿主 PHP 的 `conf.d`：其中的 `extension=...` 可能指向与私有 PHP ABI 不匹配的动态扩展。

## 运行时动态库与应用变量

| 变量 | 平台 | 作用 |
|---|---|---|
| `LD_LIBRARY_PATH` | Linux | 运行动态链接的 `tpc` 或应用时查找 `libphp.so`、`libphpx.so` 和其他共享库。部署时也可使用系统库目录或 rpath。 |
| `DYLD_LIBRARY_PATH` | macOS | 命令行启动动态链接程序时的库搜索路径之一。`.app` 发布应使用包内 Frameworks 和正确的 Mach-O 加载路径，不依赖 shell 变量。 |
| `PATH` | Windows | 让进程找到运行时 DLL；程序目录也是 Windows 常用的 DLL 部署位置。 |
| 应用自定义变量 | 生成程序 | 应用可通过 `getenv()` 读取父进程传入的变量。变量必须在**启动生成程序时**设置；只在编译时设置不会固化到二进制中。 |

`php-builder` 省去了宿主 `libphp`/`libphpx` 动态库依赖，但仍可能需要其他系统共享库。Nano 不使用 `libphp`。FPM 工作进程默认会清理环境变量；需要传入应用变量时，在 FPM pool 配置中使用 `env[NAME] = value` 或按需配置 `clear_env`。详见 [PHP-FPM 官方配置说明](https://www.php.net/manual/en/install.fpm.configuration.php)。

## 交叉编译与平台脚本

| 变量 | 场景 | 作用 |
|---|---|---|
| `ANDROID_NDK_HOME`、`ANDROID_NDK_ROOT` | Android | NDK 根目录；优先使用 `ANDROID_NDK_HOME`。 |
| `TYPEPHP_ANDROID_API` | Android | 未在 target triple 中写入 API 数字时的 API 级别，默认 `24`。若 `--target-platform=aarch64-linux-android<数字>` 已指定，triple 优先。 |
| `PHPX_ANDROID_SDK_DIR` | Android 普通模式 | 覆盖默认 `$PHPX_HOME/android/arm64-v8a` SDK 目录；Nano 源码组合构建不读取它。 |
| `PHPX_ANDROID_BUILD_DIR`、`PHPX_ANDROID_JOBS`、`PHPX_SDK_JOBS` | 构建 PHPX Android/原生 SDK | 分别控制 `phpx/android/build.sh` 的构建目录、并行数，以及 `phpx/sdk/build-native.sh` 的并行数；普通 `tpc` 编译不读取它们。 |
| `TYPEPHP_ANDROID_NM` | Android Nano | 覆盖依赖审计所用的 `llvm-nm` 路径。 |
| `PHPX_IOS_PLATFORM`、`PHPX_IOS_SDK_DIR`、`PHPX_IOS_BUILD_DIR`、`PHPX_IOS_DEPLOYMENT_TARGET`、`PHPX_IOS_JOBS` | 构建 PHPX iOS SDK | 控制 `phpx/ios/build.sh`；TypePHP 编译 iOS 应用时按 `$PHPX_HOME/ios/<目标>` 查找 SDK，不读取这些变量来重定向目标目录。 |
| `TYPEPHP_IOS_PROVISIONING_PROFILE`、`TYPEPHP_IOS_CODE_SIGN_IDENTITY`、`TYPEPHP_IOS_BUNDLE_IDENTIFIER` | 仓库 iOS 示例打包 | 供 `examples/apple-native/package-ios-app.sh` 使用；不是 `tpc` 的通用签名选项。 |
| `PATH` | WASI | `--wasm` 会在 `PATH` 中查找 WASI Clang、LLVM 工具、Wasmtime；浏览器 profile 还需 Jco，库模式还需 wit-bindgen。 |
| `PHPX_WASM_BUILD_DIR`、`PHPX_WASM_JOBS`、`PHPX_WASM_DEPS_WORK_DIR`、`PHPX_WASM_DEPS_PREFIX` | 构建 PHPX WASI SDK | 控制 `phpx/wasm/` 下的构建脚本，不是普通 `tpc --wasm` 的配置项。 |
| `TYPEPHP_WASM_SDK_BUILD_DIR`、`TYPEPHP_WASM_JOBS` | 构建 TypePHP WASI SDK | 控制仓库 `compiler/wasm/build-sdk.sh` 的默认构建目录与并行数；同名 CLI 参数优先。 |
| `XDG_CACHE_HOME` | 从底层 WASI 构建脚本直接运行产物 | 当 `TYPEPHP_WASM_RUN=1` 时控制 Wasmtime 缓存目录；未设置时脚本使用 `/tmp/typephp-wasmtime-cache`。 |

## 编译器内部变量

以下变量由 TypePHP 的 WASI 包装流程或编译器重启过程传给子进程。普通项目不需要手动设置；修改它们可能绕过工具链检查或破坏构建结果。

| 变量 | 内部用途 |
|---|---|
| `TYPEPHP_SAPI_PHP_VERSION` | 源码版编译器切换到与 php-builder 目标匹配的 PHP CLI 时，防止重复重启。 |
| `TYPEPHP_WASM_INTERNAL_COMPILE`、`TYPEPHP_GENERATED_SOURCE_LIST` | WASI 脚本递归调用编译器生成 C++，并回传源码清单。 |
| `TYPEPHP_WASM_INTERFACE_MANIFEST`、`TYPEPHP_WASM_INTERFACE_WIT`、`TYPEPHP_WASM_INTERFACE_ADAPTER`、`TYPEPHP_WASM_INTERFACE_ASYNC_EXPORTS` | WASI component/library 接口生成与适配。 |
| `TYPEPHP_WASI_CC`、`TYPEPHP_WASI_CXX`、`TYPEPHP_WASI_AR`、`TYPEPHP_WASI_RANLIB`、`TYPEPHP_WASI_NM`、`TYPEPHP_WASI_LD`、`TYPEPHP_WASI_TARGET`、`TYPEPHP_WASI_CLANG_VERSION` | 检测后的 WASI 工具路径、目标及版本信息。 |
| `TYPEPHP_WASMTIME`、`TYPEPHP_WASMTIME_VERSION`、`TYPEPHP_JCO`、`TYPEPHP_JCO_VERSION`、`TYPEPHP_WIT_BINDGEN` | WASI 运行、浏览器适配及 WIT 工具信息。 |
| `TYPEPHP_WASM_BROWSER`、`TYPEPHP_WASM_BROWSER_DIR`、`TYPEPHP_WASM_PROGRAM_BUILD_DIR`、`TYPEPHP_WASM_MODE`、`TYPEPHP_WASM_PACKAGE`、`TYPEPHP_WASM_WORLD`、`TYPEPHP_WASM_NANO` | WASI 构建模式与产物参数；由 `--wasm` 和项目配置生成。 |
| `TYPEPHP_WASM_RUN` | 仅底层 `wasm/build-*-program.sh` 识别的直接运行开关，公开 `tpc --wasm` 命令不以此作为稳定接口。 |
| `TYPEPHP_PACKAGE_VERSION`、`TYPEPHP_BUILD_ID` | 打包 TypePHP 编译器自身时选择发布版本或预览构建号；不影响应用编译或运行。 |
| `PROCESSOR_ARCHITECTURE`、`PROCESSOR_ARCHITEW6432` | Windows 上打包 TypePHP 编译器时由系统提供的架构标识；不是用户应修改的应用构建选项。 |
| `PHPX_SDK_VERSION` | 打包 PHPX SDK 时覆盖归档版本；不影响 TypePHP 应用编译或运行。 |

示例应用（如 Qt Taskboard）的 `TYPEPHP_TASKBOARD_*`、`MACDEPLOYQT` 等由示例代码和打包脚本自己定义，不是所有 TypePHP 项目通用的环境变量。

相关说明：[安装](install.md)、[PHP Builder](php-builder.md)、[Nano](nano.md)、[跨平台原生应用](mobile-native.md)和 [WASM](wasm.md)。

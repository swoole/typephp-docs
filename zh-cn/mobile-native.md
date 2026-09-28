# Android、iOS 与 macOS 原生应用

TypePHP 可以把应用逻辑和原生 UI 描述编译为移动端或桌面端原生应用。仓库提供了
完整的计数器示例，包含 Logo、名字输入、计数按钮和重置按钮：

- [Android 原生应用示例](https://github.com/swoole/typephp/tree/master/examples/android-native)
- [iOS/macOS 原生应用示例](https://github.com/swoole/typephp/tree/master/examples/apple-native)

示例把 UI 结构、状态、业务逻辑和事件处理放在 TypePHP 中。Android Java 或 Apple
Objective-C++ 代码只保留平台框架要求的桥接和生命周期入口。

## Nano 与普通运行时

原生应用可以选择两种运行时来源：

| 构建方式 | PHP/PHPX 来源 | 目标设备依赖 |
|---|---|---|
| `--nano` | 直接交叉编译 Composer 安装的 PHP Nano 与 PHPX 源码 | 不需要 `libphp`、`libphpx` 或 PHP 环境 |
| 普通模式 | 链接与目标平台、架构和 PHP ABI 匹配的 PHPX SDK | 使用 SDK 中的 PHP/PHPX 库 |

Nano 构建需要 PHP Nano 1.0.2 或更高版本：

```shell
composer require --dev swoole/typephp "swoole/php-nano:^1.0.2"
```

`tpc.php` 本身仍由构建机的 PHP 动态执行，并需要 Composer `vendor` 包。`--nano`
改变的是生成产物的运行时，不会把编译器改成 Nano 程序。Nano 构建应将
`PHPX_HOME` 指向 PHPX 源码目录；无需预先为 Android 或 iOS 构建 `libphp.a`、
`libphpx.a`。

普通模式请从 [PHPX Releases](https://github.com/swoole/phpx/releases) 下载匹配的
平台 SDK。SDK 的平台、架构、PHP ABI 与 TypePHP/PHPX 版本必须和项目配置一致。

## Android arm64-v8a

Android 示例使用 `mode: lib` 生成 JNI 共享库，目标为
`aarch64-linux-android24`（`arm64-v8a`）。需要：

- Android SDK Platform 与 Build Tools 36；
- Android NDK r27 或更高版本；
- 提供 `javac` 与 `keytool` 的 JDK；
- TypePHP Composer 依赖和 PHPX 源码目录。

构建 Nano APK：

```bash
export ANDROID_SDK_ROOT=/path/to/android-sdk
export ANDROID_NDK_HOME=/path/to/android-ndk-r27d
export PHPX_HOME=/path/to/phpx

cd examples/android-native
./build-app.sh --nano
```

产物为 `dist/typephp-android-hello-debug.apk`。连接 arm64 真机或模拟器并启用 USB
调试后安装：

```bash
./install-app.sh
```

Nano 模式直接把 PHP Nano、PHPX、生成的 TypePHP C++ 和 JNI 桥接编入
`libtypephp_android_hello.so`，不读取 `PHPX_ANDROID_SDK_DIR`。不带 `--nano` 的普通
构建才需要设置该变量并提供匹配的 Android PHPX SDK。

Android 仍需要一个很小的 Java `Activity` 来进入框架并加载共享库。JNI 桥接在首次
启动时初始化 Nano 运行时一次，此后的点击事件直接调用 TypePHP 函数，不会为每次
事件重复执行 RINIT/RSHUTDOWN。

## iOS arm64

iPhoneOS 目标为 `arm64-apple-ios15.0`。构建必须在 macOS 上安装完整 Xcode；仅有
Command Line Tools 不包含 iPhoneOS SDK。Nano 编译只需要 PHP Nano/PHPX 源码，
真机打包和安装仍需要 Apple Development 证书与 Provisioning Profile。

```bash
export PHPX_HOME=/path/to/phpx
cd /path/to/typephp
sh examples/apple-native/build-ios.sh device --nano
```

设置签名环境变量后，同一脚本会生成签名后的 `.app`：

```bash
export TYPEPHP_IOS_PROVISIONING_PROFILE=/path/to/profile.mobileprovision
export TYPEPHP_IOS_CODE_SIGN_IDENTITY='Apple Development: Your Name (TEAMID)'
# Provisioning Profile 使用其他 Bundle ID 时再设置：
export TYPEPHP_IOS_BUNDLE_IDENTIFIER='your.provisioned.bundle.identifier'
sh examples/apple-native/build-ios.sh device --nano
```

使用 Xcode 命令行工具查找设备并安装：

```bash
xcrun devicectl list devices
xcrun devicectl device install app \
    --device <device-id> \
    'examples/apple-native/dist/TypePHP iOS Hello.app'
```

Apple silicon 上也可以构建 Nano 模拟器应用；它不需要单独的 simulator PHP/PHPX
静态库：

```bash
export PHPX_HOME=/path/to/phpx
sh examples/apple-native/build-ios.sh simulator --nano
xcrun simctl install booted 'examples/apple-native/dist/TypePHP iOS Simulator Hello.app'
xcrun simctl launch booted org.swoole.typephp.ios-simulator-hello
```

不带 `--nano` 时，真机和模拟器必须分别使用为 `arm64-apple-ios` 与
`arm64-apple-ios-simulator` 构建的 PHPX SDK，二者不能混用，也不能链接 macOS 或
Homebrew 的静态库。

## macOS 原生应用

Apple 示例也可以构建 AppKit 应用。Nano 模式需要 Xcode Command Line Tools、
PHP Nano/PHPX 源码和构建机 PHP，不需要宿主机 `libphp`：

```bash
export PHPX_HOME=/path/to/phpx
cd /path/to/typephp
php bin/tpc.php examples/apple-native/project.yml --nano --no-progress
./examples/apple-native/typephp_macos_hello
```

创建可从 Finder 启动的 `.app`：

```bash
sh examples/apple-native/package-app.sh
open 'examples/apple-native/dist/TypePHP macOS Hello.app'
```

## 原生库生命周期

`mode: lib` 不生成 `main`。编译器按项目名导出运行时初始化与关闭函数，桥接代码通过
`typephp_runtime.h` 中的 `TYPEPHP_RUNTIME_INIT(project)` 和
`TYPEPHP_RUNTIME_SHUTDOWN(project)` 调用。宿主必须在第一次 TypePHP 调用前初始化，
并且只能在不再有回调进入 TypePHP 后关闭。通常应让运行时覆盖整个应用进程生命
周期，不能在每次 Activity、窗口或按钮事件中反复初始化。

Nano 不支持 `mode: ext`。项目使用的 Composer 原生扩展会静态编入 `bin` 或 `lib`
运行时，它们的 MINIT/RINIT 与 RSHUTDOWN/MSHUTDOWN 分别随上述初始化和关闭各执行
一次。

## 平台边界

| 层次 | Android | iOS/macOS |
|---|---|---|
| 框架生命周期 | 最小 Java `Activity` | 最小 Objective-C++ UIKit/AppKit 入口 |
| 原生 UI 桥接 | JNI/View 桥接 | Objective-C++ UIKit/AppKit 桥接 |
| UI 结构与文本 | TypePHP | TypePHP |
| 状态、校验与点击事件 | TypePHP | TypePHP |
| Nano 运行时 | PHP Nano/PHPX 源码直接参与目标构建 | PHP Nano/PHPX 源码直接参与目标构建 |
| 普通运行时 | Android PHPX SDK | iPhoneOS/Simulator/macOS PHPX SDK |

这一边界让平台胶水代码保持最小，也让 TypePHP 逻辑可以在真实的 Android、iOS 和
macOS 生命周期中运行。

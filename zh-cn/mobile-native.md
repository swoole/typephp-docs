# Android、iOS 与 macOS 原生应用

TypePHP 可以把应用逻辑和原生 UI 描述编译为移动端或桌面端原生应用。仓库提供了完整的计数器示例，包含 Logo、名字输入、计数按钮和重置按钮：

- [Android 原生应用示例](https://github.com/swoole/typephp/tree/master/examples/android-native)
- [iOS/macOS 原生应用示例](https://github.com/swoole/typephp/tree/master/examples/apple-native)

示例有意把 UI 结构、状态、业务逻辑和事件处理尽量放在 TypePHP 中。Android Java 或 Apple Objective-C++ 代码只保留平台框架必需的桥接与生命周期入口。

## SDK 获取方式

请从 [PHPX Releases](https://github.com/swoole/phpx/releases) 下载完整的平台 SDK。PHPX SDK 包含 TypePHP 应用链接所需的运行时、头文件与库。Swoole CLI Release 提供生成这些 SDK 所需的底层 PHP 运行时产物；普通应用开发者通常不需要自行组装这两层。

SDK 的平台、架构、PHP ABI 与 TypePHP/PHPX 版本必须和应用构建配置匹配。

## Android arm64-v8a

Android 示例目标为 `aarch64-linux-android24`（`arm64-v8a`），需要：

- Android SDK Platform 与 Build Tools 36
- Android NDK r27 或更高版本
- 提供 `javac` 与 `keytool` 的 JDK
- 匹配的 PHPX Android SDK

设置工具链路径并构建 APK：

```bash
export ANDROID_SDK_ROOT=/path/to/android-sdk
export ANDROID_NDK_HOME=/path/to/android-ndk-r27d
export PHPX_ANDROID_SDK_DIR=/path/to/phpx-android-sdk

cd examples/android-native
./build-app.sh
```

APK 会生成到 `dist/` 目录。通过 `adb` 连接真机或模拟器后，可执行：

```bash
./install-app.sh
```

Android 框架仍需要一个很小的 Java `Activity` 作为入口并加载原生库。示例通过通用 View/JNI 桥接把原生控件提供给 TypePHP，应用状态和点击行为仍由 TypePHP 实现。

## iOS arm64

iOS 目标为 iphoneos 平台的 `arm64-apple-ios15.0`。构建必须在 macOS 上安装完整 Xcode；仅有 Command Line Tools 不足以构建 iPhoneOS 应用。真机应用还需要 Apple 签名证书与 Provisioning Profile。

使用匹配的 PHPX iOS SDK，并按照 [iOS/macOS 示例](https://github.com/swoole/typephp/tree/master/examples/apple-native)中的构建说明操作。Objective-C++ 桥接负责 UIKit 生命周期边界，TypePHP 负责示例的 UI 声明、状态与事件。

## macOS 原生应用

同一个 Apple 示例也可以在 macOS 上构建 AppKit 原生应用。它需要 Xcode Command Line Tools，以及与宿主架构匹配的 PHPX/macOS SDK。桥接层使用 Objective-C++，应用行为使用 TypePHP 实现。

## 平台边界

| 层次 | Android | iOS/macOS |
|---|---|---|
| 框架生命周期 | 最小 Java `Activity` | 最小 Objective-C++ UIKit/AppKit 入口 |
| 原生 UI 桥接 | JNI/View 桥接 | Objective-C++ UIKit/AppKit 桥接 |
| UI 结构与文本 | TypePHP | TypePHP |
| 状态、校验与点击事件 | TypePHP | TypePHP |
| PHP 运行时与原生链接 | PHPX 平台 SDK | PHPX 平台 SDK |

这一边界让平台胶水代码保持最小，使示例真正体现 TypePHP 原生应用开发，而不是只调用少量 TypePHP 逻辑的 Java 或 Objective-C++ 应用。

# Android, iOS, and macOS Native Applications

TypePHP can compile application logic and native UI descriptions into native mobile or desktop applications. The repository provides complete counter demos with a logo, name input, counter button, and reset button:

- [Android native example](https://github.com/swoole/typephp/tree/master/examples/android-native)
- [iOS/macOS native example](https://github.com/swoole/typephp/tree/master/examples/apple-native)

The examples intentionally keep UI structure, state, business logic, and event handling in TypePHP. Platform-language code is limited to the bridge and lifecycle entry points required by Android or Apple frameworks.

## SDK Distribution

Download the complete platform SDK from [PHPX Releases](https://github.com/swoole/phpx/releases). A PHPX SDK contains the TypePHP-facing runtime, headers, and libraries needed to link an application. Swoole CLI releases provide the lower-level PHP runtime artifacts used to produce those SDKs; application developers normally do not need to assemble the layers themselves.

Use an SDK whose platform, architecture, PHP ABI, and TypePHP/PHPX version match the application build.

## Android arm64-v8a

The Android example targets `aarch64-linux-android24` (`arm64-v8a`). It requires:

- Android SDK Platform and Build Tools 36
- Android NDK r27 or later
- a JDK providing `javac` and `keytool`
- the matching PHPX Android SDK

Set the toolchain paths and build the APK:

```bash
export ANDROID_SDK_ROOT=/path/to/android-sdk
export ANDROID_NDK_HOME=/path/to/android-ndk-r27d
export PHPX_ANDROID_SDK_DIR=/path/to/phpx-android-sdk

cd examples/android-native
./build-app.sh
```

The APK is written below `dist/`. With a device or emulator connected through `adb`, install it with:

```bash
./install-app.sh
```

Android still requires a small Java `Activity` to enter the framework and load the native library. The demo's generic View/JNI bridge exposes native controls to TypePHP; application state and click behavior remain in TypePHP.

## iOS arm64

The iOS target is `arm64-apple-ios15.0` for `iphoneos`. Building it requires macOS with a full Xcode installation. Command Line Tools alone do not include everything needed for an iPhoneOS application. A real-device build also requires an Apple signing identity and provisioning profile.

Use the matching PHPX iOS SDK and follow the build instructions in the [iOS/macOS example](https://github.com/swoole/typephp/tree/master/examples/apple-native). The Objective-C++ bridge owns the UIKit lifecycle boundary; TypePHP owns the demo UI declaration, state, and events.

## macOS Native Application

The same Apple example also builds a native AppKit application on macOS. It needs Xcode Command Line Tools and a PHPX/macOS SDK matching the host architecture. The bridge is Objective-C++, while the application behavior is implemented in TypePHP.

## Platform Boundary

| Layer | Android | iOS/macOS |
|---|---|---|
| Framework lifecycle | Minimal Java `Activity` | Minimal Objective-C++ UIKit/AppKit entry |
| Native UI bridge | JNI/View bridge | Objective-C++ UIKit/AppKit bridge |
| UI structure and text | TypePHP | TypePHP |
| State, validation, and click events | TypePHP | TypePHP |
| PHP runtime and native linking | PHPX platform SDK | PHPX platform SDK |

This boundary keeps platform glue small while making the examples representative TypePHP applications rather than Java or Objective-C++ applications with only a small TypePHP function.

# Android, iOS, and macOS Native Applications

TypePHP can compile application logic and native UI descriptions into native
mobile or desktop applications. The repository provides complete counter demos
with a logo, name input, counter button, and reset button:

- [Android native example](https://github.com/swoole/typephp/tree/master/examples/android-native)
- [iOS/macOS native example](https://github.com/swoole/typephp/tree/master/examples/apple-native)

The examples keep UI structure, state, business logic, and event handling in
TypePHP. Android Java or Apple Objective-C++ code contains only the bridge and
lifecycle entry points required by the platform framework.

## Nano and regular runtimes

Native applications can use either runtime source:

| Build | PHP/PHPX source | Target-device dependency |
|---|---|---|
| `--nano` | Cross-compiles the PHP Nano and PHPX sources installed by Composer | No `libphp`, `libphpx`, or PHP environment |
| Regular mode | Links a PHPX SDK matching the target platform, architecture, and PHP ABI | Uses the PHP/PHPX libraries from that SDK |

Nano builds require PHP Nano 1.0.2 or later:

```shell
composer require --dev swoole/typephp "swoole/php-nano:^1.0.2"
```

The build host still runs `tpc.php` dynamically with PHP and its Composer
`vendor` packages. `--nano` changes the generated artifact's runtime; it does
not turn the compiler itself into a Nano program. For a Nano build, point
`PHPX_HOME` at the PHPX source tree. Prebuilt Android or iOS `libphp.a` and
`libphpx.a` archives are not required.

For regular mode, download a matching platform SDK from
[PHPX Releases](https://github.com/swoole/phpx/releases). Its platform,
architecture, PHP ABI, and TypePHP/PHPX version must match the project.

## Android arm64-v8a

The Android example uses `mode: lib` to produce a JNI shared library and targets
`aarch64-linux-android24` (`arm64-v8a`). It requires:

- Android SDK Platform and Build Tools 36;
- Android NDK r27 or later;
- a JDK providing `javac` and `keytool`;
- the TypePHP Composer dependencies and a PHPX source tree.

Build the Nano APK:

```bash
export ANDROID_SDK_ROOT=/path/to/android-sdk
export ANDROID_NDK_HOME=/path/to/android-ndk-r27d
export PHPX_HOME=/path/to/phpx

cd examples/android-native
./build-app.sh --nano
```

The result is `dist/typephp-android-hello-debug.apk`. With an arm64 device or
emulator connected and USB debugging enabled, install it with:

```bash
./install-app.sh
```

Nano compiles PHP Nano, PHPX, the generated TypePHP C++, and the JNI bridge into
`libtypephp_android_hello.so`. It does not read `PHPX_ANDROID_SDK_DIR`. A regular
build without `--nano` does require that variable and a matching Android PHPX
SDK.

Android still needs a small Java `Activity` to enter the framework and load the
shared library. The JNI bridge initializes the Nano runtime once on first
startup. Later click events call TypePHP functions directly and do not repeat
RINIT/RSHUTDOWN for each event.

## iOS arm64

The iPhoneOS target is `arm64-apple-ios15.0`. Building requires macOS with a
full Xcode installation; Command Line Tools alone do not contain the iPhoneOS
SDK. Nano compilation needs only the PHP Nano/PHPX sources, while packaging and
installing on a physical iPhone still requires an Apple Development identity
and provisioning profile.

```bash
export PHPX_HOME=/path/to/phpx
cd /path/to/typephp
sh examples/apple-native/build-ios.sh device --nano
```

Set the signing environment before running the same script to produce a signed
`.app`:

```bash
export TYPEPHP_IOS_PROVISIONING_PROFILE=/path/to/profile.mobileprovision
export TYPEPHP_IOS_CODE_SIGN_IDENTITY='Apple Development: Your Name (TEAMID)'
# Set this only when the profile uses another bundle identifier:
export TYPEPHP_IOS_BUNDLE_IDENTIFIER='your.provisioned.bundle.identifier'
sh examples/apple-native/build-ios.sh device --nano
```

Find a device and install the bundle with the Xcode command-line tools:

```bash
xcrun devicectl list devices
xcrun devicectl device install app \
    --device <device-id> \
    'examples/apple-native/dist/TypePHP iOS Hello.app'
```

Apple silicon can also build a Nano simulator application without separate
simulator PHP/PHPX archives:

```bash
export PHPX_HOME=/path/to/phpx
sh examples/apple-native/build-ios.sh simulator --nano
xcrun simctl install booted 'examples/apple-native/dist/TypePHP iOS Simulator Hello.app'
xcrun simctl launch booted org.swoole.typephp.ios-simulator-hello
```

Without `--nano`, physical-device and simulator builds require separate PHPX
SDKs compiled for `arm64-apple-ios` and `arm64-apple-ios-simulator`. They are not
interchangeable and cannot use macOS or Homebrew static archives.

## macOS native application

The Apple example also builds an AppKit application. Nano mode needs Xcode
Command Line Tools, the PHP Nano/PHPX sources, and build-host PHP, but it does
not need host `libphp`:

```bash
export PHPX_HOME=/path/to/phpx
cd /path/to/typephp
php bin/tpc.php examples/apple-native/project.yml --nano --no-progress
./examples/apple-native/typephp_macos_hello
```

Create a standard `.app` bundle that can be launched from Finder:

```bash
sh examples/apple-native/package-app.sh
open 'examples/apple-native/dist/TypePHP macOS Hello.app'
```

## Native-library lifecycle

`mode: lib` does not generate `main`. The compiler exports runtime initialization
and shutdown functions derived from the project name. Bridge code calls them
through `TYPEPHP_RUNTIME_INIT(project)` and `TYPEPHP_RUNTIME_SHUTDOWN(project)`
from `typephp_runtime.h`. The host must initialize before the first TypePHP call
and may shut down only after no callback can enter TypePHP. In most applications,
the runtime should cover the entire process lifetime rather than being restarted
for each Activity, window, or button event.

Nano does not support `mode: ext`. Composer native extensions used by the
project are statically compiled into its `bin` or `lib` runtime. Their
MINIT/RINIT and RSHUTDOWN/MSHUTDOWN phases run once with the initialization and
shutdown described above.

## Platform boundary

| Layer | Android | iOS/macOS |
|---|---|---|
| Framework lifecycle | Minimal Java `Activity` | Minimal Objective-C++ UIKit/AppKit entry |
| Native UI bridge | JNI/View bridge | Objective-C++ UIKit/AppKit bridge |
| UI structure and text | TypePHP | TypePHP |
| State, validation, and click events | TypePHP | TypePHP |
| Nano runtime | PHP Nano/PHPX sources compiled for the target | PHP Nano/PHPX sources compiled for the target |
| Regular runtime | Android PHPX SDK | iPhoneOS/Simulator/macOS PHPX SDK |

This boundary keeps platform glue small while allowing TypePHP logic to run in
real Android, iOS, and macOS application lifecycles.

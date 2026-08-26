# Composer Installation

You can quickly install `TypePHP` using `Composer`:

```bash
composer require swoole/typephp
```

`tpc.php` is run by the current `PHP` interpreter. It requires `PHP 8.4` or `PHP 8.5`.
`TypePHP` projects require a `GCC` or `Clang` that supports `C++17`; check that the current environment meets this requirement.

> It is recommended to use `require-dev`, since `tpc.php` is a development-time and build-time tool

## Verify the installation
```bash
vendor/bin/tpc.php --version
vendor/bin/tpc.php --help
```

## Compiling a PHP project

```bash
vendor/bin/tpc.php project.yml
vendor/bin/tpc.php hello.php
```

If the system already has mutually compatible `libphp.so` and `libphpx.so`, `TypePHP` uses them directly;
if the system is missing `libphp.so` or `libphpx.so`, `TypePHP` auto-builds them in a `Linux` environment.

## Environment requirements

- Auto-building `PHPX` requires `CMake 3.24` or higher; if missing, the installer can install it via the system package manager.
- `PHP` development header files; providing `php-config` is recommended.
- Auto-building `PHPX` requires build dependencies such as `gmp` and `mpfr`.

Common dependencies on Ubuntu / Debian:

```bash
sudo apt-get update
sudo apt-get install -y \
    build-essential cmake pkg-config \
    libgmp-dev libmpfr-dev
```

Common dependencies on Fedora / RHEL:

```bash
sudo dnf install -y \
    gcc gcc-c++ make cmake pkgconf-pkg-config \
    gmp-devel mpfr-devel
```

Specific package names may vary by distribution version. When local libraries are already available, you do not need to install these dependencies again for auto-build.

`tpc.php` does not require GCC or CMake before checking local libraries. Only after the user confirms auto-building the missing libraries does the installer detect `apt-get`, `dnf`, or `yum` and ask whether to install GCC/G++, make, CMake, pkg-config, and other missing development packages. After the local libraries are handled, the compiler performs the final toolchain validation.

Team projects should commit `composer.json` and `composer.lock`. Other developers can run:

```bash
composer install
vendor/bin/tpc.php project.yml
```

## Global installation

```bash
composer global require swoole/typephp
composer global config bin-dir --absolute
```

After adding the `Composer` global `bin` directory printed by the command to `PATH`, you can run:

```bash
tpc.php --version
tpc.php project.yml
```

Project installation pins the compiler version via `composer.lock`; production projects should prefer the project-level `vendor/bin/tpc.php`.

## Prefer existing local libraries

If you have already built `PHP Embed SAPI` and `PHPX` yourself, it is recommended to set the installation directories explicitly:

```bash
export PHP_HOME=/opt/php-8.4
export PHPX_HOME=/opt/phpx

vendor/bin/tpc.php project.yml
```

The corresponding directories should contain:

```text
$PHP_HOME/bin/php
$PHP_HOME/bin/php-config
$PHP_HOME/lib/libphp.so

$PHPX_HOME/CMakeLists.txt
$PHPX_HOME/include/
$PHPX_HOME/lib/libphpx.so
```

As long as these files exist, `tpc.php` proceeds directly to project compilation, without downloading PHP source code or running the PHPX CMake build.

`libphpx.so` must be compiled against the same PHP header files and `php-config` pointed to by `PHP_HOME`, to avoid PHP ABI mismatches.

## Composer check flow

Every compilation first checks existing local libraries and only offers the auto-build option when they are missing:

```text
System PHP
  → vendor/bin/tpc.php
  → look for an existing libphp.so
      └─ if missing, ask whether to auto-build
  → look for an existing libphpx.so
      └─ if missing, ask whether to auto-build with the same PHP
  → compile the TypePHP project
```

Auto-build must be started from `tpc.php`. The self-hosted `tpc` binary needs the system dynamic linker to load `libphp.so` and `libphpx.so` before entering the main program; if the libraries do not exist, the binary cannot start and therefore never gets a chance to run the installer.

## Auto-building libphp.so when missing

Linux binary mode requires `libphp.so` or `libphp.a` from the PHP Embed SAPI. Many distributions only provide PHP CLI/FPM; being able to run `php` does not mean the Embed library is installed.

Simply run the compile command to trigger the check:

```bash
vendor/bin/tpc.php project.yml
```

When the Embed library is missing, you will be asked:

```text
The current PHP installation does not provide libphp.so: /usr
Build a private PHP embed library now? [Y/n] y
PHP version [the full PHP_VERSION currently running tpc.php]:
Install directory [/home/user/.typephp]:
```

The installer completes the following steps in order:

1. Fetch the stable release information from PHP.net;
2. By default select the exact same PHP version that currently runs Composer and `tpc.php` — for example, if the current version is PHP 8.4.14, it builds PHP 8.4.14 by default; the user can also manually enter another stable PHP 8.4.x or 8.5.x version;
3. Download the official `.tar.xz` source package from PHP.net;
4. Verify the source package using the official SHA-256 checksum;
5. Read the current `php-config --configure-options` and preserve the current extensions and build configuration;
6. Replace the installation prefix and add `--enable-embed=shared`;
7. Detect `apt-get`, `dnf`, or `yum`, and install missing development packages after confirmation;
8. Build and install PHP using at most 8 parallel jobs;
9. Merge the current ini to generate the `php.ini` used by the private PHP.

Default installation directory:

```text
~/.typephp/
├── bin/
│   ├── php
│   └── php-config
├── include/
│   └── php/
├── lib/
│   ├── libphp.so
│   ├── php.ini
│   └── loaded-extensions.txt
└── var/
    └── build/
```

After a successful installation, the current `tpc.php` process sets the new `PHP_HOME` and continues the original compilation task.

### php.ini and extensions

The installer merges the current main `php.ini` and ini files in scanned directories.

When the old and new PHP major/minor versions are the same — for example, both PHP 8.4 — it attempts to copy the shared extensions referenced by the current configuration. Extensions that cannot be found are commented out in the new ini, to avoid PHP failing to start due to loading a non-existent `.so`.

When crossing major/minor versions — for example, building PHP 8.5 from PHP 8.4 — existing binary extensions are not copied, because extensions for different PHP module APIs cannot be reused directly.

## Auto-building libphpx.so when missing

Composer installs the `swoole/phpx` source code. TypePHP locates PHPX in the following order:

1. `PHPX_HOME`;
2. The `swoole/phpx` directory returned by Composer `InstalledVersions`;
3. `vendor/swoole/phpx` inside the TypePHP source repository.

To use a PHPX you have checked out yourself:

```bash
export PHPX_HOME=/path/to/phpx
vendor/bin/tpc.php project.yml
```

The target directory must contain `CMakeLists.txt`, `include/`, and `src/`.

When `<PHPX>/lib/libphpx.so` is missing, you will be asked:

```text
The PHPX installation does not provide libphpx.so: /path/to/phpx
Build libphpx.so now? [Y/n]
```

After confirming, it runs the equivalent of:

```bash
cmake \
    -S /path/to/phpx \
    -B /path/to/phpx/build \
    -DCMAKE_BUILD_TYPE=Release \
    -DBUILD_TESTS=OFF \
    -Dphp_dir="$PHP_HOME"

cmake \
    --build /path/to/phpx/build \
    --parallel 8 \
    --target phpx
```

The actual parallelism is the smaller of the CPU count and 8. On success, it generates:

```text
<PHPX>/lib/libphpx.so
```

### PHP ABI consistency

PHPX itself does not depend on `libphp.so`, nor does it need to link the Embed library at build time. It depends on PHP's header files and `php-config`. To guarantee final runtime ABI consistency, PHPX should be built against the same PHP build that the project ultimately uses. The auto-installer sets all of the following:

```text
PHP_HOME=<the PHP prefix where libphp.so was found or auto-installed>
PATH=$PHP_HOME/bin:$PATH
-Dphp_dir=$PHP_HOME
```

and verifies that this PHP prefix provides:

```text
$PHP_HOME/bin/php
$PHP_HOME/bin/php-config
```

If `libphp.so` was just auto-installed to `~/.typephp`, PHPX is built using `~/.typephp/bin/php` and `~/.typephp/bin/php-config`. If the user declines to build the Embed library but still chooses to build PHPX, PHPX can directly use the `php-config` corresponding to the current Composer PHP; the two builds have no dependency on each other.

After changing the PHP version, ZTS/NTS, or Debug ABI, delete the old `libphpx.so` and rebuild.

## Subsequent reuse

For subsequent compilations, set PHP and PHPX explicitly:

```bash
export PHP_HOME="$HOME/.typephp"
export PHPX_HOME=/path/to/phpx
vendor/bin/tpc.php project.yml
```

When selecting the same PHP directory and version again, the installer asks whether to reuse the existing `libphp.so`. When `libphpx.so` exists, it does not run CMake again.

## CI and non-interactive environments

Non-interactive environments do not automatically confirm downloads, system package installations, or CMake builds. Generate the two libraries during image preparation, then set:

```bash
export PHP_HOME=/opt/typephp-php
export PHPX_HOME=/opt/phpx

composer install --no-interaction
vendor/bin/tpc.php project.yml --no-progress
```

Do not reuse these local libraries across different operating systems, CPU architectures, PHP major/minor versions, or PHP ABIs.

## Manually building PHPX

If auto-build fails, run manually with the same PHP prefix:

```bash
cmake -S "$PHPX_HOME" -B "$PHPX_HOME/build" \
    -DCMAKE_BUILD_TYPE=Release \
    -DBUILD_TESTS=OFF \
    -Dphp_dir="$PHP_HOME"

cmake --build "$PHPX_HOME/build" --parallel 4 --target phpx
```

Confirm the result:

```bash
test -f "$PHPX_HOME/lib/libphpx.so"
```

## First compilation

Create `hello.php`:

```php
<?php

function main(): void
{
    echo "Hello TypePHP\n";
}
```

Compile and run:

```bash
vendor/bin/tpc.php hello.php -O2 -o hello
./hello
```

See [Compiling Projects](compile.md) and [Command-line Options](options.md) for full build options.

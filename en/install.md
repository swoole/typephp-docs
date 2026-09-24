# Installing TypePHP

TypePHP recommends directly using the pre-built, self-hosted `tpc` binary provided by GitHub Releases. This tutorial uses Linux as an example and assumes that mutually compatible `libphp.so` and `libphpx.so` have already been compiled and installed on the machine.

If you are using the Windows toolkit or prefer to run `tpc.php` via the PHP Composer package, see the other installation methods at the end of this page.

## 1. Download tpc

Open the TypePHP Releases page. Linux and macOS users must select a package that matches the operating system, CPU architecture, and host PHP version. The Windows toolkit includes PHP, so Windows users can directly choose the bundled PHP 8.4 or PHP 8.5 version they want:

<https://github.com/swoole/typephp/releases>

After downloading, extract it, for example:

```bash
mkdir -p "$HOME/typephp"
cd "$HOME/typephp"
tar -xf /path/to/tpc_v*_linux_x64_php8.4.*-zts.tar.gz
cd tpc_v*_linux_x64_php8.4.*-zts
chmod +x tpc
```

Use the actual version shown on the Releases page in place of the example. Linux and macOS provide both PHP 8.4 ZTS and PHP 8.5 ZTS builds, and each filename records the complete PHP version used for the build. Do not mix binaries for another operating system, CPU architecture, PHP version, or ZTS/NTS ABI. Windows provides complete PHP 8.4 and PHP 8.5 toolkits, each containing its matching PHP/PHPX runtime and SDK.

Linux and macOS archives contain only `tpc`, the English and Chinese READMEs, and the license. Production Composer dependencies are embedded in `tpc` through `embedded-files`; no `composer install` or disk `vendor` directory is required.

TypePHP produces native executable programs for the current platform, not PHP bytecode.

## 2. Check PHP Embed

TypePHP's binary mode requires the PHP Embed SAPI. PHP 8.4 is recommended, and ensure that PHP, header files, `php-config`, and `libphp.so` all come from the same PHP build.

Set the PHP installation prefix:

```bash
export PHP_HOME=/opt/php-8.4
```

Check the PHP version and configuration:

```bash
"$PHP_HOME/bin/php" -v
"$PHP_HOME/bin/php-config" --version
"$PHP_HOME/bin/php-config" --php-sapis
"$PHP_HOME/bin/php-config" --lib-dir
```

`--php-sapis` should include `embed`, and the PHP lib directory should contain:

```bash
test -f "$PHP_HOME/lib/libphp.so"
```

A typical PHP configure configuration includes:

```text
--enable-embed=shared
```

If ZTS is used, the PHP header files and libraries used by PHPX and `tpc` must also belong to the same ZTS build. Do not mix files from different PHP major/minor versions, ZTS/NTS, or Debug/Release ABIs.

## 3. Check PHPX

PHPX must be built against the same PHP from the previous step. PHPX depends on GMP and MPFR; using Ubuntu / Debian as an example:

```bash
sudo apt-get update
sudo apt-get install -y \
    build-essential cmake pkg-config \
    libgmp-dev libmpfr-dev
```

If PHPX has not been compiled yet, run:

```bash
git clone https://github.com/swoole/phpx.git /opt/phpx

cmake -S /opt/phpx -B /opt/phpx/build \
    -DCMAKE_BUILD_TYPE=Release \
    -DBUILD_TESTS=OFF \
    -Dphp_dir="$PHP_HOME"

cmake --build /opt/phpx/build --parallel 4 --target phpx
```

Set the PHPX path and check the result:

```bash
export PHPX_HOME=/opt/phpx
test -f "$PHPX_HOME/lib/libphpx.so"
```

`PHPX_HOME` must point to the PHPX source root directory, which should contain:

```text
$PHPX_HOME/CMakeLists.txt
$PHPX_HOME/include/
$PHPX_HOME/src/misc/
$PHPX_HOME/lib/libphpx.so
```

In addition to linking `libphpx.so`, TypePHP also uses PHPX's header files and `src/misc` source files, so you cannot keep only a single `.so` file.

## 4. Configure environment variables

You can set them directly in the current terminal:

```bash
export PHP_HOME=/opt/php-8.4
export PHPX_HOME=/opt/phpx
export PATH="$HOME/typephp:$PATH"
export LD_LIBRARY_PATH="$PHP_HOME/lib:$PHPX_HOME/lib${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}"
```

If you want to use them long-term, add these settings to `~/.bashrc` or the corresponding shell configuration file.

You can also use the system dynamic library configuration. Create `/etc/ld.so.conf.d/typephp.conf`:

```text
/opt/php-8.4/lib
/opt/phpx/lib
```

Then refresh the cache:

```bash
sudo ldconfig
```

Choose either `LD_LIBRARY_PATH` or `ldconfig`. Development environments typically use the former; fixed deployment environments can use the latter.

## 5. Verify the dynamic libraries

Check the dynamic linking result before running `tpc`:

```bash
ldd "$HOME/typephp/tpc" | grep -E 'libphp(x)?\.so'
```

Normally you should see both `libphp.so` and `libphpx.so` resolve to the expected directories, for example:

```text
libphpx.so => /opt/phpx/lib/libphpx.so
libphp.so  => /opt/php-8.4/lib/libphp.so
```

If it shows `not found`, fix `LD_LIBRARY_PATH` or `ldconfig` first. The `tpc` binary needs to load these two libraries before entering the main program, and cannot repair the environment on its own when the libraries are missing.

## 6. View the compiler

```bash
tpc --version
tpc --help
```

You can also run it directly from the extracted directory:

```bash
./tpc --version
./tpc --help
```

Basic command format:

```text
tpc <PHP file, directory, or project.yml> [options]
```

## 7. Compile your first program

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
tpc hello.php -O2 -o hello
./hello
```

Expected output:

```text
Hello TypePHP
```

For complex projects, use `project.yml`:

```yaml
name: my-app
version: 1.0.0

sources:
  - src
  - main.php

build-dir: build
```

```bash
tpc project.yml -O2
```

See [Compiling Projects](compile.md), [Command-line Options](options.md), and [project.yml Configuration](project-yml.md) for full details.

## FAQ

### tpc: command not found

Run using the full path, or add the directory containing `tpc` to `PATH`:

```bash
export PATH="$HOME/typephp:$PATH"
```

### error while loading shared libraries

Use `ldd` to see the missing library:

```bash
ldd "$HOME/typephp/tpc" | grep 'not found'
```

Confirm that `PHP_HOME`, `PHPX_HOME`, and the dynamic library search path point to the correct directories.

### libphpx.so is incompatible with the PHP ABI

Reconfigure and rebuild PHPX using `$PHP_HOME/bin/php-config`:

```bash
cmake -S "$PHPX_HOME" -B "$PHPX_HOME/build" \
    -DCMAKE_BUILD_TYPE=Release \
    -DBUILD_TESTS=OFF \
    -Dphp_dir="$PHP_HOME"

cmake --build "$PHPX_HOME/build" --parallel 4 --target phpx
```

## Other installation methods

- [Composer](composer.md): for users who already have PHP and Composer in a Linux environment. Prefers existing `libphp.so` / `libphpx.so`, and can interactively auto-build them via `vendor/bin/tpc.php` when missing.
- [Windows Environment](windows.md): directly use the TypePHP Windows toolkit, which includes the required DLLs, PHP, PHPX, and a pre-compiled `tpc.exe`.

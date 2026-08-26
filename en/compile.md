# Compiling Projects

It is recommended to use the binary compiler produced by TypePHP's self-hosting build:

```text
./tpc
```

Basic format:

```bash
./tpc <PHP file, directory, or project.yml> [options]
```

If the directory containing the binary is already in `PATH`, you can also use `tpc` directly. For the Composer installation method, see [Composer Installation](composer.md).

## Single-file compilation

The PHP file must provide a global `main()` entry point:

```php
<?php

function main(): void
{
    echo "Hello TypePHP\n";
}
```

Compile:

```bash
./tpc hello.php
```

By default, an executable named after the entry file is generated:

```bash
./hello
```

Specify the optimization level and output path:

```bash
./tpc hello.php -O2 -o build/hello
```

Compile and run immediately:

```bash
./tpc hello.php -O2 --run
```

## Compiling a directory

When a directory is passed, the compiler recursively scans the PHP files within it:

```bash
./tpc src/ -O2 -o app
```

Directory mode suits projects with a simple structure. Use `project.yml` when you need precise control over sources, ignored files, C++ source files, and linking options.

## Using project.yml

```yaml
name: my-app
version: 1.0.0

sources:
  - src
  - main.php

ignore:
  - src/DevelopmentOnly.php

build-dir: build
```

Run:

```bash
./tpc project.yml -O2
```

The configuration file path can use any filename as long as the extension is `.yml` or `.yaml`. Relative paths are resolved relative to the directory containing the configuration file.

See [project.yml Configuration](project-yml.md) for the full set of fields.

## Build modes

The default `bin` mode generates an executable program:

```bash
./tpc project.yml -m bin
```

`lib` mode generates a dynamic library that can be linked by other TypePHP projects:

```bash
./tpc project.yml -m lib
```

See [TypePHP Dynamic Libraries](library.md) for the full steps to create, publish, and reference dynamic libraries.

`ext` mode generates a PHP extension:

```bash
./tpc extension.yml -m ext -o my_extension
```

When an extension depends on other PHP modules such as `pdo_mysql` or `curl`, you can use `extension-dependencies` (abbreviated `ext-deps`) in YAML to write the required dependencies into the Zend module metadata. The two configuration names cannot appear at the same time. This configuration does not auto-load extensions, and is different from the `link-libs` of native libraries; see [project.yml: PHP Extension Dependencies](project-yml.md#php-扩展依赖) for the full configuration and deployment order.

Linux `bin` mode requires `libphp.so` or `libphp.a`. The `tpc` binary itself also depends on `libphp.so` and `libphpx.so`, so it cannot start the installer on its own when the libraries are missing. Composer users should prepare these libraries automatically via `vendor/bin/tpc.php`; see [Composer Installation](composer.md). `ext` mode does not trigger the Embed PHP installer.

## Inspecting the generated code

`--dry` only generates the C++ files without invoking the C++ compiler and linker:

```bash
./tpc project.yml --dry
```

The output directory defaults to the project's `build`, but can be specified:

```bash
./tpc project.yml --dry --build-dir /tmp/typephp-build
```

## Common commands

```bash
# View help and version
./tpc --help
./tpc --version

# Release build
./tpc project.yml -O2 -j8

# Debug build
./tpc project.yml --debug

# Force recompilation, ignoring the cache
./tpc project.yml --force

# Generate and run immediately
./tpc project.yml --run
```

See [Command-line Options](options.md) for all options.

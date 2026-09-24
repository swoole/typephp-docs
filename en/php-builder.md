# PHP Builder and SAPI Targets

TypePHP treats the artifact type, PHP process interface, and PHP runtime source as three independent settings:

| Setting | Meaning | Values |
|---|---|---|
| `mode` | Artifact to generate | `bin`, `lib`, `ext` |
| `sapi` | PHP SAPI used by a `bin` executable | `embed`, `cli`, `fpm`; multiple values are allowed |
| `php-builder` | Whether to build a private PHP runtime from php-src | A mapping containing `extensions` and `zts` |

`cli` and `fpm` are not build modes. Like `embed`, they are SAPIs, and all three produce platform-native executables such as ELF or Mach-O files. `sapi` and `php-builder` apply only to `mode: bin`.

## Choosing a Configuration

| SAPI | Without `php-builder` | With `php-builder` | `entry` |
|---|---|---|---|
| `embed` | Links the host `libphp`, preserving the original build | Builds and statically links a private Embed runtime from php-src | Not used |
| `cli` | Unsupported | Builds a PHP CLI program containing the TypePHP project | Required |
| `fpm` | Unsupported | Builds a PHP-FPM program containing the TypePHP project | Not required |

`php-builder` currently supports Linux and macOS. Windows can use the Embed runtime supplied by the toolkit or host PHP, but cannot currently build a private SAPI runtime from source.

`php-builder` does not depend on the host PHP runtime, but still uses low-level development libraries supplied by the operating system. PHP, PHPX, and selected PHP extensions are linked into the artifact, while libc, libxml, zlib, OpenSSL, and similar libraries remain system dependencies. It is therefore different from `--full-static`, which requires a dedicated SDK.

## YAML Configuration

```yaml
name: myapp
mode: bin
php-version: 8.5
sapi: [embed, cli, fpm]
entry: bin/console.php

php-builder:
  extensions: [swoole, mongodb]
  zts: on

sources:
  - src
  - bin/console.php
```

Configuration rules:

- `sapi` may be a scalar such as `sapi: cli` or a list. Its default is `embed`.
- `php-builder` must be a mapping. Use `php-builder: {}` for defaults; a bare YAML key with no value is invalid.
- `extensions` defaults to `[]` and explicitly adds extensions to the private PHP build.
- `zts` defaults to `off`; explicitly writing `on` or `off` is recommended.
- `sapi` cannot be nested under `php-builder`; it is always a top-level setting.
- If `sapi` contains `cli`, `entry` is required. Setting `entry` without `cli` is an error.
- A YAML `entry` path is resolved relative to the directory containing `project.yml`.
- Command-line settings override corresponding YAML settings.

Legacy forms such as `mode: cli`, `mode: fpm`, and `php-builder.sapi` are not supported.

## Command-Line Configuration

A valueless `--php-builder` is equivalent to `{}`:

```bash
./tpc hello.php --php-builder
```

An explicit value uses the same fields as YAML, separated by semicolons. Quote the complete value so the shell does not interpret spaces or semicolons:

```bash
./tpc project.yml \
    --sapi=embed,cli,fpm \
    --entry=bin/console.php \
    --php-builder='extensions: [swoole, mongodb]; zts: on'
```

The following forms are equivalent:

```yaml
php-builder:
  extensions: [swoole, mongodb]
  zts: on
```

```bash
--php-builder='extensions: [swoole, mongodb]; zts: on'
```

Do not put `sapi` inside `--php-builder`. Use `--sapi=cli`, `--sapi=fpm`, or a comma-separated list independently.

## Embed SAPI

### Using the Host libphp

```yaml
mode: bin
sapi: embed
```

Because `embed` is the default SAPI, it may be omitted. The artifact links `libphp.so`, `libphp.dylib`, or the platform equivalent found through `PHP_HOME` or the current PHP installation. This preserves the original TypePHP binary behavior.

If the Embed library is missing on Linux or macOS, an interactive terminal offers to enable `php-builder`. Non-interactive environments such as CI must configure `--php-builder` explicitly or the build stops.

### Using a Private Static Embed Runtime

```yaml
mode: bin
sapi: embed
php-builder: {}
```

Equivalent command:

```bash
./tpc hello.php --sapi=embed --php-builder
```

This keeps the original Embed module registration and request lifecycle. It only replaces the host dynamic `libphp` with a static `libphp.a` built from php-src.

## CLI SAPI

The CLI target uses PHP's official CLI SAPI and statically includes the generated TypePHP module. `entry` is the PHP primary script executed by Zend VM at startup; other project `sources` continue through the TypePHP AOT pipeline.

```yaml
name: worker
mode: bin
sapi: cli
entry: bin/worker.php
php-builder:
  extensions: [pcntl, sockets]
  zts: off
sources:
  - src
  - bin/worker.php
```

Even if `entry` also appears under `sources`, the compiler removes it from the AOT source set and stores it as an embedded PHP script, preventing duplicate compilation. It may contain ordinary top-level PHP code and does not need the global `main()` used by an Embed-style program.

The AOT source and entry script may also be supplied separately on the command line:

```bash
./tpc src/functions.php \
    --sapi=cli \
    --entry=bin/worker.php \
    --php-builder
```

Arguments passed to the generated executable are forwarded to the embedded entry:

```bash
./worker --queue emails
```

## FPM SAPI

The FPM target links the TypePHP project into PHP-FPM as a built-in PHP module. It has no fixed CLI entry, so `entry` is not configured. Request scripts, pools, and listeners use the ordinary PHP-FPM configuration, and request scripts can call functions and classes exported by the TypePHP AOT module.

```yaml
name: myapp-fpm
mode: bin
sapi: fpm
php-builder:
  extensions: [pdo, pdo_mysql, opcache]
  zts: off
sources:
  - src
```

Start it like a normal PHP-FPM executable, for example:

```bash
./myapp-fpm --fpm-config /etc/myapp/php-fpm.conf
```

An FPM-only target cannot be used with `tpc --run`; start it separately after preparing its FPM configuration.

## Building Multiple SAPIs

```yaml
name: myapp
mode: bin
sapi: [embed, cli, fpm]
entry: bin/console.php
php-builder:
  extensions: []
  zts: on
```

A single SAPI uses the original output name. Multiple SAPIs receive suffixes:

| Configuration | Generated files |
|---|---|
| `sapi: cli`, `name: myapp` | `myapp` |
| `sapi: [embed, cli, fpm]`, `name: myapp` | `myapp-embed`, `myapp-cli`, `myapp-fpm` |
| Multiple SAPIs with `-o build/myapp.bin` | `build/myapp-embed.bin`, `build/myapp-cli.bin`, `build/myapp-fpm.bin` |

With multiple targets, `--run` runs the first `embed` or `cli` target in `sapi` order. FPM is never selected by `--run`.

## Extension Requirement Collection

`php-builder` merges and deduplicates requirements from:

1. Explicit `php-builder.extensions` entries;
2. YAML `extension-dependencies` or `ext-deps`;
3. Extensions owning internal PHP functions and classes actually referenced by AOT sources;
4. Composer `ext-*` requirements from projects included through `embedded-files`.

```yaml
php-builder:
  extensions: [swoole]
extension-dependencies:
  - curl
embedded-files:
  - vendor
```

Names are normalized: for example, `ext-curl` becomes `curl`, and `pdo_mysql` becomes `pdo-mysql`. Bundled php-src extensions are translated into their `--enable-*` or `--with-*` configure switches. Extensions absent from php-src are downloaded from the latest stable PECL release and injected into a derived source tree. The pristine cached php-src tree is not modified.

All PHP extensions are linked statically into the private runtime; the target does not load extension `.so` files from a host PHP installation. Native development libraries required by those extensions must still be provided by the operating system.

## PHP Version, ZTS, and Toolchain

- `php-version: 8.4` or `8.5` selects both the accepted syntax and the php-src release branch. The latest stable patch release is used on the first download; an existing complete cache is preferred.
- `zts: on` builds thread-safe PHP, while `zts: off` builds NTS. Changing ZTS selects a different runtime cache.
- `--compiler` also controls the PHP builder toolchain. GCC uses matching `gcc`/`g++`/`gcc-ar`; Clang uses `clang`/`clang++`/`llvm-ar`. GCC users are not required to install LLVM.
- `-j`/`--job` controls parallelism for both TypePHP and php-src compilation.

The build host needs a C/C++ compiler, `make`, CMake, pkg-config, and operating-system development libraries required by selected PHP and PECL extensions. External PECL extensions may additionally need generation tools such as autoconf.

## Cache and Build Logs

Downloads and builds are stored under `~/.typephp`:

```text
~/.typephp/
├── archives/          # PHP and PECL source archives
├── src/               # pristine official php-src trees
├── pecl/              # extracted PECL extensions
├── php-builder-src/   # derived source trees with external extensions
└── php-builder/       # built private runtimes
```

The suffix in a directory such as `php-8.5.10-e90c19d6ad02c94f` is a compatibility fingerprint. It includes the PHP patch version, configure arguments, SAPI set, ZTS mode, external extension versions, operating system, CPU architecture, and toolchain. It does not include the project name or path. PHP source is therefore not copied per project: compatible projects share runtimes, and an existing runtime whose extension set satisfies a project can be reused.

Each runtime directory uses a file lock to protect concurrent builds. Complete PHP build output is stored in `build/.typephp-build.log` inside that runtime. The terminal normally shows a progress bar and build stages; `--no-progress` switches to line-oriented CI output.

## Network Proxy

`--proxy` is a global network setting, not a `php-builder` field. PHP release metadata, php-src, PECL metadata and source archives, and other TypePHP network operations all use it:

```bash
./tpc project.yml \
    --proxy=socks5h://127.0.0.1:1080 \
    --php-builder='extensions: [mongodb]; zts: off'
```

HTTP(S) proxies are supported, and SOCKS proxies are supported when `curl` is
available. When the downloader invokes external `curl`, it explicitly forwards
the setting through `--proxy`.

## Common Errors

| Error | Correction |
|---|---|
| `` `php-builder` must be a mapping `` | Use `php-builder: {}` or a mapping containing `extensions`/`zts`. |
| `The cli and fpm SAPIs require php-builder` | Explicitly enable `php-builder` for CLI/FPM. |
| `` `sapi` containing cli requires an `entry` PHP file `` | Select an existing PHP entry file for CLI. |
| `` `entry` requires `sapi` to contain cli `` | Remove `entry`, or add `cli` to `sapi`. |
| `` `sapi` is only supported with `mode: bin` `` | `lib` and `ext` cannot configure `sapi`. |
| configure cannot find a header or library | Install the operating-system development package required by that extension and retry. |

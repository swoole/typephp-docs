# Reducing Nano and php-builder executable size

A TypePHP `bin` output contains application code and may also contain PHP Nano,
PHPX, or a static PHP runtime. First decide what to measure: `build/` contains
intermediate files and caches, not the release package. A macOS `.app` or Windows
release directory can also contain Qt, native libraries, and resources. Measure
both the executable and the complete release directory, then rerun the same
functional checks after each change.

| Build | Main size contributors | Check first |
|---|---|---|
| `--nano` | Selected Nano/PHPX sources, application code, native dependencies | Dynamic calls, unused sources and resources, optimization level |
| `--php-builder` | Static PHP/PHPX, extensions, application code, embedded files | SAPI, extensions, ZTS, `embedded-files` |

The quickest release steps are to run `strip` on a copy of the executable on
Linux/macOS, as shown below, and to run `composer install --no-dev` in a clean
release checkout before embedding `vendor`. This keeps development packages
out of the executable.

## Make a release build first

Do not ship with `-d` / `--debug`, `--sanitize`, or `--profile`. Debug mode
disables optimization and adds debugging information. TypePHP accepts `-O0`
through `-O3` and defaults to `-O0`. Try `-O1` first, then compare the size and
performance of `-O2`. A higher optimization level does not guarantee a smaller
file. `--lto` enables optimization across compilation units at link time, but
costs build time and should be measured too.

```bash
# Nano: compile only the needed sources; -o chooses the output file
./tpc app.php --nano -O1 --lto -o app-small

# Private PHP runtime: select only the default Embed SAPI
./tpc app.php --sapi=embed \
  --php-builder='extensions: []; zts: off' \
  -O1 --lto -o app-small
```

`--lto` applies to objects TypePHP compiles and links for this build; it does
not rebuild cached php-builder static PHP/PHPX archives with LTO. If the
toolchain or a native library cannot use LTO, remove `--lto` and compare again.
Nano already enables unused section elimination on each platform (Windows
`/OPT:REF /OPT:ICF`, Linux `--gc-sections`, macOS `-dead_strip`); there is no
need to add those linker flags yourself.

## Nano: reduce selected runtime features

Nano selects extensions and some standard-library features from the functions,
classes, and types actually used by the source. Review project `sources`, use
`ignore` to exclude source files that do not belong in the release, and remove
unneeded native link inputs. Dynamic function or method calls can make the
compiler retain a wider built-in Nano feature set; explicit call targets can
usually produce a narrower selection. Keep features the program actually needs.

Nano does not support `embedded-files`, so changes to that setting for a normal
build do not affect a Nano artifact. Nano also does not use `libphp`; the
`php-builder.extensions` and `php-builder.zts` settings do not apply.

## php-builder: control the runtime and resources

1. If the program only needs the Embed SAPI, use the default `sapi: embed`;
   do not request `cli` or `fpm` as well. Those targets have their own entry
   points and runtime code.
2. Use `zts: off` (the default) if thread-safe PHP is unnecessary. Keep
   `zts: on` when the application requires it.
3. List only needed extra extensions under `extensions`. TypePHP also merges
   requirements from project sources, YAML, and Composer metadata:
   **`extensions: []` does not disable every extension**. PHP core extensions
   and features always enabled by the builder remain present.
4. If you use `embedded-files`, run `composer install --no-dev` in a clean
   release checkout before compiling. This excludes `require-dev` packages
   from `vendor`. List only files required for the release; avoid embedding
   tests, docs, examples, and large resource trees. `ignore` affects `sources`
   but does not remove files from `embedded-files`. See
   [embedded files](embedded-files.md).

For example, when the release really needs the entire production `vendor`:

```yaml
embedded-files:
  - vendor
```

Run these commands in a clean release checkout:

```bash
composer install --no-dev
./tpc project-release.yml -O1
```

`--no-dev` controls the contents of `vendor`. To reduce size further, list
only the needed subdirectories in `embedded-files`. Keep packages required at
runtime.

For example, keep `pdo_sqlite` if the application uses SQLite:

```yaml
mode: bin
sapi: embed
optimize: 1
lto: true
php-builder:
  extensions: [pdo_sqlite]
  zts: off
```

php-builder reuses a compatible private runtime from `~/.typephp/php-builder/`
and may choose one with a superset of the requested extensions. If a build
reports reuse after you remove extensions, inspect `enabled_extensions` and
`configure_options` in that cached runtime's `runtime.json`. An unchanged file
size does not prove the new configuration had no effect. To measure a newly
minimal runtime, use a clean cache in an isolated build environment rather
than deleting a cache used by other builds. `-O1` / `--lto` do not change how
an existing cached PHP runtime was compiled.

## Remove unneeded symbols before release

Keep the original build for debugging. Strip a **copy**, then rerun the program:

Linux (GNU binutils):

```bash
cp app-small app-small.release
strip --strip-unneeded app-small.release
```

macOS (run on a Mac, before signing):

```bash
cp app-small app-small.release
xcrun strip -S -x app-small.release
```

For a macOS `.app`, strip the executable and bundled native libraries before
code signing; modifying a signed bundle invalidates its signature. With MSVC
on Windows, debugging information is primarily kept in a separate PDB. Ship
the required `.exe`, DLLs, and resources, and retain the PDB separately if you
need crash analysis. Do not run a Unix `strip` tool on MSVC output.

Finally measure both the executable and the release directory. On Linux/macOS,
use `ls -lh app-small.release` and `du -sh release/`; in Windows PowerShell,
use `(Get-Item .\app-small.exe).Length`. A smaller `.exe` does not imply a smaller
complete release when it depends on shared libraries. `--full-static` pulls
more runtime libraries into the executable and is generally not a way to
reduce the size of that individual file.

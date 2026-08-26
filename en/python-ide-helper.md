# Generating IDE Hints

`--gen-python-helper` is a built-in Python IDE helper-file generation tool in the `tpc` command entry, used during development to generate an editor helper file for a Python module so that calls like `python\math\sqrt()` get autocompletion. It does not participate in compiling the final program.

## What It Can Do for You

When you call Python modules using syntax like `python\math\sqrt()` or `numpy\zeros()`, the editor does not know by default where those symbols are, what their parameters are, or their return types. `--gen-python-helper` reads the runtime information of the target Python module and generates a PHP helper file that is **indexed only by the editor**, making autocompletion, parameter hints, and go-to-definition work correctly.

## Prerequisites

The generation process actually imports the specified Python module to collect symbols, so your development machine must meet the following:

- The PHP running `tpc` has the **phpy extension** installed and enabled; the generator reflects the Python module in the current process through phpy;
- The target **Python module is installed** in the same Python environment that runs `tpc` (for example, to generate a helper file for `numpy`, run `python3 -m pip install numpy` first);
- A usable `python3` exists.

## Usage

```bash
./tpc --gen-python-helper <Python module name> [--output-dir <directory>]
```

- `<Python module name>`: a dot-separated module path, for example `math`, `numpy.linalg`;
- `--output-dir <directory>` (optional): a custom output root directory, supporting relative or absolute paths; defaults to `ide-helper` in the current directory.

Examples:

```bash
./tpc --gen-python-helper math
./tpc --gen-python-helper numpy.linalg
./tpc --gen-python-helper numpy --output-dir .ide-helper
```

## What Files Are Generated

By default, the following are generated under `ide-helper/`:

```text
ide-helper/python/math.php
ide-helper/python/numpy/linalg.php
ide-helper/python.php
ide-helper/PyObject.php
```

- `python.php`: completions for Python builtin symbols (such as `python\len()`, `python\tuple()`), generated based on the current Python environment;
- `PyObject.php`: common type hints shared by all Python module helper files; written on first generation and never overwritten afterward.

## Notes

- Helper files are **only handed to the editor for indexing**; do not `include` them, and do not add them to a TypePHP project's `sources` or compilation inputs;
- The generator imports the target module, so the module's top-level initialization code and import side effects will actually run; do not run this command against untrusted modules;
- Each generation updates `python.php` and the target module file; an existing `PyObject.php` is kept, making it easy to maintain custom IDE declarations in the project;
- Helper file names and function names must conform to PHP naming rules, so a few Python reserved words (such as `print`, `list`, `int`, `float`) cannot produce symbol declarations that are free of syntax errors; the generator marks them with comments and does not rename them on its own.

## 1. Are Frameworks and Libraries Compiled

For frameworks and libraries in the `vendor` directory, it is recommended to still load them with `Composer Autoload` without compiling. Alternatively, use a whitelist configuration to compile only certain `vendor` subdirectories separately.

If individual `PHP` files in a `vendor` subdirectory do not support static compilation, you can also configure `ignore` to skip those files.

```yaml
name: thinkphp
type: ext
version: 0.0.1
cxxflags: |
  -std=c++14
  -Wall
sources:
  - ./src/think
  - ./vendor/topthink/think-helper/src
  - ./vendor/topthink/think-orm/src
  - ./vendor/topthink/think-container/src
ignore:
  - ./vendor/topthink/think-helper/src/functions.php
```

## 2. How to Package and Distribute
The TypePHP compiler differs from static compilation approaches like `Golang`. To reduce file size, it uses dynamic linking, so it depends on the operating system's `.so` libraries. Therefore, deploying and distributing the compiled binary requires following these rules:

1. The operating system must be identical, and the list of installed base libraries must remain consistent; this can be automated with OS package management tools such as `apt/yum/dnf`
2. Build `PHP` and `PHPX` on the build machine to produce `libphp.so` and `libphpx.so`, and save them into the package
3. All `PHP` extensions are compiled statically, compiled directly into `PHP` rather than dynamically loaded

The final distributed software package should contain the following:

- the compiled binary executable
- `libphp.so` and `libphpx.so`
- the open-source framework and library files in the `vendor/` directory

> If `PHP` extensions are dynamically loaded, their `.so` files need to be copied into the package

## 3. Can `__FILE__` and `__DIR__` Be Used

Because the TypePHP compiler determines the values of the `__FILE__` and `__DIR__` magic constants at the compilation stage, the final directory is the path of the `.php` file at compile time, not at runtime. Therefore, although the magic constants can be used, their result may differ from what is expected. It is recommended to use `getcwd()` or another configuration file approach to determine the final runtime directory.

## 4. Can Dynamic Classes Be Inherited

If a class in the project must inherit from a dynamic class defined by a file in the `vendor` directory, then all classes in the entire inheritance chain must be made statically compiled classes.

For example:
`App\Controller\TestController` inherits from `Framework\Controller`, and `Framework\Controller` in turn inherits from `Framework\BaseController`. Then these `3` classes must all be statically compiled classes.

## 5. Can Function Parameter Return Values and Class Properties Be Left Unannotated

Yes. Without a type annotation, the type automatically defaults to `any`. For example:
```php
function foo($a, $b, $c) {
}

class Bar {
    public $prop1;
    static public $prop2;
}
```

Types are not mandatory; the TypePHP compiler will automatically compile and generate instructions treating them as the `any` type. However, it is recommended to annotate the types of all functions and class properties, because explicit types allow the compiler to generate better-performing code.

## 6. Nullable, UnionType, and IntersectionType
Because `Nullable`, `UnionType`, and `IntersectionType` cannot be expanded into a single definite type at the static stage, the compiler treats them as the `any` type and cannot perform further performance optimization. Runtime type checks are still preserved, to avoid bypassing PHP type constraints.

When using these types, it is recommended to add `toObject()` type continuation in the logic chain, so that the compiler can restore the concrete object type and optimize subsequent calls. If you want to keep the dynamic behavior, use `toAny()` or `std::any()` to explicitly downgrade.

```php
function foo(): ?MyClass {
}

function bar(): MyClass1 | MyClass2 {
}

function baz(): ?MyClass {
}

function main() {
    $rs = foo();
    if ($rs == null) {
        // failure branch
    }
    // type continuation
    $o = $rs->toObject(MyClass::class);
    // generate native method call
    $o->someMethod();

    $rs = bar();
    if ($rs instanceof MyClass1) {
        $o1 = $rs->toObject(MyClass1::class);
        $o1->someMethod();
    } elseif ($rs instanceof MyClass2 ) {
        $o2 = $rs->toObject(MyClass2::class);
        $o2->someMethod();
    }

    $rs = baz();
    if ($rs !== null) {
        $o = $rs->toObject(MyClass::class);
        $o->someMethod();
    }
}
```

## 7. Prefer to Avoid `readonly` and Property Hooks

Both `readonly` and Property Hooks are implemented entirely at runtime. Using either one prevents the compiler from applying any optimization to the property access, and requires an extra object-level hook to be inserted. In access-heavy code this is more than an order of magnitude slower than a plain property.

**Instead of `readonly`:** use a `private` property. It is equally inaccessible from outside the class, but keeps full optimization ability. Add `#[Getter]` when read-only access still needs to be exposed.

```php
class Config
{
    #[Getter]
    private int $port;

    public function total(int $n): int
    {
        $t = 0;
        for ($i = 0; $i < $n; $i++) {
            $t += $this->port;   // access directly, not getValue()
        }
        return $t;
    }
}
```

**Instead of Property Hooks:** access the backing property directly inside the class, and reserve the hook for the public interface.

```php
class Order
{
    public int $totalFen = 0;

    public int $totalYuan {
        get {
            return intdiv($this->totalFen, 100);
        }
    }

    public function sumFen(int $n): int
    {
        $t = 0;
        for ($i = 0; $i < $n; $i++) {
            $t += $this->totalFen;   // not $this->totalYuan
        }
        return $t;
    }
}
```

> Rewriting a Property Hook as a plain method brings no performance gain, since both are function calls; the same applies to a method generated by `#[Getter]`. The real distinction is between "property access" and "method call". Both features can be used normally on non-hot paths.

## 8. Avoid `global` Variables

A `global` variable is typed `var`. The compiler cannot prove its concrete type, so a method call on it degrades to a runtime lookup by method name, losing the `Native Call` optimization and requiring an extra runtime type check.

Pass the object as a typed parameter instead:

```php
function handle(User $user): void
{
    $user->save();   // concrete type — native call
}
```

instead of:

```php
function handle(): void
{
    global $user;
    $user->save();   // $user is var — dynamic call
}
```

# Hot

`#[Hot]` can be used on functions and methods to hint to the C++ compiler that this function belongs to a hot execution path of the program and should prioritize execution speed.

```php
namespace App;

use \Hot;

#[Hot]
function decodePacket(string $packet): array
{
    // hot path
    return [];
}

class Router
{
    #[Hot]
    public function dispatch(string $path): void
    {
        // hot path
    }
}
```

`Hot` is a purely compile-time annotation. At runtime the program does not read the Attribute, does not use reflection, and does not add dynamic invocation; TypePHP converts it directly into an optimization marker on the underlying C++ function, so the annotation mechanism itself has zero runtime overhead.

On GCC and Clang it maps to `__attribute__((hot))`. The compiler can use this to optimize the function more aggressively and adjust code layout according to the toolchain and target platform. Other C++ compilers that do not support this capability safely ignore the hint.

## When to Use

It suits marking core paths that are confirmed by profiling to be called very frequently, such as request dispatch, protocol parsing, serialization, or numerical computation. It is only an optimization hint; it does not change PHP semantics, nor does it guarantee that the function is inlined.

It is not recommended to add `Hot` in large numbers based on guesswork. A wrong hint may increase code size, or even undermine the compiler's own more accurate optimization decisions. Prefer using profiling data to decide where to place the marker.

Even when LTO is not enabled, the compiler can still use this hint when compiling the `.cc` file that contains the function's implementation; but further optimization of cross-file call sites still depends on LTO, the compiler, and the linker.

`Hot` cannot be used together with [`Cold`](cold.md) on the same function or method, otherwise compilation fails.

## Library Projects

When compiling with `-m lib`, `Hot` is preserved in the automatically generated library stub. The library implementation itself accepts the optimization hint at build time; other TypePHP projects using the stub need no runtime processing.

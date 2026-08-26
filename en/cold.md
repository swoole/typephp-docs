# Cold

`#[Cold]` can be used on functions and methods to hint to the C++ compiler that this function is normally executed rarely and should be handled as a cold path, leaving optimization resources to the normal path first.

```php
namespace App;

use \Cold;

#[Cold]
function reportProtocolError(string $message): void
{
    throw new \ValueError($message);
}

class Parser
{
    #[Cold]
    public function invalidInput(string $input): void
    {
        throw new \ValueError($input);
    }
}
```

`Cold` is a purely compile-time annotation. At runtime the program does not read the Attribute, does not use reflection, and does not add dynamic invocation; TypePHP converts it directly into an optimization marker on the underlying C++ function, so the annotation mechanism itself has zero runtime overhead.

On GCC and Clang it maps to `__attribute__((cold))`. The compiler can use this to favor reducing the code size of the function, and to lay out cold code separately from hot paths according to the toolchain and target platform. Other C++ compilers that do not support this capability safely ignore the hint.

## When to Use

It suits error handling, exception construction, diagnostic output, and fallback paths that are theoretically rarely entered. It is only an optimization hint; it does not make the function uncallable, nor does it change exception or return value semantics.

Do not equate "lower business importance" with low execution frequency. Only functions that are genuinely executed rarely are suitable for `Cold`; wrongly marking a hot function may noticeably degrade performance.

Even when LTO is not enabled, the compiler can still use this hint when compiling the `.cc` file that contains the function's implementation; how much information cross-file call sites can use depends on LTO, the compiler, and the linker.

`Cold` cannot be used together with [`Hot`](hot.md) on the same function or method, otherwise compilation fails.

## Library Projects

When compiling with `-m lib`, `Cold` is preserved in the automatically generated library stub. The library implementation itself accepts the optimization hint at build time; other TypePHP projects using the stub need no runtime processing.

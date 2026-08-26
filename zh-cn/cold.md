# Cold

`#[Cold]` 可用于函数和方法，用来提示 C++ 编译器：这个函数通常很少执行，应按低频路径处理，将优化资源优先留给正常路径。

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

`Cold` 是纯编译期注解。程序运行时不会读取 Attribute、不会使用反射，也不会增加动态调用；TypePHP 会把它直接转换为底层 C++ 函数的优化标记，因此注解机制本身是零运行时开销。

在 GCC 和 Clang 中，它会映射为 `__attribute__((cold))`。编译器可据此偏向减小该函数的代码体积，并根据工具链和目标平台将低频代码与高频路径分开布局。其他不支持该能力的 C++ 编译器会安全地忽略此提示。

## 何时使用

适合错误处理、异常构造、诊断输出和理论上很少进入的回退路径。它只是优化提示，不会使函数无法调用，也不会改变异常或返回值语义。

不要把业务上“重要性较低”等同于执行频率低。只有确实很少执行的函数才适合 `Cold`；错误标记高频函数可能明显降低性能。

即使未启用 LTO，编译器仍能在编译该函数实现的 `.cc` 文件时使用此提示；跨文件调用点能够利用多少信息，则取决于 LTO、编译器和链接器。

`Cold` 不能与 [`Hot`](hot.md) 同时用于同一个函数或方法，否则编译会失败。

## Library 项目

以 `-m lib` 编译时，`Cold` 会保留在自动生成的 library stub 中。库实现本身在构建时接受该优化提示；使用该 stub 的其他 TypePHP 项目不需要进行运行时处理。

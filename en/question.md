## Is the TypePHP Compiler Compatible with the 3.2 Encryptor

Compatible. The TypePHP compiler and the `3.2` encryptor can be used at the same time. Files that support static compilation generate binary native instructions, while files containing unsupported syntax can still be dynamically loaded via `ZendVM`, using the `3.2` encryptor to encrypt the `PHP` code.

## Is the TypePHP Compiler Open Source and Free

Yes. The TypePHP compiler is open-source software, released under the **GPL** open-source license. It is completely free and can be freely used, modified, and redistributed, including for commercial use. Whether you are an individual developer, an enterprise, a government agency, a public institution, a school, or a non-profit organization, you can use the TypePHP compiler free of charge without purchasing any license.

## Differences Between `Swoole Compiler` and `TypePHP`

Swoole Compiler is commercial software that users must purchase before use. `Swoole Compiler` focuses on protecting the source code of `PHP` projects. It is more suitable for `PHP` programs in traditional `PHP-FPM` or `PHP-CLI` modes such as `Swoole`/`Swow`/`Workerman`; users do not need to modify their existing `PHP` code.

TypePHP is an open-source project. All users can use it free of charge. It is a brand-new `PHP` compilation technology. Unlike traditional `PHP` interpretation and execution, TypePHP compiles `PHP` source code directly into binary executable programs.
Existing `PHP` code cannot run directly under TypePHP; it needs to be modified according to the compiler's information before use.

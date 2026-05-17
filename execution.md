`AOT`编译器相比`ZendPHP`最大的优势就是性能。

## ZendVM 的执行方式
`ZendVM`执行`PHP`代码分为`5`个步骤：

### 1. 词法分析 (`Lexical Analysis`)

将`PHP`代码拆分成许多`Token`词元，使用`Re2c`工具实现。在`PHP`代码可以使用`token_get_all()`模拟这一过程。

```c
// 简化的Token输出
T_OPEN_TAG, T_VARIABLE, '=', T_CONSTANT_ENCAPSED_STRING, ';',
T_ECHO, T_CONSTANT_ENCAPSED_STRING, ';', T_CLOSE_TAG
```

### 2. 语法分析（`Syntax Analysis`）

使用**Bison**解析器将Token流转换为**抽象语法树(AST)**：

```c
// AST节点结构 (Zend/zend_ast.h)
struct _zend_ast {
    zend_ast_kind kind;  // 节点类型
    zend_attr attr;      // 属性
    uint32_t lineno;     // 行号
    zend_ast_child children[1];  // 子节点
};
```

```text
AST_STMT_LIST
├── AST_ASSIGN
│   ├── AST_VAR (name: "name")
│   └── AST_ZVAL ("World")
└── AST_ECHO
    └── AST_STR ("Hello, World!")
```

### 3. **AST编译为OPCode**

遍历AST，生成**OPArray**（操作码数组），字节码。


### 4. **执行Opcode**

`ZendVM`的核心执行器位于`Zend/zend_vm_execute.h`。在`PHP`的实现中这是一个大循环，每个`opcode`都对应一个 `C函数` 实现的 `opcode handler`。

## AOT 编译器的执行方式
AOT 编译器的执行分为`4`个步骤
- 预处理：遍历整个工程目录，找出`PHP`文件和`C/C++`源代码、`ASM`汇编代码，在`Clang + macOS`平台还会自动发现`Objective-C`文件
- 转换`C++`代码：将所有`PHP`文件，经过`php-parser`处理生成`AST`后，转为`C++`代码
- 编译源代码：使用`gcc/clang/msvc`等`C/C++`编译器将源文件编译为`.o`目标文件
- 连接：将所有`.o`文件连接在一起，生成可执行文件（`ELF/PE`）或动态链接库`so/dll`文件


与`ZendVM`的执行相比，`AOT`编译后所有`PHP`代码将从`Opcode`字节码转为机器指令，执行的效率得到了大幅增强，但编译后丧失了动态性，不允许动态改变`Opcode`字节码或者动态插入新的执行指令。
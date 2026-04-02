# QT

## 1 #    define Q_DECL_IMPORT     __attribute__((visibility("default")))表示什么意思

`#define Q_DECL_IMPORT __attribute__((visibility("default")))` 是 **Qt 库中用于声明符号导入的宏**，专门用于 **GCC / Clang 编译器** 下的动态库（共享库）符号可见性控制。

------

### 1. 各部分的含义

### `__attribute__((visibility("default")))`

- 这是 **GCC / Clang** 提供的一个扩展属性，用于控制 **动态库中符号（函数、变量、类）的可见性**。
- 符号可见性决定了该符号在最终生成的共享库（`.so`）中是否可以被外部模块（即链接该库的程序或其他库）访问。
- `visibility("default")` 表示该符号具有 **默认可见性**：
  - 符号在动态库中 **对外可见**（即导出），其他模块可以链接并调用它。
  - 这是最宽松的可见性级别，相当于传统意义上的“公开符号”。

其他常见的可见性级别包括：

- `hidden`：符号在库内部可见，但对外部模块 **不可见**（类似 `static` 但作用域为动态库边界）。这可以减小符号表体积、提高加载速度、防止符号冲突。
- `internal`：类似 `hidden`，但更严格（通常很少使用）。
- `protected`：符号对外可见，但不可被覆盖（少见）。

### `Q_DECL_IMPORT`

- 这是 Qt 内部定义的一个宏，用于 **声明符号是从某个共享库中导入的**。
- 当你在 **使用** Qt 库（或其他动态库）时，在头文件中用 `Q_DECL_IMPORT` 标记一个类或函数，告诉编译器：该符号的定义在另一个共享库中，需要在链接时从该库导入。
- 对应的还有 `Q_DECL_EXPORT`，用于 **导出** 符号（即定义在库内部，提供给外部使用）。

### 整个宏定义

cpp

```
#define Q_DECL_IMPORT __attribute__((visibility("default")))
```



意味着：在使用 GCC/Clang 编译时，`Q_DECL_IMPORT` 被替换为 `__attribute__((visibility("default")))`，从而将标记的符号设置为默认可见性（即从外部可见，可以导入）。

------

### 2. 为什么需要这个宏？

**跨平台动态库符号管理**：

- Windows 上，动态库导入/导出需要使用 `__declspec(dllimport)` / `__declspec(dllexport)`。
- Linux / macOS 上（GCC/Clang），则使用 `__attribute__((visibility(...)))` 来控制。
- Qt 通过定义 `Q_DECL_IMPORT` 和 `Q_DECL_EXPORT` 宏，根据平台自动选择正确语法，让开发者只需使用 `Q_DECL_EXPORT` 导出、`Q_DECL_IMPORT` 导入，无需关心底层编译器差异。

**典型用法**（Qt 头文件中）：

cpp

```
#ifdef Q_OS_WIN
#  define Q_DECL_EXPORT __declspec(dllexport)
#  define Q_DECL_IMPORT __declspec(dllimport)
#else
#  define Q_DECL_EXPORT __attribute__((visibility("default")))
#  define Q_DECL_IMPORT __attribute__((visibility("default")))
#endif
```



**在类声明中使用**：

cpp

```
class Q_DECL_EXPORT MyClass { ... };   // 导出 MyClass（库内部定义）
class Q_DECL_IMPORT MyClass { ... };   // 导入 MyClass（库外部使用）
```



------

### 3. 对程序行为的影响

- 当编译一个共享库时，用 `Q_DECL_EXPORT` 标记的符号会被导出，对用户可见。
- 当链接该共享库时，用 `Q_DECL_IMPORT` 标记的符号会告诉编译器：“这个符号不在当前目标文件中，而是在某个动态库中，请生成重定位信息，允许运行时动态链接。”
- 如果没有正确使用可见性属性，可能导致：
  - **符号隐藏**：库内部符号意外暴露（污染全局命名空间），或需要暴露的符号被隐藏（链接时找不到符号）。
  - **性能损失**：默认情况下 GCC 会导出所有符号（类似于 `default`），导致动态库体积增大、加载变慢。通过合理设置 `hidden` 可以优化。

------

### 4. 总结

- `Q_DECL_IMPORT` 是 Qt 定义的跨平台宏，在 Linux/macOS 上展开为 `__attribute__((visibility("default")))`。
- 它的作用是 **声明一个符号是从外部共享库导入的**，并确保该符号在动态库边界具有默认可见性（对外可见）。
- 与之配对的是 `Q_DECL_EXPORT`，用于导出库内部的符号。
- 这种机制是动态库编程的基础，也是 Qt 实现模块化、插件化架构的关键技术之一。
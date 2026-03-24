# C#知识点

## 1	为什么要使用unsafe

🔒 C# 默认是“安全”的

C# 是一个 **托管语言**，默认：

* 不允许直接操作内存地址
* 不允许用指针（比如 `int*`、`byte*`）
* 所有的对象都在 CLR（.NET运行时）里受到垃圾回收（GC）管理

这就保证了：

* 内存安全（不会越界访问）
* 自动内存管理（不会内存泄漏）

------

🧨 什么时候需要 `unsafe`？

> 简单说：**你要自己操作内存或优化性能时，就需要 `unsafe`。**

✅ 常见场景：

| 场景                                                  | 原因                                                     |
| ----------------------------------------------------- | -------------------------------------------------------- |
| 🚀 **图像处理 / 性能优化**                             | 用指针直接访问 `BitmapData` 比逐像素 `GetPixel` 快几百倍 |
| 📸 **调用 C/C++ DLL（Interop）**                       | 接口返回 `IntPtr`，你需要 `byte*` 或 `结构体*` 来读内存  |
| 💾 **大数据处理 / 缓冲区操作**                         | 操作 byte 数组效率更高，不需要封装                       |
| 📦 **自定义内存管理（比如用 `Marshal.AllocHGlobal`）** | 需要手动释放、操作指针                                   |

------

示例：图像处理性能对比

使用 `GetPixel`（托管）

```
csharp


复制编辑
Color color = bmp.GetPixel(x, y);  // 非常慢
```

使用 `unsafe` + 指针（非托管）

```
csharp复制编辑unsafe
{
    byte* ptr = (byte*)scan0.ToPointer();
    byte value = ptr[y * stride + x];  // 快很多
}
```

------

❗ 使用 `unsafe` 的代价

| 风险                                          | 描述 |
| --------------------------------------------- | ---- |
| 🧨 你要自己确保不越界访问                      |      |
| 🧹 GC 不会帮你管理内存了（如果你用非托管内存） |      |
| 🔧 程序崩了自己负责（没有 CLR 帮你兜底）       |      |

------

✅ 总结一句话：

> **只有当你需要极致性能、直接操作内存或与底层系统交互时，才应该使用 `unsafe`。否则就让 C# 替你做安全检查。**

## 2	内存泄漏是什么

内存泄漏（Memory Leak）指的是：

> **程序运行时申请了内存，但用完后没有释放，导致这块内存一直被占用，不能再使用。**

------

🔍 举个很直观的例子：

你开了一家快餐店，有 100 个餐盘。

* 顾客来了，你借给他一个餐盘（分配内存）。
* 吃完饭后，他没有还你盘子（没释放内存）。
* 时间久了，盘子全被拿走，你就没法接待新顾客了（内存耗尽，程序崩溃）。

------

📦 在 C# 中内存有两种：

| 类型                                                         | 由谁管理                        | 是否会内存泄漏             |
| ------------------------------------------------------------ | ------------------------------- | -------------------------- |
| **托管内存**（比如 `new` 出来的对象）                        | .NET CLR（垃圾回收 GC）自动释放 | 通常不会，只要没有引用残留 |
| **非托管内存**（比如 `Marshal.AllocHGlobal()`、C++ DLL 返回的 `IntPtr`） | 你自己负责释放                  | **容易泄漏！❗**            |

------

✅ C# 中最常见的内存泄漏场景

| 场景                                    | 是否泄漏 | 说明                                |
| --------------------------------------- | -------- | ----------------------------------- |
| `new Bitmap()` 后没有 Dispose           | ✅ 有泄漏 | GDI 对象，必须调用 `Dispose()` 释放 |
| 事件没解除绑定（事件源存活）            | ✅ 有泄漏 | GC 认为你还在被使用，不会回收       |
| 非托管内存（IntPtr）没手动释放          | ✅ 有泄漏 | GC 不管这块内存，你得自己 `Free`    |
| 长时间缓存引用（比如 `static List<T>`） | ✅ 有泄漏 | 对象引用一直存在，GC 不会回收       |

------

🧠 检查是否内存泄漏的方法

* 使用 **任务管理器** 观察内存是否持续上涨
* 使用 Visual Studio 的 **诊断工具**
* 使用专业工具如 **dotMemory、ANTS Memory Profiler**
* 打 log 监测引用和 GC 回收情况

------

🛠 如何避免内存泄漏？

| 做法                                                     | 说明 |
| -------------------------------------------------------- | ---- |
| ✅ 使用 `using` 语句释放对象（如 `Bitmap`, `Stream`）     |      |
| ✅ 非托管内存使用后要记得释放（如 `Marshal.FreeHGlobal`） |      |
| ✅ 使用 `WeakReference` 避免长时间强引用                  |      |
| ✅ 事件订阅要记得解绑（特别是 `static` 对象订阅实例事件） |      |
| ✅ 大对象不用时置为 `null`，或手动触发 GC（少用）         |      |

------

✅ 总结一句话：

> 内存泄漏 = 拿了内存却不还。
>  在 C# 里如果你接触了 **非托管内存**、**图像处理**、**第三方 DLL**，就必须格外小心，手动释放。

## 3	Using的作用

`using` 主要有 **两种用法**，但核心都是为了 **资源管理** 和 **提高代码可读性**。

------

✨ 第一种用法：**引用命名空间**

比如：

```
csharp复制编辑using System;
using System.Collections.Generic;
```

🔵 作用：

* 告诉编译器去找 `System` 命名空间下的类型（比如 `Console`、`List<>`）。
* 否则每次用类都要写完整名字，例如 `System.Console.WriteLine(...)`，很麻烦。

简化开发，让代码更清爽。

------

✨ 第二种用法：**释放资源（IDisposable接口对象）**

比如：

```
csharp复制编辑using (var file = new System.IO.StreamWriter("test.txt"))
{
    file.WriteLine("Hello World");
}
// 出了using块，file对象自动调用 Dispose() 方法，释放文件句柄资源
```

🔵 作用：

* 自动在**作用域结束时释放资源**。
* 避免**内存泄漏**、**资源占用**（比如文件未关闭、数据库连接未释放等问题）。
* 所有实现了 `IDisposable` 接口的对象（比如文件流、数据库连接、图像对象等）都可以用 `using` 包裹。

**本质**就是帮你自动调用：

```
csharp


复制编辑
file.Dispose();
```

------

🔥 通俗总结一句话

> `using`就是为了让你**用完资源马上释放，不用自己手动调用释放函数**，防止资源泄漏。

------

🛠 实际例子（资源释放）

如果不用 `using`，你得自己写：

```
csharp复制编辑var file = new System.IO.StreamWriter("test.txt");
try
{
    file.WriteLine("Hello World");
}
finally
{
    file.Dispose(); // 手动释放
}
```

而用 `using`：

```
csharp复制编辑using (var file = new System.IO.StreamWriter("test.txt"))
{
    file.WriteLine("Hello World");
}
// 作用域结束，自动释放
```

更简单、更安全，推荐！

------

⚡总结

| 用法             | 目的             |
| ---------------- | ---------------- |
| `using 命名空间` | 引用外部命名空间 |
| `using (对象)`   | 自动释放对象资源 |

## 4	 C#中哪些对象需要主动调用dispose

我来系统地告诉你：在C#里，**所有实现了 `IDisposable` 接口的对象**，都应该在用完后主动调用 `.Dispose()`，或者用 `using` 自动处理。

------

🎯 需要主动调用 `Dispose` 的常见对象类别

| 类型                       | 具体例子                                       | 原因                                     |
| -------------------------- | ---------------------------------------------- | ---------------------------------------- |
| **文件资源**               | `FileStream`, `StreamReader`, `StreamWriter`   | 文件打开后占用系统文件句柄，需要释放     |
| **数据库连接**             | `SqlConnection`, `SqlCommand`, `SqlDataReader` | 数据库连接资源很宝贵，必须及时释放       |
| **网络资源**               | `HttpClient`, `TcpClient`, `Socket`            | 网络连接/端口占用，需要关闭连接          |
| **图形资源**               | `Bitmap`, `Image`, `Graphics`                  | GDI+对象，占用大量系统内存               |
| **内存流**                 | `MemoryStream`, `BufferedStream`               | 内存或缓冲区需要手动回收                 |
| **计时器**                 | `Timer`（如 `System.Timers.Timer`）            | 定时器内部起线程，释放可以避免内存泄漏   |
| **其他非托管资源封装对象** | `SafeHandle`, `Mutex`, `Semaphore` 等          | 涉及系统资源或线程同步控制，需要主动释放 |

------

⚡ 为什么？

* **垃圾回收器（GC）不会自动回收非托管资源**，比如文件句柄、网络连接、系统绘图对象等。
* 这些资源如果不及时释放，会导致：
  * 内存泄漏
  * 文件被锁住无法访问
  * 网络端口一直占用
  * 应用程序崩溃

------

🛠 怎么用？

有两种方法：

1. ✅ 用 `using` 自动释放（推荐）：

```
csharp复制编辑using (var stream = new FileStream("path.txt", FileMode.Open))
{
    // 使用 stream
} 
// 出作用域自动调用 Dispose()
```

1. ✅ 或者手动调用 `.Dispose()`：

```
csharp复制编辑var stream = new FileStream("path.txt", FileMode.Open);
// 使用 stream
stream.Dispose(); // 手动释放
```

------

🎯 额外提醒

* 有些对象，比如 `HttpClient`，可以复用，不建议频繁 `Dispose`。
   特殊场景（比如短时间大量创建）还是要注意释放。
* 如果忘记释放对象，有时候需要 `using` + `try-catch`保护，确保即使出错也能释放资源。
* 如果自己写类，**如果类内部持有需要释放的资源，也要让自己的类实现 `IDisposable`。**

------

🚀 总结口诀

> “**用完的流、图、网、连，记得Dispose，不要等GC！**”

## 5	ex.Message和ex.ToString的区别

`ex.Message` 和 `ex.ToString()` 都是用来获取异常（Exception）信息的，但它们之间有明显区别：

------

✅ `ex.Message`

* 返回异常的简要描述（通常是一行文字）。
* 适合用户提示或日志中简短说明。

**示例：**

```
csharp复制编辑try
{
    int x = int.Parse("abc");
}
catch (Exception ex)
{
    Console.WriteLine(ex.Message);
}
```

**输出：**

```
css


复制编辑
Input string was not in a correct format.
```

------

✅ `ex.ToString()`

* 返回异常的**完整信息**，包括：
  * 异常类型（如 `System.FormatException`）
  * `Message`
  * `StackTrace`
  * 以及可能的 `InnerException`
* 更适合调试或写入详细日志。

**示例：**

```
csharp复制编辑try
{
    int x = int.Parse("abc");
}
catch (Exception ex)
{
    Console.WriteLine(ex.ToString());
}
```

**输出（示例）：**

```
css复制编辑System.FormatException: Input string was not in a correct format.
   at System.Number.StringToNumber(...)
   at System.Int32.Parse(...)
```

------

🆚 总结对比：

| 用法            | 内容                            | 场景               |
| --------------- | ------------------------------- | ------------------ |
| `ex.Message`    | 只返回异常的说明                | 用户提示、简洁日志 |
| `ex.ToString()` | 类型 + 消息 + 堆栈 + 内部异常等 | 调试、详细日志记录 |

## 6	启动线程的方法有哪些

在 C# 中，要启动线程并在后台运行方法，有多种方式可以实现。以下是 **几种常用的方式**，包括手动线程、线程池、`Task`、`async/await` 等，并说明是否是**后台线程**：

------

✅ 1. 使用 `Thread` 类（可以设置为后台线程）

```
Thread thread = new Thread(() =>
{
    // 后台运行的方法
    DoWork();
});
thread.IsBackground = true;  // 设置为后台线程
thread.Start();
```

> ✅ 控制灵活，可设置优先级、后台等
>  ⚠️ 不推荐用于大量并发任务（每个线程开销较大）

------

✅ 2. 使用 `ThreadPool`（默认是后台线程）

```
ThreadPool.QueueUserWorkItem(state =>
{
    DoWork();
});
```

> ✅ 简单高效，线程复用
>  ❌ 无法设置线程名称、优先级，也不能轻松获取返回值

------

✅ 3. 使用 `Task`（线程池，默认后台线程）

```
Task.Run(() =>
{
    DoWork();
});
```

> ✅ 推荐方式，可配合 `async/await`，支持返回值、异常处理
>  ✅ 适合 CPU 密集型或异步 IO 场景

------

✅ 4. 使用 `async/await` 异步方法（通常结合 `Task.Run`）

```
public async Task StartBackgroundAsync()
{
    await Task.Run(() =>
    {
        DoWork();
    });
}
```

> ✅ 简洁、安全，支持取消、超时等高级控制
>  ⚠️ 适合异步操作，不是真正并发线程

------

✅ 5. 使用 `BackgroundWorker`（WinForms/WPF）

```
BackgroundWorker worker = new BackgroundWorker();
worker.DoWork += (s, e) => DoWork();
worker.RunWorkerAsync();
```

> ✅ 支持进度、完成事件，适合 UI 应用
>  ❌ 已较老，不推荐新项目使用

------

🔁 小结对比表：

| 方式                     | 后台线程 | 支持返回值 | 控制性 | 推荐用途                 |
| ------------------------ | -------- | ---------- | ------ | ------------------------ |
| `Thread`                 | ✅ 可设置 | ❌          | 高     | 需要精细控制的任务       |
| `ThreadPool`             | ✅ 默认   | ❌          | 中     | 简单短任务               |
| `Task.Run`               | ✅ 默认   | ✅          | 高     | 推荐使用，大多数场景适用 |
| `async/await + Task.Run` | ✅ 默认   | ✅          | 高     | 异步/并发、UI友好        |
| `BackgroundWorker`       | ✅ 默认   | ❌          | 中     | WinForms 老项目使用      |

------

备注：

* **后台线程（`IsBackground = true`）**：主程序退出时自动结束，不阻止进程退出；
* **前台线程**：主程序必须等它们结束才会退出。

## 7	什么是闭包引用

在 C# 中，“**闭包引用**”是指 **lambda 表达式或匿名方法捕获了其作用域外的变量**，使得这些变量即使在其原作用域已结束时仍然可以被访问和修改。

------

🔹 什么是闭包（Closure）

闭包是一个函数（通常是 lambda 或匿名方法），它**捕获并“闭住”了所在作用域中的变量**，即便这个作用域已经结束了，这些变量依然存在于内存中。

------

📌 示例

```
csharp复制编辑List<Action> actions = new List<Action>();

for (int i = 0; i < 3; i++)
{
    actions.Add(() => Console.WriteLine(i));
}

foreach (var action in actions)
    action(); // 输出：3 3 3（而不是 0 1 2）
```

🔍 解释

* `() => Console.WriteLine(i)` 是一个闭包，**捕获了变量 `i`**。
* 当循环结束后，`i == 3`，所以三个 lambda 都打印的是 3。
* **这里 `i` 不是值复制，而是被引用（闭包引用）了**，lambda 访问的是 `i` 的**原始变量地址**。

------

✅ 正确写法（避免共享引用）

```
csharp复制编辑for (int i = 0; i < 3; i++)
{
    int copy = i; // 每次创建副本，避免闭包捕获同一个变量
    actions.Add(() => Console.WriteLine(copy));
}
```

输出就变成了：`0 1 2`。

------

🔸 闭包的作用

* **延长局部变量生命周期**：变量不会在作用域结束后销毁，因为被 lambda 捕获。
* **实现回调、异步任务逻辑**。

------

🔺 闭包的常见“陷阱”

* **在线程或异步方法中使用循环变量**；
* **lambda 中访问外部变量导致值不一致**；
* **可能导致内存泄漏**，因为闭包引用阻止 GC 回收原作用域对象。

------

✅ 应用示例

```
csharp复制编辑public static Func<int> Counter()
{
    int count = 0;
    return () => ++count;  // 闭包捕获了 count
}

var counter = Counter();
Console.WriteLine(counter()); // 1
Console.WriteLine(counter()); // 2
```

------

总结

| 特性     | 说明                               |
| -------- | ---------------------------------- |
| 闭包引用 | lambda 表达式捕获外部变量          |
| 持续有效 | 即使作用域结束，变量仍保留         |
| 易错点   | 循环中使用 lambda 捕获变量         |
| 实用场景 | 回调、延迟执行、计数器、状态保存等 |

## 8	dock fill时会抢占其他dock区域怎么解决

在 WinForms 中，`Dock = Fill` 会让控件占满它所在的容器（`Panel`、`Form`、`TableLayoutPanel` 的单元格 等）剩余的所有可用空间——这就会“抢占”掉其他已经 Dock 在同一容器里的控件。

要解决这个问题，关键在于 **控制各个控件的添加顺序**（或者显示顺序），以及在必要时 **使用多个容器** 来分区布局。下面给出几种思路：

------

1. 调整控件的添加/停靠顺序

WinForms 的 `Dock` 停靠是按**控制添加到父容器的先后顺序**来计算的：

* 先添加并停靠 `Top`、`Left`、`Right`、`Bottom` 的控件
* 最后添加并停靠 `Fill` 的控件

如果你的 `Fill` 先添加，后加入的 `Top`/`Left` 即使 `Dock` 了也会被 `Fill` 覆盖。

**解决办法**：在设计器中或代码里，确保你先添加（或先初始化）所有非-Fill 的控件，再最后添加那个要 `Dock = Fill` 的控件。

```
csharp复制编辑// 伪代码示例
panel.Controls.Add(topControl);
topControl.Dock = DockStyle.Top;

panel.Controls.Add(leftControl);
leftControl.Dock = DockStyle.Left;

// 最后再添加并填充
panel.Controls.Add(fillControl);
fillControl.Dock = DockStyle.Fill;
```

------

2. 使用 `BringToFront` / `SendToBack`

如果在 Designer 里顺序不易调整，你也可以在运行时改变 Z-Order：

```
csharp复制编辑// 先停靠
topControl.Dock = DockStyle.Top;
leftControl.Dock = DockStyle.Left;

// 停靠 Fill
fillControl.Dock = DockStyle.Fill;

// 确保 Fill 在底层
fillControl.SendToBack();
// 或者让其它停靠控件浮在上层
topControl.BringToFront();
leftControl.BringToFront();
```

这样 `Fill` 控件仍然占据剩余空间，但不会遮挡 `Top/Left` 等控件。

------

3. 分区使用多个容器

如果布局比较复杂，推荐**分割面板（`SplitContainer`）\**或\**嵌套 `Panel`/`TableLayoutPanel`**：

```
scss复制编辑Form
 └─ SplitContainer (Dock = Fill)
     ├─ Panel1 (Dock = Left or Top)
     └─ Panel2 (Dock = Fill)   ← 这里再放你的“填充”控件
```

或者用 `TableLayoutPanel` 把窗口划分若干行列：

```
csharp复制编辑tableLayoutPanel.ColumnCount = 2;
tableLayoutPanel.ColumnStyles.Add(new ColumnStyle(SizeType.AutoSize));
tableLayoutPanel.ColumnStyles.Add(new ColumnStyle(SizeType.Percent, 100));
tableLayoutPanel.Controls.Add(sidebarControl, 0, 0);
sidebarControl.Dock = DockStyle.Fill;
tableLayoutPanel.Controls.Add(mainControl, 1, 0);
mainControl.Dock = DockStyle.Fill;
```

这样 `mainControl` 填充的是第二列，不会侵占第一列的空间。

------

4. 如果只是想留出边距

有时你想让 Fill 留出顶部或左侧一点空间，可以给它**设置 `Margin`** 或给父容器设置 `Padding`：

```
csharp复制编辑// 给父 Panel 留出 50px 顶部
panel.Padding = new Padding(0, 50, 0, 0);

fillControl.Dock = DockStyle.Fill;
```

------

小结

* **顺序第一**：先停靠非-Fill，再停靠 Fill。
* **Z-Order**：必要时用 `SendToBack` / `BringToFront`。
* **分区布局**：用 `SplitContainer` 或 `TableLayoutPanel` 做多区分割。
* **边距留空**：用 `Padding`/`Margin` 给 Fill 控件留出区域。

## 9	?.代表什么意思

在 C# 里，`?.` 是 **空条件成员访问运算符**（null-conditional member access operator），有时候也叫作 “安全导航运算符”（safe navigation operator）或者 “null‐conditional operator”。[维基百科+2Microsoft Learn+2](https://en.wikipedia.org/wiki/Safe_navigation_operator?utm_source=chatgpt.com)

------

理解它的作用

当你使用 `x?.Member` 或 `x?.Method()` 的时候，如果 `x` 是 `null`，整个表达式会 **直接返回 null**，不会抛出 `NullReferenceException`。如果 `x` 不为 null，则正常访问成员或调用方法。[Microsoft Learn+2维基百科+2](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/?utm_source=chatgpt.com)

------

例子

```
Person p = null;
string name = p?.Name;   // 因为 p 是 null，name 得到 null，而不是抛异常

// 链式调用的例子
int? length = p?.Name?.Length;  
// 如果 p 是 null 或者 p.Name 是 null，结果都是 null
// 否则得到字符串 Name 的长度（int 类型，但用 Nullable<int>）
```

------

和其它运算符的搭配

* 如果你希望当表达式为 null 的时候给一个默认值，可以配 `??` （空合并运算符）一起用：

  ```
  string nameOrDefault = p?.Name ?? "未知";
  ```

* 对于索引器也有类似语法 `x?[i]`，意思是如果 `x` 为 null，就不进行索引访问，返回 null。[Microsoft Learn+1](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/?utm_source=chatgpt.com)

------

注意事项

* 如果 `?.` 使用在返回值为值类型（非 Nullable）成员或属性上，结果会被提升为对应的可空类型（Nullable<T>）。例如 `obj?.Prop` 若 `Prop` 是 `int`，那么类型是 `int?`。[维基百科+1](https://en.wikipedia.org/wiki/Safe_navigation_operator?utm_source=chatgpt.com)
* 连续链式的 `?.`：只要前面某一环是 null，其后的成员访问都不会执行。[Microsoft Learn+1](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/?utm_source=chatgpt.com)

## 10	MethodInfo用法

`MethodInfo` 是 .NET Reflection（反射）里一个很重要的类，用来表示某个类型的方法（Method）。你可以用它来检查方法的信息（返回类型、参数、访问修饰符、是否静态／泛型等），也可以用它在运行时动态调用这个方法（`Invoke`）。下面我讲讲它的常见用法 + 示例 + 注意事项。

------

`MethodInfo` 基本介绍

* 在 `System.Reflection` 命名空间中。 [Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/api/system.reflection.methodinfo?view=net-9.0&utm_source=chatgpt.com)
* `MethodInfo` 继承自 `MethodBase`，再上是 `MemberInfo`，因此它有方法声明、属性等元数据可用来检查。 [Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/api/system.reflection.methodinfo?view=net-9.0&utm_source=chatgpt.com)
* 你可以用 `Type.GetMethod(...)`, `Type.GetMethods()`, `GetType()`, 或者通过 `Assembly` 加载类型然后取得 `Type`，再用 `GetMethod` 等来得到一个 `MethodInfo` 对象。 [Microsoft Learn+2.NET Academy+2](https://learn.microsoft.com/en-us/dotnet/api/system.reflection.methodinfo?view=net-9.0&utm_source=chatgpt.com)

------

常见方法／属性

下面是 `MethodInfo` 中比较常用的一些成员：

| 成员                                                         | 用途                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `Name`                                                       | 方法名                                                       |
| `ReturnType`                                                 | 返回类型（`Type`）                                           |
| `GetParameters()`                                            | 获取参数信息，返回 `ParameterInfo[]`                         |
| `IsStatic`                                                   | 是否静态方法                                                 |
| `IsPublic` / `IsPrivate` / 访问修饰等                        | 判断可见性等                                                 |
| `IsGenericMethod` / `IsGenericMethodDefinition` / `ContainsGenericParameters` | 与泛型方法相关的信息 [Microsoft Learn+1](https://learn.microsoft.com/en-us/dotnet/api/system.reflection.methodinfo?view=net-9.0&utm_source=chatgpt.com) |
| `MakeGenericMethod(Type[] typeArguments)`                    | 用于泛型方法，将泛型参数指定具体类型后得到可调用的 “closed generic method” 的 `MethodInfo`。 [Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/api/system.reflection.methodinfo.makegenericmethod?view=net-9.0&utm_source=chatgpt.com) |
| `Invoke(object obj, object[] parameters)`                    | 调用这个方法；如果是实例方法，`obj` 是实例；如果方法是静态，`obj` 可以是 `null`。参数用 `object[]` 传。 [Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/api/system.reflection.methodinfo.invoke?view=netframework-1.1&utm_source=chatgpt.com) |

------

示例

下面是一些常见用法的例子。

示例 1：获取某个类的方法并调用它

假设有个类：

```
public class Person
{
    public string SayHello(string name)
    {
        return $"Hello, {name}!";
    }

    public static int Add(int a, int b)
    {
        return a + b;
    }
}
```

用 `MethodInfo` 调用：

```
using System;
using System.Reflection;

class Program
{
    static void Main()
    {
        Type t = typeof(Person);

        // 实例方法
        MethodInfo mi1 = t.GetMethod("SayHello", new Type[] { typeof(string) });
        object personInstance = Activator.CreateInstance(t);
        object result1 = mi1.Invoke(personInstance, new object[] { "World" });
        Console.WriteLine(result1);  // "Hello, World!"

        // 静态方法
        MethodInfo mi2 = t.GetMethod("Add", BindingFlags.Public | BindingFlags.Static);
        object result2 = mi2.Invoke(null, new object[] { 3, 4 });
        Console.WriteLine(result2);  // 7
    }
}
```

示例 2：泛型方法 + MakeGenericMethod

假设有这样的泛型方法：

```
public class Utils
{
    public static T Echo<T>(T value)
    {
        return value;
    }
}
```

用 `MethodInfo` + `MakeGenericMethod` 调用：

```
Type utilsType = typeof(Utils);
MethodInfo genericMi = utilsType.GetMethod("Echo");  // 获取泛型定义
MethodInfo closedMi = genericMi.MakeGenericMethod(typeof(int));  // 指定 T = int

object result = closedMi.Invoke(null, new object[] { 42 });
Console.WriteLine(result);  // 42
```

------

性能／使用注意事项

* `MethodInfo.Invoke(...)` 比直接调用慢，因为是通过反射。每次调用会有类型检查、参数打包等开销。若频繁调用，可能成为性能瓶颈。
* 如果确定方法签名，在多次调用情况下，可以考虑把 `MethodInfo` 转成一个 delegate（通过 `Delegate.CreateDelegate(...)`）来提高性能。这样调用 delegate 比 `Invoke` 快很多。 [Jon Skeet's coding blog](https://codeblog.jonskeet.uk/2008/08/09/making-reflection-fly-and-exploring-delegates/?utm_source=chatgpt.com)
* 对参数数量和类型要注意：如果传入的 `object[]` 参数与方法声明的不匹配，或者类型不兼容，会抛异常。静态方法 `obj` 参数传 null；实例方法必须有实例对象。
* 泛型方法如果你取得的是泛型定义（未指定类型参数），调用之前必须先用 `MakeGenericMethod(...)` 来构造具体类型，否则 `Invoke` 会出错。
* 安全性、权限问题：如果方法是私有或 Internal 的，用反射可能必须带合适的 `BindingFlags` 或者有访问权限（例如在某些上下文下可能被禁止）。
* 如果目标方法可能抛异常，通过 `Invoke` 调用的异常通常被封装在 `TargetInvocationException` 中，你可能需要 `try...catch` 并查看其 `InnerException`。

------

实际应用场景

* 插件／扩展机制中自动发现带特定特性（Attribute）的函数／方法，并动态调用。
* 来自配置或用户输入的动作名称（字符串），例如 “Execute”、“Save” 等，根据名称查 `MethodInfo` 并执行对应方法。
* 跨程序集／反射式调用（例如动态载入 DLL、反射调用方法等）。
* 在序列化／映射工具、ORM、RCP / RPC 框架中可能用来生成代理、动态方法调用等。

## 11	托管资源和非托管资源是那些

在 .NET / C# 里，“托管资源（managed resource）”和“非托管资源（unmanaged resource）”是很重要的区分，特别是在释放资源／写 `Dispose` 的时候。下面帮你整理清楚：

------

基本定义

* **托管资源（Managed Resource）**：由 CLR（公共语言运行时 Common Language Runtime）自动管理其生命周期的资源。比如 C# 中的普通对象、数组、字符串、List、Dictionary、Stream 的某些封装等等。GC（垃圾回收器）会追踪这些对象，当它们不再被引用时自动清理内存。
* **非托管资源（Unmanaged Resource）**：CLR 本身不能自动回收／清理的资源。这通常是操作系统／外部库／本地 API 分配给你的资源，比如：文件句柄、数据库连接、网络 socket、本地内存、图形资源（GDI 对象）、COM 对象等。你必须显式释放这些资源（或者用某些包装类 + finalizer / safe handle）。

微软文档里有：“The most common types of unmanaged resources are objects that wrap operating system resources, such as files, windows, network connections, or database connections.” [Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/unmanaged?utm_source=chatgpt.com)

------

举例说明

下面是一些具体属于哪一类的例子：

| 示例                                             | 是托管还是非托管  | 备注                                                         |
| ------------------------------------------------ | ----------------- | ------------------------------------------------------------ |
| `new List<string>()`                             | 托管              | 完全由 CLR 管理内存，GC 会回收                               |
| 一个普通的 C# 类实例                             | 托管              | 即便它内部有其他托管对象，也都由 GC 管理                     |
| `FileStream`                                     | 托管 + 非托管包装 | `FileStream` 是托管对象，但内部封装了一个非托管文件句柄，所以需要 `Dispose()` 或使用 `using` 来释放底层句柄。 [Stack Overflow+1](https://stackoverflow.com/questions/3607213/what-is-meant-by-managed-vs-unmanaged-resources-in-net?utm_source=chatgpt.com) |
| 数据库连接（例如 `SqlConnection`）               | “托管包装”        | `SqlConnection` 本身是托管类型，但它管理的连接、socket /网络资源等是非托管的，需要显式关闭／释放。 [Stack Overflow+1](https://stackoverflow.com/questions/3607213/what-is-meant-by-managed-vs-unmanaged-resources-in-net?utm_source=chatgpt.com) |
| COM 对象                                         | 非托管            | 通常需要释放 /调用 `Release()` /使用 COM Interop 等机制      |
| 调用 `Marshal.AllocHGlobal(...)` 分配本地内存    | 非托管            | 你自己从非托管堆上分配的内存，不是 GC 跟踪的。必须用 `Marshal.FreeHGlobal(...)` 来释放。 [Code Maze+2领英+2](https://code-maze.com/csharp-managed-vs-unmanaged-code-garbage-collection/?utm_source=chatgpt.com) |
| 操作系统窗口句柄（HWND）、GDI 对象（笔、画刷等） | 非托管            | 这些 OS 资源不是 GC 自动管理的。必须通过 Windows API 手动释放。 |

------

为什么要区分 /怎样正确处理非托管资源

因为非托管资源如果不释放，会导致资源泄漏，例如文件句柄不释放导致文件锁定、内存泄漏、句柄耗尽等问题。CLR 的垃圾回收只负责托管内存，而不知道怎样“关闭”文件句柄、释放 socket、释放本地内存块等。

常见的做法包括：

* 实现 `IDisposable` 接口，在 `Dispose()` 方法里释放非托管资源。 [Microsoft Learn+1](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/unmanaged?utm_source=chatgpt.com)
* 如果类中有非托管资源，有必要提供 finalizer（析构函数 `~ClassName()`），作为备用，保证即使调用者忘了 `Dispose()`，在非确定时间点可以释放。 [Microsoft Learn+1](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/unmanaged?utm_source=chatgpt.com)
* 使用 “Dispose 模式”（dispose pattern）：在 `Dispose(bool disposing)` 方法里，当 `disposing == true` 时释放托管资源，以及始终释放非托管资源。调用 `GC.SuppressFinalize(this)` 避免 finalizer 重复执行。 [Microsoft Learn+1](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/unmanaged?utm_source=chatgpt.com)
* 使用 `SafeHandle` 类来包装非托管句柄，而不是自己写 finalizer。微软推荐这样做，因为 `SafeHandle` 的 finalize 实现更安全、更正确。 [Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/unmanaged?utm_source=chatgpt.com)

------

小结

* 托管资源是那些由 CLR／.NET GC 自动管理生命周期的对象／内存。
* 非托管资源是那些 CLR 不知道如何自动清理的资源（操作系统资源、本地内存、句柄 etc.），你必须手动释放或者用已有机制释放。
* 在写类时，如果持有非托管资源或者间接持有非托管资源（通过某个托管包装类比如 `FileStream`、数据库连接 etc.），务必要实现 `IDisposable` 来进行清理。

## 12	事件的应用

------

一、事件 (Event) 在 C# 中是什么

* 事件是一种语言级别的机制，用来让一个对象（发布者 / publisher）在某个时刻“通知”另一个对象（订阅者 / subscriber）有某件事发生。
* 底层是基于 **委托 (delegate)** 实现的。事件是一种对委托的封装，使得外部只能通过 `+=` / `-=` 订阅 /取消订阅，而不能随意调用。 [Microsoft Learn+1](https://learn.microsoft.com/en-us/dotnet/standard/events/?utm_source=chatgpt.com)
* 发布者负责“抛出 / 触发（raise / invoke）事件”；订阅者注册自己的方法，当事件触发时，它的方法就被调用。

微软文档中有关于“Handle and raise events”的说明：事件是 .NET 中基于委托模型的通知机制。 [Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/standard/events/?utm_source=chatgpt.com)

------

二、典型场景 /应用

一些常见你可能会在项目中使用事件的场景包括：

1. **UI 交互事件**
   * WinForms / WPF 控件（按钮点击、鼠标移动、文本改变、窗口大小改变等）本身就定义了很多事件。
   * 你订阅这些事件，在用户交互时执行某些逻辑。
2. **业务 /模型状态改变通知**
   * 比如在某个后台处理类里，有一个状态（如“处理完成”、“进度更新”）发生变化时，抛出一个事件，让界面、日志、监控模块等响应。
   * 比如 `ProgressChanged`、`Completed`、`DataLoaded` 等。
3. **组件解耦 /插件 /模块通信**
   * 模块 A 不直接调用模块 B，而是 A 抛出事件，B 或其他监听者来注册处理，从而降低模块间耦合。
   * 常见于 MVC / MVVM /事件总线 /中间件架构里。
4. **异步 /后台任务**
   * 异步执行时，比如后台线程完成某任务、下载完成、IO 读取完成等，使用事件通知主线程 / UI。
   * 比如 `BackgroundWorker` 里的 `ProgressChanged`、`RunWorkerCompleted`，就是事件模式。
5. **资源 /监控 /日志**
   * 当某个资源状态异常 /错误 /警告发生时发事件。
   * 日志模块、监控模块订阅这些事件，统一处理。

------

三、简单示例

下面是一个完整的示例，演示如何在一个业务类里定义事件、触发事件、订阅事件：

```
// 1. 定义一个 EventArgs 派生类，用来传递事件相关的数据
public class DataProcessedEventArgs : EventArgs
{
    public int RecordsProcessed { get; }
    public TimeSpan Duration { get; }

    public DataProcessedEventArgs(int count, TimeSpan dur)
    {
        RecordsProcessed = count;
        Duration = dur;
    }
}

// 2. 定义发布者类
public class DataProcessor
{
    // 定义事件，使用通用的 EventHandler<T>
    public event EventHandler<DataProcessedEventArgs> DataProcessed;

    // 方法：执行某处理
    public void ProcessData()
    {
        var sw = Stopwatch.StartNew();
        // 模拟处理
        Thread.Sleep(500);
        int count = 100;  // 模拟处理的记录数
        sw.Stop();

        // 处理完毕，触发事件
        OnDataProcessed(new DataProcessedEventArgs(count, sw.Elapsed));
    }

    // 这个方法封装了触发事件的逻辑
    protected virtual void OnDataProcessed(DataProcessedEventArgs e)
    {
        // “?.Invoke” 是空安全调用，如果没有订阅者就不会抛 null 异常
        DataProcessed?.Invoke(this, e);
    }
}

// 3. 用法：订阅 &处理事件
public class Program
{
    public static void Main()
    {
        DataProcessor proc = new DataProcessor();
        proc.DataProcessed += Proc_DataProcessed;

        proc.ProcessData();
    }

    private static void Proc_DataProcessed(object sender, DataProcessedEventArgs e)
    {
        Console.WriteLine($"Processed {e.RecordsProcessed} records in {e.Duration.TotalMilliseconds} ms");
    }
}
```

在这个例子里：

* `DataProcessor` 是发布者，定义并触发 `DataProcessed` 事件。
* `Program` 是订阅者，用 `+=` 注册事件处理方法 `Proc_DataProcessed`。
* 处理完数据后，发布者调用 `OnDataProcessed(...)` 触发事件，所有订阅者的方法就被调用。

------

四、在 WinForms / WPF 中应用

在 UI 程序里，事件使用几乎无处不在，比如：

```
button1.Click += Button1_Click;
textBox1.TextChanged += TextBox1_TextChanged;
```

这些都是预定义事件，你在窗体 /控件的代码里注册处理方法。

你也可能自定义控件 /子组件，在控件内部定义事件，让使用控件的外部代码来响应内部行为。

例如，在一个自定义控件里：

```
public class MyControl : Control
{
    public event EventHandler SomethingHappened;

    protected void RaiseSomethingHappened()
    {
        SomethingHappened?.Invoke(this, EventArgs.Empty);
    }

    // 在某种条件触发
    protected override void OnMouseClick(MouseEventArgs e)
    {
        base.OnMouseClick(e);
        RaiseSomethingHappened();
    }
}
```

然后在窗体里：

```
var my = new MyControl();
my.SomethingHappened += (s, e) => { MessageBox.Show("事件触发了"); };
```

------

五、注意事项 /最佳实践

1. **使用 `EventHandler<T>` 而不是自己定义 delegate**
    .NET 提供通用的 `EventHandler<TEventArgs>`，节省重复定义 delegate 的工作。除非有特殊签名需求，否则优先用它。 [Microsoft Learn+1](https://learn.microsoft.com/en-us/dotnet/standard/events/?utm_source=chatgpt.com)
2. **事件触发方法命名惯例为 `OnXxx`**
    发布者类通常写一个 `protected virtual void OnEventName(...)` 方法来封装事件触发逻辑，这样继承类可以重写。 [Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/standard/events/?utm_source=chatgpt.com)
3. **空安全触发**
    用 `eventName?.Invoke(...)` 或事先拷贝到局部变量防止并发修改（防止 NullReferenceException）。
4. **取消订阅**
    如果订阅者生命周期短于发布者，必须在不需要时 `-= 处理方法`，否则可能导致内存泄漏。特别是在 UI /控件释放时要确保解绑。
5. **线程安全 /跨线程事件**
   * 如果在非 UI 线程触发事件，而处理者在 UI 线程更新界面，要用 `Invoke` / `BeginInvoke` /`Dispatcher` 切换线程。
   * 事件内部如果有共享资源，要注意锁 /同步。
6. **避免异常传播**
    事件处理程序如果抛异常，会影响发布者的执行。发布者通常应考虑在触发事件时捕获异常或逐个调用订阅处理。
7. **事件链 /事件总线**
    对于复杂系统，可以设计一个事件总线 /消息中心，用来统一管理事件发布 /订阅。这样模块解耦更清晰。

## 13	内存泄漏表现形式是什么，哪些情况会引起内存泄漏

------

一、内存泄漏的表现形式

在程序运行时，如果出现内存泄漏，通常会有以下一些症状 /表现：

| 表现                                                         | 可能的意义 /提示                                             |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| 长时间运行下 **内存使用（Working Set /私有字节数 /托管堆占用）不断增长**，即使操作结束 | 表示一些对象持续被保留在内存中，GC 无法回收                  |
| GC 运行后依然占用大内存，不下降                              | GC 虽然尝试回收，但因为对象被根引用 “活着”，所以没被回收     |
| 应用性能逐渐变慢 / 响应变迟缓                                | 因为内存压力、GC 频繁、堆碎片、缓存膨胀等影响                |
| 最终可能抛出 `OutOfMemoryException` 或资源耗尽               | 内存被耗尽 /系统资源不足                                     |
| 资源泄漏（如文件句柄、数据库连接、GDI 对象、Windows 句柄 /句柄泄漏） | 即使托管对象被回收，但底层资源未释放 → 造成句柄 /本地资源泄漏 |
| 对象实例数不断累积 /相同类型对象持续存在                     | 使用内存剖析工具（如 Visual Studio 的内存分析、dotMemory、ANTS Memory Profiler）可以看到某些类实例数量不断上升 |

这些表现是“漏”的迹象，但需要结合分析工具（内存快照 /剖析器）进一步定位。

------

二、常见会导致“内存泄漏”的情形

下面是一些在托管 /混合环境中最常见的内存泄漏根源：

| 原因                                        | 描述 /示例                                                   | 参考 /文献 |
| ------------------------------------------- | ------------------------------------------------------------ | ---------- |
| **事件订阅未取消**                          | 发布者对象持有对订阅者对象的引用。若订阅者本应被销毁但忘了 `-= event`，订阅者就无法被回收。典型的“lapsed listener 问题”。 [markheath.net+4维基百科+4codeproject.com+4](https://en.wikipedia.org/wiki/Lapsed_listener_problem?utm_source=chatgpt.com) |            |
| **静态引用 / 全局缓存**                     | 使用 `static` 字段 / 单例 /全局集合缓存对象，导致对象始终被引用，无法回收。 [michaelscodingspot.com+2site24x7.com+2](https://michaelscodingspot.com/ways-to-cause-memory-leaks-in-dotnet/?utm_source=chatgpt.com) |            |
| **闭包 / Lambda 捕获外部变量**              | 在匿名方法 / lambda 中捕获外部对象会导致引用保留，即使看起来没有显式引用。 [michaelscodingspot.com+2Medium+2](https://michaelscodingspot.com/ways-to-cause-memory-leaks-in-dotnet/?utm_source=chatgpt.com) |            |
| **未释放 IDisposable /非托管资源**          | 如 `SqlConnection`、`FileStream`、GDI+ 对象、COM 对象、Native 内存、Interop 分配等没有调用 `Dispose` /释放相应资源。 [site24x7.com+2Medium+2](https://www.site24x7.com/learn/eliminate-net-memory-leaks.html?utm_source=chatgpt.com) |            |
| **本地 /非托管内存泄漏**                    | 通过 P/Invoke /本地库、非托管 DLL 等分配内存但不释放。GC 无法自动管理这些内存。 [Stack Overflow+2michaelscodingspot.com+2](https://stackoverflow.com/questions/134086/what-strategies-and-tools-are-useful-for-finding-memory-leaks-in-net?utm_source=chatgpt.com) |            |
| **委托链 /事件链残留**                      | 多次订阅相同事件没取消 /重复订阅，或事件链累积引用。 [Stack Overflow+2codeproject.com+2](https://stackoverflow.com/questions/51075052/event-unsubscribe-resubscribe-memory-leak-avoidance?utm_source=chatgpt.com) |            |
| **长生命周期对象引用短生命周期对象**        | 如果某对象生命周期非常长（或永存），它引用了许多临时 /短期对象，这些对象就无法回收。 |            |
| **缓存 /集合不清除**                        | 把许多对象放入集合 / Dictionary / List 中，不清理由 /淘汰机制，导致集合持续膨胀。 |            |
| **大对象 /大数组 /LOH（大对象堆）碎片问题** | 对象太大、频繁分配和释放大对象可能导致 LOH 碎片，虽然不是“泄漏”，但内存使用不回降。 |            |
| **资源句柄 /操作系统句柄泄漏**              | 虽然不是纯“托管泄漏”，但如果你打开文件、GDI+ 对象、图像句柄、窗口句柄等不关闭 /释放就会造成“句柄泄漏”（Handle Leak） [维基百科](https://en.wikipedia.org/wiki/Handle_leak?utm_source=chatgpt.com) |            |

------

三、如何排查 /定位内存泄漏

1. **使用内存剖析器 /诊断工具**
   * Visual Studio 的诊断工具（Memory Usage / Snapshots）
   * dotMemory / ANTS Memory Profiler / JetBrains profiler
   * .NET CLI 工具 `dotnet trace`, `dotnet dump`, `GCHeapDump` 等 [Microsoft Learn+1](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/debug-memory-leak?utm_source=chatgpt.com)
2. **捕获内存快照 /对比快照**
   * 在不同时间点捕获快照（如启动时、中间、结束前）
   * 比较实例数量 /保持引用链 /最大实例增长的类型
   * 看哪些类型对象没有被释放、被引用链根源是谁
3. **分析引用路径 (Retained Paths, GC Roots)**
   * 剖析器会显示某个对象为何不能被回收：它被哪些对象引用，最终根源（static、线程、事件、缓存等）
   * 根据引用链往上追，找出“是哪个引用没有释放 /哪个事件没取消”
4. **代码审查 /静态检查**
   * 在可能位置查看是否有 `+=` 而没有对应 `-=`
   * 检查静态字段、缓存集合、闭包 / lambda 捕获
   * 检查 IDisposable 对象是否被 `using` /Dispose
5. **日志 /计数 /调试辅助**
   * 在构造 /析构器 /Dispose 中加日志 /计数，监测实例创建 /销毁次数
   * 自己做弱引用 / GC 检测：例如某对象持有 `WeakReference`，定期检查 `IsAlive` 是否变 false

------

四、避免 /控制内存泄漏的策略

以下是一些实践与技巧，帮助你在项目中减少 /防止内存泄漏：

| 策略                                 | 做法 /推荐                                                   |
| ------------------------------------ | ------------------------------------------------------------ |
| **及时取消事件订阅**                 | 当对象不再需要监听时，用 `-= event` 取消；尤其是发布者生命周期较长的场景。 [Medium+3Stack Overflow+3Microsoft for Developers+3](https://stackoverflow.com/questions/4526829/why-and-how-to-avoid-event-handler-memory-leaks?utm_source=chatgpt.com) |
| **使用弱事件 (Weak Events) /弱引用** | 用弱引用 /弱事件模式，让订阅者即便被引用也不会阻止 GC 回收。 [devleader.ca+2michaelscodingspot.com+2](https://www.devleader.ca/2024/02/14/weak-events-in-c-how-to-avoid-nasty-memory-leaks?utm_source=chatgpt.com) |
| **避免静态引用 /谨慎使用静态缓存**   | 静态字段应只保存真正长期需要的数据，避免持有可能被释放的对象引用。 |
| **使用 `using` / Dispose 模式**      | 对于实现 `IDisposable` 的类（如 Streams、DbContext、Sockets、GDI 对象等），一定要确保调用 `Dispose`／`using` 块封闭它们。 |
| **清理 /剔除缓存 /集合**             | 对长期运行的缓存、集合设置适时清理 /淘汰机制 /最大容量限制。 |
| **避免闭包捕获过多外部引用**         | 在 lambda /匿名方法中捕获少量局部变量，避免捕获整个外部对象 (this) 或很大的对象。 |
| **弱引用 / WeakReference**           | 在某些缓存 /监听场景中如果你愿意牺牲一点复杂度，可以用 `WeakReference<T>` 存对象，使其若无其他强引用可回收。 |
| **周期性监测 /内存健康检查**         | 在应用运行过程中定期监控内存使用、GC 统计、对象数目变化等，及时发警告 /日志。 |

## 14	BindingList的基本使用

在 C# 中，BindingList<T> 是一个可用于数据绑定的集合类型，尤其适用于 Windows Forms 中当 UI 需要实时反映集合内容变化时。下面我来详细说明它的 **基本使用**、**优点**、**与其它集合（如 List<T>）的区别**、以及在实际项目中的注意事项。

------

✅ 基本使用

示例：创建和绑定一个 BindingList

```
// 假设有一个数据类型
public class Person
{
    public string Name { get; set; }
    public int Age { get; set; }
}

// 在窗体或 ViewModel 中
BindingList<Person> peopleList = new BindingList<Person>();

// 添加一些项
peopleList.Add(new Person { Name = "Alice", Age = 30 });
peopleList.Add(new Person { Name = "Bob", Age = 25 });

// 例如在 WinForms 的 DataGridView 中绑定
dataGridView1.DataSource = peopleList;
```

这样做后，如果你再做：

```
peopleList.Add(new Person { Name = "Charlie", Age = 40 });
```

那么绑定到 UI 的控件（如 DataGridView）会自动显示新增的 “Charlie” 行，而无需重新设置 `DataSource`。这是因为 BindingList 会触发集合改变的通知。 [Stack Overflow+1](https://stackoverflow.com/questions/16695885/binding-listt-to-datagridview-in-winform/16695924?utm_source=chatgpt.com)

示例：监听集合变化

```
peopleList.ListChanged += PeopleList_ListChanged;

private void PeopleList_ListChanged(object sender, ListChangedEventArgs e)
{
    // 例如输出是哪一种改变：新增、删除、属性改变等
    Console.WriteLine($"List changed: {e.ListChangedType} at index {e.NewIndex}");
}
```

使用 `ListChanged` 事件可以监控集合的变化。 [Telerik.com+1](https://docs.telerik.com/devtools/winforms/controls/gridview/populating-with-data/binding-to-bindinglist?utm_source=chatgpt.com)

------

📌 优点 &与 List<T> 的区别

* `List<T>` 是简单的集合，不会自动通知 UI 当你在代码里 `Add` / `Remove` 项目时。绑定到 `List<T>` 的控件（比如 DataGridView）如果在后台修改 `List<T>`，通常 **不会** 更新 UI。 [Stack Overflow](https://stackoverflow.com/questions/16695885/binding-listt-to-datagridview-in-winform?utm_source=chatgpt.com)
* `BindingList<T>` 实现了 `IBindingList` 接口或类似机制（它包含 `ListChanged` 事件），可以让 UI 知道集合发生了变化。 [Stack Overflow](https://stackoverflow.com/questions/2243950/listt-vs-bindinglistt-advantages-disadvantages/2243974?utm_source=chatgpt.com)
* 因此，在 WinForms 数据绑定场景中，如果你希望控件随集合添加／删除／更改项而动态更新，推荐使用 `BindingList<T>`。
* 另外，`BindingList<T>` 允许控制集合是否可编辑、是否可删除项、是否允许添加新项，且支持监听变化事件等。 [Telerik.com](https://docs.telerik.com/devtools/winforms/controls/gridview/populating-with-data/binding-to-bindinglist?utm_source=chatgpt.com)

------

⚠ 注意事项

* 虽然 `BindingList<T>` 支持集合项的添加与移除通知，但它 **不自动**监控每个项内部属性的改变（除非该项也实现了 `INotifyPropertyChanged` 并控件绑定支持此机制）。如果你希望 UI 随 “Person.Age” 改变自动更新，你的 `Person` 类应实现 `INotifyPropertyChanged`。
* 在 WPF 场景下，通常会使用 ObservableCollection<T> 而不是 `BindingList<T>`，因为 WPF 的绑定机制对 `INotifyCollectionChanged` 支持更好。但在 WinForms 中 `BindingList<T>` 是更自然的选项。
* 大规模项变动（例如批量添加大量项目）时，每次 `Add` 会触发一次 `ListChanged` 通知，可能带来性能问题。此时可能需要暂时禁用通知或使用其他方案（例如将集合替换一次性设置）。
* 绑定给控件（如 DataGridView）时，建议使用 `BindingSource` 作为中介，然后将 `BindingList<T>` 设为 `BindingSource.DataSource`。这样在源集合改变时，控件可以更加稳定地响应。 [Stack Overflow](https://stackoverflow.com/questions/16695885/binding-listt-to-datagridview-in-winform?utm_source=chatgpt.com)

## 15	partial关键字作用和用法

✅ 作用

使用 `partial` 的主要目的包括：

* 将一个较大的类型分布在多个源文件中，使得每个文件更专注、更易维护。 [simplilearn.com+1](https://www.simplilearn.com/tutorials/c-sharp-tutorial/partial-class-in-c-sharp?utm_source=chatgpt.com)
* 允许工具（如设计器、代码生成器、源生成器）生成类型的一部分代码，而开发者手动编写其余部分，且生成代码不会覆盖手写代码。 [Stack Overflow+1](https://stackoverflow.com/questions/3601901/when-is-it-appropriate-to-use-c-sharp-partial-classes?utm_source=chatgpt.com)
* 支持将类型的不同功能模块化拆分（如业务逻辑与 UI 代码分开）以改善可读性、减少单文件复杂度。 [ByteHide+1](https://www.bytehide.com/blog/partial-class-csharp?utm_source=chatgpt.com)

------

🛠 用法

部分类型（partial type）

你可以这样使用 `partial` 来拆分一个类型：

文件 `Employee_Part1.cs`：

```
public partial class Employee
{
    public int Id { get; set; }
    public string Name { get; set; }
}
```

文件 `Employee_Part2.cs`：

```
public partial class Employee
{
    public void DisplayInfo()
    {
        Console.WriteLine($"Id={Id}, Name={Name}");
    }
}
```

以上两个 `partial` 定义在编译时会被合并为一个 `Employee` 类。 [learn.microsoft.com+1](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/partial-classes-and-methods?utm_source=chatgpt.com)

部分成员（partial members）

在类型中还可以使用 `partial` 方法、部分属性等（支持的成员随 C# 版本而扩展）：

```
public partial class MyClass
{
    partial void OnSomethingHappened(string message);
    public void DoWork()
    {
        // …
        OnSomethingHappened("done");
    }
}

public partial class MyClass
{
    partial void OnSomethingHappened(string message)
    {
        Console.WriteLine(message);
    }
}
```

如果你不为 `OnSomethingHappened` 提供实现，调用它会在编译时被移除。 [Stack Overflow+1](https://stackoverflow.com/questions/36558244/partial-methods-in-c-sharp-explanation?utm_source=chatgpt.com)

------

⚠ 注意事项 /规则

* 所有拆分的 `partial` 部分必须在 **同一个命名空间 (namespace)**、同一个 **程序集 (assembly/module)** 中。 [learn.microsoft.com+1](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/partial-classes-and-methods?utm_source=chatgpt.com)
* 所有部分必须使用相同的访问修饰符（如 `public`、`internal`）和类名。 [Round The Code+1](https://www.roundthecode.com/dotnet-tutorials/what-are-partial-classes-csharp-why-do-we-use-them?utm_source=chatgpt.com)
* 每个部分前必须带 `partial` 关键字；如果一部分缺少，编译器会报错。 [learn.microsoft.com+1](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/partial-classes-and-methods?utm_source=chatgpt.com)
* 如果不同部分对基类或接口的声明有冲突（如一个部分继承 A，一个部分继承 B），也会有编译错误。 [Round The Code](https://www.roundthecode.com/dotnet-tutorials/what-are-partial-classes-csharp-why-do-we-use-them?utm_source=chatgpt.com)
* `partial` 关键字不能用于枚举 (enum)、委托 (delegate)（类型定义层面）等场景（但可以用于拆分成员）。 [learn.microsoft.com](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/partial-classes-and-methods?utm_source=chatgpt.com)

------

🎯 使用场景推荐

* 在使用 **可视化设计器**（如 WinForms 的 `.Designer.cs` 文件、WPF 的 XAML 生成代码）时，将生成代码与手写代码拆分，可以避免覆盖或混乱。
* 在类非常大、含有很多成员时，将不同功能模块拆分到不同文件，提高可读性。
* 在代码生成 /源生成场景（如 ORM、Web 服务代理、工具生成代码）中使用 `partial`，给开发者留下扩展钩子。
* 当团队多人同时开发同一个类但不同职责（如 UI 逻辑、业务逻辑、数据访问）时，使用 `partial` 可以减少冲突。

## 16	窗口句柄创建时机

🧩 一、什么是窗口句柄 (Handle)

在 .NET 的 `Control` / `Form` 中，每个控件都对应一个底层 Win32 窗口对象（有时没有）。
 这个对象的唯一标识符就是 **句柄 (`IntPtr Handle`)**。

> Win32 创建窗口的本质调用：
>  `HWND hwnd = CreateWindowEx(...);`
>  这个 `hwnd` 就是你在 .NET 里看到的 `control.Handle`。

------

🕒 二、句柄的创建时机（WinForms）

1️⃣ **懒加载（On-demand）机制**

控件的 `Handle` **不是在构造函数中立刻创建**，而是 **在控件第一次需要与系统交互时才创建**。
 比如：

* 控件第一次显示在屏幕上；
* 调用 `CreateControl()`；
* 调用 `Handle` 属性；
* 控件需要接收消息（例如被添加到可见父控件中）时。

------

2️⃣ **典型生命周期**

以窗体 `Form` 为例：

| 阶段                              | 时机                                      | 句柄状态     |
| --------------------------------- | ----------------------------------------- | ------------ |
| 构造函数 (`Form()` / `Control()`) | 控件仅创建托管对象，还未与 Win32 窗口绑定 | ❌ 未创建句柄 |
| `OnHandleCreated` 事件            | 调用 Win32 `CreateWindowEx` 完成后        | ✅ 句柄已创建 |
| `Load` 事件                       | 窗口加载后（句柄已存在）                  | ✅ 有效       |
| `Shown` / `Activated`             | 首次显示                                  | ✅ 有效       |
| `HandleDestroyed` / `Dispose`     | 窗口关闭或销毁时                          | ❌ 销毁句柄   |

------

🔍 检查示例

```
public partial class MyForm : Form
{
    public MyForm()
    {
        InitializeComponent();

        Console.WriteLine("构造函数: Handle created? " + this.IsHandleCreated); // false
    }

    protected override void OnHandleCreated(EventArgs e)
    {
        base.OnHandleCreated(e);
        Console.WriteLine("OnHandleCreated: " + this.Handle); // 这里才真正有句柄
    }

    private void MyForm_Load(object sender, EventArgs e)
    {
        Console.WriteLine("Load: " + this.Handle); // 句柄已创建
    }
}
```

输出示例：

```
构造函数: Handle created? False
OnHandleCreated: 0x00B305C6
Load: 0x00B305C6
```

------

⚙️ 三、常见问题

🚫 问题1：在构造函数中使用句柄

```
public MyControl()
{
    var hwnd = this.Handle; // ❌ 此时还未创建，会触发强制创建
}
```

**危险点**：
 这样会强制创建句柄，可能导致控件层级关系错误（句柄过早创建）。

✅ 推荐：
 在 `OnHandleCreated` 或 `Load` 后使用。

------

🚫 问题2：句柄被销毁后使用

句柄可能被销毁又重建（如重新创建窗口、父窗口重绘、Dock 改变等）。

```
if (!this.IsHandleCreated)
    this.CreateHandle(); // 或等待 OnHandleCreated
```

如果你持有旧的 `Handle` 值（`IntPtr`），它**不再有效**。
 应该在 `HandleCreated` 事件中重新注册或更新所有 Win32 关联（如 `SetWindowLong`、`Subclass` 等）。

------

🧠 四、控制创建句柄的时机

有时你希望**提前创建句柄**（例如要调用 Win32 API）：

```
if (!this.IsHandleCreated)
    this.CreateHandle();
```

或通过：

```
this.CreateControl(); // 递归创建当前控件及子控件的句柄
```

------

🧩 五、在 UserControl 中的特殊点

如果你写的是 `UserControl`，句柄的创建依赖父窗体。

* `UserControl` 在被加入到可见的 Form 后，父级创建时会自动创建其句柄；
* 但如果还没显示在 UI 中，`Handle` 也不存在；
* 因此在构造函数里访问 `Handle` 也会触发“强制提前创建”。

------

✅ 总结

| 阶段                | 句柄状态   | 推荐做法                  |
| ------------------- | ---------- | ------------------------- |
| 构造函数            | ❌ 未创建   | 不要访问 `Handle`         |
| `OnHandleCreated`   | ✅ 已创建   | 注册 Win32 回调等         |
| `Load` / `Shown`    | ✅ 已创建   | 可安全访问句柄            |
| `OnHandleDestroyed` | ❌ 即将销毁 | 注销 Win32 回调或释放资源 |

## 17	Queue\<T\> 为什么需要加锁？ConcurrentQueue\<T\> 是怎么做到线程安全的？

> 来源：INS 检测系统 `InsInspectionManager.cs` 中 `InspectionThread` 类使用 `lock + Queue<TaskRunParam>` 实现多线程命令队列的代码审查。

------

### 🔍 问题背景

在检测系统中，多个线程（命令管理器线程、相机回调线程、TCP 接收线程）同时调用 `Run()` 往队列里塞任务，一个工作线程在 `DoWork()` 中取任务执行。代码使用了 `lock` 保护 `Queue<T>`：

```csharp
// 生产者（多个线程调用）
public void Run(TaskRunParam taskRunParam)
{
    lock (_sync)
    {
        _waitingRuns.Enqueue(taskRunParam);
    }
}

// 消费者（单个工作线程）
private void DoWork()
{
    while (!_exit)
    {
        TaskRunParam taskRunParam = null;
        lock (_sync)
        {
            if (_waitingRuns.Count > 0)
                taskRunParam = _waitingRuns.Dequeue();
        }
        if (taskRunParam != null) { /* 处理 */ }
        else { Thread.Sleep(18); }
    }
}
```

**问题**：为什么一个简单的入队出队要加锁？不就是一行代码吗？

------

### 🧨 一、Queue\<T\> 为什么不是线程安全的

`Queue<T>` 内部是一个**环形数组**，每次操作涉及**多个字段的修改**：

```
Queue<T> 内部结构：

_array:  [ _ | A | B | C | _ | _ ]    ← 存储数组
_head:        ↑                        ← 出队指针
_tail:                    ↑            ← 入队指针
_size:   3                             ← 元素计数

Enqueue 操作 = _array[_tail] = item → _tail++ → _size++    （修改 3 个字段）
Dequeue 操作 = item = _array[_head] → _head++ → _size--    （修改 3 个字段）
```

一次操作要改 3 个字段，**不是原子操作**。两个线程同时执行，中间状态会互相踩踏。

------

### 💥 二、不加锁会出什么问题

#### 场景 1：数据丢失

```
时刻1: 线程A 读到 _tail = 3
时刻2: 线程B 也读到 _tail = 3     ← 还没来得及更新
时刻3: 线程A 写入 _array[3] = paramA, _tail = 4
时刻4: 线程B 写入 _array[3] = paramB, _tail = 4  ← paramA 被覆盖！

结果：paramA 丢失，检测命令被吞掉
```

#### 场景 2：数组越界崩溃

```
时刻1: 线程A 检查 _size < _array.Length → true（还有空间）
时刻2: 线程B 也检查 → true
时刻3: 线程A Enqueue, _size++ → 数组满了
时刻4: 线程B Enqueue → 💥 IndexOutOfRangeException
```

#### 场景 3：Count 与实际不一致

```
时刻1: Enqueue 写入了 _array[3] = param
时刻2: 还没来得及 _size++
时刻3: 另一线程读 _size → 以为队列是空的，跳过处理
```

------

### ✅ 三、ConcurrentQueue\<T\> 如何做到线程安全

`ConcurrentQueue<T>` 完全**不用锁**，采用**无锁（Lock-Free）** 设计，核心有三个技术：

------

#### 技术 1：Segment 链表（分离读写端）

```
ConcurrentQueue<T> 内部：

  _head（Segment 引用）             _tail（Segment 引用）
      │                                 │
      ▼                                 ▼
┌──────────┐    ┌──────────┐    ┌──────────┐
│ Segment 0│───>│ Segment 1│───>│ Segment 2│
│ [32个槽] │    │ [32个槽] │    │ [32个槽] │
│ 已消费完 │    │ 部分消费 │    │ 正在填充 │
└──────────┘    └──────────┘    └──────────┘
```

- 每个 Segment 是一个固定 32 个元素的小数组
- **Enqueue** 只操作 `_tail` 端的 Segment
- **Dequeue** 只操作 `_head` 端的 Segment
- 大部分时间**读写在不同 Segment，没有竞争**

------

#### 技术 2：Interlocked 原子操作（替代 lock）

```csharp
// Queue<T> —— 不安全
_array[_tail] = item;
_tail++;        // ← 不是原子的

// ConcurrentQueue<T> —— 原子操作
int slot = Interlocked.Increment(ref _tail);  // ← CPU 级别原子指令 lock xadd
_array[slot] = item;
```

`Interlocked.Increment` 是 CPU 硬件指令，**不可能被打断**：

```
线程A: Interlocked.Increment(ref _tail) → 得到 3
线程B: Interlocked.Increment(ref _tail) → 得到 4  ← 保证不同，永远不会冲突
```

------

#### 技术 3：CAS（Compare-And-Swap）+ SpinWait 自旋

```csharp
// CAS 的语义："如果当前值 == 期望值，才更新为新值"
if (Interlocked.CompareExchange(ref _slots[tail].Seq, tail + 1, tail) == tail)
{
    _slots[tail].Item = item;  // 成功占位，写入数据
}
else
{
    // 槽位被别人抢了，SpinWait 自旋重试
}
```

对比 lock：
- **lock**：竞争时线程**挂起**（内核态切换，~1-15μs 开销）
- **CAS**：竞争时线程**自旋重试**（用户态，纳秒级，不需要系统调用）

------

### 🆚 四、对比总结

| 维度 | `lock + Queue<T>` | `ConcurrentQueue<T>` |
|------|-------------------|----------------------|
| **并发策略** | 悲观锁（互斥） | 乐观无锁（CAS） |
| **阻塞行为** | 竞争时线程挂起，等待唤醒 | 竞争时自旋重试，不挂起 |
| **上下文切换** | 有（内核态切换，开销大） | 无（用户态自旋） |
| **吞吐量** | 中等 | 高（高并发下优势明显） |
| **读写分离** | 否，同一把锁 | 是，读写在不同 Segment |
| **内存增长** | 数组满时整体扩容（翻倍） | 按 Segment（32 个）链式增长 |
| **适用场景** | 低并发，或锁内含复杂逻辑 | 高并发，纯入队出队 |

------

### 🧠 五、什么时候该用哪个

#### 用 `ConcurrentQueue<T>` 的场景

纯粹的多线程入队出队，不需要额外原子操作：

```csharp
private ConcurrentQueue<TaskRunParam> _waitingRuns = new();

public void Run(TaskRunParam param)
{
    _waitingRuns.Enqueue(param);  // 无锁，线程安全
}

private void DoWork()
{
    while (!_exit)
    {
        if (_waitingRuns.TryDequeue(out var param))  // 无锁
            ProcessParam(param);
        else
            Thread.Sleep(18);
    }
}
```

#### 继续用 `lock + Queue<T>` 的场景

锁内不仅仅是入队，还有多个操作需要作为原子事务一起执行：

```csharp
lock (_sync)
{
    _waitingSemaphore.Release();                    // ① 信号量
    taskRunParam.TickNum = DateTime.Now.ToString();  // ② 赋时间戳
    InsMessageManager.Warn("等待运行...");           // ③ 日志
    _waitingRuns.Enqueue(taskRunParam);              // ④ 入队
    InsMessageManager.Warn("等待数量：" + Count);    // ⑤ 队列长度日志
}
```

`ConcurrentQueue` 只能保证**单次 Enqueue/Dequeue 原子**，无法保证 ①②③④⑤ 整体原子。如果需要多个操作不可分割，仍然需要 `lock`。

------

### ✅ 总结一句话

> `Queue<T>` 一次操作改 3 个内部字段，不是原子的，多线程必须加锁。`ConcurrentQueue<T>` 用 Segment 链表分离读写 + CPU 级原子指令（Interlocked/CAS）实现无锁并发，性能更高但只能保证单次操作原子。如果锁内包含多步关联操作，仍需 `lock`。

## 18	.NET GC 分代回收机制（Gen 0 / Gen 1 / Gen 2 / LOH）

> 来源：INS 检测系统 `InsInspectionManager.cs` 中 `LoadInspectionByIndex()` 方法使用 `GC.Collect()` 引发的讨论，以及 VisionPro ToolBlock 的 `GarbageCollectionEnabled` / `GarbageCollectionFrequency` 属性分析。

------

### 🧠 一、分代假说（Generational Hypothesis）

.NET GC 基于一个经过实践验证的统计规律：

1. **大多数对象"朝生暮死"** — 刚分配的对象很快就不再使用
2. **存活越久的对象，越可能继续存活** — 老对象往往是长期驻留的

基于这个假设，GC 把托管堆分成三代 + 大对象堆，避免每次都扫描整个堆：

```
┌─────────────────────────────────────────────────────────┐
│                    托管堆 (Managed Heap)                  │
├──────────┬──────────────┬──────────────────┬────────────┤
│  Gen 0   │    Gen 1     │      Gen 2       │    LOH     │
│ (最年轻)  │  (缓冲区)     │    (老年代)       │ (大对象堆)  │
│ ~256KB   │   ~2MB       │    动态增长       │  ≥85KB对象  │
│ 回收最频繁 │  较少回收      │   Full GC时回收   │ 与Gen2一起  │
└──────────┴──────────────┴──────────────────┴────────────┘
```

------

### 🟢 二、Gen 0 — 新生代

**特征：**
- 所有新分配的对象首先进入 Gen 0（大对象除外）
- 空间最小，约 256KB ~ 几 MB
- 回收频率最高，速度最快（通常 < 1ms）

**工作方式：**
```
分配前:
┌──────────────────────────────────┐
│ ObjA  ObjB  ObjC  ObjD  ObjE   │ ← 分配指针
└──────────────────────────────────┘

GC 扫描后（ObjB、ObjD 无引用 → 标记为垃圾）:
┌──────────────────────────────────┐
│ ObjA  ░░░░  ObjC  ░░░░  ObjE   │
└──────────────────────────────────┘

压缩整理后:
┌──────────────────────────────────┐
│ ObjA  ObjC  ObjE │              │ ← 分配指针前移
└──────────────────────────────────┘
  这3个存活对象 → 晋升到 Gen 1
```

**触发条件：** Gen 0 空间用完时自动触发

------

### 🟡 三、Gen 1 — 缓冲代

**特征：**
- 从 Gen 0 存活下来的对象晋升到这里
- 相当于"缓冲区"，给短命对象第二次被回收的机会
- 回收频率中等

**设计意图：**
```
场景：某个对象在 Gen 0 回收时恰好还被引用（比如刚好在一个方法中间）
→ 晋升到 Gen 1
→ 方法结束后，下次 Gen 1 回收时清除

没有 Gen 1 的话，这类对象会直接进 Gen 2 → 导致 Gen 2 膨胀 → Full GC 更慢
```

**回收时机：** 当 Gen 0 回收后空间仍不足，或 Gen 1 自身空间不足时触发。Gen 1 回收会**同时回收 Gen 0**。

------

### 🔴 四、Gen 2 — 老年代

**特征：**
- 经过两次 GC 仍存活的对象最终到达这里
- 空间最大，动态增长
- 回收代价最高（称为 **Full GC**）
- 回收时会暂停所有托管线程（Stop-The-World）

**典型的 Gen 2 对象：**
```csharp
// 单例对象、长期存活的集合
private static InsCommandManager _this = null;     // 单例，应用生命周期
private Queue<CommandSettings> _commandsInComing;   // 长期存活的队列
private CogToolBlock _toolBlock;                    // VisionPro 工具块
```

**Full GC 的代价：** 扫描整个托管堆（Gen 0 + Gen 1 + Gen 2 + LOH），在工业检测系统中可能导致 **10~100ms 的暂停**。

------

### 🟣 五、LOH — 大对象堆（Large Object Heap）

**特征：**
- 存放 **≥ 85,000 字节**（约 85KB）的对象
- 直接归入 Gen 2 管理，与 Gen 2 一起回收
- **默认不压缩**（移动大对象代价太高）→ 容易产生内存碎片

**工业视觉中的典型场景：**
```csharp
// VisionPro 图像对象通常很大
ICogImage image = camera.AcquireImage();
// 一张 2048x2048 灰度图 = 4MB → 直接进 LOH

// CogSerializer 反序列化的 VPP 文件
var obj = CogSerializer.LoadObjectFromFile(path);
// 文件几十MB → 中间对象都在 LOH
```

------

### 🔄 六、分代晋升流程

```
对象分配
   │
   ▼
┌────────┐  存活   ┌────────┐  存活   ┌────────┐
│ Gen 0  │ ──────→ │ Gen 1  │ ──────→ │ Gen 2  │
│        │         │        │         │        │
│ 死亡→回收│         │ 死亡→回收│         │ 死亡→回收│
└────────┘         └────────┘         └────────┘
                                           │
                                      Full GC 时
                                      才会清理

≥85KB 的对象 ──────────────────────→  LOH (随 Gen 2 回收)
```

------

### ⚙️ 七、GC 回收三阶段

每次 GC 发生时，都执行这三步：

| 阶段 | 动作 | 说明 |
|------|------|------|
| **1. 标记（Mark）** | 从 GC Root 出发，遍历所有可达对象 | GC Root = 静态变量、栈变量、CPU 寄存器、终结队列 |
| **2. 清除（Sweep）** | 未标记的对象视为垃圾 | 回收其占用的内存 |
| **3. 压缩（Compact）** | 存活对象向前移动，消除碎片 | Gen 0/1/2 压缩，LOH 默认不压缩 |

**Write Barrier（写屏障）：** 当只回收 Gen 0 时，GC 通过 Card Table 记录"老代对象引用新代对象"的情况，避免扫描整个 Gen 2。

------

### 🔧 八、GC.Collect() 的使用

```csharp
GC.Collect();              // 回收所有代（0+1+2+LOH），最慢
GC.Collect(0);             // 只回收 Gen 0，最快
GC.Collect(1);             // 回收 Gen 0 + Gen 1
GC.Collect(2);             // 回收所有代 = GC.Collect()
```

**何时可以调用 GC.Collect()：**

| 场景 | 是否合适 | 原因 |
|------|---------|------|
| 检测循环中（热路径） | ❌ 不合适 | Full GC 暂停会打断实时检测 |
| 加载 VPP 文件后（冷路径） | ✅ 合适 | 低频操作，大量临时对象需要清理 |
| 切换产品/配方时 | ✅ 合适 | 旧对象已无用，趁机清理 |
| 应用空闲时 | ✅ 合适 | 不影响用户体验 |

------

### 🆚 九、VisionPro GC vs .NET GC

| | VisionPro GC | .NET GC |
|---|---|---|
| **管理对象** | ToolBlock 内部中间结果（临时图像、几何记录） | 所有托管对象 |
| **触发方式** | 每 N 次 Run 执行一次（`GarbageCollectionFrequency`） | Gen 0 满时自动触发 |
| **控制方式** | `GarbageCollectionEnabled` / `GarbageCollectionFrequency` | `GC.Collect()` / 自动 |
| **相互关系** | 完全独立，互不影响 | 完全独立，互不影响 |

------

### ✅ 总结一句话

> .NET GC 采用分代策略：Gen 0（新生、小、快）→ Gen 1（缓冲）→ Gen 2（老年、大、慢），大对象（≥85KB）直接进 LOH。GC 每次只扫描必要的代，存活对象逐代晋升。`GC.Collect()` 应只在低频冷路径（如加载 VPP 文件后）调用，检测循环中绝对不要调用。VisionPro 的 GC 与 .NET GC 完全独立。

## 19	反序列化为什么会产生大量临时对象

> 来源：INS 检测系统 `InsInspectionManager.cs` 中 `CogSerializer.LoadObjectFromFile()` 之后调用 `GC.Collect()` 引发的讨论。

------

### 🔍 一、核心原因

反序列化不是"一步到位"把字节变成最终对象的，而是经历多个中间阶段，每个阶段都会产生临时对象：

```
磁盘上的 .ins 文件 (二进制)
        │
        ▼
  ┌─────────────┐
  │ 1. 读取文件   │ → 产生 byte[] 缓冲区（文件几十MB → 直接进 LOH）
  └──────┬──────┘
        ▼
  ┌─────────────┐
  │ 2. 解析流结构 │ → 产生 Stream 包装对象、Header 元数据对象
  └──────┬──────┘
        ▼
  ┌─────────────────┐
  │ 3. 读取类型信息    │ → 产生 Type 描述字符串、Assembly 名称字符串
  └──────┬──────────┘
        ▼
  ┌─────────────────┐
  │ 4. 逐个构造对象    │ → 每个对象先 new 出来，再填充字段
  └──────┬──────────┘     某些字段需要类型转换 → 产生中间转换对象
        ▼
  ┌─────────────────┐
  │ 5. 重建引用关系    │ → 临时的引用表 / ID 映射字典
  └──────┬──────────┘
        ▼
  最终的 Dictionary<string, List<CogToolBlock>>
  （只有这个是你需要的，上面的中间产物全是"垃圾"）
```

------

### 🧨 二、具体有哪些临时对象

#### 1. I/O 缓冲区
```csharp
FileStream.Read() → byte[8192] 默认缓冲区
// 如果文件 30MB，整个内容可能被读入一个大 byte[] → 直接进 LOH
```

#### 2. 字符串（大量）
```csharp
// 反序列化时需要读取每个字段名、类型名
"Cognex.VisionPro.ToolBlock.CogToolBlock"  // 类型全名
"InputTerminal_0"                           // 字段名
// 这些字符串在解析过程中创建，解析完就没用了
```

#### 3. 中间集合
```csharp
// 反序列化器内部维护的临时数据结构
Dictionary<int, object> objectTable;    // 对象ID → 对象实例的映射
List<FixupRecord> fixupList;            // 延迟修复的引用记录
Stack<object> deserializationStack;     // 嵌套对象的解析栈
```

#### 4. 装箱（Boxing）
```csharp
// 值类型字段反序列化时，往往先作为 object 读出再拆箱
object rawValue = reader.ReadInt32();  // int → 装箱为 object（临时对象）
field.SetValue(target, rawValue);       // 赋值后 rawValue 不再需要
```

#### 5. 类型转换的中间产物
```csharp
// 老版本保存的数据可能需要类型适配
ArrayList temp = DeserializeArrayList(stream);  // 先反序列化为旧类型
List<T> result = temp.Cast<T>().ToList();        // 转换为新类型
// temp 和 Cast 产生的迭代器都是临时对象
```

------

### 🆚 三、不同反序列化方式的临时对象量对比

| 反序列化方式 | 临时对象量 | 原因 |
|------------|----------|------|
| **BinaryFormatter**（.NET 原生） | 很多 | 完整的类型元数据解析、对象图重建、大量反射 |
| **CogSerializer**（VisionPro） | 很多 | 基于 BinaryFormatter，还有 VisionPro 内部对象图（图像、几何） |
| **JSON（Newtonsoft/System.Text.Json）** | 中等 | 字符串解析 + token 化 + 中间 JObject/JsonElement |
| **System.Text.Json（Utf8）** | 较少 | 直接操作 UTF-8 字节，减少字符串分配 |
| **Protobuf / MessagePack** | 最少 | 二进制协议，schema 预编译，几乎无反射 |
| **手动 BinaryReader** | 最少 | 你完全控制分配，只创建最终对象 |

**关键规律：** 反序列化框架越"智能"（自动处理类型发现、版本兼容、引用循环），产生的临时对象越多。

------

### 💡 四、直观对比

```csharp
// === 方式 1: CogSerializer（项目中的方式）===
var obj = CogSerializer.LoadObjectFromFile("test.ins");
// 内部：byte[] → 流解析 → 类型查找 → 反射创建对象 → 引用修复
// 临时对象：几千到几万个

// === 方式 2: 手动二进制读取（假设格式已知）===
using (var reader = new BinaryReader(File.OpenRead("data.bin")))
{
    var result = new MyObject();
    result.Width = reader.ReadInt32();    // 直接读，无中间对象
    result.Height = reader.ReadInt32();
    result.Data = reader.ReadBytes(n);    // 只有最终的 byte[]
}
// 临时对象：几乎为零
```

------

### ✅ 总结一句话

> 反序列化过程中，框架需要读取 I/O 缓冲区、解析类型元数据字符串、构建对象引用映射表、装箱值类型、做类型转换等多步操作，每一步都会产生只在解析期间存在的临时对象。框架越智能（如 BinaryFormatter / CogSerializer），临时对象越多。这就是为什么在 `CogSerializer.LoadObjectFromFile()` 之后调用 `GC.Collect()` 是合理的 — 清理几万个临时对象和 LOH 碎片。

## 21	C# 中的线程/并发形式全览与选型指南

> 来源：INS 检测系统 `InsTcpServer.cs` 中 `ThreadPool.QueueUserWorkItem` 用于长期阻塞任务引发的讨论，结合项目中 Thread、ThreadPool、Task、Timer 的使用场景总结。

------

### 📋 一、一览表

| 形式 | 本质 | 适用场景 |
|------|------|---------|
| **Thread** | 操作系统原生线程 | 长期运行、需要精细控制 |
| **ThreadPool** | 线程复用池 | 短期小任务 |
| **Task / Task.Run** | 线程池的高级包装 | 通用异步/并行（首选） |
| **async/await** | 异步状态机 | I/O 等待（不占线程） |
| **Parallel** | 数据并行 | 批量数据同构处理 |
| **Timer** | 定时触发 | 周期性任务 |

------

### 🔧 二、Thread — 操作系统线程

一个 Thread 对应一个操作系统线程，完全由你控制生命周期。

```csharp
Thread thread = new Thread(() =>
{
    while (!exit)
    {
        // 永久循环干活
    }
});
thread.IsBackground = true;                     // 应用退出时自动结束
thread.Priority = ThreadPriority.Highest;        // 可控优先级
thread.Name = "Inspection Thread";               // 可命名，方便调试
thread.SetApartmentState(ApartmentState.MTA);    // 可设置 COM 套间模型
thread.Start();
```

**适用场景：**
- 长期运行的循环（检测线程、消息处理线程）
- 需要设置优先级（如检测线程用 `Highest`）
- 需要设置 ApartmentState（如 VisionPro 需要 MTA）
- 需要命名（方便调试定位）

**不适用：** 大量短任务（每个线程创建销毁开销约 1MB 内存 + 几百微秒时间）

------

### 🏊 三、ThreadPool — 线程复用池

系统维护一组线程，任务完成后线程回到池中等待复用。

```csharp
ThreadPool.QueueUserWorkItem(state =>
{
    string result = ProcessData();  // 几百毫秒的短任务
    // 完成后线程自动归还池中
});
```

**关键限制：**
- 初始线程数 = CPU 核心数，新线程注入速率约 **500ms 一个**
- 无法设置线程优先级、名称、ApartmentState
- 无法获取返回值、无法捕获异常

**适用场景：** 短期任务（几毫秒到几秒），大量并发小任务

**不适用：** 长期阻塞任务（借了不还，池子被耗干 → 线程饥饿）

------

### 🚀 四、Task / Task.Run — 线程池的高级包装

底层用的就是 ThreadPool，但增加了返回值、异常传播、取消、延续等能力。

```csharp
// 基本用法 — 等同于 ThreadPool.QueueUserWorkItem，但更强大
Task.Run(() => DoSomething());

// 有返回值
Task<int> task = Task.Run(() => CalculateResult());
int result = await task;  // 等待结果

// 异常可以捕获（ThreadPool 做不到）
try
{
    await Task.Run(() => { throw new Exception("出错了"); });
}
catch (Exception ex)
{
    Console.WriteLine(ex.Message);  // 能正常 catch 到
}

// 任务编排
await Task.WhenAll(task1, task2, task3);  // 等待全部完成
await Task.WhenAny(task1, task2);         // 任一完成即返回

// 长期任务 — 告诉调度器创建专用线程，不占线程池
Task.Factory.StartNew(AcceptProc, TaskCreationOptions.LongRunning);
```

**适用场景：** **绝大多数场景的首选** — 需要返回值、异常传播、取消（`CancellationToken`）、任务编排

------

### ⏳ 五、async/await — 异步编程

**核心：不创建新线程！** 等待期间释放当前线程去做别的事，I/O 完成后回来继续。

```csharp
// ❌ 同步 — 线程阻塞在 Receive，干等着什么也不干
int ret = socket.Receive(buffer);   // 阻塞 200ms，线程被占着

// ✅ 异步 — 等待期间线程被释放
int ret = await socket.ReceiveAsync(
    new ArraySegment<byte>(buffer), SocketFlags.None);  // 等待期间不占线程
```

**直观对比：**

```
同步（Thread/ThreadPool）：
  线程: [=======等网络=======][处理结果]
        ↑ 这段时间线程被占着，什么也不干

异步（async/await）：
  线程: [发起请求][释放，去服务其他任务...]
  线程: [回来处理结果]
        ↑ 等待期间没有线程被占用
```

**适用场景：** I/O 密集型（网络请求、文件读写、数据库查询）、Web 高并发、UI 不卡顿

**不适用：** CPU 密集型计算（async/await 不是为计算设计的，应该用 `Task.Run`）

------

### 🔀 六、Parallel — 数据并行

把一个集合的处理工作自动分配到多个线程池线程上并行执行。

```csharp
// 串行：逐个处理 1000 张图片，8 核只用了 1 核
foreach (var image in images)
    ProcessImage(image);

// 并行：自动拆分到多核，8 核全部跑满
Parallel.ForEach(images, image =>
    ProcessImage(image));

// Parallel.For
Parallel.For(0, 100, i =>
    results[i] = HeavyCalculation(i));
```

**适用场景：** 大量**同构**数据需要**相同处理**（批量图像处理、批量文件压缩），且数据之间无依赖

**不适用：** 数据量少（并行开销 > 收益）、数据有依赖关系

------

### ⏰ 七、Timer — 定时器

C# 有三种 Timer：

```csharp
// ① System.Threading.Timer — 线程池线程上回调，最轻量
var timer = new System.Threading.Timer(
    callback: _ => CheckStatus(),
    state: null, dueTime: 0, period: 1000);

// ② System.Timers.Timer — 线程池线程上触发事件
var timer = new System.Timers.Timer(1000);
timer.Elapsed += (s, e) => CheckStatus();
timer.Start();

// ③ System.Windows.Forms.Timer — UI 线程上触发，可直接操作控件
var timer = new System.Windows.Forms.Timer();
timer.Interval = 100;
timer.Tick += (s, e) => label1.Text = "更新UI";  // UI 线程，安全
timer.Start();
```

| Timer 类型 | 执行线程 | 适用场景 |
|-----------|---------|---------|
| `Threading.Timer` | 线程池 | 后台定时任务 |
| `Timers.Timer` | 线程池 | 服务端定时任务 |
| `Windows.Forms.Timer` | UI 线程 | 刷新界面 |

**注意：** `Threading.Timer` 和 `Timers.Timer` 的回调在线程池线程上执行，如果要更新 UI 控件需要 `Invoke`。`Windows.Forms.Timer` 直接在 UI 线程上执行，可以安全操作控件但精度较低（约 15ms）。

------

### 🧭 八、选型决策流程

```
需要并发/异步？
    │
    ├─ 任务是 I/O 等待（网络/文件/数据库）？
    │     └─ async/await ✅（不占线程，最高效）
    │
    ├─ 任务是短期计算/处理（< 几秒）？
    │     └─ Task.Run ✅（线程池复用）
    │
    ├─ 批量数据同构并行处理？
    │     └─ Parallel.ForEach ✅（自动分配多核）
    │
    ├─ 长期运行的循环 / 永久阻塞？
    │     ├─ 需要设置优先级/名称/ApartmentState？
    │     │     └─ new Thread ✅
    │     └─ 不需要特殊设置？
    │           └─ Task.Factory.StartNew(LongRunning) ✅
    │
    └─ 定时周期执行？
          ├─ 需要更新 UI？ → Windows.Forms.Timer ✅
          └─ 后台任务？    → Threading.Timer ✅
```

------

### ✅ 总结一句话

> `Task.Run` 是大多数场景的首选；I/O 等待用 `async/await`（不占线程）；长期运行的循环用 `new Thread` 或 `Task.Factory.StartNew(LongRunning)`；批量同构数据处理用 `Parallel`；定时任务用 `Timer`。`ThreadPool.QueueUserWorkItem` 虽然能用但能力有限，基本被 `Task.Run` 取代。

## 22	ApartmentState（STA / MTA）— COM 线程套间模型

> 来源：INS 检测系统 `InsMessageManager.cs`、`InsCommandManager.cs` 中 `_thread.SetApartmentState(ApartmentState.MTA)` 的代码分析，以及 VisionPro COM 组件并发调用场景。

------

### 🔍 一、什么是 ApartmentState

`ApartmentState` 决定了线程如何与 COM（Component Object Model）组件交互。C# 中有三个枚举值：

| 值 | 含义 | 线程与 COM 对象的关系 |
|---|---|---|
| **STA** | Single-Threaded Apartment（单线程套间） | 一个线程独占一个套间，COM 调用自动串行化 |
| **MTA** | Multi-Threaded Apartment（多线程套间） | 多个线程共享一个套间，COM 调用可并发 |
| **Unknown** | 未设置（默认值） | 首次调用 COM 时由系统自动决定（通常分配到 MTA） |

------

### 🆚 二、STA vs MTA 核心区别

```
STA（单线程套间）：
┌──────────────────────────┐
│ 套间 A（线程1 独占）        │
│                           │
│  COM 对象 X               │
│  COM 对象 Y               │
│                           │
│  线程2 想调用对象 X？       │
│  → 必须排队，通过消息泵     │
│    转发到线程1 执行         │
└──────────────────────────┘
所有调用被串行化，COM 对象不需要自己处理线程安全

MTA（多线程套间）：
┌──────────────────────────┐
│ 套间 B（所有 MTA 线程共享）  │
│                           │
│  COM 对象 Z               │
│                           │
│  线程1 调用对象 Z  ─┐      │
│  线程2 调用对象 Z  ─┤ 并发  │
│  线程3 调用对象 Z  ─┘      │
│  → COM 对象自己负责线程安全  │
└──────────────────────────┘
调用可以并发，但 COM 对象必须线程安全
```

------

### 🎯 三、各自的使用场景

#### STA — UI 线程和非线程安全的 COM 组件

```csharp
// WinForms / WPF 主线程必须是 STA
[STAThread]
static void Main()
{
    Application.Run(new MainForm());
}
// UI 控件底层是 COM 组件，要求所有操作通过消息循环在同一线程上串行执行
```

其他需要 STA 的场景：Office Interop（Word/Excel 自动化）、剪贴板操作、拖放操作。

#### MTA — 后台工作线程和线程安全的 COM 组件

```csharp
// 后台线程调用线程安全的 COM 组件（如 VisionPro）
Thread thread = new Thread(WorkerProc);
thread.SetApartmentState(ApartmentState.MTA);
thread.Start();
```

------

### 🏭 四、工业视觉中的实际应用（VisionPro）

VisionPro 的核心对象（`CogToolBlock`、`CogImage` 等）底层是 COM 组件。套间模型直接影响检测吞吐量：

```
STA 线程调用 VisionPro：
  检测任务1: ToolBlock.Run() → [等消息泵调度] → 执行 → 完成
  检测任务2:                    [排队等待...]   → 执行 → 完成
  → 多个 ToolBlock 也只能串行，吞吐量低

MTA 线程调用 VisionPro：
  检测任务1: ToolBlock.Run() → 直接执行 ┐
  检测任务2: ToolBlock.Run() → 直接执行 ┤ 真正并发
  检测任务3: ToolBlock.Run() → 直接执行 ┘
  → 多通道并行检测，吞吐量高
```

------

### ⚠️ 五、注意事项

```csharp
// 1. 必须在 Start() 之前调用，否则抛 InvalidOperationException
Thread thread = new Thread(WorkerProc);
thread.SetApartmentState(ApartmentState.MTA);  // ✅ Start 之前
thread.Start();

// 2. 线程池线程和 Task.Run 的线程默认都是 MTA，且不能修改
// 如果需要 STA 线程，只能用 new Thread
Thread staThread = new Thread(StaWork);
staThread.SetApartmentState(ApartmentState.STA);
staThread.Start();
```

------

### ✅ 总结一句话

> STA 让 COM 调用自动串行化（UI 线程必须用 STA），MTA 允许多线程并发调用 COM 组件（后台工作线程用 MTA）。在 VisionPro 工业视觉项目中，检测线程必须用 MTA 才能实现多通道并行检测。线程池线程默认 MTA 且不可修改，需要 STA 时只能用 `new Thread`。

## 23	COM 组件（Component Object Model）是什么

> 来源：讨论 `SetApartmentState(ApartmentState.MTA)` 时引出的 COM 基础概念，结合 VisionPro 在项目中的使用场景。

------

### 🔍 一、定义

COM 是微软在 1993 年提出的一套**二进制级别的组件接口标准**。它定义了组件之间如何通信，**与编程语言无关** — 用 C++、C#、VB、Delphi 写的代码，只要遵循 COM 接口规范，就能互相调用。

```
传统调用：
  C# 程序 → 直接调用 C# 类库（同语言/同平台）

COM 调用：
  C# 程序 → COM 接口 → C++ 写的 DLL（跨语言，双方遵守 COM 规范即可）
  VB 程序 → COM 接口 → 同一个 C++ DLL（换语言也能调同一个组件）
```

------

### 📋 二、核心特征

| 特征 | 说明 |
|------|------|
| **接口驱动** | 通过接口（IUnknown、IDispatch 等）通信，不直接访问实现 |
| **二进制兼容** | 编译后的 DLL/EXE 直接互操作，不需要源码 |
| **语言无关** | C++、VB、C#、Delphi 等都能创建和使用 COM 组件 |
| **引用计数** | 通过 `AddRef()` / `Release()` 管理生命周期（不是 GC） |
| **注册到系统** | 通过注册表（CLSID/ProgID）注册，任何程序都能找到并使用 |

------

### 🗂️ 三、常见的 COM 组件

#### 1. Windows 系统内置

```
Shell32.dll        → 文件对话框、快捷方式、回收站操作
DirectShow         → 音视频播放和采集
DirectX            → 游戏图形渲染
WinHTTP / XMLHTTP  → 早期 HTTP 请求组件
```

#### 2. UI 控件（ActiveX）

WinForms 控件底层就是 ActiveX（COM 的 UI 扩展）：

```csharp
Button button = new Button();              // 底层 Windows USER32 + COM
WebBrowser browser = new WebBrowser();     // IE 的 ActiveX 控件
```

这就是 WinForms 主线程必须标记 `[STAThread]` 的根本原因。

#### 3. Office 自动化

```csharp
var excel = new Microsoft.Office.Interop.Excel.Application();  // COM
var word = new Microsoft.Office.Interop.Word.Application();    // COM
```

#### 4. 工业视觉 / 自动化

```csharp
// VisionPro — Cognex 视觉库，核心全是 COM 组件
CogToolBlock toolBlock = new CogToolBlock();      // COM
CogImage8Grey image = new CogImage8Grey();         // COM
CogSerializer.LoadObjectFromFile("test.ins");      // COM

// OPC（OLE for Process Control）— 工业通信标准，早期版本基于 COM
```

------

### 🔗 四、COM 与 .NET 的关系

```
COM 时代（1993~2002）：C++ / VB6 写 COM → 注册到系统 → 任何语言调用
.NET 时代（2002~至今）：.NET 类库逐渐替代 COM → 但大量存量 COM 仍在使用

桥梁：
  .NET 调用 COM → RCW（Runtime Callable Wrapper）自动包装
  COM 调用 .NET → CCW（COM Callable Wrapper）反向包装
```

```csharp
// 实际调用链：
CogToolBlock toolBlock = new CogToolBlock();
// C# 代码 → RCW 包装层 → VisionPro COM DLL（C++ 实现）→ 底层算法
// new 的不是纯 C# 对象，而是通过 RCW 创建了一个 COM 对象
```

------

### ⚠️ 五、COM 对象的生命周期管理

COM 用**引用计数**而非 GC 管理生命周期，所以在 C# 中使用 COM 对象时需要注意手动释放：

```csharp
// COM 对象需要 Dispose，不能完全依赖 GC
CogToolBlock toolBlock = new CogToolBlock();
try
{
    toolBlock.Run();
}
finally
{
    toolBlock.Dispose();  // ← 释放 COM 引用计数，回收非托管资源
}
```

如果不 Dispose，COM 对象的非托管内存不会立即释放，需要等 GC 调用终结器（Finalizer）时才通过 `Marshal.ReleaseComObject` 释放，时机不可控。

------

### 🔎 六、怎么判断是不是 COM 组件

```
1. DLL 需要用 regsvr32 注册 → 是 COM
2. 注册表 HKEY_CLASSES_ROOT\CLSID 下有条目 → 是 COM
3. VS 中"添加引用"在 COM 标签页出现 → 是 COM
4. 引用的程序集名称包含"Interop" → 是 COM 的 .NET 包装
5. 实现了 IUnknown 接口 → 是 COM
```

------

### ✅ 总结一句话

> COM 是微软定义的跨语言二进制组件标准，Windows 系统、Office、工业视觉库（VisionPro）等大量使用。C# 通过 RCW 自动包装调用 COM 组件，但需要注意两点：线程套间模型（STA/MTA）影响并发行为，引用计数机制要求及时 Dispose 而非依赖 GC。

------

## 24 WinForms 跨线程 UI 操作：UI 线程、消息循环与 Invoke

### 🧵 一、UI 线程是什么

**UI 线程就是调用 `Application.Run()` 的那个线程。**

WinForms 程序的入口 `Program.cs`：

```csharp
static void Main()
{
    Application.EnableVisualStyles();
    Application.Run(new FormDemo());  // ← 这行所在的线程就是 UI 线程
}
```

`Application.Run()` 在该线程上启动一个 **Windows 消息循环**，线程从此"卡"在循环里不断取消息并分发，直到程序退出。所有的窗口事件（点击、重绘、键盘输入）都在这个线程上执行。

------

### 🔄 二、Windows 消息循环

`Application.Run()` 内部本质上是：

```c
while (GetMessage(&msg, NULL, 0, 0))
{
    TranslateMessage(&msg);
    DispatchMessage(&msg);   // 分发给对应窗口的 WndProc 处理
}
```

常见消息类型：鼠标点击（`WM_LBUTTONDOWN`）、键盘输入（`WM_KEYDOWN`）、重绘（`WM_PAINT`）、自定义消息（`WM_USER + N`）。

**为什么控件操作必须在 UI 线程上？**
Win32 规定：**HWND（窗口句柄）只能被创建它的线程访问和操作**。WinForms 的所有控件底层都是 HWND，因此修改控件属性、操作控件集合等，都必须在 UI 线程上执行，否则行为未定义（通常抛 `InvalidOperationException`）。

------

### 🪝 三、Handle 如何绑定到 UI 线程

WinForms 中 `Control.Handle`（HWND）是**懒创建**的——`new Form()` 不会立刻创建 Handle，而是在控件第一次需要与系统交互时才触发 Win32 `CreateWindowEx()`。

**`Show()` 触发 Handle 创建，且必须在 UI 线程调用**，因此 Handle 的归属线程 = 调用 `Show()` 的线程。

**"傀儡 Form" 技巧**（来自本框架 `InsGlobalValue.Init()`）：

```csharp
// 在程序启动（UI 线程）时执行
InvokeForm = new Form();
InvokeForm.Show();   // ← 强制在 UI 线程创建 Handle，HWND 归属 UI 线程
InvokeForm.Hide();   // ← 立即隐藏，用户不可见
```

这个不可见的 Form 唯一的作用是**持有 UI 线程的 HWND**，让框架层的后台线程可以通过它 marshal 到 UI 线程，无需持有业务层主窗体的引用。

检查 Handle 是否已创建：

```csharp
bool created = InvokeForm.IsHandleCreated;  // true = Handle 已存在
```

------

### 🔍 四、InvokeRequired 的底层原理

`InvokeRequired` 本质是调用 Win32 API `GetWindowThreadProcessId(Handle)` 取得**创建该 HWND 的线程 ID（即 UI 线程 ID）**，再与当前线程 ID 做相等比较：

| 比较结果 | 含义 | `InvokeRequired` |
|----------|------|-----------------|
| 两者**相等** | 当前就是 UI 线程 | `false`，可直接操作控件 |
| 两者**不相等** | 当前是后台线程 | `true`，必须通过 `Invoke()` marshal |

**标准跨线程操作模板：**

```csharp
if (control.InvokeRequired)
{
    control.Invoke(new Action(() => { /* UI 操作 */ }));
}
else
{
    /* UI 操作 */
}
```

------

### 📨 五、Control.Invoke() 的工作原理

`Control.Invoke(delegate)` 内部通过 **`SendMessage(HWND, WM_INVOKE, ...)`** 向 UI 线程的消息队列投递一条携带委托的自定义消息，然后**阻塞等待**。UI 线程消息循环取到该消息后执行委托，执行完毕后解除阻塞。

```
后台线程
  └─ control.Invoke(action)
       └─ SendMessage(HWND, WM_INVOKE, ...)   ← 阻塞，等待 UI 线程执行完
            └─ UI 线程消息循环收到
                 └─ 执行 action()
                      └─ 返回，后台线程继续
```

`BeginInvoke` 则使用 `PostMessage`（异步投递，立即返回，不等待执行完成）。

| 方法 | 阻塞调用方？ | 对应 Win32 API | 适用场景 |
|------|------------|---------------|---------|
| `Invoke` | ✅ 是 | `SendMessage` | 需要等待 UI 操作完成后继续 |
| `BeginInvoke` | ❌ 否 | `PostMessage` | 只需通知 UI 更新，不关心完成时机 |

------

### 🏭 六、应用场景：框架层后台线程操作 VisionPro CogToolBlock

来自本框架 `InsImageSaveManager.InspectionRan()`：

```csharp
// InspectionRan 在后台任务线程调用
if (InsGlobalValue.GetSingleInstance.InvokeForm.InvokeRequired)
{
    InsGlobalValue.GetSingleInstance.InvokeForm.Invoke(new Action(() =>
    {
        mInspection.Inputs.Add(new CogToolBlockTerminal("Guid", ""));
    }));
}
else
{
    mInspection.Inputs.Add(new CogToolBlockTerminal("Guid", ""));
}
```

`CogToolBlock.Inputs.Add()` 是 VisionPro 的 UI 相关操作，底层涉及 COM STA 线程模型，**必须在 UI 线程上执行**。框架层无法直接引用业务层主窗体，因此用 `InvokeForm`（傀儡 Form）作为跳板。

------

### ✅ 总结

> WinForms 的 UI 线程 = `Application.Run()` 所在线程，通过消息循环独占 HWND 的操作权。后台线程要改控件或操作 COM UI 对象，必须通过 `Invoke()`/`BeginInvoke()` 将委托投递到 UI 线程执行。"傀儡 Form"模式（`Show()` 后立即 `Hide()`）是框架层在无主窗体引用时获得 UI 线程 marshal 能力的常用技巧。

------

## 25 is 类型检查与循环内重复强转

### 🔍 一、问题：循环内反复强转

当用 `is` 做类型判断后，若没有保存转换结果，循环体内每次索引都需要重新强转：

```csharp
// ❌ 低效写法：每次循环都执行一次 (ArrayList)value
else if (value is ArrayList && ((ArrayList)value).Count > 0)
{
    for (int j = 0; j < ((ArrayList)value).Count; j++)
    {
        var img = (ICogImage)((ArrayList)value)[j];  // 每次迭代强转一次
    }
}
```

外层 `value is ArrayList` 只做了类型检查，**没有保存转换结果**，导致循环内 `((ArrayList)value)` 重复执行类型转换。

------

### ⚡ 二、引用类型强转的性能

引用类型的强转（downcast）本质是：

```
检查 value 的类型指针 == 目标类型指针 → 直接返回同一个对象指针
```

**不分配内存，不复制数据**，JIT 会进一步优化，单次纳秒级。但在代码层面属于冗余操作，可读性差，应当避免。

------

### ✅ 三、解决方案：C# 7 模式匹配绑定变量

`is Type variable` 语法在**一句话内同时完成类型检查和变量绑定**，彻底消除循环内重复强转：

```csharp
// ✅ 正确写法：is 检查时直接绑定 al，后续直接用
else if (value is ArrayList al && al.Count > 0 && al[0] is ICogImage)
{
    for (int j = 0; j < al.Count; j++)
    {
        var img = (ICogImage)al[j];  // al 已经是 ArrayList，无需再 cast value
    }
}
```

`al` 就是通过类型检查后的 `value` 引用，作用域从 `&&` 之后一直延伸到 `else if` 块结束。

------

### 🔄 四、进一步优化：归一化为统一集合类型

当同一逻辑需要处理多种容器类型（`ICogImage`、`ArrayList`、`List<ICogImage>`、`ICogImage[]`）时，可以先归一化为 `IEnumerable<ICogImage>`，核心逻辑只写一份：

```csharp
IEnumerable<ICogImage> images = null;
if      (value is ICogImage single)                              images = new[] { single };
else if (value is ICogImage[] arr   && arr.Length > 0)          images = arr;
else if (value is List<ICogImage> list && list.Count > 0)       images = list;
else if (value is ArrayList al && al.Count > 0
         && al[0] is ICogImage)                                  images = al.Cast<ICogImage>();

if (images != null)
{
    int j = 0;
    foreach (var img in images)
    {
        // 核心逻辑只写一次，不随容器类型数量增加而膨胀
        ProcessImage(img, j++);
    }
}
```

`al.Cast<ICogImage>()` 是 LINQ 懒求值，不会提前分配额外集合。

------

### ✅ 总结

> 用 `is Type variable` 模式匹配代替裸 `is` + 循环内重复强转，既消除冗余，又提升可读性。多容器类型分支重复同一逻辑时，应先归一化为统一的 `IEnumerable<T>`，使核心逻辑只出现一次。

------

## 26 单例模式与 Lazy\<T\>

### 🏗️ 一、单例模式的标准写法

工控框架中常见写法：

```csharp
public class InsImageSaveManager
{
    private static readonly Lazy<InsImageSaveManager> _instance =
        new Lazy<InsImageSaveManager>(() => new InsImageSaveManager());

    private InsImageSaveManager() { }

    public static InsImageSaveManager GetSingleInstance() => _instance.Value;
}
```

### 💤 二、为什么用 `Lazy<T>` 而不是直接赋值

**方式一：直接赋值（饿汉式）**

```csharp
private static readonly InsImageSaveManager _instance = new InsImageSaveManager();
```

类加载时立即构造，无论是否用到都会执行初始化逻辑（开线程、分配队列等）。

**方式二：手写双重检查锁（懒汉式）**

```csharp
private static InsImageSaveManager _instance;
private static readonly object _lock = new object();

public static InsImageSaveManager GetSingleInstance()
{
    if (_instance == null)
        lock (_lock)
            if (_instance == null)
                _instance = new InsImageSaveManager();
    return _instance;
}
```

线程安全，但代码冗长且容易写错（漏掉内层 null 检查、忘记 volatile 等）。

**方式三：`Lazy<T>`（推荐）**

```csharp
private static readonly Lazy<InsImageSaveManager> _instance =
    new Lazy<InsImageSaveManager>(() => new InsImageSaveManager());
```

- **懒加载**：第一次调用 `.Value` 时才构造，从不使用则从不初始化
- **线程安全**：默认使用 `LazyThreadSafetyMode.ExecutionAndPublication`，内部已处理锁，等价于双重检查锁但不会写错
- **简洁**：一行替代上面十几行

### 🔁 三、工控软件为什么大量使用单例

**合理的原因：**

1. **硬件资源天然唯一**：相机、PLC、数据库连接在物理上只有一套，单例在软件层面强制这个约束，避免多处各自创建导致资源冲突或重复初始化
2. **全局状态需要共享**：任务管理器、日志、配置等整个进程生命周期内所有模块都访问同一份状态，单例提供统一入口，无需到处传引用
3. **单进程单机运行**：工控软件跑在专用工控机上，不存在多实例部署场景，单例的主要弊端几乎不会触发

### ⚠️ 四、单例模式的代价（互联网系统中更突出）

**1. 隐式依赖**

依赖藏在方法体内部，从类的构造函数签名完全看不出来：

```csharp
// 显式依赖：构造函数一眼看出依赖什么
public class TaskProcessor {
    public TaskProcessor(ILogger logger, IDatabase db) { ... }
}

// 隐式依赖：依赖藏在方法里，外部不可见
public class TaskProcessor {
    public void Run() {
        InsLogManager.GetSingleInstance().Log("start");      // 隐式依赖
        InsDatabaseManager.GetSingleInstance().Save(result); // 隐式依赖
    }
}
```

依赖越多调用越深，越难搞清楚改动某个单例会影响哪些地方。

**2. 难以单元测试**

单例无法被替换为 Mock 对象，测试时要么依赖真实硬件/数据库，要么直接崩溃：

```csharp
// 单例：测试时 InsDatabaseManager 无真实连接，直接崩
[Test]
public void Run_WithInvalidData_ShouldNotSave() {
    var processor = new TaskProcessor();
    processor.Run(invalidData); // 内部调用单例，测试环境无法控制
}

// 接口注入：测试时可替换为 Mock，精确验证行为
[Test]
public void Run_WithInvalidData_ShouldNotSave() {
    var mockDb = new Mock<IDatabase>();
    var processor = new TaskProcessor(mockDb.Object);
    processor.Run(invalidData);
    mockDb.Verify(db => db.Save(It.IsAny<Result>()), Times.Never); // 精确断言
}
```

### ✅ 总结

| | 工控软件 | 互联网业务系统 |
|---|---|---|
| 需求变化频率 | 低，交付后稳定 | 高，频繁迭代 |
| 单元测试必要性 | 低，以系统集成测试为主 | 高，回归风险大 |
| 隐式依赖影响 | 小，小团队模块边界固定 | 大，多人协作易踩坑 |
| 硬件全局唯一 | 是，单例符合现实 | 否 |

> 单例模式没有对错之分，代价在工控场景里几乎不会兑现，而带来的便利（无需传引用、初始化顺序简单）却实实在在。

---

## 27 HTTP 方法全览与对比

### 方法一览

| 方法 | 语义 | 请求体 | 幂等 | 安全 | 常见用途 |
|------|------|--------|------|------|---------|
| **GET** | 获取资源 | ❌ | ✅ | ✅ | 查询数据、读取配置 |
| **POST** | 提交/创建资源 | ✅ | ❌ | ❌ | 新增记录、触发操作 |
| **PUT** | 全量替换资源 | ✅ | ✅ | ❌ | 更新整条记录 |
| **PATCH** | 局部更新资源 | ✅ | ❌ | ❌ | 只改某几个字段 |
| **DELETE** | 删除资源 | ❌/可选 | ✅ | ❌ | 删除记录 |
| **HEAD** | 只取响应头 | ❌ | ✅ | ✅ | 检查资源是否存在/获取大小 |
| **OPTIONS** | 查询服务器支持的方法 | ❌ | ✅ | ✅ | CORS 预检请求 |

> - **幂等**：多次相同请求的结果与执行一次相同（PUT 写同样数据结果不变；DELETE 删了还是删）
> - **安全**：不修改服务器资源（GET / HEAD / OPTIONS 只读）

---

### PUT vs PATCH 的区别

```
现有记录：{ "sn": "A001", "result": "OK", "score": 95 }

PUT  传入：{ "sn": "A001", "result": "NG" }
结果：    { "sn": "A001", "result": "NG" }              ← score 字段丢失

PATCH 传入：{ "result": "NG" }
结果：    { "sn": "A001", "result": "NG", "score": 95 } ← 只改 result
```

PUT 是**全量替换**，传入什么就是什么，未传字段视为删除；
PATCH 是**局部更新**，只改传入的字段，其余字段保持不变。

---

### 工业视觉场景的典型用法

```
GET    /api/product/{sn}    → 查 SN 对应的产品信息
POST   /api/result          → 上传检测结果（新增一条）
PUT    /api/recipe/{id}     → 替换整条配方参数
PATCH  /api/result/{id}     → 只更新 OK/NG 判定字段
DELETE /api/image/{id}      → 删除指定存图记录
HEAD   /api/health          → 心跳检测（不需要响应体，节省流量）
```

> **实际注意**：与 MES 对接时，MES 接口通常不严格遵循 REST 语义，大量操作都走 POST，方法名和语义通过请求体或 URL 路径来区分，而不是靠 HTTP 方法。

---

### C# 中的用法（HttpWebRequest / HttpClient）

```csharp
// HttpWebRequest（InsHttpHelper 的写法）
var request = (HttpWebRequest)WebRequest.Create(url);
request.Method = "PUT";   // 传入方法名字符串即可切换

// HttpClient（更现代的写法）
var client = new HttpClient();
await client.GetAsync(url);
await client.PostAsync(url, content);
await client.PutAsync(url, content);
await client.PatchAsync(url, content);
await client.DeleteAsync(url);
await client.SendAsync(new HttpRequestMessage(HttpMethod.Head, url));
```

## 28 List<T> vs 数组（Array）

### 核心区别

| 特性 | `T[]` 数组 | `List<T>` |
|------|-----------|-----------|
| 大小 | 固定，创建后不可变 | 动态，可增删 |
| 内存布局 | 连续内存块 | 内部包装一个数组 |
| 类型 | 引用类型（对象头 + 长度字段 + 元素） | 引用类型（对象头 + 字段） |

---

### 托管堆上的内存布局

**数组 `int[] arr = new int[4]`**

```
托管堆
┌─────────────────────────────────┐
│ Object Header (8 bytes, sync/MT) │
│ Method Table Pointer             │
│ Length = 4                       │
├─────────────────────────────────┤
│ [0] │ [1] │ [2] │ [3]           │  ← 连续内存，元素紧接在后
└─────────────────────────────────┘
```

**`List<int> list = new List<int>()`**

```
托管堆
┌─────────────────────────┐
│  List<int> 对象          │
│  _items  ──────────────────┐   ← 指向内部数组
│  _size   = 0              │
│  _version = 0             │
└─────────────────────────┘   │
                               ↓
                        ┌─────────────────┐
                        │  int[] (空数组)  │  默认 Array.Empty<T>
                        └─────────────────┘
```

> `List<T>` 本身是个**包装器对象**，真正的数据存在其内部的 `_items` 数组里。

---

### 扩容过程

**扩容触发时机**

```csharp
public void Add(T item) {
    if (_size == _items.Length)  // 容量满了
        EnsureCapacity(_size + 1);
    _items[_size++] = item;
}
```

**扩容策略：2 倍增长**

```csharp
private void EnsureCapacity(int min) {
    int newCapacity = _items.Length == 0
        ? DefaultCapacity          // 初始容量 = 4
        : _items.Length * 2;       // 否则翻倍

    if (newCapacity < min) newCapacity = min;
    Capacity = newCapacity;
}
```

容量增长序列：`0 → 4 → 8 → 16 → 32 → 64 → ...`

**数据迁移过程**

```
扩容前（_items.Length = 4, _size = 4）:
┌──────────────┐       ┌───────────────────┐
│  List 对象   │       │  旧数组 int[4]     │
│  _items ─────┼──────▶│ [1][2][3][4]      │
│  _size = 4   │       └───────────────────┘
└──────────────┘

扩容后（新数组 int[8]）:
┌──────────────┐       ┌───────────────────────────────┐
│  List 对象   │       │  新数组 int[8]                 │
│  _items ─────┼──────▶│ [1][2][3][4][ ][ ][ ][ ]      │
│  _size = 4   │       └───────────────────────────────┘
└──────────────┘
                        旧数组 int[4] → 变成垃圾，等待 GC 回收
```

迁移使用 `Array.Copy`（底层是 `memmove`，一次性内存复制）：

```csharp
set Capacity(int value) {
    T[] newItems = new T[value];
    if (_size > 0)
        Array.Copy(_items, newItems, _size);  // O(n) 复制
    _items = newItems;  // 替换引用
}
```

---

### 时间复杂度

- **单次 Add**：均摊 O(1)（偶尔触发 O(n) 复制，但扩容总次数是 log n）
- **扩容时内存峰值**：瞬间同时持有新旧两个数组，约为 **3 倍**内存占用

---

### 实践建议

```csharp
// 已知大小时预设容量，避免多次扩容
var list = new List<int>(1000);

// 用完后释放多余容量
list.TrimExcess();  // 将 _items 缩减到实际 _size 大小
```

## 29 线程间通信方式

### 1. 共享变量 + lock / Monitor

最基础的方式，通过共享内存传递数据，用锁保护访问。

```csharp
private readonly object _lock = new object();
private int _sharedData;

// 线程 A 写
lock (_lock) { _sharedData = 42; }

// 线程 B 读
lock (_lock) { Console.WriteLine(_sharedData); }
```

---

### 2. volatile（轻量级可见性保证）

禁止 CPU/编译器对变量读写进行缓存优化，确保每次都从主内存读取。

```csharp
private volatile bool _running = true;

// 线程 A
_running = false;  // 立即对其他线程可见

// 线程 B
while (_running) { /* 工作 */ }
```

> 只保证可见性，**不保证原子性**，不能替代 lock。

---

### 3. Interlocked（原子操作）

对简单数值的无锁原子读写，性能比 lock 好。

```csharp
private int _counter = 0;

Interlocked.Increment(ref _counter);
Interlocked.Add(ref _counter, 5);
int old = Interlocked.Exchange(ref _counter, 0);           // 原子赋值
int val = Interlocked.CompareExchange(ref _counter, 1, 0); // CAS
```

---

### 4. ManualResetEventSlim / AutoResetEvent（信号通知）

一个线程等待，另一个线程发信号唤醒。

```csharp
var signal = new AutoResetEvent(false);

// 线程 B：等待信号
signal.WaitOne();

// 线程 A：发出信号（唤醒一个等待者）
signal.Set();
```

| 类型 | 行为 |
|------|------|
| `AutoResetEvent` | Set 后自动复位，只唤醒一个线程 |
| `ManualResetEvent` | Set 后保持开启，唤醒所有等待线程，需手动 Reset |
| `ManualResetEventSlim` | 轻量版，短等待时用自旋代替内核等待，性能更好 |

---

### 5. SemaphoreSlim（限流信号量）

控制同时访问某资源的线程数量。

```csharp
var semaphore = new SemaphoreSlim(3); // 最多 3 个线程同时进入

await semaphore.WaitAsync();
try { /* 访问受限资源 */ }
finally { semaphore.Release(); }
```

---

### 6. BlockingCollection\<T\>（生产者/消费者队列，同步）

内置阻塞语义，消费者队列为空时自动阻塞等待。

```csharp
var queue = new BlockingCollection<int>(capacity: 100);

// 生产者线程
Task.Run(() => {
    for (int i = 0; i < 10; i++)
        queue.Add(i);         // 满了会阻塞
    queue.CompleteAdding();
});

// 消费者线程
Task.Run(() => {
    foreach (var item in queue.GetConsumingEnumerable())
        Console.WriteLine(item);
});
```

---

### 7. Channel\<T\>（现代异步管道）

.NET Core 引入，专为 async/await 设计，是 BlockingCollection 的异步替代。

```csharp
var channel = Channel.CreateBounded<int>(100);

// 生产者
await channel.Writer.WriteAsync(42);
channel.Writer.Complete();

// 消费者
await foreach (var item in channel.Reader.ReadAllAsync())
    Console.WriteLine(item);
```

---

### 8. TaskCompletionSource\<T\>（一次性异步结果）

将回调通知转换为可 await 的 Task，适合一次性结果传递。

```csharp
var tcs = new TaskCompletionSource<string>();

// 线程 A：等待结果
string result = await tcs.Task;

// 线程 B：设置结果（触发线程 A 继续执行）
tcs.SetResult("done");
```

---

### 选型参考

| 场景 | 推荐方式 |
|------|---------|
| 简单数值计数/标志位 | `Interlocked` / `volatile` |
| 通用数据保护 | `lock` |
| 一个线程通知另一个线程 | `AutoResetEvent` / `ManualResetEventSlim` |
| 限制并发数 | `SemaphoreSlim` |
| 持续传递数据流（同步） | `BlockingCollection<T>` |
| 持续传递数据流（异步） | `Channel<T>` |
| 一次性异步结果 | `TaskCompletionSource<T>` |

---

## 30 实战：两个重载线程的资源调度（Channel + SemaphoreSlim）

### 场景描述

- **线程 1**：不定时接收任务，执行时大量占用内存、CPU、IO
- **线程 2**：不定时接收任务，执行时大量占用 IO
- **目标**：线程 1 空闲时，线程 2 能立即全速处理积压任务，且不会长期饥饿

### 架构图

```
任务源 A ──▶ Channel<WorkA> ──▶ 线程1 worker
                                     │ 持有/释放
                                  ResourceGate (SemaphoreSlim)
                                     │ 等待/获取
任务源 B ──▶ Channel<WorkB> ──▶ 线程2 worker
```

### 基础实现

```csharp
var resourceGate = new SemaphoreSlim(1, 1);
var queueA = Channel.CreateUnbounded<WorkItemA>();
var queueB = Channel.CreateUnbounded<WorkItemB>();

// 线程 1：高优先级，持有门控期间独占资源
Task.Run(async () =>
{
    await foreach (var item in queueA.Reader.ReadAllAsync())
    {
        await resourceGate.WaitAsync();
        try   { await DoHeavyWorkAsync(item); }
        finally { resourceGate.Release(); }
    }
});

// 线程 2：见缝插针，线程 1 空闲时全速处理
Task.Run(async () =>
{
    await foreach (var item in queueB.Reader.ReadAllAsync())
    {
        await resourceGate.WaitAsync();
        try   { await DoHeavyIOAsync(item); }
        finally { resourceGate.Release(); }
    }
});
```

### 防饥饿：线程 1 连续占用时主动让步

```csharp
var resourceGate = new SemaphoreSlim(1, 1);
var thread2WaitingSince = new ConcurrentDictionary<int, DateTime>();

// 线程 1
Task.Run(async () =>
{
    await foreach (var item in queueA.Reader.ReadAllAsync())
    {
        // 线程 2 等待超过 500ms，主动短暂让步一次
        bool thread2Starving = thread2WaitingSince.Values
            .Any(t => (DateTime.UtcNow - t).TotalMilliseconds > 500);

        if (thread2Starving)
            await Task.Delay(10);

        await resourceGate.WaitAsync();
        try   { await DoHeavyWorkAsync(item); }
        finally { resourceGate.Release(); }
    }
});

// 线程 2
Task.Run(async () =>
{
    await foreach (var item in queueB.Reader.ReadAllAsync())
    {
        thread2WaitingSince[item.Id] = DateTime.UtcNow;

        await resourceGate.WaitAsync();
        thread2WaitingSince.TryRemove(item.Id, out _);
        try   { await DoHeavyIOAsync(item); }
        finally { resourceGate.Release(); }
    }
});
```

### 时序效果

```
任务流：  A1  A2        A3   A4
线程1：  ████████░░░░░████████░░░░░
线程2：  ░░░░░░░░████░░░░░░░░░████
                 ↑窗口期         ↑窗口期
               线程2积压任务全速处理
```

### 线程 2 强实时性：线程 1 任务分段执行

线程 1 的重载任务拆为多个阶段，阶段间短暂释放门控：

```csharp
async Task DoHeavyWorkAsync(WorkItemA item)
{
    await PhaseMemoryLoad(item);
    resourceGate.Release();           // 阶段间释放，给线程2插入机会
    await Task.Yield();
    await resourceGate.WaitAsync();

    await PhaseCpuCompute(item);
    resourceGate.Release();
    await Task.Yield();
    await resourceGate.WaitAsync();

    await PhaseIO(item);
}
```

### 方案选型

| 需求 | 方案 |
|------|------|
| 线程 2 偶尔等待可接受 | 双 Channel + SemaphoreSlim |
| 线程 2 不能饥饿 | 加超时让步检测 |
| 线程 2 强实时性 | 线程 1 任务拆段，阶段间释放门控 |
| 线程 1 任务可中断 | `CancellationToken` 配合，线程 2 高优先级时取消线程 1 当前任务 |

## 31 WinForms 控件分辨率自适应

### 1. AutoScaleMode（必须设置）

```csharp
this.AutoScaleMode = AutoScaleMode.Dpi;  // 推荐：按屏幕 DPI 缩放
```

| 值 | 行为 |
|----|------|
| `Dpi` | 按屏幕 DPI 比例缩放，适合高分屏 |
| `Font` | 按系统默认字体大小缩放，传统方式 |
| `None` | 不缩放（不推荐） |

---

### 2. Anchor（控件跟随窗体边缘）

```csharp
btnOK.Anchor   = AnchorStyles.Bottom | AnchorStyles.Right;      // 贴右下角
txtInput.Anchor = AnchorStyles.Left | AnchorStyles.Right | AnchorStyles.Top; // 横向填满
```

---

### 3. Dock（填充父容器）

```csharp
panel.Dock     = DockStyle.Fill;
menuStrip.Dock = DockStyle.Top;
statusBar.Dock = DockStyle.Bottom;
```

---

### 4. TableLayoutPanel（比例布局，最灵活）

```csharp
var table = new TableLayoutPanel();
table.Dock = DockStyle.Fill;
table.ColumnCount = 2;
table.RowCount = 3;

table.ColumnStyles.Add(new ColumnStyle(SizeType.Percent, 30F));
table.ColumnStyles.Add(new ColumnStyle(SizeType.Percent, 70F));
table.RowStyles.Add(new RowStyle(SizeType.Percent, 10F));
table.RowStyles.Add(new RowStyle(SizeType.Percent, 80F));
table.RowStyles.Add(new RowStyle(SizeType.Percent, 10F));
```

---

### 5. 高 DPI 支持（app.manifest）

不配置此项，系统会对程序做模糊放大处理。

```xml
<application xmlns="urn:schemas-microsoft-com:asm.v3">
  <windowsSettings>
    <dpiAware xmlns="http://schemas.microsoft.com/SMI/2005/WindowsSettings">
      true/PM
    </dpiAware>
    <dpiAwareness xmlns="http://schemas.microsoft.com/SMI/2016/WindowsSettings">
      PerMonitorV2
    </dpiAwareness>
  </windowsSettings>
</application>
```

程序入口代码（.NET 6+）：

```csharp
Application.SetHighDpiMode(HighDpiMode.PerMonitorV2);
Application.EnableVisualStyles();
Application.Run(new MainForm());
```

| 模式 | 说明 |
|------|------|
| `SystemAware` | 按主屏 DPI 缩放，副屏可能模糊 |
| `PerMonitor` | 每个显示器独立 DPI |
| `PerMonitorV2` | 推荐，子窗体、对话框也跟着缩放 |

---

### 6. 字体使用 Point 单位

```csharp
label.Font = new Font("微软雅黑", 9F, FontStyle.Regular, GraphicsUnit.Point);
```

---

### 7. 手动 DPI 缩放（兜底方案）

```csharp
private float GetDpiScale()
{
    using var g = this.CreateGraphics();
    return g.DpiX / 96f;
}

protected override void OnDpiChanged(DpiChangedEventArgs e)
{
    base.OnDpiChanged(e);
    float scale = GetDpiScale();
    btnOK.Width  = (int)(80 * scale);
    btnOK.Height = (int)(30 * scale);
}
```

---

### 推荐组合

```
必做：AutoScaleMode = Dpi + app.manifest PerMonitorV2 + SetHighDpiMode
布局：TableLayoutPanel 比例布局 + 叶子控件 Anchor/Dock + 字体 Point 单位
兜底：重写 OnDpiChanged 手动重算尺寸
```

## 32 反射（Reflection）

### 什么是反射

在**运行时**动态获取类型信息、并操作对象的能力。允许你在运行时查询类型结构，并动态创建对象、调用方法、读写属性。

```csharp
// 正常方式：编译时确定
var obj = new MyClass();
obj.DoSomething();

// 反射方式：运行时确定
Type type = Type.GetType("MyNamespace.MyClass");
object obj = Activator.CreateInstance(type);
MethodInfo method = type.GetMethod("DoSomething");
method.Invoke(obj, null);
```

---

### 底层机制

#### 元数据（Metadata）是基础

编译器将源码编译为 IL 时，会在程序集中写入元数据表：

```
程序集文件结构
┌─────────────────────────────────┐
│  元数据表（Metadata Tables）     │  ← 反射的数据来源
│   TypeDef   - 所有类型定义       │
│   MethodDef - 所有方法定义       │
│   FieldDef  - 所有字段定义       │
│   CustomAttribute - 特性标注     │
│   ...共 40+ 张表                 │
├─────────────────────────────────┤
│  IL 代码（方法体）               │
└─────────────────────────────────┘
```

#### Type 对象：运行时的类型描述符

CLR 为每个已加载的类型在托管堆上创建唯一一个 `RuntimeType` 对象：

```
RuntimeType（MyClass）
├─ Name = "MyClass"
├─ Methods[]    → [MethodInfo, MethodInfo, ...]
├─ Fields[]     → [FieldInfo, ...]
├─ Properties[] → [PropertyInfo, ...]
└─ BaseType     → RuntimeType（object）
```

获取 Type 的三种方式：

```csharp
Type t1 = typeof(MyClass);          // 编译时已知，最快
Type t2 = obj.GetType();            // 从实例获取
Type t3 = Type.GetType("全限定名"); // 按名字查找，最慢
```

#### MethodInfo.Invoke() 的调用路径

```
method.Invoke(obj, args)
  1. 参数合法性检查
  2. object[] 参数装箱/拆箱
  3. 权限检查
  4. 通过方法表找到 IL 地址
  5. 调用目标方法
  6. 返回值装箱为 object
```

相比直接调用慢约 **10~100 倍**，主要开销在装箱和参数数组分配。

---

### 常用 API

```csharp
Type type = typeof(MyClass);

MethodInfo   method   = type.GetMethod("DoWork", BindingFlags.Public | BindingFlags.Instance);
PropertyInfo property = type.GetProperty("Name");
FieldInfo    field    = type.GetField("_value", BindingFlags.NonPublic | BindingFlags.Instance);

object instance = Activator.CreateInstance(type);

method.Invoke(instance, new object[] { arg1, arg2 });
property.SetValue(instance, "hello");
field.SetValue(instance, 42);          // 可绕过访问修饰符读写私有字段

var attr = type.GetCustomAttribute<ObsoleteAttribute>();
```

---

### 性能优化

| 问题 | 优化方式 |
|------|---------|
| 反复调用同一方法 | 缓存 `MethodInfo`，避免重复 `GetMethod` |
| `Invoke` 装箱开销 | `CreateDelegate` 转为强类型委托，接近直接调用 |
| 大量动态创建对象 | `Expression.Lambda` 编译为委托后缓存 |
| .NET 7+ | Source Generator 在编译期生成代码，彻底消除运行时反射 |

```csharp
// 转为委托，后续调用无反射开销
var action = (Action<MyClass>)method.CreateDelegate(typeof(Action<MyClass>));
action(instance);
```

---

### 典型使用场景

- **依赖注入容器**（Autofac、Microsoft.DI）：扫描程序集，自动注册和创建对象
- **ORM**（EF Core、Dapper）：将数据库列映射到实体属性
- **序列化**（System.Text.Json、Newtonsoft.Json）：遍历对象属性读写
- **单元测试框架**（xUnit、NUnit）：发现并执行标有 `[Fact]` / `[Test]` 的方法
- **插件系统**：运行时加载外部 dll，动态调用其中实现了某接口的类

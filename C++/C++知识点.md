# C++知识点

## 1	ostringstream怎么使用

**`ostringstream` 简介**

`ostringstream` 是 C++ 标准库 `<sstream>` 中的一个类，用于像操作流（如 `cout`）一样构建字符串。它可以将各种类型的数据（如 `int`、`double`、`string` 等）拼接成一个完整的字符串。

------

**基本用法**

```
#include <sstream>
#include <iostream>

int main() {
    std::ostringstream oss;  // 创建一个字符串输出流
    oss << "数字: " << 123 << ", 浮点数: " << 3.14;  // 写入数据
    
    std::string result = oss.str();  // 获取拼接后的字符串
    std::cout << result << std::endl;  // 输出: "数字: 123, 浮点数: 3.14"
    
    return 0;
}
```

------

**核心功能**

1. **自动类型转换**：数字、布尔值等会自动转成字符串。
2. **高效拼接**：比 `+` 或 `append` 拼接字符串更高效（尤其多次操作时）。
3. **格式化控制**：可结合 `std::fixed`、`std::setprecision` 等控制格式。

------

**常用方法**

| 方法             | 说明                       |
| :--------------- | :------------------------- |
| `.str()`         | 返回当前存储的字符串       |
| `.str("新内容")` | 重置流的内容               |
| `.clear()`       | 清除错误状态（不影响内容） |

------

**进阶示例（格式化）**

```
#include <sstream>
#include <iomanip>  // 用于格式控制

int main() {
    std::ostringstream oss;
    oss << std::fixed << std::setprecision(2) << 3.14159;  // 保留两位小数
    std::cout << oss.str() << std::endl;  // 输出 "3.14"
    return 0;
}
```

------

**适用场景**

* 动态生成复杂字符串（如日志、JSON 拼接）。
* 需要混合输出多种数据类型时（比 `printf` 更安全）。
* 替代 `to_string` + `+` 的繁琐拼接方式。

------

**相关类**

* `istringstream`：从字符串读取数据（类似 `cin`）。
* `stringstream`：既可读又可写的字符串流。

---

## 2 Rule of Three / Five / Zero

### 核心思想

> 如果一个类需要自定义析构函数，说明它在管理资源（堆内存、文件句柄等），此时编译器自动生成的拷贝/移动操作往往是错误的浅拷贝，必须一并处理。

---

### 三个原则

**Rule of Zero（最推荐）**

类成员全部使用值类型或智能指针，无需自定义任何特殊函数，编译器生成的完全正确。

```cpp
class Person {
    std::string name_;  // string 自己管理内存
    int age_;
    // 什么都不用写
};
```

**Rule of Five（管理原始资源时必须）**

一旦自定义了析构函数，就必须同时考虑以下五个函数：
1. 析构函数
2. 拷贝构造函数
3. 拷贝赋值运算符
4. 移动构造函数
5. 移动赋值运算符

**Rule of Three（C++11 之前）**

移动语义不存在，只需考虑析构、拷贝构造、拷贝赋值。

---

### 编译器默认行为

| 你声明了 | 拷贝构造 | 拷贝赋值 | 移动构造 | 移动赋值 |
|---|---|---|---|---|
| 什么都没写 | 自动生成 | 自动生成 | 自动生成 | 自动生成 |
| 析构函数 | 自动生成⚠️ | 自动生成⚠️ | **不生成** | **不生成** |
| 拷贝构造 | — | 自动生成⚠️ | **不生成** | **不生成** |
| 移动构造 | **删除** | **删除** | — | **不生成** |

⚠️ 虽然会生成，但可能是错误的浅拷贝。

---

### Rule of Five 完整示例

```cpp
class Buffer {
public:
    explicit Buffer(size_t size, const char* data = nullptr) : size_(size) {
        data_ = new char[size_];
        if (data) std::memcpy(data_, data, size_);
        else      std::memset(data_, 0, size_);
    }

    // 1. 析构函数
    ~Buffer() {
        delete[] data_;
    }

    // 2. 拷贝构造 —— 深拷贝
    Buffer(const Buffer& other) : size_(other.size_) {
        data_ = new char[size_];
        std::memcpy(data_, other.data_, size_);
    }

    // 3. 拷贝赋值 —— copy-and-swap 惯用法（强异常安全）
    Buffer& operator=(const Buffer& other) {
        if (this == &other) return *this;
        Buffer tmp(other);   // 若 new 抛异常，this 完全未动
        swap(tmp);           // swap 不会抛异常
        return *this;        // tmp 析构时自动释放旧资源
    }

    // 4. 移动构造 —— 转移所有权，置空原对象
    Buffer(Buffer&& other) noexcept : data_(other.data_), size_(other.size_) {
        other.data_ = nullptr;
        other.size_ = 0;
    }

    // 5. 移动赋值 —— 先释放自身旧资源，再转移
    Buffer& operator=(Buffer&& other) noexcept {
        if (this == &other) return *this;
        delete[] data_;          // 必须先释放，否则内存泄漏
        data_ = other.data_;
        size_ = other.size_;
        other.data_ = nullptr;
        other.size_ = 0;
        return *this;
    }

private:
    void swap(Buffer& other) noexcept {
        std::swap(data_, other.data_);
        std::swap(size_, other.size_);
    }

    char*  data_ = nullptr;
    size_t size_ = 0;
};
```

---

### 关键设计点

| 函数 | 要点 |
|---|---|
| 析构 | `delete[]` 释放资源 |
| 拷贝构造 | `new` + `memcpy`，深拷贝 |
| 拷贝赋值 | copy-and-swap，天然异常安全且处理自赋值 |
| 移动构造 | 转移指针后**置空原对象**，必须加 `noexcept` |
| 移动赋值 | 先 `delete` 释放旧资源再转移，必须加 `noexcept` |

**`noexcept` 必须加**：`std::vector` 扩容时只有移动函数标了 `noexcept` 才会调用移动而非拷贝。

---

### 为什么拷贝赋值用 copy-and-swap 而不直接赋值

直接赋值的问题：

```cpp
// 危险写法
Buffer& operator=(const Buffer& other) {
    delete[] data_;            // 1. 旧资源已释放
    data_ = new char[size_];   // 2. 若此处抛异常，data_ 成悬空指针，对象损坏
    ...
}
```

copy-and-swap 保证**强异常安全**：若拷贝构造失败抛异常，`*this` 完全不受影响，要么成功，要么不变。

---

### 为什么移动赋值必须先 delete

```cpp
b = std::move(a);   // b 原来有自己的内存

// 若不先 delete，直接覆盖 b.data_：
// b 原来的那块内存没有任何指针指向它 → 永久泄漏
```

---

### 不可拷贝的类（如互斥锁）

```cpp
class NonCopyable {
public:
    NonCopyable(const NonCopyable&) = delete;
    NonCopyable& operator=(const NonCopyable&) = delete;
};
```

---

### 各函数存在的意义

| 函数 | 解决的问题 |
|------|-----------|
| 拷贝构造 | 浅拷贝导致 double-free |
| 拷贝赋值 | 赋值时内存泄漏 + double-free |
| 移动构造 | 构造时不必要的深拷贝 |
| 移动赋值 | 赋值时不必要的深拷贝 |

**核心思想：** 拷贝系列保证**正确性**，移动系列保证**性能**。

---

### 移动构造与移动赋值的触发时机

**触发移动构造：**

```cpp
// 1. 用临时对象（右值）构造
Buffer b1 = Buffer(1024);        // → 移动构造

// 2. 函数返回值
Buffer makeBuffer() {
    Buffer tmp(1024);
    return tmp;
}
Buffer b2 = makeBuffer();        // → 移动构造（或 RVO 优化）

// 3. std::move 显式转移
Buffer b3(512);
Buffer b4 = std::move(b3);       // → 移动构造，b3 变空壳
```

**触发移动赋值：**

```cpp
// 1. 将临时对象赋值给已存在的对象
Buffer b5(10);
b5 = Buffer(1024);               // → 移动赋值

// 2. std::move 显式转移
Buffer b6(512), b7(10);
b7 = std::move(b6);              // → 移动赋值，b6 变空壳

// 3. 函数返回值赋给已存在对象
b7 = makeBuffer();               // → 移动赋值
```

**与拷贝的判断规则：**

```
右值（临时对象、std::move 的结果）→ 触发移动
左值（有名字的变量）              → 触发拷贝
```

> `std::move` 并不真正移动任何东西，只是把左值**强制转换为右值引用**，让编译器选择移动版本的函数。

---

## 3 深拷贝与浅拷贝

### 浅拷贝（Shallow Copy）

只复制指针本身，不复制指针指向的内容，两个对象**共享同一块内存**。

```cpp
Buffer b = a;   // 编译器默认的浅拷贝
```

```
a.data_ ──┐
           ├──→ [ 同一块内存 0x100 ]
b.data_ ──┘
```

**问题1：修改互相影响**
```cpp
b.data_[0] = 'X';   // a.data_[0] 也变成了 'X'
```

**问题2：double-free 崩溃**
```cpp
// a 和 b 析构时都会 delete[] 0x100 → 崩溃
```

### 深拷贝（Deep Copy）

重新分配内存并复制内容，两个对象**各自独立**。

```cpp
Buffer(const Buffer& other) : size_(other.size_) {
    data_ = new char[size_];
    std::memcpy(data_, other.data_, size_);
}
```

```
a.data_ ──→ [ 0x100: "hello" ]
b.data_ ──→ [ 0x200: "hello" ]   ← 独立的副本
```

### 对比

| | 浅拷贝 | 深拷贝 |
|---|---|---|
| 复制内容 | 只复制指针地址 | 复制指针指向的数据 |
| 内存关系 | 共享同一块内存 | 各自独立的内存 |
| 修改影响 | 互相影响 | 互不影响 |
| 析构 | double-free 崩溃 | 安全 |
| 性能 | 快 | 慢（需分配+复制） |

### 何时浅拷贝安全

成员全是**值类型**时，浅拷贝等价于深拷贝，编译器默认行为完全正确：

```cpp
class Point {
    int x_;   // 值类型，复制数值本身
    int y_;
};
```

**只要类中有裸指针，就必须自己实现深拷贝**，这正是 Rule of Five 存在的原因。

---

## 4 悬挂指针（Dangling Pointer）

> 指针指向的内存已经被释放，但指针本身还存在，仍然保存着那个地址。

### 三种典型场景

**场景1：指向已释放的堆内存**
```cpp
int* p = new int(42);
delete p;
*p = 100;    // 悬挂指针！写入已释放的内存，行为未定义
```

**场景2：指向已离开作用域的局部变量**
```cpp
int* p = nullptr;
{
    int x = 42;
    p = &x;   // p 指向局部变量 x
}             // x 离开作用域，栈内存被回收
*p;           // 悬挂指针！x 已不存在
```

**场景3：shared_ptr 双重管理**
```cpp
auto p1 = std::make_shared<A>();
auto p2 = std::shared_ptr<A>(p1.get());  // 创建独立控制块，危险！
p1.reset();     // A 被析构
p2->DoSth();    // 悬挂指针！A 已不存在
```

### 危害

| 操作 | 后果 |
|---|---|
| 读取 | 垃圾值 / 崩溃 |
| 写入 | 破坏其他数据 / 崩溃 |
| 再次 delete | double-free，堆损坏 |
| 表面正常运行 | 最危险，隐藏 bug |

表面正常是最可怕的情况——内存释放后不会立刻清零，旧数据可能还在，程序"侥幸"读到正确值，但这是未定义行为，随时可能崩溃。

### 与空指针的区别

```cpp
int* null_ptr = nullptr;
*null_ptr;      // 立刻崩溃，好调试

int* dangling = new int(1);
delete dangling;
*dangling;      // 可能不崩溃，极难调试
```

---

## 5 shared_from_this

### 问题：在类内部用 this 构造 shared_ptr

```cpp
struct A {
    std::shared_ptr<A> GetSelf() {
        return std::shared_ptr<A>(this);  // 危险！创建了独立的控制块
    }
};

auto p1 = std::make_shared<A>();  // 控制块①，refcount=1
auto p2 = p1->GetSelf();          // 控制块②，refcount=1（独立的！）
```

```
p1 ──→ [ 控制块① refcount=1 ] ──→ [ A对象 0x100 ]
                                        ↑
p2 ──→ [ 控制块② refcount=1 ] ──────────┘
```

两个独立的引用计数管理同一块内存：
- `p1` 析构 → 控制块①归零 → delete 0x100
- `p2` 析构 → 控制块②归零 → 再次 delete 0x100 → **double-free 崩溃**
- 若 `p1` 先析构，`p2` 仍持有地址 → **悬挂指针**

### 解决：继承 enable_shared_from_this

```cpp
struct A : std::enable_shared_from_this<A> {
    std::shared_ptr<A> GetSelf() {
        return shared_from_this();   // 复用已有控制块
    }
};

auto p1 = std::make_shared<A>();  // 控制块①，refcount=1
auto p2 = p1->GetSelf();          // 控制块①，refcount=2（共享！）
```

```
p1 ──→ ┐
        ├──→ [ 控制块① refcount=2 ] ──→ [ A对象 0x100 ]
p2 ──→ ┘
```

### 原理：内部存了一个 weak_ptr

```cpp
template<typename T>
class enable_shared_from_this {
    mutable std::weak_ptr<T> weak_this_;  // make_shared 时自动设置

    shared_ptr<T> shared_from_this() {
        return weak_this_.lock();   // 从已有控制块创建 shared_ptr
    }
};
```

`make_shared` 创建对象时会自动将 `weak_this_` 指向该控制块，`shared_from_this()` 调用 `lock()` 复用已有控制块，而非创建新的。

### 典型使用场景：异步回调中传递 this

```cpp
struct Server : std::enable_shared_from_this<Server> {
    void Start() {
        // 错误：lambda 捕获裸 this，回调执行时 Server 可能已析构
        async_read([this]() { HandleData(); });

        // 正确：延长 Server 生命周期到回调执行完
        async_read([self = shared_from_this()]() {
            self->HandleData();
        });
    }
};
```

### 注意事项

```cpp
// 必须由 shared_ptr 管理，才能调用 shared_from_this
A* raw = new A();
raw->shared_from_this();  // 抛异常！weak_this_ 从未被设置

// 不能在构造函数中调用
struct A : enable_shared_from_this<A> {
    A() { shared_from_this(); }  // 崩溃！控制块还未创建
};
```

### 总结

| | 裸 `shared_ptr(this)` | `shared_from_this()` |
|---|---|---|
| 控制块 | 创建新的 | 复用已有的 |
| 引用计数 | 两套独立的 | 同一套 |
| 析构 | double-free | 安全 |
| 悬挂指针 | 会产生 | 不会 |

---

## 6 指针与引用

### `*` 的三种含义

```cpp
int* p;          // 1. 声明指针：p 是一个指向 int 的指针

int val = *p;    // 2. 解引用：取指针指向的值
*p = 100;        //    通过指针修改原变量

int r = 3 * 4;   // 3. 乘法运算符
```

### `&` 的三种含义

```cpp
int& ref = x;    // 1. 声明引用：ref 是 x 的别名

int* p = &x;     // 2. 取地址：获取变量的内存地址

int r = 6 & 3;   // 3. 按位与运算符
```

### 指针 vs 引用

```cpp
int x = 42;
int* p = &x;   // 指针：存储地址，可以为 nullptr，可以改变指向
int& r = x;    // 引用：别名，必须初始化，不能改变绑定对象
```

| | 指针 `*` | 引用 `&` |
|---|---|---|
| 可以为空 | 可以（`nullptr`） | 不可以 |
| 可以改变指向 | 可以 | 不可以 |
| 需要解引用 | 需要（`*p`） | 不需要，直接用 |
| 本质 | 存储地址的变量 | 变量的别名 |

### 函数参数中的区别

```cpp
void f1(int x);   // 值传递：复制一份，修改不影响外部
void f2(int* x);  // 指针传递：传地址，可修改原变量，可传 nullptr
void f3(int& x);  // 引用传递：别名，可修改原变量，不能为 nullptr
```

```cpp
int a = 10;
f1(a);    // a 不变
f2(&a);   // 传 a 的地址
f3(a);    // 直接传 a，函数内可修改
```

### `&&` — 右值引用（C++11）

```cpp
int&& r = 42;        // 绑定到右值（临时值）
void f(int&& x);     // 接受右值，用于移动语义
```

这是移动构造 `Buffer(Buffer&& other)` 中 `&&` 的含义，表示接受一个即将销毁的临时对象，可以安全地"偷走"它的资源。

### `*` 和 `&` 在数组中的应用

**数组名本身就是指针**

```cpp
int arr[] = {1, 2, 3, 4, 5};

int* p = arr;       // 数组名隐式转换为首元素指针
int* p = &arr[0];   // 等价写法，完全一样

std::cout << *p;    // 输出: 1
std::cout << p[2];  // 输出: 3，指针可以用下标访问
```

**指针遍历数组**

```cpp
int arr[] = {10, 20, 30};

for (int* p = arr; p != arr + 3; ++p) {
    std::cout << *p << " ";   // 输出: 10 20 30
}
```

**数组的引用（保留大小信息）**

```cpp
int arr[5] = {1, 2, 3, 4, 5};

int* p = arr;         // 指针：丢失了数组大小，sizeof(p) = 8
int (&ref)[5] = arr;  // 数组的引用：保留大小，sizeof(ref) = 20

// 应用：模板函数自动推导数组大小
template<size_t N>
void printArray(int (&arr)[N]) {
    for (size_t i = 0; i < N; i++)
        std::cout << arr[i] << " ";
}

printArray(arr);   // 编译器自动推导 N=5
```

**指向数组的指针 vs 指向元素的指针**

```cpp
int arr[5] = {1, 2, 3, 4, 5};

int*    p1 = arr;      // 指向 int 元素的指针，+1 跳 4 字节
int (*p2)[5] = &arr;   // 指向"整个数组"的指针，+1 跳 20 字节

p1 + 1;   // → arr[1]，跳 4 字节
p2 + 1;   // → arr 末尾之后，跳 20 字节（整个数组的大小）
```

**二维数组中的指针**

```cpp
int matrix[3][4] = {{1,2,3,4}, {5,6,7,8}, {9,10,11,12}};

int (*row)[4] = matrix;   // 指向"含4个int的数组"的指针，指向第0行
std::cout << row[1][2];   // 输出: 7（第1行第2列）
++row;                    // 跳到下一行（跳 16 字节）
```

**函数参数中传数组**

```cpp
// 这三种写法等价，都退化为指针，丢失大小
void f1(int* arr,   int n) { }
void f2(int arr[],  int n) { }
void f3(int arr[5], int n) { }   // [5] 被忽略！

// 保留大小：使用数组引用（只接受大小正好为 N 的数组）
void f4(int (&arr)[5]) { }

int a[5], b[3];
f4(a);   // OK
f4(b);   // 编译错误！大小不匹配
```

**用 `&` 取数组中间某元素的地址**

```cpp
int arr[] = {10, 20, 30, 40};

int* p = &arr[2];      // p 指向第3个元素（值30）
std::cout << *p;       // 30
std::cout << p[-1];    // 20（向前偏移，合法）
std::cout << p[1];     // 40（向后偏移）
```

**总结对比**

| 写法 | 类型 | 说明 |
|---|---|---|
| `arr` | `int*` | 数组名退化为首元素指针 |
| `&arr[0]` | `int*` | 首元素地址，同上 |
| `&arr` | `int(*)[N]` | 整个数组的地址，值相同但类型不同 |
| `int (&ref)[N]` | 数组引用 | 绑定整个数组，保留大小信息 |
| `int (*p)[N]` | 数组指针 | 指向整个数组，+1 跳过整个数组 |

### 函数参数传递方式对比

| 传参类型 | 修改参数会影响实参？ | 能否传临时值 |
|---|---|---|
| 值传递 `int x` | 否 | 是 |
| 引用传递 `int& x` | **是** | 否 |
| 常量引用 `const int& x` | 否 | **是** |
| 值传指针 `char* p` | 否（只改局部指针） | 是 |
| 引用指针 `char*& p` | **是**（直接改外部指针） | 是 |

```cpp
// 1. 值传递：副本，修改不影响实参
void byValue(int x) { x = 999; }

// 2. 引用传递：别名，修改影响实参，不能传临时值
void byRef(int& x) { x = 999; }

// 3. 常量引用：只读，不能修改，但能接受临时值
void byConstRef(const int& x) { /* x = 999; 编译错误 */ }

// 4. 值传指针：指针本身是副本，改 p 不影响外部，但能改 *p 内容
void byPtr(char* p) {
    p[0] = 'X';    // 改了内容，影响外部
    p = nullptr;   // 只改了局部副本，外部指针不变
}

// 5. 引用指针：指针本身是引用，能直接改外部指针指向谁
void byPtrRef(char*& p) {
    p = nullptr;   // 直接改了外部指针！
}

int main() {
    int a = 1;
    byValue(a);        // a 仍为 1

    int b = 1;
    byRef(b);          // b 变为 999
    // byRef(42);      // 编译错误！不能传临时值

    byConstRef(42);    // OK！const 引用可以接受临时值

    char buf[] = "hello";
    char* p1 = buf;
    byPtr(p1);
    // p1 仍非 null，但 buf[0] 已变为 'X'（"Xello"）

    char* p2 = buf;
    byPtrRef(p2);
    // p2 变成 nullptr！外部指针被改了
}
```

**`char*` 和 `char*&` 的本质区别**：

```cpp
void f1(char* p)  { p = nullptr; }   // p 是副本，外部不变
void f2(char*& p) { p = nullptr; }   // p 是引用，外部指针被改

char* ptr = buf;
f1(ptr);   // ptr 仍然指向 buf
f2(ptr);   // ptr 变成 nullptr！
```

---

### 常量指针与指针常量

#### 记忆口诀

> **`const` 修饰离它最近的那个东西；以 `*` 为分界线，`const` 在左修饰值，`const` 在右修饰指针。**

#### 4 种组合

```cpp
char* p               // 普通指针：指向可变，值可变
const char* p         // 常量指针：指向可变，值不可变（等价于 char const* p）
char* const p         // 指针常量：指向不可变，值可变
const char* const p   // 双重const：指向不可变，值不可变
```

| 写法 | 改指向 | 改值 |
|------|--------|------|
| `char* p` | ✅ | ✅ |
| `const char* p` / `char const* p` | ✅ | ❌ |
| `char* const p` | ❌ | ✅ |
| `const char* const p` | ❌ | ❌ |

#### 代码验证

```cpp
char a = 'A', b = 'B';

const char* p2 = &a;
p2 = &b;   // ✅ 可以改指向
*p2 = 'C'; // ❌ 编译错误，不能改值

char* const p3 = &a;
p3 = &b;   // ❌ 编译错误，不能改指向
*p3 = 'C'; // ✅ 可以改值

const char* const p4 = &a;
p4 = &b;   // ❌
*p4 = 'C'; // ❌
```

#### `int* p` 与 `int *p`

两种写法完全等价，只是风格不同。多变量声明时注意：

```cpp
int* a, b;   // a 是 int*，b 是 int（容易误解）
int *a, *b;  // 两个都是指针，更清晰
// 推荐：每行只声明一个指针变量
```

---

## 7 手写线程池

```c++
#include<functional>
#include<vector>
#include<thread>
#include<queue>
#include<mutex>

using Task = std::function<void()>;
class ThreadPool{
public:
  explicit ThreadPool(size_t numThreads) : stop(false){
    for(size_t i = 0;i < numThreads; ++i){
      workers.emplace_back([this] { this->workThread();});
    }
  }
  ~ThreadPool(){
    {
    	std::unique_lock<std::mutex> lock(queueMutex);
    	stop = true;
    }
    condition.notify_all();
    for(auto& worker:workers){
      work.join();
    }
  }
  void enqueueTask(const Task& task){
    std::unique_lock<std::mutex> lock(queueMutex);
    tasks.push(task);
    condition.notify_one();
  }
  void workThread(){
    while(true){
      Task task;
      {
        std::unique_lock<std::mutex> lock(this->queueMutex);
        this.condition.wait(lock, [this](){
          return this->stop || !this->tasks.empty();
        });
        if(this->stop && this->tasks.empty()) return;
        task = std::move(this->tasks.front());
        this->tasks.pop();
      }
      task();
    }
  }
private:
  std::vector<std::thread> workers;
  std::queue<Task> tasks;
  std::mutex queueMutex;
  std::condition_variable condition;
  bool stop;
};
```

### 关键设计点

**1. `[this]` 捕获列表**

lambda 默认无法访问外部变量，`[this]` 将当前对象指针传入 lambda，使其能访问类的成员：

```cpp
workers.emplace_back([this] { workThread(); });
// 等价于把 this 指针存入 lambda，内部访问 stop/tasks 均通过 this->
```

常见捕获方式：

| 写法 | 含义 |
|---|---|
| `[]` | 不捕获任何变量 |
| `[this]` | 捕获 this 指针，访问类成员 |
| `[=]` | 按值捕获所有外部变量 |
| `[&]` | 按引用捕获所有外部变量 |

---

**2. 为什么用 `size_t` 不用 `int`**

`numThreads` 表示数量，天然不可能为负，`size_t` 是无符号类型，语义更准确：

```cpp
for (size_t i = 0; i < numThreads; ++i)  // 类型一致，无符号比较，无编译警告
```

- `int` 可以为负，语义不准确
- `size_t` 与 `vector.size()`、`strlen()` 等 STL 返回类型一致，避免有符号/无符号比较警告
- 表示"个数/大小/索引"时优先用 `size_t`，只有可能为负的差值才用 `int`

---

**3. 为什么用 `const Task&` 而不是直接传值**

```cpp
void enqueueTask(const Task& task)  // 当前写法：传引用，入参不拷贝
void enqueueTask(Task task)         // 按值：调用时多一次拷贝构造，有额外开销
```

`std::function` 内部可能持有 lambda 捕获的数据，拷贝有开销。`const Task&` 传引用零拷贝，只在 `push` 入队时才做一次必要的拷贝。

更完整的写法同时支持左值和右值：

```cpp
void enqueueTask(const Task& task) { tasks.push(task); }            // 左值：拷贝入队
void enqueueTask(Task&& task)      { tasks.push(std::move(task)); } // 右值：移动入队
```

---

**4. 为什么 wait 后还要再判断 stop && tasks.empty()**

`condition.wait` 的谓词 `stop || !tasks.empty()` 满足任意一个就会返回，有三种情况：

```
stop=false, tasks非空  → 正常取任务，继续执行
stop=true,  tasks非空  → 还有任务，继续执行完（优雅关闭）
stop=true,  tasks为空  → 真正退出
```

只有第三种情况才应该退出，这个 `if` 判断正是区分"收到停止信号但任务未清空"和"可以真正退出"的关键，保证已入队的任务不会丢失。

---

**5. 为什么取任务要 front() + pop() 而不是直接 pop()**

`std::queue::pop()` 返回 `void`，不返回元素值，这是 STL 的异常安全设计：

```cpp
// 若 pop() 返回值，则需要先拷贝再删除
// 拷贝若抛异常 → 元素已删除，值也丢失

// 正确做法：分两步
task = std::move(tasks.front());  // 先移动取值（移动不抛异常）
tasks.pop();                       // 再安全删除
```

用 `std::move` 而非直接赋值，是因为队列里的元素即将被删除，直接转移资源比拷贝更高效。

---

**6. 析构函数为什么要先释放锁再 notify 和 join**

```cpp
~ThreadPool() {
    {
        std::unique_lock<std::mutex> lock(queueMutex);
        stop = true;
    }                        // ← 锁在这里释放
    condition.notify_all();
    for (auto& worker : workers) worker.join();
}
```

若持锁时 join，工作线程在 wait 被唤醒后需要重新获取锁才能继续执行，但主线程持着锁不释放 → **死锁**。必须先释放锁，工作线程才能正常退出。

---

## 8 虚函数（Virtual Function）

### 核心概念

虚函数是 C++ 实现**运行时多态**的机制：通过基类指针/引用调用函数时，根据对象的**实际类型**决定调用哪个实现。

```cpp
class Animal {
public:
    virtual void speak() {        // 虚函数：加 virtual 关键字
        std::cout << "Animal sound\n";
    }
    void breathe() {              // 普通函数：非虚，静态绑定
        std::cout << "Breathing\n";
    }
};

class Dog : public Animal {
public:
    void speak() override {       // override 让编译器检查是否真的重写了虚函数
        std::cout << "Woof!\n";
    }
};

int main() {
    Animal* a = new Dog();
    a->speak();    // 输出: Woof!   —— 运行时决定调用 Dog::speak
    a->breathe();  // 输出: Breathing —— 编译期决定，始终调用 Animal::breathe
}
```

### 虚函数 vs 普通函数

| 特性 | 普通函数 | 虚函数 |
|---|---|---|
| 绑定时机 | 编译期（静态绑定） | 运行期（动态绑定） |
| 调用依据 | 指针/引用的**类型** | 指针/引用指向的**对象实际类型** |
| 额外开销 | 无 | 一次 vtable 间接跳转 |

### 虚函数表（vtable）机制

每个含虚函数的类都有一张 **vtable**，对象内部有隐藏指针 `vptr` 指向它：

```
Dog 对象内存布局:
┌──────────┬──────────────────────────────┐
│  vptr    │ ──→  Dog::vtable              │
│          │      [0] Dog::speak()         │
├──────────┤      [1] ~Dog()               │
│  成员变量 │                              │
└──────────┴──────────────────────────────┘

调用 a->speak():
  1. 通过 a 的 vptr 找到 Dog 的 vtable
  2. 从 vtable[0] 拿到 Dog::speak 的函数地址
  3. 跳转执行
```

### 纯虚函数与抽象类

```cpp
class Shape {
public:
    virtual double area() = 0;   // 纯虚函数：= 0，强制派生类实现
    virtual void draw() = 0;
    virtual ~Shape() {}          // 析构函数也必须是虚的！
};

class Circle : public Shape {
    double r;
public:
    Circle(double r) : r(r) {}
    double area() override { return 3.14 * r * r; }
    void draw() override { std::cout << "Drawing circle\n"; }
};

// Shape s;                       // 错误！含纯虚函数的类不能实例化
Shape* s = new Circle(5.0);      // 正确，多态使用
```

**基类有纯虚函数时，派生类不一定必须直接实现所有这些纯虚函数，但要想“变成可实例化的具体类（concrete class）”就必须实现所有继承下来的纯虚函数**。如果派生类没有实现其中某一个纯虚函数，那么这个派生类**自身仍然是抽象类**，不能创建对象。

**纯虚函数等价于抽象方法。**

一旦在基类中声明了虚函数,该函数在派生类中自动成为虚函数,不需要再显式声明virtual关键字。这是C++的语言特性,目的是为了简化代码编写并保持继承关系的一致性。

### 虚析构函数（必须记住）

通过基类指针 `delete` 对象时，若析构函数不是虚函数，只会调用基类析构，派生类的资源无法释放：

```cpp
class Base {
public:
    ~Base() { }            // 非虚！
};
class Derived : public Base {
    int* data;
public:
    Derived() : data(new int[100]) {}
    ~Derived() { delete[] data; }  // 不会被调用
};

Base* p = new Derived();
delete p;   // 只调用 ~Base，~Derived 未调用 → 内存泄漏
```

**结论：只要类会被继承并通过基类指针 delete，析构函数就必须声明为 `virtual`。**

### 与 C# abstract 的对比

C++ 的 `virtual` 概念涵盖了 C# 中 `virtual` 和 `abstract` 两种情况：

| C++ | C# | 含义 |
|---|---|---|
| `virtual void f() { }` | `virtual void f() { }` | 有默认实现，子类可选重写 |
| `virtual void f() = 0` | `abstract void f()` | 无实现，子类必须重写 |
| 含纯虚函数的类（自动） | `abstract class` | 不可实例化的抽象类 |
| 全是纯虚函数的类 | `interface` | 纯接口 |

### 典型使用场景（策略模式/插件系统）

```cpp
class Renderer {
public:
    virtual void render(Scene& s) = 0;
    virtual ~Renderer() = default;
};

class OpenGLRenderer : public Renderer {
    void render(Scene& s) override { /* OpenGL 实现 */ }
};

class VulkanRenderer : public Renderer {
    void render(Scene& s) override { /* Vulkan 实现 */ }
};

// 调用方无需关心具体实现，运行时切换
std::unique_ptr<Renderer> r = std::make_unique<VulkanRenderer>();
r->render(scene);
```

### 关键点总结

| 关键字/概念 | 作用 |
|---|---|
| `virtual` | 声明虚函数，启用运行时多态 |
| `= 0` | 纯虚函数，类变为抽象类，不可实例化 |
| `override` | 让编译器检查是否真正重写了虚函数（推荐写） |
| `virtual ~Base()` | 虚析构函数，通过基类指针 delete 时必须加 |
| vtable | 运行时查找实际函数地址的机制，每次虚函数调用多一次间接跳转 |

---

## 9 默认构造函数与成员初始化

### C++ 默认构造函数的实际行为

"编译器提供的默认构造函数不执行任何操作"是过度简化，准确行为是：

```cpp
class Foo {
    int x;           // 内置类型：不初始化（垃圾值）⚠️
    std::string s;   // 类类型：会调用 string 的默认构造函数
};

Foo f;
// f.x → 不确定值（未定义行为，可能是任意数字）
// f.s → 正常空字符串（string 的构造函数被调用了）
```

编译器生成的默认构造函数：
- **会调用**所有类类型成员的默认构造函数
- **不会初始化** `int`、`double`、指针等内置类型成员

### C# 默认构造函数的行为

C# 在 CLR 层面保证所有字段在构造前先**零初始化**：

```csharp
class Foo {
    int x;      // 自动初始化为 0
    string s;   // 自动初始化为 null
    bool b;     // 自动初始化为 false
}

var f = new Foo();
// f.x → 0    保证
// f.s → null 保证
// f.b → false 保证
```

### C++ vs C# 对比

| | C++ 默认构造函数 | C# 默认构造函数 |
|---|---|---|
| 内置类型成员（int 等） | **不初始化**，垃圾值 | **零初始化**（0 / false / null） |
| 类类型成员 | 调用其默认构造函数 | 调用其默认构造函数 |
| 安全性 | 读未初始化变量是 UB | 始终有确定默认值，安全 |

### C++ 中的陷阱

```cpp
class Node {
    int value;      // 未初始化！
    Node* next;     // 未初始化！可能是野指针
};

Node n;
if (n.next != nullptr) { }  // 行为未定义，next 是垃圾值
```

### 正确做法：在初始化列表中显式初始化所有内置类型成员

```cpp
class Node {
    int value;
    Node* next;
public:
    Node() : value(0), next(nullptr) { }  // 明确初始化，安全
};
```

或使用 C++11 成员默认初始化：

```cpp
class Node {
    int value = 0;        // 就地初始化
    Node* next = nullptr;
};
```

---

## 10 char 类型与整数表示

### char 占 1 字节，但最大值是 127

```cpp
std::cout << sizeof(char);  // 1（字节）
```

`char` 默认是**有符号（signed）**，8 位中最高位作符号位，只有 7 位存数值：

```
8位二进制：
┌───┬───────────────┐
│ S │  7位数值位    │
└───┴───────────────┘
  ↑
 符号位：0=正数，1=负数

0111 1111 = 127   （最大正数，7位全1）
1000 0000 = -128  （补码，最小负数）
1111 1111 = -1
```

### signed vs unsigned char

```cpp
signed char   s = 200;   // 溢出！实际存储为 -56
unsigned char u = 200;   // 正确，范围 0~255

std::cout << (int)s;  // -56
std::cout << (int)u;  // 200
```

**存储字节数据（图像像素、二进制协议）时，应显式用 `unsigned char` 或 `uint8_t`，避免符号歧义。**

### 补码：为什么是 -128 而不是 -127

有符号整数用补码表示：若用原码，`1000 0000` 表示 -0，与 +0 重复浪费了一个位置。补码将 `1000 0000` 定义为 -128，消除了 -0，因此多表示了一个负数：

```
有符号 8位：-128 ~ 127（128个负数 + 0 + 127个正数 = 256个）
无符号 8位：   0 ~ 255（256个非负数）
```

规律：**n 字节有符号整数范围 = -2^(8n-1) ~ 2^(8n-1)-1**，最小值绝对值比最大值大 1。

### char 的符号性是平台相关的

```cpp
char          c;   // 有符号或无符号，取决于编译器/平台（x86 通常是 signed）
signed char   s;   // 明确：有符号，-128~127
unsigned char u;   // 明确：无符号，0~255
```

### C++ vs C# 的 char 对比

C# 的 `char` 是 UTF-16 编码单元，与 C++ 定位完全不同：

| | C++ `char` | C# `char` |
|---|---|---|
| 大小 | **1 字节** | **2 字节** |
| 符号性 | 有符号（平台相关） | 无符号（固定） |
| 范围 | -128 ~ 127 | 0 ~ 65535 |
| 本质 | 整数类型，可直接做算术 | Unicode UTF-16 码元 |
| 中文支持 | 不支持（需多字节编码） | 直接支持 |

```csharp
// C# char 直接存中文
char c = '中';                  // 合法
Console.WriteLine(char.MaxValue); // 65535
```

```cpp
// C++ char 存不下中文，UTF-8 中文字符串是多个 char 拼成的
const char* s = "中";
std::cout << strlen(s);  // 输出: 3（UTF-8 每个汉字占3字节）

// C++11 提供专用宽字符类型
wchar_t  w = L'中';   // 2或4字节（平台相关）
char32_t u = U'中';   // 4字节，固定
```

### 各整型大小总结

| 类型 | 字节 | 有符号范围 | 无符号范围 |
|---|---|---|---|
| `char` | 1 | -128 ~ 127 | 0 ~ 255 |
| `short` | 2 | -32768 ~ 32767 | 0 ~ 65535 |
| `int` | 4 | -2³¹ ~ 2³¹-1 | 0 ~ 2³²-1 |
| `long long` | 8 | -2⁶³ ~ 2⁶³-1 | 0 ~ 2⁶⁴-1 |

---

## 11 类型转换：static_cast 与 dynamic_cast

### 向上转型：`Base* b = new Dog()`

用基类指针指向派生类对象，这是多态的基础写法：

```cpp
Base* b = new Dog();   // Dog 继承自 Base，向上转型，天然安全
```

```
     b (Base*)
      │
      ▼
┌─────────────────────┐
│   Dog 对象           │
│   vptr → Dog vtable │
│   Dog 的成员变量     │
└─────────────────────┘
```

通过 `b` 只能访问 `Base` 中声明的接口，但虚函数调用仍走 Dog 的实现：

```cpp
b->speak();    // 虚函数 → 调用 Dog::speak（运行时决定）
b->breathe();  // 普通函数 → 调用 Base::breathe（编译期决定）
b->fetch();    // 编译错误！Base* 不知道 fetch 存在
```

---

### static_cast — 编译期转换，不做安全检查

```cpp
// 基本类型转换
double d = 3.14;
int i = static_cast<int>(d);   // 3，截断小数

// 类层次转换（编译器信任你，不检查运行时类型）
Base* b = new Dog();
Derived* d = static_cast<Derived*>(b);  // 你保证 b 确实是 Derived*
```

**特点**：编译期决定，零运行开销，转错了是未定义行为。

---

### dynamic_cast — 运行期检查，只用于多态类

```cpp
class Base {
public:
    virtual ~Base() { }   // 必须有虚函数！否则编译错误
};
class Dog : public Base { public: void bark(); };
class Cat : public Base { public: void meow(); };

Base* b = new Dog();

// 转换成功：返回有效指针
Dog* dog = dynamic_cast<Dog*>(b);   // dog != nullptr
if (dog) dog->bark();

// 转换失败：返回 nullptr（不崩溃）
Cat* cat = dynamic_cast<Cat*>(b);   // cat == nullptr
if (cat) cat->meow();               // 不会执行
```

**引用转换失败抛 `std::bad_cast`**：

```cpp
try {
    Cat& cat = dynamic_cast<Cat&>(*b);  // 抛异常
} catch (std::bad_cast&) {
    std::cout << "转换失败\n";
}
```

---

### 核心区别

| | `static_cast` | `dynamic_cast` |
|---|---|---|
| 检查时机 | 编译期 | **运行期** |
| 安全性 | 不检查，转错是 UB | 失败返回 nullptr / 抛异常 |
| 性能开销 | 零开销 | 有开销（查 vtable/RTTI） |
| 要求 | 无 | 基类**必须有虚函数** |
| 适用场景 | 明确知道类型 | 不确定类型，需安全转换 |

---

### 何时用哪个

```cpp
// static_cast：100% 确定类型时
Derived* d = static_cast<Derived*>(b);  // 业务逻辑保证 b 一定是 Derived*

// dynamic_cast：需要根据实际类型做不同处理
void process(Base* b) {
    if (auto* dog = dynamic_cast<Dog*>(b)) {
        dog->bark();
    } else if (auto* cat = dynamic_cast<Cat*>(b)) {
        cat->meow();
    }
}
```

**频繁使用 `dynamic_cast` 是设计问题的信号**——如果需要大量判断子类类型，通常应改用虚函数消除这些判断。

---

### dynamic_cast 的独特能力：多继承中的横向转换（Cross-cast）

`static_cast` 只能在有直接继承关系的类之间转换，**无法跨越无关的兄弟类**；而 `dynamic_cast` 能在多继承中从一个基类横向转换到另一个基类。

```cpp
struct A1 { virtual ~A1() {} };
struct A2 { virtual ~A2() {} };
struct B1 : A1, A2 {};           // B1 同时继承 A1 和 A2

B1 d;
A1* pb1 = &d;

A2* pb2  = dynamic_cast<A2*>(pb1);  // L1：成功，pb2 有效
A2* pb22 = static_cast<A2*>(pb1);   // L2：编译错误！
```

**L1 成功原因**：`dynamic_cast` 运行时通过 RTTI 找到实际对象（`B1`），发现 `B1` 继承了 `A2`，成功定位到 `A2` 子对象：

```
B1 对象内存布局（多继承）：
┌──────────────────┐
│  A1 子对象        │  ← pb1 指向这里
│  vptr → A1 vtable│
├──────────────────┤
│  A2 子对象        │  ← pb2 被正确导向这里
│  vptr → A2 vtable│
├──────────────────┤
│  B1 自身成员      │
└──────────────────┘
```

**L2 编译错误原因**：`A1` 和 `A2` 互相没有继承关系，`static_cast` 编译期看不到任何路径可以从 `A1*` 转到 `A2*`，直接拒绝编译。

| | L1 `dynamic_cast` | L2 `static_cast` |
|---|---|---|
| 结果 | 运行时成功，返回有效 `A2*` | **编译错误** |
| 原因 | 运行期查 RTTI，找到 B1 的 A2 子对象 | A1 和 A2 无继承关系，编译期拒绝 |

---

## 12 进程与线程

### 基本概念

| 概念 | 定义 |
|------|------|
| **进程 (Process)** | 操作系统分配资源的基本单位，是程序的一次执行实例 |
| **线程 (Thread)** | CPU 调度的基本单位，是进程内的一个执行流 |

---

### 核心区别

| 维度 | 进程 | 线程 |
|------|------|------|
| **地址空间** | 独立地址空间，互相隔离 | 共享所在进程的地址空间 |
| **资源** | 拥有独立资源（文件描述符、堆等） | 共享进程资源，但有独立栈和寄存器 |
| **开销** | 创建/切换开销大 | 创建/切换开销小 |
| **通信** | IPC（管道、消息队列、共享内存等） | 直接读写共享内存（需同步） |
| **崩溃影响** | 一个进程崩溃不影响其他进程 | 一个线程崩溃可能导致整个进程崩溃 |
| **安全性** | 隔离性强，更安全 | 隔离性弱，需小心数据竞争 |

---

### 联系

- 一个进程至少包含一个线程（主线程）
- 线程是进程的执行单元，进程是线程的容器
- 同一进程内所有线程共享：堆、全局变量、静态变量、代码段、文件描述符
- 每个线程独有：栈、程序计数器（PC）、寄存器状态

---

### C++ 中的体现

```cpp
#include <thread>
#include <iostream>

int shared_data = 0;  // 所有线程共享

void worker(int id) {
    shared_data++;    // 危险！需要加锁
    std::cout << "Thread " << id << "\n";
}

int main() {
    std::thread t1(worker, 1);
    std::thread t2(worker, 2);
    t1.join();
    t2.join();
    // main() 是主线程，t1/t2 是子线程
    // 三个线程同属一个进程，共享 shared_data
}
```

---

### 何时用进程 vs 线程

| 场景 | 选择 |
|------|------|
| 需要强隔离（如浏览器每个 Tab） | 进程 |
| 需要高性能、频繁通信 | 线程 |
| 并行计算、任务分解 | 线程（线程池） |
| 运行独立程序 | 进程（`fork`/`exec`） |

---

## 13 C++ vs C# 优势对比

### 各自优势

| 维度 | C++ 优势 | C# 优势 |
|------|----------|----------|
| **性能** | 接近硬件，无运行时开销 | 开发效率高，JIT 有一定优化 |
| **内存控制** | 手动管理，精确控制生命周期 | GC 自动管理，不易内存泄漏 |
| **平台** | 跨平台，可运行于嵌入式/裸机 | 依赖 .NET 运行时 |
| **系统级** | 可直接操作硬件、内核、驱动 | 受沙箱限制，难以操作底层 |
| **生态** | 游戏引擎、操作系统、数据库 | Windows 应用、企业软件、Web 后端 |
| **开发效率** | 低（复杂语法、手动管理） | 高（LINQ、反射、丰富标准库） |
| **安全性** | 易出现内存错误 | 类型安全，内存访问受保护 |
| **学习曲线** | 陡峭 | 平缓 |

---

### 为什么 C++ 更适合高性能场景

#### 1. 无垃圾回收（GC）停顿

C# 的 GC 会在不确定的时机触发 **Stop-the-World** 暂停，造成延迟抖动：

```csharp
// C# - GC 可能在任意时刻暂停程序几毫秒
var list = new List<byte[]>();
for (int i = 0; i < 10000; i++)
    list.Add(new byte[1024]);  // 频繁分配触发 GC
```

```cpp
// C++ - 完全由程序员控制，零暂停
std::vector<std::vector<uint8_t>> list;
list.reserve(10000);  // 预分配，避免重分配
for (int i = 0; i < 10000; i++)
    list.emplace_back(1024);
```

> 在实时系统（游戏、金融交易、音频处理）中，几毫秒的 GC 暂停是不可接受的。

---

#### 2. 零成本抽象（Zero-Cost Abstraction）

C++ 的高级特性（模板、内联、constexpr）在**编译期**展开，运行时没有额外开销：

```cpp
// 模板在编译期生成专用代码，相当于手写 int 版本
template<typename T>
T add(T a, T b) { return a + b; }

// constexpr 在编译期计算，运行时是常量
constexpr int factorial(int n) {
    return n <= 1 ? 1 : n * factorial(n - 1);
}
constexpr int val = factorial(10);  // 编译期得出结果
```

C# 的泛型是运行时机制（类型擦除 + 装箱），有额外开销。

---

#### 3. 直接内存布局控制

C++ 可以精确控制数据在内存中的排列方式，充分利用 CPU 缓存：

```cpp
// Cache-friendly：连续内存，对 CPU 缓存友好
struct Particle {
    float x, y, z;   // 12字节，紧密排列
};
std::vector<Particle> particles(10000);  // 连续内存块

// C# 的 class 是堆对象，引用分散，缓存命中率低
// class Particle { float x, y, z; }
// var particles = new Particle[10000];  // 数组元素是引用，数据在堆上分散
```

---

#### 4. 无虚拟机/解释层

| | C++ | C# |
|--|-----|----|
| 编译结果 | 直接机器码 | IL 字节码 |
| 运行方式 | CPU 直接执行 | CLR 解释 + JIT 编译 |
| 启动开销 | 极小 | 需加载运行时 |
| 运行时优化 | 编译器静态优化 | JIT 动态优化（有限） |

---

#### 5. SIMD / 内联汇编

C++ 可以直接使用 CPU 的向量指令集，一次处理多个数据：

```cpp
#include <immintrin.h>

// AVX2：一次并行处理 8 个 float
void add_arrays(float* a, float* b, float* result, int n) {
    for (int i = 0; i < n; i += 8) {
        __m256 va = _mm256_load_ps(a + i);
        __m256 vb = _mm256_load_ps(b + i);
        _mm256_store_ps(result + i, _mm256_add_ps(va, vb));
    }
}
```

---

### 选择建议

| 场景 | 选择 |
|------|------|
| 游戏引擎、操作系统、数据库、音视频编解码、嵌入式 | **C++** |
| 企业应用、Web 后端、桌面工具、快速原型 | **C#** |

---

## 14 map 与 unordered_map

### 核心区别

| 维度 | `map` | `unordered_map` |
|------|-------|-----------------|
| **底层结构** | 红黑树（平衡二叉搜索树） | 哈希表 |
| **键的要求** | 必须可比较（`<` 运算符） | 必须可哈希（`std::hash` 特化） |
| **元素顺序** | 按键有序排列 | 无序 |
| **查找复杂度** | O(log n) | O(1) 平均，O(n) 最坏 |
| **插入复杂度** | O(log n) | O(1) 平均 |
| **内存占用** | 每个节点独立分配，有指针开销 | 哈希桶 + 链表，有负载因子开销 |
| **迭代顺序** | 有序（中序遍历） | 不保证顺序 |
| **缓存友好性** | 差（节点分散在堆上） | 相对较好 |

---

### 代码示例

```cpp
#include <map>
#include <unordered_map>
#include <iostream>

int main() {
    std::map<std::string, int> m;
    m["banana"] = 2;
    m["apple"]  = 1;
    m["cherry"] = 3;
    // 输出有序：apple=1, banana=2, cherry=3
    for (auto& [k, v] : m)
        std::cout << k << "=" << v << "\n";

    std::unordered_map<std::string, int> um;
    um["banana"] = 2;
    um["apple"]  = 1;
    um["cherry"] = 3;
    // 输出无序，顺序不确定
    for (auto& [k, v] : um)
        std::cout << k << "=" << v << "\n";
}
```

---

### 哈希冲突与 O(n) 退化

`unordered_map` 内部维护一个**桶数组**，插入时通过哈希函数决定元素放在哪个桶：

```
hash("apple")  % 8 = 3  → 放入第3号桶
hash("banana") % 8 = 6  → 放入第6号桶
hash("cherry") % 8 = 1  → 放入第1号桶

桶数组:
[1] → "cherry"=3
[3] → "apple"=1
[6] → "banana"=2
```

查找时计算哈希直接定位桶，O(1)。

当两个键哈希到**同一个桶**时发生冲突，C++ 标准库用**链地址法**解决——同桶元素用链表串起来：

```
hash("abc") % 8 = 3
hash("xyz") % 8 = 3  ← 冲突

[3] → ("abc"=1) → ("xyz"=2) → ("qwe"=3) → null
```

查找时需遍历链表逐一比较。若所有 n 个元素都冲突到同一个桶，查找退化为 **O(n)**。

> 类比：把哈希表想象成一排抽屉，正常情况直接拉开对应抽屉（O(1)）；所有东西塞进同一个抽屉，就得在里面逐个翻找。

---

### 负载因子与扩容

```
负载因子 = 元素数量 / 桶数量
```

负载因子超过阈值（默认 1.0）时自动 **rehash**（扩容重新分配），保证平均 O(1)。

```cpp
std::unordered_map<std::string, int> um;
um.max_load_factor(0.7);  // 更激进地扩容，减少冲突
um.reserve(1000);         // 预分配桶，避免频繁 rehash
```

---

### 自定义哈希函数

```cpp
struct MyHash {
    size_t operator()(const std::string& s) const {
        return std::hash<std::string>{}(s) ^ 0x9e3779b9;
    }
};
std::unordered_map<std::string, int, MyHash> safe_map;
```

---

### 选择建议

| 场景 | 选择 |
|------|------|
| 需要按键有序遍历 | `map` |
| 需要范围查询（`lower_bound` / `upper_bound`） | `map` |
| 只需快速查找，不关心顺序 | `unordered_map` |
| 键是自定义类型且难以哈希 | `map` |
| 高频查找，性能敏感 | `unordered_map` |

**一般原则**：默认用 `unordered_map`，有顺序需求时用 `map`。

---

## 15 Most Vexing Parse

C++ 的一条语法规则：**如果一段代码既能被解析为函数声明，也能被解析为对象定义，编译器一律当作函数声明。**

### 经典例子

```cpp
Buffer buf(Buffer());
```

你以为是：创建一个 `Buffer` 对象 `buf`，用一个临时 `Buffer()` 初始化它。

编译器认为是：声明一个函数 `buf`，它接受一个**返回 `Buffer` 的函数指针**作为参数，返回 `Buffer`。

```cpp
// 更简单的例子
Buffer buf();   // ❌ 不是创建对象，是声明一个返回 Buffer 的函数
Buffer buf;     // ✅ 默认构造一个对象
```

---

### 解决方法

**1. 用大括号初始化（C++11 推荐）**

```cpp
Buffer buf(Buffer{});   // ✅ 明确是对象
Buffer buf{};           // ✅ 明确是对象
```

**2. 用额外的括号**

```cpp
Buffer buf((Buffer()));  // ✅ 多一层括号，消除歧义
```

**3. 用具名变量**

```cpp
Buffer tmp;
Buffer buf(tmp);         // ✅ 没有歧义
```

---

### 典型陷阱

```cpp
std::lock_guard<std::mutex> lock(std::mutex());  // Most Vexing Parse！
                                                 // 实际是函数声明，不是加锁
```

> 现代 C++ 推荐**统一用 `{}` 初始化**，可以完全避免此问题。

---

## 16 const 与 static 修饰成员

### `const` 修饰成员变量

- 只能在**构造函数初始化列表**中赋值，不能在构造函数体内赋值
- 每个对象有自己的副本，初始化后不可修改

```cpp
class Foo {
    const int id_;
public:
    Foo(int id) : id_(id) {}   // ✅ 初始化列表
    // Foo(int id) { id_ = id; }  // ❌ 编译错误
};
```

---

### `const` 修饰成员函数

- 函数内**不能修改成员变量**（除非成员标记 `mutable`）
- **`const` 对象只能调用 `const` 成员函数**

```cpp
class Foo {
    int value_ = 0;
public:
    int get() const { return value_; }
};

const Foo f;
f.get();    // ✅
f.set(1);   // ❌ const 对象不能调用非 const 函数
```

---

### `static` 修饰成员变量

- **所有对象共享同一份**，不属于某个具体对象
- 必须在类外定义并初始化
- 通过类名访问：`Foo::count_`

```cpp
class Foo {
    static int count_;
};
int Foo::count_ = 0;   // 类外定义
```

---

### `static` 修饰成员函数

- **没有 `this` 指针**，不依附于具体对象
- 只能访问静态成员，不能访问普通成员变量
- 通过类名直接调用：`Foo::getCount()`

```cpp
class Foo {
public:
    static int getCount() { return count_; }
};

Foo::getCount();   // ✅ 不需要对象
```

---

### `static const` 组合

```cpp
class Foo {
    static const int MAX = 100;   // ✅ 可以直接在类内初始化
};
```

---

### 总结对比

| | `const` 成员变量 | `const` 成员函数 | `static` 成员变量 | `static` 成员函数 |
|--|----------------|----------------|-----------------|-----------------|
| 所属 | 每个对象独有 | 每个对象独有 | 所有对象共享 | 无对象概念 |
| `this` 指针 | 有 | 有（只读） | 有 | **无** |
| 修改成员 | 初始化后不可改 | 不能修改成员 | 可修改 | 只能访问静态成员 |
| 调用方式 | — | 需要对象 | `类名::变量` | `类名::函数()` |

---

## 17 堆与栈

### 内存中的堆与栈

| | 栈（Stack） | 堆（Heap） |
|--|------------|-----------|
| **管理方式** | 自动，由编译器管理 | 手动，程序员负责 `new`/`delete` |
| **分配速度** | 极快（移动栈指针） | 较慢（需要找空闲内存块） |
| **大小限制** | 小（通常 1~8 MB） | 大（受物理内存限制） |
| **生命周期** | 随作用域自动释放 | 手动释放，或程序结束时 |
| **内存碎片** | 无 | 频繁分配释放会产生碎片 |
| **增长方向** | 向低地址增长 | 向高地址增长 |

**栈存储：**

```cpp
void foo() {
    int a = 10;          // 局部变量 → 栈
    char buf[256];       // 固定大小数组 → 栈
    Buffer b(10);        // 对象本身 → 栈
}                        // 函数返回，以上全部自动释放
```

**堆存储：**

```cpp
int* p = new int(10);         // → 堆，必须手动 delete
char* buf = new char[1024];   // → 堆
Buffer* b = new Buffer(10);   // → 堆
```

**注意：对象在栈，成员可能在堆**

```cpp
void foo() {
    Buffer b(1024);
    // b 本身（data_ 指针 + size_）→ 栈
    // b.data_ 指向的 1024 字节   → 堆（构造函数里 new 的）
}
// 函数返回：b 析构 → delete[] data_ 释放堆内存，b 本身随栈自动释放
```

---

### 数据结构中的堆与栈

**栈（Stack）— 后进先出 LIFO**

```cpp
std::stack<int> s;
s.push(1); s.push(2); s.push(3);
s.top();   // 3
s.pop();   // 弹出 3
```

应用场景：函数调用栈、括号匹配、撤销操作、DFS

**堆（Heap）— 优先队列，最值优先出**

```cpp
std::priority_queue<int> maxHeap;  // 默认大根堆
maxHeap.push(3); maxHeap.push(1); maxHeap.push(4);
maxHeap.top();   // 4（最大值永远在顶部）
maxHeap.pop();   // 弹出 4
```

内部是完全二叉树，保证父节点 ≥ 子节点（大根堆）。

应用场景：任务调度、求第 K 大/小值、Dijkstra 算法

**数据结构对比：**

| | 栈（Stack） | 堆（Heap） |
|--|------------|-----------|
| **结构** | 线性 | 完全二叉树 |
| **出入规则** | 后进先出（LIFO） | 最值优先出 |
| **复杂度** | push/pop O(1) | push/pop O(log n) |
| **C++ 容器** | `std::stack` | `std::priority_queue` |

> 内存层面的堆/栈 与 数据结构的堆/栈，名字相同但完全无关。

---

### 栈溢出（Stack Overflow）

栈空间耗尽时发生，常见原因：

**1. 无限递归（最常见）**

```cpp
void infinite() { infinite(); }  // 栈帧不断堆积直到耗尽
```

**2. 递归深度过大**

```cpp
void recurse(int n) { recurse(n + 1); }
recurse(0);   // 百万级深度 → 栈溢出
```

**3. 栈上分配过大的局部变量**

```cpp
void foo() {
    char buf[10 * 1024 * 1024];  // 10 MB，超过栈大小
}
```

**解决方法：**

| 问题 | 解决方案 |
|------|---------|
| 无限递归 | 检查终止条件 |
| 递归过深 | 改用循环 + `std::stack` |
| 局部数组过大 | 改用 `std::vector` / `make_unique` 分配到堆 |

```cpp
// 大数组放堆上
void foo() {
    auto buf = std::make_unique<char[]>(10 * 1024 * 1024);  // ✅ 堆上
}
```

---

## 18 软件工程与面向对象设计原则

### 标准软件开发流程

```
需求分析 → 系统设计 → 编码实现 → 测试验证 → 部署上线 → 维护迭代
```

| 模型 | 特点 | 适用场景 |
|------|------|---------|
| **瀑布模型** | 各阶段严格顺序，不可回头 | 需求明确、变化少（政府系统、嵌入式） |
| **敏捷开发** | 2~4 周一个 Sprint，持续交付 | 需求频繁变化的互联网产品 |

---

### 软件工程的本质

> **在约束条件下（时间、人力、成本），持续管理复杂度的过程。**

软件最大的敌人是**复杂度**。随着代码增长，复杂度呈指数上升，工程化的目的就是控制这种上升：

- **模块化**：把复杂系统拆分为可独立理解的部分
- **抽象**：隐藏实现细节，暴露最小接口
- **自动化**：CI/CD、测试、代码检查，减少人为失误
- **文档与规范**：让知识不依赖于某个人

---

### SOLID 面向对象设计原则

#### S — 单一职责原则（SRP）

> 一个类只负责一件事

```cpp
// ❌ 一个类既处理数据又负责输出
class Report {
    void calcData() { ... }
    void printPDF() { ... }
};

// ✅ 拆分职责
class ReportData    { void calc() { ... } };
class ReportPrinter { void print(ReportData&) { ... } };
```

**适用场景**：类开始变得庞大、一个改动影响不相关功能时

---

#### O — 开闭原则（OCP）

> 对扩展开放，对修改关闭

```cpp
// ❌ 每增加一种形状就要改 calcArea
double calcArea(Shape* s) {
    if (s->type == CIRCLE) ...
    if (s->type == RECT)   ...
}

// ✅ 新增形状只需新增类，不改原有代码
class Shape  { virtual double area() = 0; };
class Circle : public Shape { double area() override { ... } };
class Rect   : public Shape { double area() override { ... } };
```

**适用场景**：需求频繁新增类型、策略经常变化时

---

#### L — 里氏替换原则（LSP）

> 子类必须能完全替换父类，不破坏程序正确性

```cpp
// ❌ 正方形继承矩形，修改宽会破坏"面积=长×宽"的约定
class Rectangle { virtual void setWidth(int w); virtual void setHeight(int h); };
class Square : public Rectangle {
    void setWidth(int w) override { width = height = w; }  // 破坏了矩形语义
};

// ✅ 不要强行用继承表达不符合替换语义的关系，改用组合
```

**适用场景**：设计继承体系时，验证子类是否真正满足父类契约

---

#### I — 接口隔离原则（ISP）

> 不要强迫类实现它不需要的接口

```cpp
// ❌ 普通打印机被迫实现扫描功能
class IMachine {
    virtual void print() = 0;
    virtual void scan()  = 0;
};

// ✅ 拆分接口
class IPrinter { virtual void print() = 0; };
class IScanner { virtual void scan()  = 0; };
class MultiFunctionPrinter : public IPrinter, public IScanner { ... };
```

**适用场景**：接口臃肿、实现类中存在大量空实现时

---

#### D — 依赖倒置原则（DIP）

> 高层模块不依赖低层模块，两者都依赖抽象

```cpp
// ❌ 高层直接依赖低层，换数据库就要改业务代码
class OrderService {
    MySQLDatabase db;
};

// ✅ 依赖抽象接口，注入任何实现
class IDatabase { virtual void save() = 0; };
class OrderService {
    IDatabase& db;
};
class MySQLDatabase : public IDatabase { ... };
class RedisDatabase  : public IDatabase { ... };
```

**适用场景**：需要替换底层实现、单元测试需要 Mock 依赖时

---

### 总结

| 原则 | 一句话 | 主要解决 |
|------|--------|---------|
| SRP | 一个类一件事 | 类过于臃肿 |
| OCP | 扩展不修改 | 频繁改动原有代码 |
| LSP | 子类能替换父类 | 继承体系设计错误 |
| ISP | 接口要精简 | 接口强迫实现无用方法 |
| DIP | 依赖抽象不依赖实现 | 模块间耦合过紧 |

> SOLID 不是金科玉律，小型项目过度应用反而增加复杂度。核心是**在合适的时机，用合适的原则降低耦合、提高可维护性**。

## 19 为什么要用size_t

`size_t` 是 C/C++ 中用于表示**对象大小或数组索引**的无符号整数类型。之所以广泛使用它，主要有以下原因：

### 1. 保证足够大的范围

- `size_t` 的大小由平台决定（通常是 `sizeof(size_t) == sizeof(void*)`），能够容纳理论上最大可能的对象尺寸。例如在 64 位系统上，`size_t` 通常为 64 位无符号整数，可以表示最大 `2^64-1` 字节，而 `int` 通常只有 32 位，无法表示超过 2GB 的对象。
- 标准库函数（如 `malloc`、`strlen`、`vector::size()`）均使用 `size_t` 作为长度或计数的类型，保证跨平台兼容性。

### 2. 语义清晰：表示“大小”或“计数”

- `size_t` 的名字直接表明它用于存储**大小**（size），让代码自文档化。使用 `int` 可能会让读者困惑：它是表示大小还是普通整数？是否可能为负？
- 无符号特性强调大小不可能为负数，避免不必要的符号检查。

### 3. 避免符号不匹配警告

- 如果混用有符号整数（如 `int`）和 `size_t`（如 `vector::size()` 的返回值），编译器会产生有符号/无符号不匹配的警告。使用 `size_t` 接收标准库的返回值可以消除这些警告。

### 4. 性能优化

- 某些平台对无符号整数的运算可能比有符号整数更快（尽管差异微小）。更重要的是，使用与机器字长匹配的类型（如 `size_t`）可以减少符号扩展指令，提高访问效率。

### 5. 防止负数错误

- 索引或大小本不应为负。如果错误地传入负数给 `size_t` 参数，会自动转换为一个极大的正数（如 `-1` 转为 `SIZE_MAX`），往往导致程序崩溃或逻辑错误，从而**快速暴露 bug**，而不是静默地产生错误行为。

### 常见使用场景

cpp

```
// 1. 数组索引
for (size_t i = 0; i < array.size(); ++i) { ... }

// 2. 内存大小
size_t size = sizeof(MyClass);
void* p = malloc(size);

// 3. 字符串长度
size_t len = strlen("hello");

// 4. 容器大小
std::vector<int> v(100);
size_t s = v.size();   // size() 返回 size_t
```



### 注意事项

- 当 `size_t` 与有符号整数混合运算时，需小心**隐式转换**带来的意外（如 `-1 > 0u` 为真）。

- 当需要表示**可能为负的差值**（如指针相减）时，应使用 `ptrdiff_t`（有符号版本）。

- 循环递减时使用 `size_t` 要格外小心，因为 `size_t` 永远 `>= 0`，可能导致无限循环：

  cpp

  ```
  for (size_t i = 10; i >= 0; --i)   // 错误！i 永远不小于 0
  ```

  

总之，`size_t` 是为表示大小/计数而设计的正确类型，使用它能提高代码的可移植性、清晰度和安全性。

## 20 设计生产者-消费者模型

使用多线程实现生产者和消费者。 使用缓冲区进行数据传递。 需要同步生产者和消费者，避免数据丢失和重复消费。 要求： 使用适当的同步机制（如互斥锁、条件变量等）确保线程安全。 使用代码展示生产者和消费者的工作流程。

### 1 代码

```c++
// 有界缓冲区类
class BoundedBuffer {
public:
    explicit BoundedBuffer(size_t capacity) : capacity_(capacity), done_(false) {}

    // 生产者：向缓冲区添加数据（阻塞直到有空位）
    void produce(int value) {
        std::unique_lock<std::mutex> lock(mtx_);
        // 等待缓冲区未满
        not_full_.wait(lock, [this] { return buffer_.size() < capacity_ || done_; });
        if (done_) return;  // 如果已经结束，不再生产（可选，实际中通常生产者先结束）
        buffer_.push(value);
        std::cout << "Produced: " << value << ", buffer size: " << buffer_.size() << '\n';
        not_empty_.notify_one();
    }

    // 消费者：从缓冲区取出数据（阻塞直到有数据或生产结束）
    std::optional<int> consume() {
        std::unique_lock<std::mutex> lock(mtx_);
        // 等待缓冲区非空或生产结束
        not_empty_.wait(lock, [this] { return !buffer_.empty() || done_; });
        if (buffer_.empty() && done_) {
            return std::nullopt;  // 无更多数据，消费者可退出
        }
        int value = buffer_.front();
        buffer_.pop();
        std::cout << "Consumed: " << value << ", buffer size: " << buffer_.size() << '\n';
        not_full_.notify_one();
        return value;
    }

    // 通知所有消费者生产已结束（由主线程或最后一个生产者调用）
    void set_done() {
        {
            std::lock_guard<std::mutex> lock(mtx_);
            done_ = true;
        }
        not_empty_.notify_all();
        not_full_.notify_all();
    }

private:
    std::queue<int> buffer_;
    const size_t capacity_;
    bool done_;
    std::mutex mtx_;
    std::condition_variable not_full_;
    std::condition_variable not_empty_;
};

// 生产者线程函数
void producer_task(BoundedBuffer& buffer, int start, int end) {
    for (int i = start; i <= end; ++i) {
        buffer.produce(i);
        // 模拟生产耗时
        std::this_thread::sleep_for(std::chrono::milliseconds(20));
    }
}

// 消费者线程函数
void consumer_task(BoundedBuffer& buffer, int id) {
    while (true) {
        auto val = buffer.consume();
        if (!val.has_value()) {
            std::cout << "Consumer " << id << " exiting (no more data).\n";
            break;
        }
        // 模拟消费耗时
        std::this_thread::sleep_for(std::chrono::milliseconds(50));
    }
}

int main() {
    constexpr size_t BUFFER_CAPACITY = 5;
    constexpr int PRODUCE_COUNT = 20;   // 生产 1..20

    BoundedBuffer buffer(BUFFER_CAPACITY);

    // 创建生产者和消费者线程（可多个，这里各一个）
    std::thread producer(producer_task, std::ref(buffer), 1, PRODUCE_COUNT);
    std::thread consumer(consumer_task, std::ref(buffer), 1);

    // 等待生产者完成
    producer.join();
    // 通知消费者生产已结束
    buffer.set_done();
    // 等待消费者退出
    consumer.join();

    std::cout << "All threads finished.\n";
    return 0;
}
```

```c++
not_full_.wait(lock, [this] { return buffer_.size() < capacity_ || done_; });
等价于
while(!(buffer_.size() < capacity_ || done_)){
    not_full_.wait(lock);
}
```

### 2 伪代码

```
// 共享资源
缓冲区: 队列（固定容量 N）
互斥锁: mutex
条件变量: not_empty（表示缓冲区非空）
条件变量: not_full（表示缓冲区未满）
结束标志: done（初始 false）

// 生产者线程（可多个）
procedure producer():
    while 有数据需要生产:
        item = 生产一个数据
        mutex.lock()
        while 缓冲区.size() == N and not done:
            not_full.wait(mutex)   // 等待缓冲区有空位
        if done:
            mutex.unlock()
            break
        缓冲区.push(item)
        print("生产了", item, "缓冲区大小", 缓冲区.size())
        not_empty.signal()         // 通知消费者
        mutex.unlock()
    // 所有数据生产完毕
    mutex.lock()
    done = true
    not_empty.broadcast()          // 唤醒所有消费者
    mutex.unlock()

// 消费者线程（可多个）
procedure consumer():
    while true:
        mutex.lock()
        while 缓冲区.empty() and not done:
            not_empty.wait(mutex)  // 等待数据或结束
        if 缓冲区.empty() and done:
            mutex.unlock()
            break                  // 没有更多数据，退出
        item = 缓冲区.pop()
        print("消费了", item, "缓冲区大小", 缓冲区.size())
        not_full.signal()          // 通知生产者有空位
        mutex.unlock()
        处理(item)                  // 模拟消费耗时

// 主线程
主():
    创建 M 个生产者线程
    创建 N 个消费者线程
    等待所有生产者线程结束
    // 生产者已设置 done，消费者会自动退出
    等待所有消费者线程结束
```

### 3 问题

1. 为什么生产者和消费者在访问缓冲区时都要使用同一个互斥锁？如果各自用不同的锁会怎样？

   生产者和消费者需要**共享同一个互斥锁**，因为缓冲区（`buffer`）和条件变量（`not_empty`/`not_full`）是它们共同访问的共享资源。

   - **互斥锁的作用**：保证对缓冲区的操作（`push`、`pop`、检查 `size()`）以及条件变量等待谓词中涉及的状态（如 `done`、`buffer.empty()`）是原子且互斥的。
   - **如果用不同锁**：生产者持锁 A 修改缓冲区，消费者持锁 B 读取缓冲区，就无法保证互斥，可能导致数据竞争（例如消费者读取到不一致的缓冲区状态）。条件变量的 `wait` 也要求与互斥锁配对，使用不同的锁会导致未定义行为或死锁。

   **结论**：必须使用**同一把互斥锁**保护所有共享状态的访问。

2. 在 `produce` 函数中，为什么要用 `while (buffer.size() == N)` 而不是 `if`？如果用 `if` 会有什么风险？

   使用 `while` 是为了**处理虚假唤醒**和**多消费者竞争**的情况。

   - **虚假唤醒**：线程可能在没有被 `notify` 的情况下唤醒，此时缓冲区可能仍然满着，用 `if` 会直接向下执行，导致缓冲区溢出（超出容量）。
   - **多消费者竞争**：当多个生产者等待 `not_full` 时，一个消费者取走一个数据后调用 `not_full.notify_one()`，只有一个生产者会被唤醒。但如果用 `if`，被唤醒的生产者会假定缓冲区有空位，但实际上可能另一个生产者已经先被调度并填满了缓冲区，导致该生产者仍然面临满缓冲区。`while` 循环会重新检查条件，避免出错。

   **风险**：使用 `if` 可能导致**缓冲区溢出**（数据丢失或覆盖）或**程序崩溃**。

3. 条件变量的 `wait` 为什么要传入一个已经锁住的 `unique_lock`？`lock_guard` 可以吗？为什么？

   - `wait` 需要**解锁并重新锁定**互斥锁的能力：调用 `wait` 时，线程会原子地释放锁并进入阻塞状态；被唤醒后重新获取锁再返回。
   - `lock_guard` 只提供 RAII 锁定，**不支持手动解锁和重新锁定**，因此无法满足 `wait` 的需求。
   - `unique_lock` 提供 `lock()`、`unlock()` 方法，且支持移动语义，可以灵活管理锁的生命周期，所以条件变量要求使用 `unique_lock`。

   **结论**：`lock_guard` 不可以，必须使用 `std::unique_lock`。

4. 伪代码中生产者结束后设置了 `done = true` 并调用了 `not_empty.broadcast()`。如果不调用 `broadcast()`，只调用 `notify_one()` 可能有什么问题？

   如果只调用 `notify_one()`，只会唤醒**一个**正在等待 `not_empty` 的消费者线程。

   - 如果有多个消费者线程，其他消费者可能仍然阻塞在 `not_empty.wait()` 上，永远不会被唤醒，导致这些线程无法退出（如果主线程等待它们 `join`，程序会卡死）。
   - 即使只有一个消费者，若该消费者在唤醒后因某些原因（例如检查条件发现 `buffer.empty() && done` 前又被其他逻辑干扰？实际上单消费者不会卡死，但多消费者时肯定有问题）。

   **结论**：必须使用 `broadcast()`（或 `notify_all()`）唤醒所有等待的消费者，让它们检查 `done` 并退出。

   `notify_all()`（或称 `broadcast`）会唤醒条件变量等待队列中的所有线程，但**这些线程不会按队列顺序依次出队执行**。具体行为如下：

   1. **所有等待线程被移至就绪队列**：
      调用 `notify_all()` 后，条件变量内部等待队列中的所有线程都被标记为可运行（不再是阻塞状态），它们会进入操作系统的就绪队列，等待 CPU 调度。
   2. **唤醒后必须重新获取互斥锁**：
      每个被唤醒的线程从 `wait()` 返回前，都需要重新获取与条件变量关联的互斥锁。此时所有被唤醒的线程会**竞争**这个锁，**谁先获得锁，谁就从 `wait()` 返回并继续执行**，其他线程则继续在锁上阻塞（不是条件变量上）。
   3. **执行顺序不确定**：
      - 线程获得锁的顺序**不保证**与它们在条件变量队列中的排队顺序一致。这取决于操作系统的线程调度策略（如优先级、时间片、随机性等）。
      - 即使条件变量内部实现是 FIFO 队列（某些实现可能如此），被唤醒后由于锁竞争和调度，实际执行顺序也可能被打乱。
   4. **标准不保证顺序**：
      C++ 标准明确指出，`notify_all()` 不提供任何关于线程唤醒顺序的保证。程序不应依赖任何特定的唤醒顺序。

5. 如果将 `done` 声明为 `bool` 而不是 `std::atomic<bool>`，并且访问时不加锁，会出现什么问题？

   `done` 被多个线程（生产者和消费者）读写。如果不加锁且不用原子类型，会导致**数据竞争**（data race），这是未定义行为。

   - 编译器可能优化掉对 `done` 的重复读取（例如将其缓存在寄存器中），导致一个线程永远看不到另一个线程对 `done` 的修改。
   - CPU 可能乱序执行，导致可见性问题。
   - 即使没有立即崩溃，逻辑可能出错：消费者可能永远看不到 `done=true`，从而无限等待。

   **正确做法**：要么用互斥锁保护对 `done` 的访问，要么声明为 `std::atomic<bool>` 并使用 `load()`/`store()`（但注意与缓冲区的复合条件仍需锁，否则条件检查非原子）。在生产者-消费者模型中，由于 `done` 常与 `buffer.empty()` 一起判断，通常仍放在锁保护下更简单安全。

6. 生产者和消费者函数内部，`mutex.unlock()` 之后才执行 `print` 或 `处理(item)`，这是为什么？如果一直持有锁直到整个循环结束会怎样？

   - **原因**：`print` 和 `处理(item)` 通常是不涉及共享资源的耗时操作（如 I/O、计算）。在持有锁期间执行这些操作会**延长锁的持有时间**，降低并发性能，甚至导致其他线程长时间阻塞（例如生产者等待空位时，消费者无法取数据）。
   - **后果**：如果一直持有锁直到循环结束，那么：
     - 生产者生产一个数据后，直到下次循环开始前才释放锁，消费者无法及时消费，可能使缓冲区很快填满，导致生产者频繁等待，降低吞吐量。
     - 更严重的是，如果 `处理(item)` 中又尝试获取同一把锁（例如递归调用），可能导致死锁。

   **最佳实践**：只在**访问共享资源的最小临界区**内持有锁，将非共享操作移出临界区。

7. 为什么 `produce` 和 `consume` 中的 `while` 循环要检查 `!done`？如果去掉这个条件会有什么后果？

   检查 `!done` 是为了在**生产已结束**的情况下，让等待中的生产者或消费者能够及时退出，避免永久阻塞。

   - 在 `produce` 中：如果生产者线程正在等待 `not_full`，而消费者已经全部退出（或永远不会有消费者取数据），此时 `done` 可能被设为 `true`（被其他生产者），如果不检查 `!done`，生产者将永远等待。
   - 在 `consume` 中：如果消费者正在等待 `not_empty`，而生产者已经结束且缓冲区为空，`done` 为 `true`，消费者需要知道不再有新数据，从而退出。

   **去掉的后果**：

   - 消费者可能永远阻塞在 `not_empty.wait()`，因为生产者不会再产生数据。
   - 生产者（如果有多个）可能永远阻塞在 `not_full.wait()`，因为消费者已退出。

   所以必须检查 `done` 以支持优雅退出。

8. 伪代码中使用了 `not_full.signal()` 和 `not_empty.signal()`。如果改为 `notify_all()` 会有什么不同？哪种更高效？

   - `notify_one()`（`signal()`）只唤醒一个等待线程，而 `notify_all()` 唤醒所有等待线程。
   - **效果差异**：在生产者-消费者模型中，通常只有一个消费者能从缓冲区取数据，唤醒所有消费者会导致“惊群效应”（thundering herd）：多个消费者被唤醒，但只有一个能获得锁并消费，其他消费者会重新进入等待，浪费 CPU 资源。
   - **效率**：`notify_one()` 更高效，因为它避免了不必要的上下文切换和锁竞争。
   - **何时用 `notify_all()`**：当有多个线程等待同一个条件，且条件变化后所有线程都可能需要重新检查（如 `done` 变为 `true` 时），这时应该用 `notify_all()` 让所有消费者退出。

   在常规的 `produce` 和 `consume` 操作中，用 `notify_one()` 即可。

9. 如果有多个生产者线程，伪代码中的结束逻辑（`done = true`）需要怎么修改才能正确通知所有消费者？

   单生产者时，生产者结束设置 `done = true` 并唤醒消费者。多生产者时，需要**最后一个结束的生产者**来设置 `done` 并唤醒消费者。常见做法：

   - 使用一个**原子计数器**记录活跃生产者数量，每个生产者结束时递减计数。当计数变为 0 时，由最后一个生产者设置 `done = true` 并调用 `not_empty.notify_all()`。
   - 或者让主线程等待所有生产者线程结束，然后由主线程设置 `done` 并唤醒消费者（如之前的示例）。

   **伪代码修改**：

   ```
   active_producers = M  // 原子整数
   producer():
       生产所有数据
       if (--active_producers == 0):
           mutex.lock()
           done = true
           not_empty.broadcast()
           mutex.unlock()
   ```

   注意：修改 `active_producers` 需要原子操作或锁保护。

10. `unique_lock` 和 `lock_guard` 的区别是什么？在条件变量的 `wait` 中为什么必须用 `unique_lock`？

    - **`lock_guard`**：简单的 RAII 锁包装器，构造时锁定，析构时解锁，**不能手动解锁或重新锁定**。
    - **`unique_lock`**：更灵活的 RAII 锁包装器，支持：
      - 延迟锁定（`std::defer_lock`）
      - 手动 `lock()`/`unlock()`
      - 尝试锁定（`try_lock`）
      - 移动语义
      - 与条件变量配合（`wait` 需要解锁/重新锁定能力）

    **条件变量 `wait` 必须用 `unique_lock`**，因为 `wait` 内部会调用 `unlock()` 挂起线程，并在被唤醒后重新调用 `lock()`。`lock_guard` 无法提供这些操作。

## 21 RAII是什么

**RAII** 是 **Resource Acquisition Is Initialization** 的缩写，中文常译为“资源获取即初始化”。它是 C++ 中一种重要的资源管理编程惯用法，核心思想是：

> **将资源的生命周期与对象的生命周期绑定**：在对象构造时获取资源（如动态内存、文件句柄、互斥锁等），在对象析构时自动释放资源。

### 工作原理

- **构造函数**：负责获取资源并初始化对象。如果资源获取失败，可以抛出异常，避免创建不完整对象。
- **析构函数**：负责释放资源。无论对象是正常离开作用域还是因异常导致栈展开，析构函数都会被调用，从而保证资源一定会被释放。

### 优点

1. **防止资源泄漏**：资源释放自动发生，无需程序员手动调用释放函数（如 `delete`、`free`、`CloseHandle` 等）。
2. **异常安全**：即使发生异常，已构造的局部对象也会被自动销毁，从而释放其管理的资源，避免资源泄漏。
3. **代码简洁**：将资源管理封装在对象中，减少重复的释放代码，提高可维护性。

### 典型应用

- **智能指针**：`std::unique_ptr`、`std::shared_ptr` 管理动态内存，析构时自动 `delete`。
- **锁管理**：`std::lock_guard`、`std::unique_lock` 在构造时锁定互斥量，析构时自动解锁。
- **文件流**：`std::fstream` 在析构时自动关闭文件。
- **自定义资源**：如数据库连接、套接字等，可以封装为 RAII 类。

### 示例（互斥锁）

cpp

```
#include <mutex>
std::mutex mtx;

void safe_func() {
    std::lock_guard<std::mutex> lock(mtx);  // 构造时加锁
    // ... 操作共享数据 ...
    // 离开作用域时，lock 析构，自动解锁
}
```



### 总结

RAII 是 C++ 资源管理的基石，它让资源释放变得自动且可靠，是编写异常安全、无泄漏代码的关键技术。

## 22 条件变量为什么会出现信号丢失和虚假唤醒问题，怎么解决

条件变量的**信号丢失**和**虚假唤醒**是并发编程中的常见问题，原因和解决方案如下：

### 1. 信号丢失（Lost Wakeup）

**原因**：
生产者发送信号时，消费者尚未进入等待状态（例如还没调用 `wait`），导致信号被忽略，消费者可能永远阻塞。
典型错误顺序：

cpp

```
// 消费者线程
if (!ready) cv.wait(lock);  // 先检查条件再等待，但检查与等待之间可能被中断

// 生产者线程
ready = true;
cv.notify_one();            // 如果此时消费者还没开始 wait，信号丢失
```



**解决方案**：
使用标准模式：**在持有锁的情况下检查条件，然后调用 `wait`**，并且生产者必须在修改条件时也持有同一把锁。

cpp

```
// 消费者
std::unique_lock<std::mutex> lock(mtx);
cv.wait(lock, []{ return ready; });  // 内部自动检查条件，不存在检查与等待的间隙

// 生产者
{
    std::lock_guard<std::mutex> lock(mtx);
    ready = true;
}
cv.notify_one();
```



因为条件检查和 `wait` 是一个原子步骤（`wait` 内部会释放锁并进入等待，但条件检查是在锁保护下完成的），所以不会丢失信号。

### 2. 虚假唤醒（Spurious Wakeup）

**原因**：
某些操作系统或线程库的实现中，条件变量的 `wait` 可能在没有收到 `notify` 的情况下自发返回（为了简化实现或允许某些优化）。这是标准允许的行为。

**解决方案**：
使用**循环检查条件**（或谓词版本的 `wait`），每次被唤醒后重新验证条件是否真正满足。

cpp

```
// 原始方式
while (!ready) {
    cv.wait(lock);
}

// 推荐方式（内部自动循环）
cv.wait(lock, []{ return ready; });
```



即使发生虚假唤醒，条件不满足时会继续等待，不会错误地继续执行。

### 总结

| 问题     | 根本原因                                        | 解决方法                                                     |
| :------- | :---------------------------------------------- | :----------------------------------------------------------- |
| 信号丢失 | 检查和等待不是原子操作，notify 发生在 wait 之前 | 确保条件检查和 wait 在锁保护下原子进行，使用标准 `wait` 谓词版本 |
| 虚假唤醒 | 操作系统/库实现允许不合理的唤醒                 | 使用循环或谓词重新检查条件                                   |

正确使用条件变量的模式（带谓词的 `wait` + 互斥锁保护条件变量）能同时解决这两个问题。

## 23 条件变量.wait(lock)内部会做哪些操作

`std::condition_variable::wait(lock)` 的内部操作可以概括为以下步骤（伪代码级别，基于典型实现如 POSIX 的 `pthread_cond_wait` 或 Windows 的 `SleepConditionVariableCS`）：

1. **前置条件**：调用线程已经持有 `lock`（`std::unique_lock` 保证此时锁被当前线程锁定）。
2. **将当前线程加入条件变量的等待队列**：条件变量内部维护一个等待队列，用于记录所有因调用 `wait` 而阻塞的线程。
3. **原子地释放互斥锁并阻塞线程**：这是最关键的步骤。
   - 底层会执行一个**原子操作**：先调用 `lock.unlock()`，然后立即将当前线程挂起（进入等待状态）。
   - 这两步之间不能被打断，否则可能导致信号丢失（即 `notify` 发生在 `unlock` 之后、挂起之前，但线程还未阻塞，就会错过唤醒）。因此条件变量的实现通常会利用操作系统的同步原语（如 Linux 的 `futex`）来保证释放锁和进入睡眠是原子的。
4. **等待被唤醒**：线程挂起，直到以下情况发生：
   - 另一个线程调用了 `notify_one()` 或 `notify_all()` 并选择了该线程；
   - 或者发生**虚假唤醒**（spurious wakeup，操作系统或库允许的无通知唤醒）；
   - （对于带超时的 `wait_for`/`wait_until`，超时时间到达）。
5. **被唤醒后，重新获取互斥锁**：
   - 线程从挂起状态恢复，首先会尝试重新获取 `lock`（与 `lock.lock()` 行为相同）。
   - 这可能会阻塞，如果其他线程正持有锁。成功获得锁后，`wait` 函数才返回。
6. **返回值**：`wait` 返回时，`lock` 已经被当前线程重新锁定，且线程处于条件变量等待队列之外（即已移除）。

### 扩展：带谓词的 `wait(lock, predicate)` 版本

标准库的 `wait` 重载：

cpp

```
template <class Predicate>
void wait(unique_lock<mutex>& lock, Predicate pred);
```



其内部等价于：

cpp

```
while (!pred()) {
    wait(lock);
}
```



因此，它会在每次（包括虚假唤醒）后重新检查条件，直到条件满足才退出。

### 底层系统调用示例（Linux pthreads）

在 pthreads 中，`pthread_cond_wait(&cond, &mutex)` 的典型实现步骤：

- 将当前线程放入 `cond` 的等待队列。
- 原子地调用 `pthread_mutex_unlock(&mutex)` 并执行 `futex_wait` 挂起线程。
- 被唤醒后，重新调用 `pthread_mutex_lock(&mutex)` 获取锁，然后返回。

### 总结

`wait(lock)` 内部完成了 **“解锁 + 挂起”** 的原子操作，以及唤醒后的 **“重新锁定”**，从而保证线程安全地等待条件变化。这就是它必须与 `std::unique_lock`（可解锁/锁定）而不是 `std::lock_guard` 搭配使用的原因。

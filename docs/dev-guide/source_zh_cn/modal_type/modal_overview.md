# 模态类型概述

## 为什么需要 local 模式

仓颉的对象默认分配在 GC 堆上，由垃圾回收器在不确定的时间回收。函数内部经常会创建一些只在本函数里使用的临时对象，例如从文件或网络读入的字符串、解析得到的中间结构、计算用的临时数组。这些对象在函数返回后就不再需要，但在下一次垃圾回收之前仍然占用堆内存，在调用频繁或数据量大的场景下会推高程序的峰值内存。

local 模式用类型来描述“这个对象不会离开当前函数”。编译器据此检查对象确实没有逃逸，运行时则把这类对象分配在当前函数专属的一块内存区域（称为 local region）中：函数返回时整块 local region 一次性释放，不需要等待垃圾回收。

> **说明：**
>
> 本章所说的 local region 指为 local 对象划分的内存区域，与 GC 堆内部的 region 不是同一个概念。

## 开启 local 模式

local 模式目前是实验特性，默认关闭。使用时需要同时加上 `--experimental` 和 `--enable-local-mode` 两个编译选项，例如：

```shell
cjc --experimental --enable-local-mode main.cj -o main
```

使用 cjpm 构建时，可以在 `cjpm.toml` 的 `compile-option` 字段中加入这两个选项。`compile-option` 只对当前模块生效，用到 local 模式的依赖模块需要在各自的 `cjpm.toml` 中开启。也可以在入口模块的 `override-compile-option` 字段中统一开启，但这样所有依赖模块中的 `exclave`、`demode` 都会成为关键字。参见[cjpm 介绍](../compile_and_build/cjpm_usage.md)。本章的示例都按开启 local 模式编写。

local 模式统一使用 cjc 的实验性编译选项，cjpm 没有单独的 local 模式配置项。本章介绍的 local 语义规则及 `Copyable` 限制只在开启 local 模式后生效并进行检查。

未开启 local 模式时：

- `exclave`、`demode` 是普通标识符，已有代码中同名的变量、函数等不受影响。
- 使用 `@ local!`、`@ local?`、`@ ~local` 等模式标注会报错 `local mode is disabled`，并提示开启方式。
- 调用参数或返回值本身带 `@ local!` 或 `@ local?` 的函数，调用 `this @ local!` 的成员函数或构造函数，或访问类型为 `@ local!` 的属性，需要开启 local 模式。
- 上游包还可能导出函数类型的变量或常量。如果该函数类型的参数或返回值带有 `@ local!` 或 `@ local?`，使用这些变量或常量也需要开启 local 模式。
- 导入包含上述声明的包不受影响，包中不涉及 local 模式的函数和变量可以照常使用。

上述模式及其在构造函数、成员函数和属性中的用法，见后续的[模式与模态类型](#模式与模态类型)和[local 模式的使用规则](./local_mode_rules.md)。

> **注意：**
>
> - local 模式仍属于实验特性，相关的语法、规则和编译选项可能在后续版本中发生变化，不保证兼容。
> - local 模式的编译产物不保证跨 SDK 版本兼容。如果一个包使用了 local 模式，它和依赖它的包都应使用同一版本的 SDK 编译；升级 SDK 后，需要用新版本重新编译这些包。
> - 编译选项只决定源码中能否使用 local 模式。标准库中有些 API 的内部实现用到了 local 模式，无论是否开启，这些实现都照常生效。
> - 工具链默认关闭 local 特性。即使通过实验性选项启用，也不保证 cjfmt、cjlint、LSP、cjdb 等工具能识别或完整支持 local 模式。

## 模式与模态类型

在类型后面加上 `@` 和一个模式（mode），就得到模态类型（modal type）。与 local 相关的模式有三种：

| 写法 | 含义 |
|------|------|
| `T @ ~local` | 非 local。对象可以自由地被保存、返回、传递，与普通的仓颉对象完全相同 |
| `T @ local!` | 一定是 local。对象的生命周期限定在当前函数（或调用者）的 local region 内，不能逃逸出去 |
| `T @ local?` | 可能是 local。值可能是 local 的，也可能不是；使用时按 local 的要求加以限制 |

不写模式等价于写 `@ ~local`，因此开启 local 模式后，已有代码中的类型含义不变。开启后关键字的变化见[关键字](../Appendix/keyword.md)。

`T @ ~local` 和 `T @ local!` 都是 `T @ local?` 的子类型：`@ local?` 的变量或参数既可以接收 local 值，也可以接收非 local 值。`@ ~local` 与 `@ local!` 之间没有子类型关系；对于未实现 `Copyable` 的类型，不能在这两种模式之间赋值。local 值不能当作非 local 值使用，因为它会随函数返回而释放；反过来，非 local 值也不能当作 `@ local!` 使用，因为程序可以把 local 对象保存到 `@ local!` 值的成员或元素中（参见 [exclave 表达式](./exclave.md)），普通对象一旦引用了 local 对象，该对象释放后就会留下悬空引用。需要 local 版本时，应构造一个 local 副本，见[常用标准库类型的 local 写法](#常用标准库类型的-local-写法)。`Copyable` 类型可以按值复制转换模式，详见[Copyable 与 demode](./copyable_and_demode.md)。

书写时需要注意：

- `@` 与模式之间可以有空格，但模式本身是一个整体，`~local`、`local!`、`local?` 内部不能有空格，`@local !`、`@~ local` 都是错误写法。本手册推荐在 `@` 与模式之间写一个空格，如 `@ local!`。
- `local` 不是关键字，只有出现在 `@` 之后时才表示模式，在其他位置仍然可以作为普通标识符使用。

## 可以标注模式的位置

模式可以写在以下常见位置：

- 局部变量的类型
- 函数的参数类型和返回类型
- 函数类型中的参数类型和返回类型，如 `(String @ local?) -> Int64`
- 整个函数类型，用于指定函数对象本身的模式，如 `(() -> Int64) @ local?`，参见[闭包与嵌套函数](./local_mode_rules.md#闭包与嵌套函数)
- 构造调用之后，用于指定构造出的实例的模式，如 `C() @ local!`
- 实例成员属性的类型。属性类型的模式同时也是其 getter 和 setter 中 `this` 的模式，例如 `prop name: String @ local!` 只能通过 `@ local!` 的实例访问，参见[其他限制速查](./local_mode_rules.md#其他限制速查)
- 静态成员变量的类型，但只能采用 `@ ~local` 模式

在函数类型中，`->` 右侧的模式属于返回类型：`() -> String @ local?` 表示返回 `String @ local?` 的函数。要标注函数对象本身的模式，需要给整个函数类型加上括号，写作 `(() -> String) @ local?`。

嵌套函数是声明，没有写类型的位置，因此在 `func` 前写 `@ local?` 来标注它作为函数对象的模式，例如 `@ local? func length(): Int64` 的类型是 `(() -> Int64) @ local?`。这样它就可以捕获外层的非 `Copyable` 类型的 `@ local?` 变量，参见[闭包与嵌套函数](./local_mode_rules.md#闭包与嵌套函数)。

以下位置不能标注模式：

- class、struct 的实例成员变量：成员的模式由所在实例决定，详见[成员与实例的模式](./local_mode_rules.md#成员与实例的模式)。
- 泛型实参本身不能带模式，`Array<String @ local!>` 是错误写法。local 字符串数组应写作 `Array<String> @ local!`：`String` 指定元素类型，`@ local!` 指定整个数组的模式；从该数组读取的字符串也受 local 限制，详见[成员与实例的模式](./local_mode_rules.md#成员与实例的模式)。
- 元组的元素类型和 `?T` 中的 `T` 同样不能单独带模式，`(Int64, String @ local!)`、`?(String @ local?)` 都是错误写法。模式要标注在整个类型上，如 `(Int64, String) @ local!`、`?String @ local?`（即 `Option<String> @ local?`）。

## 第一个示例

下面的示例在 `sumOfSquares` 中创建一个临时数组，把它标注为 `@ local!`，并交给只读的辅助函数 `total` 求和。`total` 的参数标注为 `@ local?`，因此它既能接收 local 数组，也能接收普通数组。`getAllocatedLocalObjectSize()` 返回当前活跃的 local region 所占用的内存，用来观察 local 数组何时被回收：

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
import std.runtime.getAllocatedLocalObjectSize

// 只读访问数组，local 数组和普通数组都可以传入
func total(xs: Array<Int64> @ local?): Int64 {
    var sum = 0
    for (i in 0..xs.size) {
        sum += xs[i]
    }
    sum
}

func sumOfSquares(n: Int64): Int64 {
    // 临时数组只在本函数内使用，函数返回时随 local region 一起释放
    let squares = Array<Int64>(n, {i => i * i}) @ local!
    println("in sumOfSquares, local memory in use: ${getAllocatedLocalObjectSize() > 0}")
    total(squares)
}

main() {
    let before = getAllocatedLocalObjectSize()
    let result = sumOfSquares(1000)
    let after = getAllocatedLocalObjectSize()
    println("result: ${result}")
    println("local memory released: ${after == before}")
    println(total([1, 2, 3]))
}
```

执行结果为：

```text
in sumOfSquares, local memory in use: true
result: 332833500
local memory released: true
6
```

local 对象不能离开创建它的函数。如果试图把上例中的 `squares` 作为返回值，编译器会报错：

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func squaresOf(n: Int64): Array<Int64> {
    let squares = Array<Int64>(n, {i => i * i}) @ local!
    squares // Error, a @ local! value cannot be returned as @ ~local
}
```

## 常用标准库类型的 local 写法

标准库中的常用类型提供了接收或返回 local 值的重载，但并非每个 API 都支持所有模式。下面列出入门时常用的写法；`ArrayList` 和 `HashMap` 需要导入 `std.collection` 包。

| 操作 | 写法 | 说明 |
|---|---|---|
| 从普通字符串构造 local 副本 | `String(text) @ local!` | 直接从字符串构造 local 副本 |
| 切分 local 字符串 | `localText.split(",")` | `localText` 为 `@ local!` 时，结果为 `Array<String> @ local!` |
| 构造 local 数组 | `Array<Int64>(3, {i => i * 10}) @ local!` | 按元素初始化函数构造 |
| 构造 local 动态数组 | `ArrayList<Int64>() @ local!` | 构造空列表，再用 `add` 添加元素 |
| 构造 local 哈希表 | `HashMap<Int64, Int64>() @ local!` | 构造空哈希表，再用 `add` 添加键值对 |

对于非 `Copyable` 类型，给变量加上 `@ local!` 不会自动把已有的普通对象变成 local 对象。需要 local 副本时，应使用对应的 local 构造函数。下面从普通字符串重新构造 local 字符串，再调用 local 版本的 `split`：

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func splitLocal(text: String) {
    let localText = String(text) @ local!
    let parts: Array<String> @ local! = localText.split(",")
    println(parts.size)
    println(parts[0])
    println(parts[1])
}

main() {
    splitLocal("a,b")
}
```

执行结果为：

```text
2
a
b
```

这里创建的是新的 local 字符串，原来的 `text` 仍是普通字符串。普通字符串的 `split` 返回普通数组，不能直接赋给 local 数组变量；`String @ local?` 也没有匹配的 `split` 重载：

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func splitPlain(text: String) {
    let parts: Array<String> @ local! = text.split(",") // Error, the result is @ ~local
}

func splitView(text: String @ local?): Int64 {
    text.split(",").size // Error, split has no overload for a @ local? string
}
```

下面的集合都在当前函数中构造为 local 对象，可以添加元素并读取；函数返回时，它们随 local region 释放：

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
import std.collection.*

func useCollections() {
    let values = Array<Int64>(3, {i => i * 10}) @ local!
    println(values[2])
    let list = ArrayList<Int64>() @ local!
    list.add(7)
    println(list[0])
    let map = HashMap<Int64, Int64>() @ local!
    map.add(1, 9)
    println(map[1])
}

main() {
    useCollections()
}
```

执行结果为：

```text
20
7
9
```

标准库 API 的模式支持需检查对应的重载签名。

## exclave 与调用者的 local region

当需要把函数内新建的 local 对象返回给调用者，或把它保存到调用者传入的 local 对象中时，可以使用 `exclave { ... }`，让新对象分配在调用者的 local region 中；local 构造函数也必须使用 exclave。具体用法、执行过程和限制见 [exclave 表达式](./exclave.md)。

## 何时值得使用 local 模式

local 模式的收益体现在峰值内存，而不是吞吐量：它让临时对象在函数返回时立即释放，从而降低内存占用的峰值。适合的场景是函数内创建、函数内用完、体积较大或数量较多的临时对象。

评估改写效果时，应在相同输入和负载下对比改写前后的峰值内存。改写后，还可以观察 local region 的占用：

- 使用 `std.runtime` 包的 `getAllocatedLocalObjectSize()` 获取当前活跃 local region 占用的内存。它统计的是已向 local region 分配的内存块容量（包含内存块中尚未用完的部分），local 对象不计入 `getAllocatedHeapSize()` 的结果。
- 调用 `getAllocatedLocalObjectSize()` 时会暂停所有仓颉线程，不宜在性能敏感的路径上频繁调用。该函数的详细说明参见标准库 `std.runtime` 包的 API 文档。

单个 local region 的大小有上限，超出上限时会抛出 `LocalRegionOutOfMemoryError`。上限可以通过环境变量 `cjLocalRegionMaxSize` 配置，参见[运行时环境变量](../Appendix/runtime_env.md#cjlocalregionmaxsize)。

## 本章内容

- [local 模式的使用规则](./local_mode_rules.md)：赋值与逃逸、成员的模式、构造 local 实例、重载、闭包，以及使用约束。
- [exclave 表达式](./exclave.md)：如何修改 local 参数、如何把 local 对象返回给调用者。
- [Copyable 与 demode](./copyable_and_demode.md)：不受 local 限制的值类型，以及让成员变量不继承实例模式的方法。

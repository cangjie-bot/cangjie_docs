# local 模式的使用规则

> **说明：**
>
> local 模式是实验特性。本节示例都按开启 local 模式编写，编译时需要加上 `--experimental --enable-local-mode` 编译选项，参见[开启 local 模式](./modal_overview.md#开启-local-模式)。

本节介绍 local 模式的编译期约束。

## 赋值、返回与逃逸

赋值时，除了数据类型相容，还需要满足值的模式与生命周期要求。

`Copyable` 是用于标识可按值复制、并在不同模式间转换的类型的接口。数值、`Bool`、`Rune` 等类型已经实现了该接口，例如下面的 `Int64` 值可以从 `@ local!` 复制为 `@ ~local`，然后返回。完整规则见 [Copyable 与 demode](./copyable_and_demode.md)。

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func count(): Int64 {
    let n: Int64 @ local! = 42
    let m: Int64 = n // OK, Int64 is Copyable
    return n + m
}

main() {
    println(count())
}
```

执行结果为：

```text
84
```

以下赋值规则针对非 `Copyable` 类型。

`@ local!` 的值分为外部和内部两类：

- **外部 local!**（external）：函数的 `@ local!` 参数，以及通过它访问到的成员、元组元素和 `VArray` 元素。它们由调用者创建，属于调用者的 local region。
- **内部 local!**（internal）：在当前函数中创建的 `@ local!` 值，属于当前函数的 local region，函数返回时就会释放。

区分二者是为了避免生命周期较长的对象引用生命周期较短的对象：调用者的对象在当前函数返回后可能仍被使用，不能把本函数内创建的 local 对象保存到它的成员中。

在 exclave 之外，非 `Copyable` 类型的外部 local! 不能直接作为实参继续传给其他函数，即使形参是 `@ local?`。这是子类型关系之外的调用约束。

例如，`f` 接收到的外部 local! 参数 `p` 属于 `f` 的调用者。如果允许 `f` 直接把 `p` 传给 `g` 的 `@ local!` 形参，`g` 会把它视为属于直接调用者 `f` 的 local region。此时，`g` 在 exclave 中新建对象并保存到 `p` 的成员里，新对象会分配在 `f` 的 local region，而 `p` 实际属于更外层的调用者。`f` 返回后，`p` 的成员就会引用已释放的对象。

只需读取时，可以先把外部值赋给一个 `@ local?` 局部变量，再传给 `@ local?` 形参；此后对该引用的操作受 `@ local?` 限制，例如不能通过它给非 `Copyable` 成员赋值。需要在调用者的 region 中操作时，使用 exclave。完整示例见[传递 @ local! 参数](#传递--local-参数)。

### 变量的初始化和整体赋值

对于 `var x: T @ 模式`，整体赋值 `x = value` 的规则如下。`let` 变量初始化时也需要满足相同的模式要求，但初始化后不能再次赋值。

| 目标变量的模式及赋值位置 | 可以赋入的值 |
|---|---|
| `@ ~local` | 只能是 `@ ~local` |
| `@ local!` | 只能是 `@ local!`，而且目标和值必须同为内部或同为外部 |
| `@ local?`，在定义该变量的函数或 lambda 内初始化或赋值 | 可以是 `@ ~local`、`@ local!` 或 `@ local?` |
| `@ local?`，在捕获该变量的 lambda 或嵌套函数内部赋值 | 只能是 `@ ~local` |

对于被捕获的、非 `Copyable` 类型的 `@ local?` 变量，在捕获它的 lambda 或嵌套函数内部给它赋值时，只能赋 `@ ~local` 值。不能给它赋一个在该函数中创建的 local 值，否则闭包返回后该变量可能仍引用已释放的对象。`Copyable` 类型的变量不受这条赋值限制。

声明为 `@ local!` 的局部变量总是内部 local!，因此外部 local!（`Copyable` 类型除外）不能用来初始化 `@ local!` 局部变量。需要在局部变量中保留对外部值的引用时，可以把该变量声明为 `@ local?`，示例见[传递 @ local! 参数](#传递--local-参数)。

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func keep(s: String @ local!) {
    let alias: String @ local! = s // Error, cannot initialize an internal local! with an external local!
}
```

下面的示例中，`@ local?` 变量 `any` 先后接收一个 `@ local!` 数组和一个 `@ ~local` 数组：

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func assign() {
    var plain: Array<Int64> = [1, 2, 3] // @ ~local variable
    var loc: Array<Int64> @ local! = Array<Int64>(2, {i => i * 10}) @ local! // @ local! variable
    var any: Array<Int64> @ local? = plain // @ local? variable

    any = loc // OK, @ local? accepts a @ local! value
    println(any.size)
    any = plain // OK, @ local? also accepts a @ ~local value
    println(any.size)
}

main() {
    assign()
}
```

执行结果为：

```text
2
3
```

`@ ~local` 与 `@ local!` 变量之间不能互相赋值：

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func assign() {
    var plain: Array<Int64> = [1, 2, 3]
    var loc: Array<Int64> @ local! = Array<Int64>(2, {i => i * 10}) @ local!
    plain = loc // Error, a @ ~local variable cannot accept a @ local! value
    loc = plain // Error, a @ local! variable cannot accept a @ ~local value
}
```

非 `Copyable` 类型的 local 值不能保存到全局变量或静态成员变量中，因为它们只能采用 `@ ~local` 模式；也不能把 local 值直接作为 `@ ~local` 返回值返回：

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
var cache: String = ""

func process(): String {
    let s: String @ local! = "temporary"
    let t: String = s // Error, @ local! cannot be assigned to @ ~local
    cache = s // Error, @ local! cannot be stored in a global variable
    return s // Error, @ local! cannot be returned as @ ~local
}
```

### 成员与实例的模式

class 和 struct 的实例成员变量声明时不写模式，它的模式由所在实例决定：通过 `@ local!` 的实例访问到的成员是 `@ local!`，通过 `@ local?` 的实例访问到的成员是 `@ local?`。同理，`Array<T> @ local!` 的元素也是 `@ local!`。

元组的模式由各元素共同决定：如果元素中既有 `@ local!` 的值又有 `@ ~local` 的值，元组的模式是 `@ local?`。

### 实例成员变量的赋值

对于非 `Copyable` 类型的实例成员变量，赋值规则如下：

| 成员的模式 | 可以赋入的值 |
|---|---|
| `@ ~local` | 只能是 `@ ~local` |
| `@ local!` | 只能是 `@ local!`，而且成员和值必须同为内部或同为外部 |
| `@ local?` | 通常不允许赋值 |

`@ local?` 的实例实际可能是非 local 的，也可能是 `@ local!` 的：前者的成员不能引用 local 对象，后者的成员必须是 local 对象。无论赋入哪种模式的值，都可能不满足其中一种情况，因此不允许赋值。

构造函数、实例成员函数和终结器可以在参数列表开头用 `this @ 模式` 指定 `this` 的模式，具体语法见[构造 local 实例](#构造-local-实例)、[成员函数的 this 模式](#成员函数的-this-模式)和[其他限制速查](#其他限制速查)。在 `this @ local?` 构造函数或终结器中，允许给 `this` 的成员赋数据类型相容、任意模式的值。

`Copyable` 类型的成员不受上述模式限制；`demode` 成员始终采用 `@ ~local` 模式，见[demode 成员变量](./copyable_and_demode.md#demode-成员变量)。

数组元素的规则相同：不能给 `@ local?` 数组的非 `Copyable` 元素赋值，`Copyable` 元素不受此限制。下面的示例依次演示：通过 `@ local!` 实例读出的成员也是 `@ local!`，不能保存到全局变量；不能给 `@ local?` 实例或数组的非 `Copyable` 成员、元素赋值；`Copyable` 成员和元素可以赋值：

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
class Order {
    var item: String
    var count: Int64
    init(item: String, count: Int64) {
        this.item = item
        this.count = count
    }
}

var lastItem: String = ""

func record(o: Order @ local!) {
    lastItem = o.item // Error, o.item has type String @ local!
}

func update(o: Order @ local?) {
    o.count = 2 // OK, Int64 is Copyable
    o.item = "pen" // Error, cannot assign to non-Copyable member of a @ local? instance
}

func fill(names: Array<String> @ local?, counts: Array<Int64> @ local?) {
    counts[0] = 0 // OK, Int64 is Copyable
    names[0] = "none" // Error, cannot assign to non-Copyable element of a @ local? array
}
```

### 返回值

函数内部创建的非 `Copyable` local 值不能在函数返回后继续使用。外部 `@ local!` 参数及其成员属于调用者的 local region，可以作为 `@ local!` 返回值返回给调用者。需要返回函数中新建的 local 对象时，使用 [exclave](./exclave.md#返回-local-对象)。

`@ local?` 的变量或参数可以接收 local 值和非 local 值，因此常用于只读取、不保存数据的函数参数：

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func describe(s: String @ local?): String {
    if (s.isEmpty()) {
        "empty"
    } else {
        "size ${s.size}"
    }
}

main() {
    let a: String @ local! = "local"
    let b: String = "not local"
    println(describe(a))
    println(describe(b))
}
```

执行结果为：

```text
size 5
size 9
```

由于 `@ local?` 的值可能是 local 的，它不能作为 `@ ~local` 返回值返回：

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func keep(s: String @ local?): String {
    return s // Error, @ local? cannot be returned as @ ~local
}
```

在 exclave 之外，`@ local?` 参数本身，以及通过参数访问到的（包括嵌套的）字段，可以直接作为 `@ local?` 返回值返回。返回值仍受 local 限制，并没有转换为 `@ ~local`：

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func identity(s: String @ local?): String @ local? {
    s
}

main() {
    let local: String @ local! = "local"
    println(identity(local))
    println(identity("plain"))
}
```

执行结果为：

```text
local
plain
```

数组元素不在此列；`@ local?` 局部变量也不能这样返回，因为它可能引用本函数创建的 local 对象：

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func firstOf(xs: Array<String> @ local?): String @ local? {
    xs[0] // Error, an array element is not a field of the parameter
}

func viaLocal(s: String @ local?): String @ local? {
    let v: String @ local? = s
    v // Error, a @ local? local variable cannot be returned
}
```

构造新的数组或元组，与访问外部值的成员不同。数组或元组字面量要采用 `@ local!` 模式，所有元素都必须是内部值；不能因为字面量中含有外部 local! 元素，就把整个字面量视为外部 local! 并直接返回。需要将外部 local! 元素组合成 local 返回值时，应在 exclave 中构造并返回，参见[返回 local 对象](./exclave.md#返回-local-对象)。

## 构造 local 实例

对于非 `Copyable` 类型，实例的模式由构造函数决定。在 `init` 的参数列表开头写上 `this @ local!` 或 `this @ local?`，该构造函数构造出的就是相应模式的实例；不写时等价于 `this @ ~local`。主构造函数总是构造 `@ ~local` 的实例。`Copyable` 类型还可以在普通构造后按值转换模式，见下方示例。

`this @ local!` 和 `this @ local?` 的构造函数，函数体必须以 exclave 表达式开头，通常直接用 `exclave` 修饰 `init` 来表达（参见 [exclave 表达式](./exclave.md)）。在这样的构造函数中可以给成员赋值，适用[实例成员变量的赋值](#实例成员变量的赋值)中对 `this` 初始化的例外规则。

构造实例时，可以在构造调用之后显式写上模式来选择构造函数，例如 `Order("pen", 2) @ local!`。省略模式时，编译器会按上下文推断，例如变量类型、形参类型或返回类型；若无法唯一推断，则默认为 `@ ~local`。确定模式后，从 `this` 模式与之相同的构造函数中，按函数重载规则选择。下面通过变量类型选择构造函数：

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
class Order {
    var item: String
    var count: Int64
    init(item: String, count: Int64) {
        this.item = item
        this.count = count
    }
    exclave init(this @ local!, item: String @ local!, count: Int64) {
        this.item = item
        this.count = count
    }
}

main() {
    let a = Order("book", 1) // @ ~local instance
    let b = Order("pen", 2) @ local! // @ local! instance
    let c: Order @ local! = Order("ink", 3) // 根据变量类型选择 this @ local! 的构造函数
    println("${a.item} ${b.item} ${c.item}")
}
```

执行结果为：

```text
book pen ink
```

`this @ local?` 的构造函数用于参数模式不确定的情况：它的参数可以声明为 `@ local?`，从而同时接收 local 值和非 local 值并保存到成员中。构造出的实例是 `@ local?` 的，与 local 实例一样随 local region 释放。它不能代替 `this @ local!` 或 `this @ ~local` 的构造函数：如果只有 `this @ local?` 的构造函数，就不能构造 `@ local!` 或 `@ ~local` 的实例。

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
class Tag {
    var name: String
    init(name: String) {
        this.name = name
    }
    exclave init(this @ local?, name: String @ local?) {
        this.name = name
    }
}

func size(t: Tag @ local?): Int64 {
    t.name.size
}

func use(text: String @ local?): Int64 {
    let t = Tag(text) @ local? // text may or may not be local
    size(t)
}

main() {
    let local: String @ local! = "local"
    println(use(local))
    println(use("plain"))
}
```

执行结果为：

```text
5
5
```

如果非 `Copyable` 类型没有对应模式的构造函数，就不能构造该模式的实例。例如下面的 `Point` 只有默认模式的构造函数：

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
class Point {
    var x: Int64
    init(x: Int64) {
        this.x = x
    }
}

main() {
    let p = Point(1) @ local! // Error, no @ local! constructor
}
```

`Copyable` 类型可以先使用普通构造函数，再把结果按值复制到其他模式的变量中，因此不需要为这种转换添加 local 构造函数。下面的 `P` 只定义了普通构造函数：

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
struct P <: Copyable {
    var x: Int64
    init(x: Int64) {
        this.x = x
    }
}

main() {
    let p: P @ local! = P(1) // constructed by the ordinary constructor, then copied
    let plain = P(2)
    let copied: P @ local! = plain
    println("${p.x} ${copied.x}")
}
```

执行结果为：

```text
1 2
```

在构造调用之后显式写 `P(1) @ local!`，仍表示选择 `this @ local!` 构造函数。如果没有该构造函数，即使 `P` 实现了 `Copyable`，也会报错；需要使用普通构造后复制的方式时，采用上例的变量类型标注写法：

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
struct P <: Copyable {
    var x: Int64
    init(x: Int64) {
        this.x = x
    }
}

func create() {
    let p = P(1) @ local! // Error, the call explicitly requires a @ local! constructor
}
```

`this @ local!` 或 `this @ local?` 的构造函数如果没有写成 exclave，编译器会报错：

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
class Label {
    var text: String
    init(text: String) {
        this.text = text
    }
    init(this @ local!, text: String @ local!) { // Error, body must be an exclave
        this.text = text
    }
}
```

在构造函数中通过 `this(...)` 调用当前类型的另一个构造函数，或通过 `super(...)` 调用父类构造函数时，被调用的构造函数必须与当前构造函数的 `this` 模式相同。在 `this @ local!` 或 `this @ local?` 的构造函数中，该调用还必须是 exclave 块中的第一个表达式：

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
open class Base {
    var n: Int64 = 7
    init() {}
    exclave init(this @ local!) {}
}

class Child <: Base {
    exclave init(this @ local!) {
        super() // OK, calls Base's this @ local! constructor
    }
}

main() {
    let child = Child() @ local!
    println(child.n)
}
```

执行结果为：

```text
7
```

如果父类只有默认的 `this @ ~local` 构造函数，local 构造函数不能调用它：

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
open class Base {
    init() {}
}

class Child <: Base {
    exclave init(this @ local!) {
        super() // Error, no constructor with this @ local!
    }
}
```

## 成员函数的 this 模式

实例成员函数中 `this` 的模式也需要确定。在参数列表开头写 `this @模式` 来指定，例如 `func show(this @ local?): String`；不写时等价于 `this @ ~local`。`this @...` 只是对 `this` 的说明，不是参数，调用时不需要也不能为它传值。

通过实例调用成员函数时，实例的模式必须是 `this` 模式的子类型：

| 实例的模式 | 可以调用的成员函数 |
|-----------|------------------|
| `@ ~local` | `this @ ~local`（即不写）、`this @ local?` |
| `@ local!` | `this @ local!`、`this @ local?` |
| `@ local?` | `this @ local?` |

因此，如果希望某个只读的成员函数对任何模式的实例都可用，应把它声明为 `this @ local?`。

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
class Counter {
    var n: Int64 = 0
    init() {}
    exclave init(this @ local!) {}
    func value(): Int64 {
        n
    }
    func reset(this @ local!): Unit {
        n = 0
    }
}

func use(c: Counter @ local?) {
    let a = Counter() @ local!
    a.value() // Error, a @ local! instance cannot call a this @ ~local member function
    c.value() // Error, a @ local? instance cannot call a this @ ~local member function
    Counter().reset() // Error, a @ ~local instance cannot call a this @ local! member function
}
```

同一个成员函数可以只按 `this` 模式重载。在可通过类型检查的候选中，`this` 模式的匹配优先于参数类型：先选择 `this` 模式与实例模式最匹配的版本，只有无法据此确定时才继续比较参数类型：

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
class C {
    init() {}
    exclave init(this @ local!) {}
    func name(this @ ~local): String {
        "this @ ~local"
    }
    func name(this @ local?): String {
        "this @ local?"
    }
    func name(this @ local!): String {
        "this @ local!"
    }
}

func test(p: C @ local?) {
    let a = C()
    let b = C() @ local!
    println(a.name())
    println(b.name())
    println(p.name())
}

main() {
    test(C())
}
```

执行结果为：

```text
this @ ~local
this @ local!
this @ local?
```

实现接口函数或覆盖父类函数时，`this` 模式必须与被实现或被覆盖的函数完全相同：

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
interface Shape {
    func area(this @ local?): Int64
}

class Square <: Shape {
    let side: Int64
    init(side: Int64) {
        this.side = side
    }
    public func area(this @ local?): Int64 { // same this mode as Shape.area
        side * side
    }
}

func areaOf(s: Shape @ local?): Int64 {
    s.area()
}

main() {
    println(areaOf(Square(3)))
}
```

执行结果为：

```text
9
```

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
interface Shape {
    func area(this @ local?): Int64
}

class Square <: Shape { // Error, area(this @ local?) is not implemented
    public func area(): Int64 {
        1
    }
}

open class Base {
    public open func id(this @ local?): Int64 {
        1
    }
}

class Derived <: Base {
    public override func id(): Int64 { // Error, this mode differs from the overridden function
        2
    }
}
```

## 按参数模式重载

函数可以只按参数的模式重载。标准库中常见的做法是同时提供 `f(x: T)` 和 `f(x: T @ local?)` 两个版本。调用时，编译器选择参数模式与实参模式最匹配的版本：

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func f(_: String): String {
    "f(String)"
}

func f(_: String @ local?): String {
    "f(String @ local?)"
}

func g(_: String @ local!): String {
    "g(String @ local!)"
}

func g(_: String @ local?): String {
    "g(String @ local?)"
}

func test(p: String @ local?) {
    let a: String = "a"
    let b: String @ local! = "b"
    println(f(a))
    println(f(b))
    println(f(p))
    println(g(a))
    println(g(b))
    println(g(p))
}

main() {
    test("p")
}
```

执行结果为：

```text
f(String)
f(String @ local?)
f(String @ local?)
g(String @ local?)
g(String @ local!)
g(String @ local?)
```

即：实参为 `@ ~local` 时优先选 `@ ~local` 版本，没有时选 `@ local?` 版本；实参为 `@ local!` 时优先选 `@ local!` 版本，没有时选 `@ local?` 版本；实参为 `@ local?` 时只能选 `@ local?` 版本。

如果没有任何版本能接收实参的模式，调用会报错。例如只提供 `@ local!` 版本时，不能传入 `@ ~local` 或 `@ local?` 的实参：

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func h(s: String @ local!): Int64 {
    s.size
}

func test(p: String @ local?) {
    let a: String = "a"
    h(a) // Error, no h accepts a @ ~local argument
    h(p) // Error, no h accepts a @ local? argument
}
```

## 传递 @ local! 参数

外部与内部 local! 的区别及传参限制，见[赋值、返回与逃逸](#赋值返回与逃逸)。局部变量初始化的限制，见[变量的初始化和整体赋值](#变量的初始化和整体赋值)。

在 exclave 之外，外部 local!（`Copyable` 类型除外）还有两条限制：

- 不能直接作为实参传给其他函数；
- 不能用来调用 `this @ local!` 的成员函数。

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
class Doc {
    var title: String
    init(title: String) {
        this.title = title
    }
    exclave init(this @ local!, title: String @ local!) {
        this.title = title
    }
    exclave func clear(this @ local!): Unit {
        title = ""
    }
}

func size(s: String @ local?): Int64 {
    s.size
}

func process(d: Doc @ local!): Int64 {
    d.clear() // Error, cannot call a this @ local! member function on a @ local! parameter
    size(d.title) // Error, cannot pass a @ local! parameter (or its member) as an argument
}
```

需要传递时，可以采用以下两种写法之一：

- 先把参数赋给一个 `@ local?` 变量，再把该变量传给 `@ local?` 形参；
- 在 [exclave](./exclave.md) 中传递或调用，此时外部 local! 视同内部 local!，可以传给 `@ local!` 形参，也可以调用 `this @ local!` 的成员函数。

通过外部 local! 调用 `this @ local?` 的成员函数、访问 `Copyable` 类型的成员，不受上述限制。

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func size(s: String @ local?): Int64 {
    s.size
}

func sizeL(s: String @ local!): Int64 {
    s.size
}

func forward1(s: String @ local!): Int64 {
    let t: String @ local? = s
    size(t)
}

exclave func forward2(s: String @ local!): Int64 {
    sizeL(s)
}

main() {
    let s: String @ local! = "abc"
    println(forward1(s))
    println(forward2(s))
}
```

执行结果为：

```text
3
3
```

## 闭包与嵌套函数

lambda 表达式和嵌套函数会捕获外部变量，捕获规则如下：

- 不能捕获非 `Copyable` 类型的 `@ local!` 变量。
- 捕获了非 `Copyable` 类型的 `@ local?` 变量的 lambda，其自身的模式是 `@ local?`，因此不能逃逸，例如不能赋值给 `@ ~local` 的变量。只捕获 `Copyable` 类型的 local 变量时，lambda 的模式仍是 `@ ~local`。
- 不能创建 `@ local!` 的函数对象，即 lambda 的类型不能是 `@ local!` 的函数类型。

`Copyable` 类型的变量按值捕获：闭包保存的是值的副本，不引用 local region 中的对象，因此不受上述限制。例如下面的 lambda 捕获了 `Int64` 类型的 local 变量，它的模式仍是 `@ ~local`，可以在函数返回后继续使用：

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func makeCounter(): () -> Int64 {
    let n: Int64 @ local! = 42
    {=> n} // OK, Int64 is Copyable and captured by value
}

main() {
    println(makeCounter()())
}
```

执行结果为：

```text
42
```

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
var onDone: () -> Int64 = {=> 0}

func run() {
    let text: String @ local! = "abc"
    let view: String @ local? = text
    let byText = {=> text.size} // Error, cannot capture a @ local! variable
    let byView = {=> view.size}
    onDone = byView // Error, a @ local? lambda cannot be assigned to @ ~local
    let fixed: (() -> Int64) @ local! = {=> 1} // Error, a lambda cannot have a @ local! function type
}
```

而捕获非 `Copyable` 类型的变量时，闭包引用的是 local region 中的对象，因此受到上述限制：

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func makeLabel(): () -> Int64 {
    let s: String @ local! = "abc"
    {=> s.size} // Error, cannot capture a @ local! variable
}
```

捕获 `@ local?` 变量的 lambda 可以在当前函数内正常调用。写出它的类型时，要给整个函数类型加上括号，写作 `(() -> Int64) @ local?`；`() -> Int64 @ local?` 标注的是返回值的模式：

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func run() {
    let text: String @ local! = "hello"
    let view: String @ local? = text
    let length: (() -> Int64) @ local? = {=> view.size}
    println(length())
}

main() {
    run()
}
```

执行结果为：

```text
5
```

嵌套函数默认是 `@ ~local` 的，与普通仓颉中一样可以自由返回和保存，因此不能捕获非 `Copyable` 类型的 `@ local?` 和 `@ local!` 变量。要捕获非 `Copyable` 类型的 `@ local?` 变量，在 `func` 前写上 `@ local?`。这个模式标注的是嵌套函数作为函数对象的模式，而不是返回值的模式：下例中 `length` 的类型是 `(() -> Int64) @ local?`，与捕获 `@ local?` 变量的 lambda 一样不能逃逸。无论是否写 `@ local?`，嵌套函数都不能捕获非 `Copyable` 类型的 `@ local!` 变量；`Copyable` 类型的 local 变量不受这些捕获限制：

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func run() {
    let text: String @ local! = "hello"
    let view: String @ local? = text
    @ local? func length(): Int64 {
        view.size
    }
    println(length())
}

main() {
    run()
}
```

执行结果为：

```text
5
```

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func run() {
    let text: String @ local! = "hello"
    let view: String @ local? = text
    func length1(): Int64 {
        view.size // Error, a @ ~local nested function cannot capture a @ local? variable
    }
    @ local? func length2(): Int64 {
        text.size // Error, cannot capture a @ local! variable
    }
}
```

## 约束

以下场景对 local 值有额外限制：

- **spawn**：spawn 块中不能使用块外声明的、非 `Copyable` 类型的 `@ local!` 或 `@ local?` 变量，块的结果只能是 `@ ~local` 的值。spawn 块中可以创建自己的 local 对象，它们在块执行结束时释放。spawn 块中不能使用 exclave 表达式。
- **异常**：`throw` 的操作数只能是 `@ ~local` 的值；catch 中绑定的异常变量是 `@ ~local` 的，可以自由保存。
- **泛型**：泛型实参不能带模式（如 `Array<String @ local!>`）。类型参数也不会带上实参的模式：传入 `String @ local!` 时，`T` 推断为 `String`，形参 `x: T` 仍是 `@ ~local` 的。如果要接收 local 值，需要写成 `x: T @ local?` 等形式，见下方示例。
- **C 语言互操作**：`@C` 修饰的函数，其参数类型和返回类型只能采用 `@ ~local` 模式。
- **exclave 的使用位置**：不能在 spawn 块、`main`、终结器和静态初始化器中直接使用 exclave；也不能在 try 或 catch 块中用 exclave 结束包含该块的函数，详见 [exclave 表达式](./exclave.md#使用限制)。

下面的示例在 spawn 块中使用外层的 `Copyable` 变量，并在块内创建自己的 local 数组。数组在 spawn 块结束时释放，求和结果是可返回的普通 `Int64` 值：

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func sumInThread(n: Int64): Int64 {
    let limit: Int64 @ local! = n // Int64 is Copyable, can be used in spawn
    let fut = spawn {
        let squares = Array<Int64>(limit, {i => i * i}) @ local! // local to the spawn block
        var sum = 0
        for (i in 0..squares.size) {
            sum += squares[i]
        }
        sum
    }
    fut.get()
}

main() {
    println(sumInThread(4))
}
```

执行结果为：

```text
14
```

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func f() {
    let s: String @ local? = "s"
    spawn {
        println(s) // Error, cannot use a @ local? variable declared outside spawn
    }
}

func g(e: Exception @ local?) {
    throw e // Error, the thrown value must be @ ~local
}

func id<T>(x: T): Unit {}

func h() {
    let s: String @ local! = "s"
    id(s) // Error, T is inferred as String, so the parameter x: T is @ ~local
}

@C
func fromC(p: CPointer<Int8> @ local!): Unit {} // Error, @C function cannot use modal types
```

泛型函数要接收 local 值时，把形参标注为 `@ local?`。从 `Array<T> @ local?` 中读出的元素类型是 `T @ local?`，同样受 local 限制；要把元素作为 `T` 返回，需要约束 `T <: Copyable`：

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func count<T>(xs: Array<T> @ local?): Int64 {
    xs.size
}

func first<T>(xs: Array<T> @ local?): T where T <: Copyable {
    xs[0] // OK, the element is copied out because T is Copyable
}

main() {
    let names: Array<String> @ local! = ["a", "b", "c"]
    println(count(names))
    let nums: Array<Int64> @ local! = [7, 8, 9]
    println(first(nums))
}
```

执行结果为：

```text
3
7
```

没有 `Copyable` 约束时，元素不能作为 `T` 返回：

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func firstAny<T>(xs: Array<T> @ local?): T {
    xs[0] // Error, the element is T @ local?
}
```

> **注意：**
>
> 通过 `@ local?` 的值调用接口函数时，调用的是接口中 `this @ local?` 的版本，其返回值也可能带有模式。标准库的部分接口（如 `ToString`）为这类函数提供的默认实现会抛出 `UnimplementedException`。例如对 `x: T @ local?`（`T <: ToString`）调用 `x.toString()` 时，如果实际类型只实现了普通的 `toString()`，程序可以编译通过，但运行时会抛出异常。

## 其他限制速查

以下限制不常遇到，这里只简要列出：

1. 终结器的 `this` 模式必须是所有构造函数 `this` 模式的父类型。例如同时有 `this @ ~local` 和 `this @ local!` 的构造函数时，终结器需写成 `~init(this @ local?)`。
2. 如果一个类型的构造函数能构造出不同模式的实例，那么它的非 `Copyable` 类型的成员变量不能在声明处写初始值，`demode` 成员变量除外。
3. 同名的实例成员属性可以按模式重载，但它们的数据类型必须相同。
4. `abstract` 函数不能用 `exclave` 修饰。
5. 在 exclave 之外，非 `Copyable` 类型的外部 `@ local!` 参数不能直接作为实参传给其他函数，见[传递 @ local! 参数](#传递--local-参数)。
6. 对 `@ local!` 参数做模式匹配时，绑定得到的变量是 `@ local?` 而不是 `@ local!`。
7. `this` 模式说明用于实例成员函数、构造函数和终结器，不能用于全局函数或静态函数。
8. 模式只能写在构造调用之后，如 `C() @ local!`，写在普通函数调用之后会报错。
9. 实例成员属性的 getter 和 setter 按对应的成员函数调用及重载规则处理，其 `this` 模式与属性类型的模式相同。例如 `prop name: String @ local!` 只能通过 `@ local!` 的实例访问；希望任何模式的实例都能读取时，可以把属性类型写成 `@ local?`。
10. 主构造函数的普通参数只能采用 `@ ~local` 模式。
11. 宏的参数和返回值只能采用 `@ ~local` 模式。
12. 效应处理中的 `perform` 操作数只能是 `@ ~local` 的值。

可选参数的默认值表达式不能引用其他参数，也不能捕获非 `Copyable` 类型的 `@ local!` 变量。

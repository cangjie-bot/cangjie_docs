# Copyable 与 demode

> **说明：**
>
> local 模式是实验特性。本节示例都按开启 local 模式编写，编译时需要加上 `--experimental --enable-local-mode` 编译选项，参见[开启 local 模式](./modal_overview.md#开启-local-模式)。

## Copyable 接口

有些类型的值不包含任何引用，赋值时复制的就是值本身，例如 `Int64`、`Bool`。对这样的类型来说，“对象是否逃逸出函数”没有意义，local 模式的限制也就不必施加在它们身上。标准库用 `Copyable` 接口来标识这类类型。

`Copyable` 定义在 `std.core` 包中，是一个空接口。以下类型自动实现了 `Copyable`：

- 所有数值类型，以及 `Rune`、`Bool`、`Unit`、`Nothing`；
- 所有实现了 `CType` 的类型，如 `CPointer<T>` 和 `@C` 修饰的 struct；
- 所有元素类型均实现了 `Copyable` 的元组类型；
- 元素类型 `T` 实现了 `Copyable` 的 `VArray<T, $N>`。

实现了 `Copyable` 的类型的值，可以在 `@ local!`、`@ local?`、`@ ~local` 之间直接转换。转换只是按值复制，不需要额外的运行时操作。

元组的所有元素类型都实现 `Copyable` 时，可以把 `@ local?` 元组按值返回为普通元组：

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func toPlain(t: (Int64, Bool) @ local?): (Int64, Bool) {
    t // OK, the tuple type is Copyable
}

main() {
    let t: (Int64, Bool) @ local! = (1, true)
    let plain: (Int64, Bool) = toPlain(t)
    println("${plain[0]} ${plain[1]}")
}
```

执行结果为：

```text
1 true
```

如果元组中包含未实现 `Copyable` 的类型，例如 `String`，就不能进行同样的转换：

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func toPlain(t: (Int64, String) @ local?): (Int64, String) {
    t // Error, cannot convert @ local? to @ ~local
}
```

同样，元素类型为 `Copyable` 的 `VArray` 可以从 `@ local!` 复制为 `@ ~local`：

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
main() {
    let local: VArray<Int64, $2> @ local! = [1, 2]
    let plain: VArray<Int64, $2> = local
    println(plain[1])
}
```

执行结果为：

```text
2
```

在 exclave 之外，`@ local!` 或 `@ local?` 的 `Copyable` 值不能直接作为接口类型返回，需要先复制到 `@ ~local` 变量再返回。下面分别接收两种模式的值，复制为普通 `Int64` 后以 `ToString` 接口返回：

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func fromLocal(n: Int64 @ local!): ToString {
    let plain: Int64 = n
    plain
}

func fromMaybe(n: Int64 @ local?): ToString {
    let plain: Int64 = n
    plain
}

main() {
    let n: Int64 @ local! = 42
    println(fromLocal(n).toString())
    println(fromMaybe(n).toString())
}
```

执行结果为：

```text
42
42
```

## 为 struct 实现 Copyable

如果 struct 的所有实例成员变量的类型都实现了 `Copyable`，可以声明该 struct 实现 `Copyable`。泛型 struct 可以通过泛型约束来满足这一条件。实现后，普通构造函数产生的值也可以按值复制到 local 模式的变量中，不需要额外的 local 构造函数，见[构造 local 实例](./local_mode_rules.md#构造-local-实例)：

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
struct Point <: Copyable {
    var x: Int64
    var y: Int64
    init(x: Int64, y: Int64) {
        this.x = x
        this.y = y
    }
}

struct Pair<A, B> <: Copyable where A <: Copyable, B <: Copyable {
    var first: A
    var second: B
    init(first: A, second: B) {
        this.first = first
        this.second = second
    }
}

func keep(p: Point @ local?): Point {
    p // OK, Point is Copyable
}

main() {
    let p: Point @ local! = Point(1, 2)
    let r: Point = keep(p) // OK, copy a local Point as a non-local value
    let pr: Pair<Int64, Bool> @ local! = Pair<Int64, Bool>(3, true)
    let pr2: Pair<Int64, Bool> = pr
    println("${r.x} ${r.y} ${pr2.first}")
}
```

执行结果为：

```text
1 2 3
```

仅成员类型满足条件还不够，struct 需要显式声明实现 `Copyable`。下面的 `Plain` 虽然只有一个 `Int64` 成员，但没有实现 `Copyable`，因此不能进行上述转换：

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
struct Plain {
    var x: Int64
    init(x: Int64) {
        this.x = x
    }
}

func keep(p: Plain @ local?): Plain {
    p // Error, Plain does not implement Copyable
}
```

也可以通过扩展为已有的 struct 实现 `Copyable`，规则与在声明处实现相同。例如，已有一个尚未实现 `Copyable`、且所有实例成员变量的类型都实现了 `Copyable` 的 struct `Plain`，可以写 `extend Plain <: Copyable {}`。

以下情况不能实现 `Copyable`：

- struct 中有任何一个实例成员变量的类型没有实现 `Copyable`；
- class 和 enum 类型，即使它们不包含任何引用类型的成员。

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
struct Named <: Copyable {
    var name: String = "" // Error, String is not Copyable
}

class Box <: Copyable {} // Error, a class cannot implement Copyable

enum Color <: Copyable { // Error, an enum cannot implement Copyable
    | Red | Green
}
```

## Copyable 的使用位置

`Copyable` 只能用于类型声明或扩展声明的父接口列表，或作为泛型参数的上界（如 `where T <: Copyable`）使用，不能作为变量、参数等的类型：

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func f(c: Copyable): Unit {} // Error, Copyable cannot be used as a parameter type

func g() {
    let c: Copyable = 1 // Error, Copyable cannot be used as a variable type
}
```

作为泛型约束时，可以让泛型函数对任何模式的实参都返回非 local 的结果：

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func plain<T>(x: T @ local?): T where T <: Copyable {
    x
}

main() {
    let n: Int64 @ local! = 42
    let m: Int64 = plain(n)
    println(m)
}
```

执行结果为：

```text
42
```

## demode 成员变量

实例成员变量的模式通常与所在实例相同（参见[成员与实例的模式](./local_mode_rules.md#成员与实例的模式)）。如果 local 对象需要持有一个普通对象的引用，例如指向全局配置或缓存，成员就不能跟着实例变成 local 的。这时可以用 `demode` 修饰该成员变量：

- `demode` 成员变量的模式始终是 `@ ~local`，与实例的模式无关；
- 因此可以把 `@ ~local` 的值赋给它，读出的值也是 `@ ~local` 的，可以自由保存和返回；
- 相应地，不能把非 `Copyable` 类型的 `@ local!` 或 `@ local?` 值赋给它；`Copyable` 值可以按值复制。

下面通过 `this @ local!` 的构造函数创建 local 对象，再观察其 `demode` 成员的模式。该构造函数必须使用 `exclave`，参见[构造 local 实例](./local_mode_rules.md#构造-local-实例)。

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
var defaultTag: String = "default"

class Node {
    demode var tag: String
    var label: String
    exclave init(this @ local!, label: String @ local!) {
        this.tag = ""
        this.label = label
    }
}

func tagOf(): String {
    let n = Node("temp") @ local!
    n.tag = defaultTag // OK, n.tag is always @ ~local
    n.tag // OK, can be returned
}

main() {
    println(tagOf())
}
```

执行结果为：

```text
default
```

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
var defaultTag: String = "default"

class Node {
    demode var tag: String
    var label: String
    exclave init(this @ local!, label: String @ local!) {
        this.tag = ""
        this.label = label
    }
}

func f() {
    let n = Node("temp") @ local!
    n.label = defaultTag // Error, n.label is @ local!
    let s: String @ local! = "s"
    n.tag = s // Error, n.tag is @ ~local
}
```

## 元素非 local 的 local 数组

`Array<T> @ local!` 的元素也是 local 的。如果想要一个随函数释放的临时数组，但其中存放的是普通对象，可以使用带有 `demode` 成员的包装 struct。下面的 `Demode<T>` 是本示例定义的包装类型，用作数组的元素：

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
struct Demode<T> {
    demode var value: T
    init(v: T) {
        this.value = v
    }
    exclave init(this @ local!, v: T) {
        this.value = v
    }
}

func pickSecond(a: String, b: String): String {
    let arr: Array<Demode<String>> @ local! = [Demode(a) @ local!, Demode(b) @ local!]
    arr[1].value // OK, value is @ ~local
}

main() {
    println(pickSecond("x", "y"))
}
```

执行结果为：

```text
y
```

这样，`Array<Demode<String>> @ local!` 这个数组在函数返回时随 local region 释放，而每个元素的 `value` 引用的是普通对象，不受影响，可以被返回或保存。

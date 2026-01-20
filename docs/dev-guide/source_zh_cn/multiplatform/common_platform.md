# 跨平台

仓颉提供了跨平台开发特性，可以在跨端开发场景中，解决代码复用的问题。用户可以通过区分公共部分代码和平台部分代码，来完成在不同平台共享代码，减少不同平台开发维护相同代码所花费的时间。

> **注意：**
>
> 跨平台开发特性为实验性功能，使用该特性可能有风险。

## 跨平台开发特性介绍

### 公共部分代码和平台部分代码

代码库中与平台无关的部分称作公共部分代码，它包含了可以在所有目标平台上运行的代码，这些代码通常是算法、业务逻辑或其他不依赖于具体平台功能的模块。代码库中与平台相关的部分称作平台部分代码，它包含了只能在特定平台上运行的代码，这些代码通常涉及对操作系统、硬件或其他平台特定功能的调用。公共部分代码与平台部分代码都属于同一个包，平台文件可以依赖公共文件，但是公共文件不可以依赖平台文件。公共部分代码用于在不同平台间共享，其内容可以使用 common 修饰。平台部分代码用于区分不同平台的实现，其内容可以使用 platform 修饰。使用 common/specific 修饰符需要满足如下规则限制：

- common 修饰符只可以出现在公共部分代码中，specific 修饰符只可以出现在平台部分代码中。
- 和 private/const/foreign 修饰符冲突，不可以同时使用。

如下示例，定义了公共部分代码和全局函数 foo。

<!-- compile -->
```cangjie
package cmp
​
public common func foo(): Unit {
    println("I am common")
}
```

如下示例，定义了平台部分代码和全局函数 foo。

<!-- compile -->
```cangjie
package cmp
​
public specific func foo(): Unit {
    println("I am platform")
}
```

更多的细节将在跨平台开发章节详细描述。

### 支持跨平台开发特性的类型和语法特性

下面对支持跨平台开发特性的类型和语法特性介绍详细使用规则。其中一些特性与普通声明基本一致，不做展开介绍，例如：

- common 和 specific 的声明支持异常抛出，异常抛出的栈信息与实际所使用的声明地址一致。
- common 声明中支持存在 Deprecated 声明，其 Deprecated 注解将传播给 specific 声明。不允许在 specific 声明中使用 Deprecated。
- common 声明与对应的 specific 声明必须使用相同的注解（Deprecated 注解除外）。
- 对于跨平台开发中的函数（包括全局函数、成员函数、构造函数），默认参数只能在 common 声明或 specific 声明中的一侧定义。若 common 声明的某参数已指定默认值，则 specific 声明对应位置的参数必须使用相同的参数名作为命名参数，且不得再设置默认值。

#### 全局函数

全局函数支持跨平台特性，用户可以使用 common 和 specific 修饰全局函数。
common 全局函数，可以包含函数实现，也可以不包含函数实现。

<!-- compile -->
```cangjie
common func foo(): Int64
common func goo(a: Int64): Int64 { 1 }
```

如上示例，定义了两个 common 全局函数，其中函数 foo 不包含函数体，goo 包含函数体，都是合法的 common 全局函数定义。
common/specific 全局函数必须满足如下限制：

- common 全局函数必须定义函数返回值类型。
- 当 common 全局函数有完整实现时，可以不定义 specific 全局函数；当 common 全局函数无完整实现时，必须定义 specific 全局函数。
- specific 全局函数的函数签名必须与同包的 common 全局函数匹配，即参数类型必须一致，返回值类型可以是相同类型或子类型，并需要同时满足以下规则：
    - common 全局函数与全局平台函数必须使用相同的修饰符，如 public，unsafe 等，common/specific 除外。
    - 当 common 全局函数使用命名参数时，specific 全局函数对应位置必须使用相同名字的命名参数。
    - 每个 specific 全局函数必须匹配唯一的 common 全局函数，不可以出现多个平台全局函数匹配相同的 common 全局函数。
    - 如果是全局泛型函数，还需满足以下泛型特定限制：
        - common 全局泛型函数和 specific 全局泛型函数必须具有相同个数的类型形参。
        - 当 common 全局泛型函数有泛型约束时，specific 全局泛型函数对应类型形参的泛型约束必须保持一致或者更宽松。
        - common 全局泛型函数和 specific 全局泛型函数类型形参允许重命名，但类型形参结构和泛型约束必须匹配。

示例：

在公共文件中，可以定义一些 common 全局函数。

<!-- compile -->
```cangjie
// common file
package cjmp
​
common func foo1()   // error: 'common' function return type must be specified
common func foo2(): Unit   // ok
common func foo3(a!: Int64): Unit   // ok
common func foo4(a!: Int64 = 1): Unit   // ok
common func foo5(a: Int64): Unit { println("hello world") }   // ok

common func printValue1<T>(value: T): Unit where T <: ToString
common func printValue2<T>(value: T): Unit where T <: ToString
```

在平台文件中，基于 common 全局函数，定义 specific 全局函数。

<!-- compile -->
```cangjie
// platform file
package cjmp
​
specific func foo2(a: Int64): Unit {}   // error: different arguments
specific func foo2(): Int64 {}   // error: different return type
public specific func foo2(): Int64 {}   // error: different modifiers
specific func foo2(): Unit {}   // ok

specific func foo3(a!: Int64): Unit { println("hello world") }   // ok

specific func foo4(a!: Int64 = 1): Unit {}   // error: 'specific' function parameter can not have default value
specific func foo4(a!: Int64): Unit {}   // ok

// common func foo5 有完整实现，无需在 specific 中定义。

specific func printValue1<R>(value: R): Unit  where R <: ToString {
    println(value)
}
specific func printValue2<T>(value: T): Unit {}


```

#### class

仓颉 class 支持跨平台特性，用户可以使用 common 和 specific 修饰 class 及其部分成员。

对于 common class，至多存在一个与之匹配的 specific class。当 common 侧所有成员都有完整实现时，可以省略 specific class。若存在 specific class，则需要满足以下要求：

- common class 和 specific class 可见性必须相同。
- common class 和 specific class 接口实现性必须相同。
- common class 和 specific class 继承性必须相同。
- common open class 匹配 specific open class。
- common abstract class 匹配 specific abstract class。
- common sealed abstract class 匹配 specific sealed abstract class。
- common abstract class 匹配 specific sealed abstract class。
- 如果是 common 修饰的泛型类，还需满足以下泛型特定限制：
    - common 泛型类和 specific 泛型类必须具有相同个数的类型形参。
    - 当 common 泛型类有泛型约束时，specific 泛型类对应类型形参的泛型约束必须保持一致或者更宽松。
    - common 泛型类和 specific 泛型类类型形参允许重命名，但参数结构和约束必须匹配。
- common class 不允许存在隐式的无参构造函数，必须有至少一个显式声明的构造函数。
- common class 不允许在构造函数中对 common let 变量赋值。

普通类示例：

<!-- compile -->
```cangjie
// common file
package cmp

common class A {
    common var a: Int64 = 1
    common init()
    common func foo(): Unit
    common prop p: Int64
}
```

<!-- compile -->
```cangjie
// platform file
package cmp

specific class A {
    specific var a: Int64 = 2
    specific init() {}
    specific func foo(): Unit {}
    specific prop p: Int64 {
        get() { a }
    }
}
```

泛型类示例：

<!-- compile -->
```cangjie
// common file
package cmp

common class Container<T> where T <: Comparable<T> {
    common var value: T
    common init(value: T) { this.value = value }
    common func get(): T
    common func set(newValue: T): Unit
    common func map<R>(convert: (T) -> R): Container<R> where R <: Comparable<R>
}
```

<!-- compile -->
```cangjie
// platform file
package cmp

specific class Container<T> where T <: Comparable<T> {
    specific var value: T
    specific init(value: T) { this.value = value }
    specific func get(): T { value }
    specific func set(newValue: T): Unit { value = newValue }
    specific func map<R>(convert: (T) -> R): Container<R> where R <: Comparable<R> {
        return Container<R>(convert(value))
    }
}
```

##### class 构造函数

构造函数和主构造函数均已支持跨平台特性。使用中需要满足以下要求：

- common init 可以有具体实现，也可以仅保留函数签名，由 specific init 实现。
- 若 common init 有完整实现，则可省略 specific init，否则必须存在一个匹配的 specific init。
- common init 和 specific init 的可见性必须相同。
- specific init 实现会覆盖 common init 实现。
- 主构造函数不可以被 common 或 specific 修饰。
- common/specific class 支持普通构造函数，在 common class 或 specific class 中均可以定义。
- common class 或 specific class 中必须存在至少一个显式定义的构造函数。
- 静态初始化器不支持被 common/specific 修饰。

<!-- compile -->
```cangjie
// common file
package cmp

common class A {
    common A()
    common init(a: String) {}
    init(a: Bool) {}
}
```

<!-- compile -->
```cangjie
// platform file
package cmp

specific class A {
    specific A() {}
    specific init(a: String) {
        println(a)
    }
    init(a: Int64) {}
}
```

##### class 成员变量

common class 和 specific class 的成员变量需要满足如下限制：

- common/specific 成员变量必须定义变量类型，但有初始值时可以省略变量类型声明。
- common 成员变量和 specific 成员变量的类型、可变性和可见性必须相同。
- common 成员变量可以直接赋初值或在构造函数中赋初值，也可以仅保留类型声明，在 specific 侧赋初值。
- common/specific class 支持普通成员变量，且 common class 或 specific class 中均可以定义。
- class 的静态成员变量暂不支持跨平台特性，将会在后续的版本中支持。

<!-- compile -->
```cangjie
// common file
package cmp

common class A {
    common let a: Int64 = 1
    common var b: Int64
    common var c: Int64

    init() {
        b = 1
        c = 1
    }
}
```

<!-- compile -->
```cangjie
// platform file
package cmp

specific class A {
    specific let a: Int64 = 2
    specific let b: Int64 = 2

    init(input: Int64) { c = input }
}
```

##### class 成员函数

common class 和 specific class 的成员函数需要满足如下限制：

- common 成员函数可以有具体实现，也可以仅保留函数签名，由 specific 成员函数实现。
- 若 common 成员函数有完整实现，则可省略 specific 成员函数，否则必须存在一个匹配的 specific 成员函数。
- common 成员函数和 specific 成员函数的参数必须相同，返回值类型可以是相同类型或子类型，修饰符（common/specific 除外）必须相同。
- common/specific class 支持普通成员函数，且 common class 或 specific class 中均可以定义。
- common 泛型成员函数和 specific 泛型成员函数，还需满足泛型特定限制，规则同全局泛型函数。

<!-- compile -->
```cangjie
// common file
package cmp

common class A {
    common func foo1(a: Int64): Unit
    common func foo2(): Unit {}
    common func foo3(): Unit {}
    func foo4() {}
}
```

<!-- compile -->
```cangjie
// platform file
package cmp

specific class A {
    specific func foo1(a: Int64): Unit { println(a) }
    specific func foo3(): Unit { println("platform") }
    func foo5(): Int64 { 1 }

    init() {}
}
```

##### class 属性

common class 和 specific class 的属性需要满足如下限制：

- common 属性可以有具体实现，也可以仅保留属性签名，由 specific 属性实现。
- 若 common 属性有完整实现，则可省略 specific 属性，否则必须存在一个匹配的 specific 属性。
- common 属性和 specific 属性的类型、可见性和可赋值性必须相同。
- common/specific class 支持普通属性，且 common class 或 specific class 中均可以定义。

<!-- compile -->
```cangjie
// common file
package cmp

common class A {
    common prop a: Int64
    common prop b: Int64 {
        get() { 1 }
    }
    common prop c: Int64 {
        get() { 1 }
    }
    prop d: Int64 {
        get() { 1 }
    }
}
```

<!-- compile -->
```cangjie
// platform file
package cmp

specific class A {
    specific prop a: Int64 {
        get() { 1 }
    }
    specific prop c: Int64 {
        get() { 2 }
    }
    prop e: Int64 {
        get() { 1 }
    }

    init() {}
}
```

##### class 的继承

common/specific class 支持继承，其继承关系的处理与 common/specific 的可见性相关，当子类仅在 specific 中时，在 common 部分中，其不可见。

> **注意：**
>
> common sealed class 的所有直接子类型必须定义在同一个 common 包中。

具体示例如下：

<!-- compile -->
```cangjie
// common file
package cmp

public common interface I {
}

public open common class A <: I {
    public init() {}
    public open func foo5(): Unit { println("A::foo5 common") }
}

public common class A2 <: A {
    public init() {}
}

public common class B <: I {
    public init() {}
}

public func runCommonA(a: A) {
    a.foo5()
}

public func runCommonA2(a: A2) {
    a.foo5()
}
```

<!-- compile -->
```cangjie
// platform
package cmp

public specific interface I {
    func foo5(): Unit { println("I::foo5 platform") }
}

public open specific class A <: I {
}

public specific class A2 <: A {
    public func foo5(): Unit { println("A2::foo5 platform") }
}

public specific class B <: I {}

public class C <: I {}

public func runPlatformI(a: I) {
    a.foo5()
}

public func runPlatformA(a: A) {
    a.foo5()
}

public func runPlatformA2(a: A2) {
    a.foo5()
}
```

<!-- run -->
```cangjie
// m_common.cj
import cmp.*

main() {
    runCommonA(A())
    runCommonA(A2())
    runCommonA2(A2())
}
```

输出如下：

```plain
A::foo5 common
A::foo5 common
A::foo5 common
```

<!-- run -->
```cangjie
// m_platform.cj
import cmp.*

main() {
    runCommonA(A())
    runCommonA(A2())
    runCommonA2(A2())
    println("=")
    let i1: I = A()
    let i2: I = A2()
    let i3: I = B()
    let i4: I = C()
    runPlatformI(i1)
    runPlatformI(i2)
    runPlatformI(i3)
    runPlatformI(i4)
    println("=")
    runPlatformI(A())
    runPlatformI(A2())
    runPlatformA(A())
    runPlatformA(A2())
    runPlatformA2(A2())
}
```

输出如下：

```plain
A::foo5 common
A2::foo5 platform
A2::foo5 platform
=
A::foo5 common
A2::foo5 platform
I::foo5 platform
I::foo5 platform
=
A::foo5 common
A2::foo5 platform
A::foo5 common
A2::foo5 platform
A2::foo5 platform
```

当前存在如下限制：

- 在 A 包中 specific 中将子类的成员挪到父类声明中，B 包导入 A 包，common/specific 部分的行为不符合预期。

##### abstract class

当 common/specific 为抽象函数时，新增了如下规则：

- 如果成员函数/属性没有 body 体，则必须有 `abstract` 修饰符。
- 支持 `common abstract`。
- `abstract common` 修饰的成员可以被 `open specific` 修饰的成员替换。

示例如下:

<!-- compile -->

```cangjie
// common part
public common abstract class A {
    init() {}
    public common func a(): Int{1}
    public open func b(): Int{2}
    public common open func c(): Int {3}
    public common func d(): Int
    public common abstract func e(): Int
    public common abstract func f(): Int
    public abstract func g(): Int

    public common open prop prop_a: Int{ get() { 1 } }
    public common prop prop_b: Int { get() { 1 } }
    public abstract prop prop_c: Int
}

public class B <: A {
  public func b(): Int { a() + 10 }
  public func c(): Int { a() + 20 }
  public func e(): Int { a() + 30 }
  public func f(): Int { a() + 40 }
  public func g(): Int { a() + 50 }

  public prop prop_c: Int { get() { 10 } }
}
```

<!-- compile -->

```cangjie
// platform part
public specific abstract class A {
    public specific func a(): Int{4}
    public specific open func c(): Int {5}
    public specific func d(): Int {6}
    public specific open func e(): Int {7}
    public specific abstract func f(): Int

    public specific open prop prop_a: Int { get() { 2 } }
}
```

> **注意：**
>
> 不可以在 abstract specific class 中额外添加抽象成员。

#### struct

仓颉 struct 支持跨平台特性，用户可以使用 common 和 specific 修饰 struct 及其部分成员。

对于 common struct，至多存在一个与之匹配的 specific struct。当 common 侧所有成员都有完整实现时，可以省略 specific struct。若存在 specific struct，则需要满足以下要求：

- common struct 和 specific struct 可见性必须相同。
- common struct 和 specific struct 接口实现性必须相同。
- common struct 和 specific struct 必须同时被 @C 修饰或同时不被修饰。
- 如果是 common 修饰的泛型 struct ，还需满足以下泛型特定限制：
    - common 泛型 struct 和 specific 泛型 struct 必须具有相同个数的类型形参。
    - 当 common 泛型 struct 有泛型约束时，specific 泛型 struct 对应类型形参的泛型约束必须保持一致或者更宽松。
    - common 泛型 struct 和 specific 泛型 struct 类型形参允许重命名，但参数结构和泛型约束必须匹配。
- common struct 不允许存在隐式的无参构造函数，必须有至少一个显式声明的构造函数。
- common struct 不允许在构造函数中对 common let 变量赋值。

普通 struct 示例：

<!-- compile -->
```cangjie
// common file
package cmp

common struct A {
    common var a: Int64 = 1
    common init()
    common func foo(): Unit
    common prop p: Int64
}
```

<!-- compile -->
```cangjie
// platform file
package cmp

specific struct A {
    specific var a: Int64 = 2
    specific init() {}
    specific func foo(): Unit {}
    specific prop p: Int64 {
        get() { a }
    }
}
```

泛型 struct 示例：

<!-- compile -->
```cangjie
// common file
package cmp

interface Add {
    common operator func +(right: Add): Add
}

common struct Point<T> where T <: Add {
    common var x: T
    common var y: T
    common init(x: T, y: T) { this.x = x; this.y = y }
}
```

<!-- compile -->
```cangjie
// platform file
package cmp

specific struct Point<T> where T <: Add {
    specific var x: T
    specific var y: T
    specific init(x: T, y: T) { this.x = x; this.y = y }
}
```

##### struct 构造函数

构造函数已支持跨平台特性。使用中需要满足以下要求：

- common init 可以有具体实现，也可以仅保留函数签名，由 specific init 实现。
- 若 common init 有完整实现，则可省略 specific init，否则必须存在一个匹配的 specific init。
- common init 和 specific init 的可见性必须相同。
- specific init 实现会覆盖 common init 实现。
- 主构造函数不可以被 common 或 specific 修饰。
- common/specific struct 支持普通构造函数，在 common struct 或 specific struct 中均可以定义。
- 静态初始化器不支持被 common/specific 修饰。

<!-- compile -->
```cangjie
// common file
package cmp

common struct A {
    common init(a: String) {}
    init(a: Bool) {}
}
```

<!-- compile -->
```cangjie
// platform file
package cmp

specific struct A {
    specific init(a: String) {
        println(a)
    }
    init(a: Int64) {}
}
```

##### struct 成员变量

common struct 和 specific struct 的成员变量需要满足如下限制：

- common/specific 成员变量必须定义变量类型，但有初始值时可以省略变量类型声明。
- common 成员变量和 specific 成员变量的类型、可变性和可见性必须相同。
- common 成员变量可以直接赋初值或在构造函数中赋初值，也可以仅保留类型声明，在 specific 侧赋初值。
- common/specific struct 支持普通成员变量，且 common struct 或 specific struct 中均可以定义。
- struct 的静态成员变量暂不支持跨平台特性，将会在后续的版本中支持。

<!-- compile -->
```cangjie
// common file
package cmp

common struct A {
    common let a: Int64 = 1
    common var b: Int64
    common var c: Int64

    init() {
        b = 1
        c = 1
    }
}
```

<!-- compile -->
```cangjie
// platform file
package cmp

specific struct A {
    specific let a: Int64 = 2
    specific let b: Int64 = 2

    init(input: Int64) { c = input }
}
```

##### struct 成员函数

common struct 和 specific struct 的成员函数需要满足如下限制：

- common 成员函数可以有具体实现，也可以仅保留函数签名，由 specific 成员函数实现。
- 若 common 成员函数有完整实现，则可省略 specific 成员函数，否则必须存在一个匹配的 specific 成员函数。
- common 成员函数和 specific 成员函数的参数必须相同，返回值类型可以是相同类型或子类型，修饰符（common/specific 除外）必须相同。
- common/specific struct 支持普通成员函数，且 common struct 或 specific struct 中均可以定义。
- common 泛型成员函数和 specific 泛型成员函数，还需满足泛型特定限制，规则同全局泛型函数。

<!-- compile -->
```cangjie
// common file
package cmp

common struct A {
    common func foo1(a: Int64): Unit
    common func foo2(): Unit {}
    common func foo3(): Unit {}
    func foo4() {}
}
```

<!-- compile -->
```cangjie
// platform file
package cmp

specific struct A {
    specific func foo1(a: Int64): Unit { println(a) }
    specific func foo3(): Unit { println("platform") }
    func foo5(): Int64 { 1 }

    init() {}
}
```

##### struct 属性

common struct 和 specific struct 的属性需要满足如下限制：

- common 属性可以有具体实现，也可以仅保留属性签名，由 specific 属性实现。
- 若 common 属性有完整实现，则可省略 specific 属性，否则必须存在一个匹配的 specific 属性。
- common 属性和 specific 属性的类型、可见性和可赋值性必须相同。
- common/specific struct 支持普通属性，且 common struct 或 specific struct 中均可以定义。

<!-- compile -->
```cangjie
// common file
package cmp

common struct A {
    common prop a: Int64
    common prop b: Int64 {
        get() { 1 }
    }
    common prop c: Int64 {
        get() { 1 }
    }
    prop d: Int64 {
        get() { 1 }
    }
}
```

<!-- compile -->
```cangjie
// platform file
package cmp

specific struct A {
    specific prop a: Int64 {
        get() { 1 }
    }
    specific prop c: Int64 {
        get() { 2 }
    }
    prop e: Int64 {
        get() { 1 }
    }

    init() {}
}
```

#### enum

仓颉 enum 支持跨平台特性，用户可以使用 common 和 specific 修饰 enum 及其部分成员。

<!-- compile -->
```cangjie
// common file
package cmp

common enum A {
    | ELEMENT
    common func foo(): Unit
    common prop p: Int64
}
```

<!-- compile -->
```cangjie
// platform file
package cmp

specific enum A {
    | ELEMENT
    specific func foo(): Unit {}
    specific prop p: Int64 {
        get() { 1 }
    }
}
```

对于 common enum，至多存在一个与之匹配的 specific enum。当 common 侧所有成员都有完整实现时，可以省略 specific enum。若存在 specific enum，则需要满足以下要求：

- common enum 和 specific enum 可见性必须相同。
- common enum 和 specific enum 接口实现性必须相同。
- common enum 和 specific enum 中对应的构造器必须是相同类型。
- 如果 common enum 是 exhaustive enum，则 specific enum 必须也是 exhaustive enum；如果 common enum 是 non-exhaustive enum，specific 可以是 exhaustive enum。
    - 对于 exhaustive specific enum 中必须包含 common enum specific enum 中不可以增加新的构造器。
    - 对于 non-exhaustive specific enum 中必须包含 common enum specific enum 中可以增加新的构造器。
- 如果是 common 修饰的泛型 enum，还需满足以下泛型特定限制：
    - common 泛型 enum 和 specific 泛型 enum 必须具有相同个数的类型形参。
    - 当 common 泛型 enum 有泛型约束时，specific 泛型 enum 对应类型形参的泛型约束必须保持一致或者更宽松。
    - common 泛型 enum 和 specific 泛型 enum 类型形参允许重命名，但参数结构和泛型约束必须匹配。

<!-- compile -->
```cangjie
// common file
package cmp

common enum A { ELEMENT1 | ELEMENT2 }
common enum B { ELEMENT1 | ELEMENT2 }
common enum C { ELEMENT1 | ELEMENT2 }
common enum D { ELEMENT1 | ELEMENT2 | ... }
common enum E { ELEMENT1 | ELEMENT2 | ... }

common enum Either<L, R> where L <: Equatable<L>, R <: Equatable<R> {
    | Left(L)
    | Right(R)

    common func goo(x:L): Unit
    common func foo<M>(x:M): Unit
}
```

<!-- compile -->
```cangjie
// platform file
package cmp

specific enum A { ELEMENT1 | ELEMENT2 }                   // ok
specific enum B { ELEMENT1 | ELEMENT2 | ELEMENT3 }        // error: exhaustive enum cannot add new constructor
specific enum C { ELEMENT1 | ELEMENT2 | ... }             // error: exhaustive 'common' enum cannot be matched with non-exhaustive 'platform' enum
specific enum D { ELEMENT1 | ELEMENT2 | ELEMENT3 }        // ok
specific enum E { ELEMENT1 | ELEMENT2 | ELEMENT3 | ... }  // ok

specific enum Either<L, R> where L <: Equatable<L>, R <: Equatable<R> {
    | Left(L)
    | Right(R)

    specific func goo(x:L): Unit {}
    specific func foo<M>(x:M): Unit{}
}
```

##### enum 成员函数

common enum 和 specific enum 的成员函数需要满足如下限制：

- common 成员函数可以有具体实现，也可以仅保留函数签名，由 specific 成员函数实现。
- 若 common 成员函数有完整实现，则可省略 specific 成员函数，否则必须存在一个匹配的 specific 成员函数。
- common 成员函数和 specific 成员函数的参数必须相同，返回值类型可以是相同类型或子类型，修饰符（common/specific 除外）必须相同。
- common/specific enum 支持普通成员函数，且 common enum 或 specific enum 中均可以定义。
- common 泛型成员函数和 specific 泛型成员函数，还需满足泛型特定限制，规则同全局泛型函数。

<!-- compile -->
```cangjie
// common file
package cmp

common enum A {
    | ELEMENT

    common func foo1(a: Int64): Unit
    common func foo2(): Unit {}
    common func foo3(): Unit {}
    func foo4() {}
}
```

<!-- compile -->
```cangjie
// platform file
package cmp

specific enum A {
    | ELEMENT

    specific func foo1(a: Int64): Unit { println(a) }
    specific func foo3(): Unit { println("platform") }
    func foo5(): Int64 { 1 }
}
```

##### enum 属性

common enum 和 specific enum 的属性需要满足如下限制：

- common 属性可以有具体实现，也可以仅保留属性签名，由 specific 属性实现。
- 若 common 属性有完整实现，则可省略 specific 属性，否则必须存在一个匹配的 specific 属性。
- common 属性和 specific 属性的类型、可见性和可赋值性必须相同。
- common/specific enum 支持普通属性，且 common enum 或 specific enum 中均可以定义。

<!-- compile -->
```cangjie
// common file
package cmp

common enum A {
    | ELEMENT

    common prop a: Int64
    common prop b: Int64 {
        get() { 1 }
    }
    common prop c: Int64 {
        get() { 1 }
    }
    prop d: Int64 {
        get() { 1 }
    }
}
```

<!-- compile -->
```cangjie
// platform file
package cmp

specific enum A {
    | ELEMENT

    specific prop a: Int64 {
        get() { 1 }
    }
    specific prop c: Int64 {
        get() { 2 }
    }
    prop e: Int64 {
        get() { 1 }
    }
}
```

#### interface

仓颉 interface 支持跨平台特性，用户可以使用 common 和 specific 修饰 interface 及其部分成员。

<!-- compile -->
```cangjie
// common file
package cmp

common interface A {
    common func foo(): Unit
    common prop p: Int64
}
```

<!-- compile -->
```cangjie
// platform file
package cmp

specific interface A {
    specific func foo(): Unit {}
    specific prop p: Int64 {
        get() { 1 }
    }
}
```

对于 common interface，至多存在一个与之匹配的 specific interface。当 common 侧所有成员都有完整实现时，可以省略 specific interface。若存在 specific interface，则需要满足以下要求：

- common interface 和 specific interface 可见性必须相同。
- common interface 和 specific interface 接口实现性必须相同。
- common sealed interface 匹配 specific sealed interface。
- sealed interface 的直接子类型必须定义在同一个 common 包里。
- 如果是 common 修饰的泛型 interface ，还需满足以下泛型特定限制：
    - common 泛型 interface 和 specific 泛型 interface 必须具有相同个数的类型形参。
    - 当 common 泛型 interface 有泛型约束时，specific 泛型 interface 对应类型形参的泛型约束必须保持一致或者更宽松。
    - common 泛型 interface 和 specific 泛型 interface 类型形参允许重命名，但参数结构和泛型约束必须匹配。

<!-- compile -->
```cangjie
// common file
package cmp

common interface Entity {
    common prop id: String
}

common interface Repository<T> where T <: Entity {
    common func save(entity: T): Unit
}
```

<!-- compile -->
```cangjie
// platform file
package cmp

specific interface Entity {
    specific prop id: String
}

specific interface Repository<T> where T <: Entity {
    specific func save(entity: T): Unit { }
}
```

##### interface 成员函数

common interface 和 specific interface 的成员函数需要满足如下限制：

- 无论 common 成员函数是否有完整实现，specific 成员函数都可以省略。
- common 成员函数和 specific 成员函数的参数必须相同，返回值类型可以是相同类型或子类型，修饰符（common/specific 除外）必须相同。
- common 成员函数如果包含具体实现，则 specific 成员函数必须也包含具体实现。
- common/specific interface 支持普通成员函数，且 common interface 或 specific interface 中均可以定义。
- specific interface 新增的普通函数必须包含完整实现。

<!-- compile -->
```cangjie
// common file
package cmp

common interface A {
    common func foo1(a: Int64): Unit
    common func foo2(): Unit
    common func foo3(): Unit {}
    func foo4(): Int64
}
```

<!-- compile -->
```cangjie
// platform file
package cmp

specific interface A {
    specific func foo1(a: Int64): Unit { println(a) }
    specific func foo3(): Unit { println("platform") }
    func foo5(): Int64 { 1 }
}
```

##### interface 属性

common interface 和 specific interface 的属性需要满足如下限制：

- 无论 common 属性是否有完整实现，specific 属性都可以省略。
- common 属性和 specific 属性的类型、可见性和可赋值性必须相同。
- common 属性如果包含具体实现，则 specific 属性必须也包含具体实现。
- common/specific interface 支持普通属性，且 common interface 或 specific interface 侧均可以存在。
- specific interface 新增的属性必须包含完整实现。

<!-- compile -->
```cangjie
// common file
package cmp

common interface A {
    common prop a: Int64
    common prop b: Int64
    common prop c: Int64 {
        get() { 1 }
    }
    prop d: Int64
}
```

<!-- compile -->
```cangjie
// platform file
package cmp

specific interface A {
    specific prop a: Int64 {
        get() { 1 }
    }
    specific prop c: Int64 {
        get() { 2 }
    }
    prop e: Int64 {
        get() { 1 }
    }
}
```

#### extend

仓颉 extend 支持跨平台特性，用户可以使用 common 和 specific 修饰 extend 及其成员。

> **注意：**
>
> common extend 成员函数或属性不能同时用 common 和 private 来修饰。

<!-- compile -->
```cangjie
// common file
package cmp

class A{}

common extend A {
    common func foo(): Unit
    common prop p: Int64
}
```

<!-- compile -->
```cangjie
// platform file
package cmp

specific extend A {
    specific func foo(): Unit {}
    specific prop p: Int64 {
        get() { 1 }
    }
}
```

对于 common extend，至多存在一个与之匹配的 specific extend。当 common 侧所有成员都有完整实现时，可以省略 specific extend。若存在 specific extend，则需要满足以下要求：

- 当[直接扩展](../extension/direct_extension.md)被 common 修饰时， 必须存在唯一的 specific extend。
- 当[接口扩展](../extension/interface_extension.md)被 common 修饰时， common extend 和 specific extend 必须具有完全相同的接口集合。
- 如果是 common extend 泛型声明，还需满足以下泛型特定限制：
    - common extend 泛型声明和 specific extend 泛型声明必须具有相同个数的类型形参。
    - 当 common extend 泛型声明有泛型约束时，specific extend 泛型声明对应类型形参的泛型约束必须保持一致。
    - common extend 泛型声明和 specific extend 泛型声明形参允许重命名，但参数结构和泛型约束必须匹配。

示例 1：

<!-- compile -->
```cangjie
// common file
package cmp
common extend Int32 {}
common extend Int32 {}
common extend Int64 {}

interface I {}
class A {}
common extend A <: I {}

class C {}
common extend C <: I {}
```

<!-- compile -->
```cangjie
// platform file
platform extend Int32 {} // ok
platform extend Int64 {}
platform extend Int64 {} // error: direct extension of Int64 redefinition

interface B {}
platform extend A <: I {} // ok

platform extend C <: B {} // error: the interfaces of platform extend do not match those on common extend
```

示例 2：

<!-- compile -->
```cangjie
// common file
class Container<T> {
    var item: ?T = Option<T>.None
}

common extend<T> Container<T> {
    common func setItem(newItem: T): Unit
    common func getItem(): ?T
}
```

<!-- compile -->
```cangjie
// platform file
specific extend<T> Container<T> {
    specific func setItem(newItem: T) {
        item = newItem
    }
    specific func getItem(): ?T {
        item
    }
}
```

##### extend 成员函数

common extend 和 specific extend 的成员函数需要满足如下限制：

- common 成员函数可以有具体实现，也可以仅保留函数签名，由 specific 成员函数实现。
- 若 common 成员函数有完整实现，则可省略 specific 成员函数，否则必须存在一个匹配的 specific 成员函数。
- common 成员函数和 specific 成员函数的参数必须相同，返回值类型可以是相同类型或子类型，修饰符（common/specific 除外）必须相同。
- common/specific extend 支持普通成员函数，且 common extend 或 specific extend 中均可以定义。
- common 泛型成员函数和 specific 泛型成员函数，还需满足泛型特定限制，规则同全局泛型函数。

<!-- compile -->
```cangjie
// common file
package cmp

class A {}

common extend A {
    common func foo1(a: Int64): Unit
    common func foo2(): Unit { println("common") }
    func foo3(): Unit {}
}

```

<!-- compile -->
```cangjie
// platform file
package cmp

specific extend A {
    specific func foo1(a: Int64): Unit { println(a) }
    specific func foo2(): Unit { println("platform") }
    func foo4(): Int64 { 1 }
}
```

##### extend 属性

common extend 和 specific extend 的属性需要满足如下限制：

- common 属性可以有具体实现，也可以仅保留属性签名，由 specific 属性实现。
- 若 common 属性有完整实现，则可省略 specific 属性，否则必须存在一个匹配的 specific 属性。
- common 属性和 specific 属性的类型、可见性和可赋值性必须相同。
- common/specific extend 支持普通属性，且 common extend 或 specific extend 中均可以定义。

<!-- compile -->
```cangjie
// common file
package cmp

class A {}

common extend A {
    common prop a: Int64
    common prop b: Int64 {
        get() { 1 }
    }
    prop c: Int64 {
        get() { 1 }
    }
}
```

<!-- compile -->
```cangjie
// platform file
package cmp

specific extend A {
    specific prop a: Int64 {
        get() { 1 }
    }
    specific prop b: Int64 {
        get() { 2 }
    }
    prop d: Int64 {
        get() { 1 }
    }
}
```

#### 导入导出

common 和 specific 的声明支持导入与导出，其规则与其他类型的导入导出规则一致。

具体示例如下：

<!-- compile -->
```cangjie
// common file
package cmp

public common func foo() { println("common func foo") }
```

<!-- compile -->
```cangjie
// platform file
package cmp
public import std.sort.*

public specific func foo() { println("platform func foo") }

public func goo(){
    println("platform func goo")
    sort([1, 4, 3])
}
```

<!-- compile -->
```cangjie
// common file
import cmp.*

main() {
    foo() // 来自 common 的 foo
}
```

<!-- compile -->
```cangjie
// platform file
import cmp.*

main() {
    foo() // 来自 platform 的 foo
    goo() // 仅在 platform 可见
    sort([1,4,3]) // 仅在 platform 可见，由于仅在 platform 做了 public import
}
```

导入导出的可见性与普通声明在 common/specific 的可见性一致，当定义或重导出只在 specific 中时，在 common 部分中，其不可见。

### 跨平台编译

用户可以使用 cjc 进行跨平台包的编译。

> **注意：**
>
> 跨平台包的平台部分代码中的导入语句需要是公共部分代码中导入语句的超集，否则可能会有编译错误。

#### cjc 编译

如下目录组织

```text
cjmp_project(package cjmp)
├── common
│      └── common.cj
├── platform
│      └── platform.cj
└── main.cj
```

1. 首先编译公共部分代码所在的文件。

    ```shell
    cjc --experimental common/common.cj --output-type=chir --output-dir ./common
    ```

2. 其次编译平台部分代码所在的文件。

    ```shell
    cjc --experimental platform/platform.cj common/common.chir --common-part-cjo=./common/cjmp.cjo --output-type=dylib --output-dir ./platform
    ```

3. 当需要调用不同平台的代码时，可以通过指定编译平台文件产生的 .so 文件，指定使用的平台。

    ```shell
    cjc main.cj -o main --import-path=./platform -L./platform -lcjmp
    ```

## 跨平台开发示例

### 使用 Platform() 接口，获取平台名字

公共定义文件。

<!-- compile -->
```cangjie
// common.cj
package example.cmp
// 获取平台信息
public common func Platform(): String
```

Linux 平台文件。

<!-- compile -->
```cangjie
// linux.cj
package example.cmp
public specific func Platform(): String {
    "Linux"
}
```

Windows 平台文件。

<!-- compile -->
```cangjie
// windows.cj
package example.cmp
public specific func Platform(): String {
    "Win64"
}
```

macOS 平台文件。

<!-- compile -->
```cangjie
// macos.cj
package example.cmp
public specific func Platform(): String {
    "Mac"
}
```

应用侧代码。

<!-- compile -->
```cangjie
// app.cj
import example.cmp.Platform
​
main() {
    println("${Platform()}")
}
```

## 约束

当前不支持 @Frozen 修饰 common/specific 声明，行为存在异常。

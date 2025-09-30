# 跨平台

仓颉提供了跨平台开发特性，可以在跨端开发场景中，解决代码复用的问题。用户可以通过区分公共部分代码和平台部分代码，来完成在不同平台共享代码，减少不同平台开发维护相同代码所花费的时间。

> **注意：**
>
> 跨平台开发特性为实验性功能，使用该特性可能有风险。

## 跨平台开发特性介绍

### 公共部分代码和平台部分代码

代码库中与平台无关的部分称作公共部分代码，它包含了可以在所有目标平台上运行的代码，这些代码通常是算法、业务逻辑或其他不依赖于具体平台功能的模块。代码库中与平台相关的部分称作平台部分代码，它包含了只能在特定平台上运行的代码，这些代码通常涉及对操作系统、硬件或其他平台特定功能的调用。公共部分代码与平台部分代码都属于同一个包，平台文件可以依赖公共文件，但是公共文件不可以依赖平台文件。公共部分代码用于在不同平台间共享，其内容可以使用 common 修饰。平台部分代码用于区分不同平台的实现，其内容可以使用 platform 修饰。使用 common/platform 修饰符需要满足如下规则限制：

- common 修饰符只可以出现在公共部分代码中，platform 修饰符只可以出现在平台部分代码中。
- 和 private/const/foreign 修饰符冲突，不可以同时使用。

如下示例，定义了公共部分代码和全局函数 foo。

```cangjie
package cmp
​
public common func foo(): Unit {
    println("I am common")
}
```

如下示例，定义了平台部分代码和全局函数 foo。

```cangjie
package cmp
​
public platform func foo(): Unit {
    println("I am platform")
}
```

更多的细节将在跨平台开发章节详细描述。

### 支持跨平台开发特性的类型

下面对支持跨平台开发特性的类型介绍详细使用规则。

#### 全局函数

全局函数支持跨平台特性，用户可以使用 common 和 platform 修饰全局函数。
common 全局函数，可以包含函数实现，也可以不包含函数实现。

```cangjie
common func foo(): Int64
common func goo(a: Int64): Int64 { 1 }
```

如上示例，定义了两个 common 全局函数，其中函数 foo 不包含函数体，goo 包含函数体，都是合法的 common 全局函数定义。
common/platform 全局函数必须满足如下限制：

- common 全局函数必须定义函数返回值类型。
- 当 common 全局函数有完整实现时，可以不定义 platform 全局函数；当 common 全局函数无完整实现时，必须定义 platform 全局函数。
- platform 全局函数的函数签名必须与同包的 common 全局函数匹配，即参数类型和返回值类型必须一致，并需要同时满足以下规则：
    - common 全局函数与全局平台函数必须使用相同的修饰符，如 public，unsafe 等，common/platform 除外。
    - 当 common 全局函数使用命名参数时，platform 全局函数对应位置必须使用相同名字的命名参数。
    - 当 common 全局函数包含默认值时，platform 全局函数的相应位置必须为同参数名的命名参数，platform 全局函数不支持默认值。
    - 每个 platform 全局函数必须匹配唯一的 common 全局函数，不可以出现多个平台全局函匹配相同的 common 全局函数。

示例：

在公共文件中，可以定义一些 common 全局函数。

```cangjie
// common file
pkg cjmp
​
common func foo1()   // error: 'common' function return type must be specified
common func foo2(): Unit   // ok
common func foo3(a!: Int64): Unit   // ok
common func foo4(a!: Int64 = 1): Unit   // ok
common func foo5(a: Int64): Unit { println("hello word") }   // ok
```

在平台文件中，基于 common 全局函数，定义 platform 全局函数。

```cangjie
// platform file
pkg cjmp
​
platform func foo2(a: Int64): Unit {}   // error: different arguments
platform func foo2(): Int64 {}   // error: different return type
public platform func foo2(): Int64 {}   // error: different modifiers
platform func foo2(): Unit {}   // ok

platform func foo3(a!: Int64): Unit { println("hello word") }   // ok

platform func foo4(a!: Int64 = 1): Unit {}   error: 'platform' function parameter can not have default value
platform func foo4(a!: Int64): Unit {}   // ok

// common func foo5 有完整实现，无需在 platform 中定义。
```

#### class

仓颉 class 支持跨平台特性，用户可以使用 common 和 platform 修饰 class 及其部分成员。

```cangjie
// common file
package cmp

common class A {
    common var a: Int64 = 1
    common init()
    common func foo(): Unit
    common prop p: Int64
}

// platform file
package cmp

platform class A {
    platform var a: Int64 = 2
    platform init() {}
    platform func foo(): Unit {}
    platform prop p: Int64 {
        get() { a }
    }
}
```

若存在一个 common class，则必须存在与之匹配的 platform class，具体要求如下：

- common class 和 platform class 可见性必须相同。
- common class 和 platform class 接口实现性必须相同。
- common class 和 platform class 继承性必须相同。
- common open class 匹配 platform open class。
- common abstract class 匹配 platform abstract class。
- common sealed abstract class 匹配 platform sealed abstract class。

##### class 构造函数

构造函数和主构造函数均已支持跨平台特性。使用中需要满足以下要求：

- common init 可以有具体实现，也可以仅保留函数签名，由 platform init 实现。
- 若 common init 有完整实现，则可省略 platform init，否则必须存在一个匹配的 platform init。
- common init 和 platform init 的可见性必须相同。
- platform init 实现会覆盖 common init 实现。
- 主构造函数规则与构造函数一致。
- common/platform class 支持普通构造函数，在 common class 或 platform class 中均可以定义。
- common class 或 platform class 中必须存在至少一个显示定义的构造函数。
- 静态初始化器不支持被 common/platform 修饰。

```cangjie
// common file
package cmp

common class A {
    common A()
    common init(a: String) {}
    init(a: Bool) {}
}

// platform file
package cmp

platform class A {
    platform A() {}
    platform init(a: String) {
        println(a)
    }
    init(a: Int64) {}
}
```

##### class 成员变量

common class 和 platform class 的成员变量需要满足如下限制：

- common/platform 成员变量必须定义变量类型。
- common 成员变量和 platform 成员变量的类型、可变性和可见性必须相同。
- common 成员变量可以直接赋初值或在构造函数中赋初值，也可以仅保留类型声明，在 platform 侧赋初值。
- common/platform class 支持普通成员变量，且 common class 或 platform class 中均可以定义。
- class 的静态成员变量暂不支持跨平台特性，将会在后续的版本中支持。

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

// platform file
package cmp

platform class A {
    platform let a: Int64 = 2
    platform let b: Int64 = 2

    init(input: Int64) { c = input }
}
```

##### class 成员函数

common class 和 platform class 的成员函数需要满足如下限制：

- common 成员函数可以有具体实现，也可以仅保留函数签名，由 platform 成员函数实现。
- 若 common 成员函数有完整实现，则可省略 platform 成员函数，否则必须存在一个匹配的 platform 成员函数。
- common 成员函数和 platform 成员函数的参数、返回值和修饰符（common/platform 除外）必须相同。
- common/platform class 支持普通成员函数，且 common class 或 platform class 中均可以定义。

```cangjie
// common file
package cmp

common class A {
    common func foo1(a: Int64): Unit
    common func foo2(): Unit {}
    common func foo3(): Unit {}
    func foo4() {}
}

// platform file
package cmp

platform class A {
    platform func foo1(a: Int64): Unit { println(a) }
    platform func foo3(): Unit { println("platform") }
    func foo5(): Int64 { 1 }

    init() {}
}
```

##### class 属性

common class 和 platform class 的属性需要满足如下限制：

- common 属性可以有具体实现，也可以仅保留属性签名，由 platform 属性实现。
- 若 common 属性有完整实现，则可省略 platform 属性，否则必须存在一个匹配的 platform 属性。
- common 属性和 platform 属性的类型、可见性和可赋值性必须相同。
- common/platform class 支持普通属性，且 common class 或 platform class 中均可以定义。

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

// platform file
package cmp

platform class A {
    platform prop a: Int64 {
        get() { 1 }
    }
    platform prop c: Int64 {
        get() { 2 }
    }
    prop e: Int64 {
        get() { 1 }
    }

    init() {}
}
```

##### class 的继承

common/platform class 的继承暂不支持跨平台特性，将会在后续的版本中支持。

#### struct

仓颉 struct 支持跨平台特性，用户可以使用 common 和 platform 修饰 struct 及其部分成员。

```cangjie
// common file
package cmp

common struct A {
    common var a: Int64 = 1
    common init()
    common func foo(): Unit
    common prop p: Int64
}

// platform file
package cmp

platform struct A {
    platform var a: Int64 = 2
    platform init() {}
    platform func foo(): Unit {}
    platform prop p: Int64 {
        get() { a }
    }
}
```

若存在一个 common struct，则必须存在与之匹配的 platform struct，具体要求如下：

- common struct 和 platform struct 可见性必须相同。
- common struct 和 platform struct 接口实现性必须相同。
- common struct 和 platform struct 必须同时被 @C 修饰或同时不被修饰。

##### struct 构造函数

构造函数已支持跨平台特性。使用中需要满足以下要求：

- common init 可以有具体实现，也可以仅保留函数签名，由 platform init 实现。
- 若 common init 有完整实现，则可省略 platform init，否则必须存在一个匹配的 platform init。
- common init 和 platform init 的可见性必须相同。
- platform init 实现会覆盖 common init 实现。
- common/platform struct 支持普通构造函数，在 common struct 或 platform struct 中均可以定义。
- 静态初始化器不支持被 common/platform 修饰。

```cangjie
// common file
package cmp

common struct A {
    common init(a: String) {}
    init(a: Bool) {}
}

// platform file
package cmp

platform struct A {
    platform init(a: String) {
        println(a)
    }
    init(a: Int64) {}
}
```

##### struct 成员变量

common struct 和 platform struct 的成员变量需要满足如下限制：

- common/platform 成员变量必须定义变量类型。
- common 成员变量和 platform 成员变量的类型、可变性和可见性必须相同。
- common 成员变量可以直接赋初值或在构造函数中赋初值，也可以仅保留类型声明，在 platform 侧赋初值。
- common/platform struct 支持普通成员变量，且 common struct 或 platform struct 中均可以定义。
- struct 的静态成员变量暂不支持跨平台特性，将会在后续的版本中支持。

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

// platform file
package cmp

platform struct A {
    platform let a: Int64 = 2
    platform let b: Int64 = 2

    init(input: Int64) { c = input }
}
```

##### struct 成员函数

common struct 和 platform struct 的成员函数需要满足如下限制：

- common 成员函数可以有具体实现，也可以仅保留函数签名，由 platform 成员函数实现。
- 若 common 成员函数有完整实现，则可省略 platform 成员函数，否则必须存在一个匹配的 platform 成员函数。
- common 成员函数和 platform 成员函数的参数、返回值和修饰符（common/platform 除外）必须相同。
- common/platform struct 支持普通成员函数，且 common struct 或 platform struct 中均可以定义。

```cangjie
// common file
package cmp

common struct A {
    common func foo1(a: Int64): Unit
    common func foo2(): Unit {}
    common func foo3(): Unit {}
    func foo4() {}
}

// platform file
package cmp

platform struct A {
    platform func foo1(a: Int64): Unit { println(a) }
    platform func foo3(): Unit { println("platform") }
    func foo5(): Int64 { 1 }

    init() {}
}
```

##### struct 属性

common struct 和 platform struct 的属性需要满足如下限制：

- common 属性可以有具体实现，也可以仅保留属性签名，由 platform 属性实现。
- 若 common 属性有完整实现，则可省略 platform 属性，否则必须存在一个匹配的 platform 属性。
- common 属性和 platform 属性的类型、可见性和可赋值性必须相同。
- common/platform struct 支持普通属性，且 common struct 或 platform struct 中均可以定义。

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

// platform file
package cmp

platform struct A {
    platform prop a: Int64 {
        get() { 1 }
    }
    platform prop c: Int64 {
        get() { 2 }
    }
    prop e: Int64 {
        get() { 1 }
    }

    init() {}
}
```

#### enum

仓颉 enum 支持跨平台特性，用户可以使用 common 和 platform 修饰 enum 及其部分成员。

```cangjie
// common file
package cmp

common enum A {
    | ELEMENT
    common func foo(): Unit
    common prop p: Int64
}

// platform file
package cmp

platform enum A {
    | ELEMENT
    platform func foo(): Unit {}
    platform prop p: Int64 {
        get() { 1 }
    }
}
```

若存在一个 common enum，则必须存在与之匹配的 platform enum，具体要求如下：

- common enum 和 platform enum 可见性必须相同。
- common enum 和 platform enum 接口实现性必须相同。
- common enum 和 platform enum 中对应的构造器必须是相同类型。
- 如果 common enum 是 exhaustive enum，则 platform enum 必须也是 exhaustive enum；如果 common enum 是 non-exhaustive enum，platform 可以是 exhaustive enum。
    - 对于 exhaustive enum，platform enum 中必须包含 common enum 的全部构造器，platform enum 中不可以增加新的构造器。
    - 对于 non-exhaustive enum，platform enum 中必须包含 common enum 的全部构造器，platform enum 中可以增加新的构造器。

```cangjie
// common file
package cmp

common enum A { ELEMENT1 | ELEMENT2 }
common enum B { ELEMENT1 | ELEMENT2 }
common enum C { ELEMENT1 | ELEMENT2 }
common enum D { ELEMENT1 | ELEMENT2 | ... }
common enum E { ELEMENT1 | ELEMENT2 | ... }

// platform file
package cmp

platform enum A { ELEMENT1 | ELEMENT2 }                   // ok
platform enum B { ELEMENT1 | ELEMENT2 | ELEMENT3 }        // error: exhaustive enum cannot add new constructor
platform enum C { ELEMENT1 | ELEMENT2 | ... }             // error: exhaustive 'common' enum cannot be matched with non-exhaustive 'platform' enum
platform enum D { ELEMENT1 | ELEMENT2 | ELEMENT3 }        // ok
platform enum E { ELEMENT1 | ELEMENT2 | ELEMENT3 | ... }  // ok
```

##### enum 成员函数

common enum 和 platform enum 的成员函数需要满足如下限制：

- common 成员函数可以有具体实现，也可以仅保留函数签名，由 platform 成员函数实现。
- 若 common 成员函数有完整实现，则可省略 platform 成员函数，否则必须存在一个匹配的 platform 成员函数。
- common 成员函数和 platform 成员函数的参数、返回值和修饰符（common/platform 除外）必须相同。
- common/platform enum 支持普通成员函数，且 common enum 或 platform enum 中均可以定义。

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

// platform file
package cmp

platform enum A {
    | ELEMENT

    platform func foo1(a: Int64): Unit { println(a) }
    platform func foo3(): Unit { println("platform") }
    func foo5(): Int64 { 1 }
}
```

##### enum 属性

common enum 和 platform enum 的属性需要满足如下限制：

- common 属性可以有具体实现，也可以仅保留属性签名，由 platform 属性实现。
- 若 common 属性有完整实现，则可省略 platform 属性，否则必须存在一个匹配的 platform 属性。
- common 属性和 platform 属性的类型、可见性和可赋值性必须相同。
- common/platform enum 支持普通属性，且 common enum 或 platform enum 中均可以定义。

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

// platform file
package cmp

platform enum A {
    | ELEMENT

    platform prop a: Int64 {
        get() { 1 }
    }
    platform prop c: Int64 {
        get() { 2 }
    }
    prop e: Int64 {
        get() { 1 }
    }
}
```

#### interface

仓颉 interface 支持跨平台特性，用户可以使用 common 和 platform 修饰 interface 及其部分成员。

```cangjie
// common file
package cmp

common interface A {
    common func foo(): Unit
    common prop p: Int64
}

// platform file
package cmp

platform interface A {
    platform func foo(): Unit {}
    platform prop p: Int64 {
        get() { 1 }
    }
}
```

若存在一个 common interface，则必须存在匹配的 platform interface，且需要满足以下要求：

- common interface 和 platform interface 可见性必须相同。
- common interface 和 platform interface 接口实现性必须相同。
- common sealed interface 匹配 platform sealed interface。
- sealed interface 的直接子类型必须定义在在同一个 common 包里。

##### interface 成员函数

common interface 和 platform interface 的成员函数需要满足如下限制：

- 无论 common 成员函数是否有完整实现，platform 成员函数都可以省略。
- common 成员函数和 platform 成员函数的参数、返回值和修饰符（common/platform 除外）必须相同。
- common 成员函数如果包含具体实现，则 platform 成员函数必须也包含具体实现。
- common/platform interface 支持普通成员函数，且 common interface 或 platform interface 中均可以定义。
- platform interface 新增的普通函数必须包含完整实现。

```cangjie
// common file
package cmp

common interface A {
    common func foo1(a: Int64): Unit
    common func foo2(): Unit
    common func foo3(): Unit {}
    func foo4(): Int64
}

// platform file
package cmp

platform interface A {
    platform func foo1(a: Int64): Unit { println(a) }
    platform func foo3(): Unit { println("platform") }
    func foo5(): Int64 { 1 }
}
```

##### interface 属性

common interface 和 platform interface 的属性需要满足如下限制：

- 无论 common 属性是否有完整实现，platform 属性都可以省略。
- common 属性和 platform 属性的类型、可见性和可赋值性必须相同。
- common 属性如果包含具体实现，则 platform 属性必须也包含具体实现。
- common/platform interface 支持普通属性，且 common interface 或 platform interface 侧均可以存在。
- platform interface 新增的属性必须包含完整实现。

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

// platform file
package cmp

platform interface A {
    platform prop a: Int64 {
        get() { 1 }
    }
    platform prop c: Int64 {
        get() { 2 }
    }
    prop e: Int64 {
        get() { 1 }
    }
}
```

### extend

仓颉 extend 支持跨平台特性，用户可以使用 common 和 platform 修饰 extend 及其成员。

> **注意：**
>
> 泛型 extend 暂不支持此特性。

```cangjie
// common file
package cmp

class A{}

common extend A {
    common func foo(): Unit
    common prop p: Int64
}

// platform file
package cmp

platform extend A {
    platform func foo(): Unit {}
    platform prop p: Int64 {
        get() { 1 }
    }
}
```

若存在一个或多个 common extend，则必须存在唯一的 platform extend 与其匹配，且需要满足以下要求：

- 当存在多个未声明接口的 common extend 时， 必须存在唯一的 platform extend，禁止多个 common extend 中声明同名私有函数。
- 当存在声明接口的 common extend 时， common extend 和 platform extend 必须具有完全相同的接口集合。

#### extend 成员函数

common extend 和 platform extend 的成员函数需要满足如下限制：

- common 成员函数可以有具体实现，也可以仅保留函数签名，由 platform 成员函数实现。
- 若 common 成员函数有完整实现，则可省略 platform 成员函数，否则必须存在一个匹配的 platform 成员函数。
- common 成员函数和 platform 成员函数的参数、返回值和修饰符（common/platform 除外）必须相同。
- common/platform extend 支持普通成员函数，且 common extend 或 platform extend 中均可以定义。

```cangjie
// common file
package cmp

class A{}

common extend A {
    common func foo1(a: Int64): Unit
    common func foo2(): Unit { println("common") }
    func foo3(): Unit{}
}

// platform file
package cmp

platform extend A {
    platform func foo1(a: Int64): Unit { println(a) }
    platform func foo2(): Unit { println("platform") }
    func foo4(): Int64 { 1 }
}
```

##### extend 属性

common extend 和 platform extend 的属性需要满足如下限制：

- common 属性可以有具体实现，也可以仅保留属性签名，由 platform 属性实现。
- 若 common 属性有完整实现，则可省略 platform 属性，否则必须存在一个匹配的 platform 属性。
- common 属性和 platform 属性的类型、可见性和可赋值性必须相同。
- common/platform extend 支持普通属性，且 common class 或 platform extend 中均可以定义。

```cangjie
// common file
package cmp

class A{}

common extend A {
    common prop a: Int64
    common prop b: Int64 {
        get() { 1 }
    }
    prop c: Int64{
        get() { 1 }
    }
}

// platform file
package cmp

platform extend A {
    platform prop a: Int64 {
        get() { 1 }
    }
    platform prop b: Int64 {
        get() { 2 }
    }
    prop d: Int64 {
        get() { 1 }
    }
}
```

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

```cangjie
// common.cj
package example.cmp
// 获取平台信息
public common func Platform(): String
```

Linux 平台文件。

```cangjie
// linux.cj
package example.cmp
public platform func Platform(): String {
    "Linux"
}
```

Windows 平台文件。

```cangjie
// windows.cj
package example.cmp
public platform func Platform(): String {
    "Win64"
}
```

macOs 平台文件。

```cangjie
// macos.cj
package example.cmp
public platform func Platform(): String {
    "Mac"
}
```

应用侧代码。

```cangjie
// app.cj
import example.cmp.Platform
​
main() {
    println("${Platform()}")
}
```

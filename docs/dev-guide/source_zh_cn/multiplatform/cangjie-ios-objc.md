# 仓颉-ObjC 互操作

> **注意**：
>
> Objective-C 互操作特性为实验特性，尚在持续完善中。

仓颉跨平台方案支持开发者将仓颉语言接入 iOS 应用开发，无论是项目中尚未实现的新逻辑，还是已有逻辑，都可通过仓颉语言完成开发与适配。

镜像类型是仓颉跨平台实现跨语言、跨运行时互操作的核心机制。它允许一门语言中定义的类型向另一门语言暴露接口，进而实现该类型在不同语言环境中的直接使用。

在仓颉侧，镜像类型让仓颉 `class` 在保持仓颉语法和语义的前提下，能够继承 Objective-C `@interface`、实现 Objective-C `@protocol`。在 Objective-C 侧，镜像类型则可将仓颉类型表示为 Objective-C 类型。仓颉跨平台让仓颉和 Objective-C 在 iOS 应用工程中尽可能无缝衔接，开发者可在仓颉代码中通过跨语言互操作调用 iOS 操作系统的 API。

## 互操作实现思路与底层机制

仓颉和 Objective-C 虽然都是支持继承和多态的面向对象范式语言，但两者在语义、底层对象模型和执行模型等方面存在显著差异，因此，无法在 Objective-C 代码中直接使用仓颉，反之亦然。

两种语言各自拥有独立的托管运行时，自动内存管理、线程模型、异常处理等底层特性各不相同。若让两个复杂编程语言的运行时相互感知来实现互操作，整个应用的复杂度将急剧增加。

因此，仓颉跨平台的互操作实现策略是：仓颉和 Objective-C 各自将对方视作底层语言。具体来说，仓颉与 Objective-C 通过运行时模块 API 实现互通。运行时模块 API 虽然功能强大，但作为底层 API，手写绑定层费时费力，CJMP 提供了相应的工具链，有效降低了使用复杂度。

## 核心概念

### 镜像类型

镜像类型的含义如下：仓颉和 Objective-C 语言之间进行互操作，若一种语言 A 的源码中定义有镜像类型 `T'`，则意味着在另一种语言 B 的源码中实际存在由 B 语言定义的类型 `T`。于是，在语言 A 的源码中就可以通过直接使用镜像类型 `T'` 来实现间接使用类型 `T`，达到在语言 A 中如同直接使用语言 B 的类型般的效果。该操作存在特定限制，将在下文中详细说明。

Objective-C 视角下，Objective-C 的 `int` 类型就是仓颉 `Int32` 类型在 Objective-C 侧的镜像类型；反过来，仓颉视角下，其 `Int32` 类型就是 Objective-C 的 `int` 类型在仓颉侧的镜像类型。不过，对于部分无法建立对应关系的数值类型来说，这个镜像关系就是不存在的了，例如仓颉的 `Float16` 在 Objective-C 侧就没有任何类型能够与之对应，故在 Objective-C 视角下就不存在一种镜像类型来匹配仓颉的 `Float16` 类型，也可以理解为，仓颉的 `Float16` 类型无法被镜像为任何 Objective-C 基本类型。

对于 `class`、`struct`、`interface` 和 `enum` 等用户自定义类型，对于语言 A 中的类型 `T`，其在语言 B 中的镜像类型 `T'` 是语言 B 中与之最接近的等价类型。例如，仓颉的 `struct` 类型在 Objective-C 中所能找到的最佳等价类型是附加了 `objc_subclassing_restricted` 属性的 Objective-C `interface`。

若要在语言 B 中通过镜像类型使用语言 A 定义的类型，该镜像类型仅会暴露语言 A 的源类型中“理论上可被语言 B 访问和调用”的成员与构造函数。例如：若某个仓颉成员函数的返回类型为 `Float16`，由于 `Float16` 无法被镜像为 Objective-C 类型，该仓颉成员函数也无法生成对应的镜像，导致 Objective-C 侧无法通过镜像类型调用此函数，这类场景需根据实际情况采用特定技巧解决。

正常情况下，无论是仓颉类型的镜像类型还是 Objective-C 类型的镜像类型，以及镜像类型本身依赖的其他类型的镜像类型，都能够以某种方式自动生成获得。CJMP 提供了[Objective-C 镜像生成器](#objective-c-镜像生成器参考)，支持 Objective-C 类型自动生成镜像类型。仓颉类型镜像同样可自动生成：配置对应编译选项执行 `cjc` 编译时，会将仓颉类型的镜像类型定义作为副产品生成，后续章节将对完整操作步骤展开详细讲解。

**将 Objective-C 类型镜像为仓颉类型：**

cjc 在编译过程中会将所有仓颉源码中用到的 Objective-C 镜像类型替换为相应的胶水代码，这意味着，真正对编译结果起作用的核心信息只有两点：一是所用 Objective-C 镜像类型的名称，二是该镜像类型中各可用成员的名称及其类型。因此在编写仓颉代码时，Objective-C 镜像类型定义中只需要包含各个可用成员的声明就够了，即 Objective-C 镜像类型中并不需要保留构造函数体、成员函数体和成员属性体，成员变量也不需要初始化器。另一方面，Objective-C 类型中定义的 `@private` 成员对仓颉侧来说不可见，因此这类成员同样不会出现在 Objective-C 镜像类型定义中。

显然，上述 Objective-C 镜像类型定义的写法是不符合仓颉语法/语义规格的，故 Objective-C 镜像类型定义必须带有 `@ObjCMirror` 注解，该注解用于在编译期协助 `cjc` 区分正常的仓颉类型定义与 Objective-C 镜像类型定义，从而对后者进行特殊处理。

示例如下，假设存在如下的 Objective-C `interface`：

```objectivec
@interface Node : NSObject {
}
- (id)initWith:(int)x;
- (int)getX;
@end
```

其对应的 Objective-C 镜像类型定义可能如下：

<!-- compile -->
```cangjie
@ObjCMirror
public open class Node <: NSObject {
    @ForeignName["initWith:"]
    public init(x: Int32)
    public open func getX(): Int32
}
```

### 全局函数镜像

Objective-C 与仓颉均支持全局函数。Objective-C 全局函数以镜像全局函数的形式暴露至仓颉侧，本质上是自动生成的胶水代码，用于在语言间传递控制权和数据。镜像规则详见 [顶层函数](#顶层函数)。

### 互操作类

互操作类本质上是一个仓颉 `class`，其从一到若干个镜像类型派生而来，这种仓颉 `class` 可供 Objective-C 侧使用，这是因为其所有构造函数和非继承而来的 `public` 成员函数，都会通过一个由 cjc 在编译它时自动生成的对应的 Objective-C 包装 `interface`，对 Objective-C 代码暴露。这个 Objective-C 包装 `interface` 本身可能会定义若干辅助方法，但对于 Objective-C 侧代码来说，能调用的方法只有从仓颉侧暴露而来的，以及该 `interface` 继承而来的；仓颉侧代码也是同理。

接下来将举例说明，当使用 cjc 编译以下互操作类时：

<!-- compile -->
```cangjie
@ObjCImpl
public class BooleanNode <: Node {
    private let _flag: Bool
    @ForeignName["initWithX:AndFlag:"]
    public init(x: Int32, flag: Bool) {
        super(x)
        this._flag = flag
    }
    public func isFlagged(): Bool {
        _flag
    }
}
```

cjc 将同时生成一对 Objective-C 源码，其内容类似于以下代码块：

```objectivec
// BooleanNode.h
@interface BooleanNode : Node
/* 胶水层代码 */
- (id)initWithX:(int32_t)x AndFlag:(BOOL)flag;
- (BOOL)isFlagged;
/* 其他胶水层代码 */
@end
```

```objectivec
// BooleanNode.m
@implementation BooleanNode : Node
/* 胶水层代码 */
- (id)initWithX:(int32_t)x AndFlag:(BOOL)flag {
    /* 胶水代码：构造一个仓颉 BooleanNode(x, flag) 实例，
    *  并将其与正在构造的 Objective-C 实例（即 'self'）关联起来。
    */
}
- (BOOL)isFlagged {
    /* 胶水代码：调用与 'self' 关联的仓颉 BooleanNode 实例的 'isFlagged' 成员函数，
    *  并返回其结果。
    */
}
/* 其他胶水层代码 */
@end
```

### 外部类型

镜像类型和互操作类均有别于语言本身的用户自定义类型，为简洁起见，本文档将它们统称为外部类型。

### Objective-C 兼容类型

以下仓颉类型均为 Objective-C 兼容类型：

* 所有拥有等价的 Objective-C 基本类型的仓颉值类型，例如 `Int16` 拥有等价的 Objective-C 基本类型 `int16_t`，故 `Int16` 为 Objective-C 兼容类型；而 `Float16` 无等价的 Objective-C 基本类型，故 `Float16` 不是 Objective-C 兼容类型
* 所有外部类型
* `Option<T>` 类型，且其中类型变元 `T` 为[外部类型](#外部类型)，（详情请参考[空引用值处理](#objective-c-侧-nil-值处理)）
* `ObjCPointer<T>`、`ObjCFunc<F>`、`ObjCBlock<F>` 等内置类型
* `@C struct` 结构体
* `CFunc<F>` C 函数类型
* `CString`
* 作为函数返回值类型的 `Unit`

> **注意**：
>
> 当前版本中，`CPointer<T>` 与 `VArray<T, $N>` **本身**不是 Objective-C 兼容类型，但这并不限制它们在 `@C struct` 或 `CFunc<F>` 函数类型中的使用。

## 在仓颉侧使用 Objective-C

通过以下步骤来实现 iOS 应用中 Objective-C 与仓颉的互操作：

1. 基于 Objective-C 类型和函数，设计互操作胶水层 API，由开发者完成互操作胶水层的设计（以 Objective-C 伪代码形式呈现）。

2. 根据上一步设计的胶水层，借助 Objective-C 镜像生成器，为所有现存相关的 Objective-C 类和协议生成仓颉侧可用的 `@ObjCMirror` 类型定义，即将 .h 头文件转换为 .cj 镜像类型定义文件。

3. 使用仓颉语言编写实现互操作层，在仓颉代码中按需使用 `@ObjCMirror` 镜像类型，例如创建镜像类型的实例、调用其成员函数等。开发者依据互操作胶水层设计和 .cj 镜像类型定义，完成 .cj 胶水层实现代码的编写。

4. 将 `@ObjCMirror` 镜像类型定义和第 3 步中仓颉实现的互操作层代码一并使用 cjc 编译器进行编译，编译产物包括：

    * 包含互操作层逻辑的动态库（.dylib 文件）。
    * 若干 Objective-C 侧可用的镜像类型定义源文件（.h / .m 文件）。

    即：.cj 源文件（镜像类型定义 + 胶水层实现）经 cjc 编译后，生成 .dylib 动态库和 .h / .m 胶水层镜像类型定义。

5. 将以下中间产物添加进 XCode 工程：

    * 第 4 步中由 cjc 编译产生的若干 .h / .m 源文件，其中包含后续 Objective-C 侧可能用到的互操作胶水层代码。
    * 第 4 步中由 cjc 编译得到的 .dylib 动态库文件，其中包含了由仓颉实现的胶水层逻辑。
    * 仓颉 SDK 中所有必要的运行时库，包括 .dylib 等。

    接着，在 Objective-C 侧编写必要的代码，对胶水层中提供的镜像类型进行实例化和方法调用，完成后使用 iOS 工具链重构建工程，即可生成最终的 iOS 应用。

### 从零实现互操作层

#### 步骤一：设计互操作层

在这一步，开发者需要 从 Objective-C 源码的视角，来设计一到若干个互操作类。互操作类由仓颉编写实现，但最终会由 cjc 编译生成镜像类以便 Objective-C 侧使用，因此从 Objective-C 侧的角度，开发者并不需要关心互操作类的具体实现，只需要关心 Objective-C 侧需要哪些功能。因此，对每个互操作类，开发者只需要考虑以下要点：

* 互操作类是继承 `NSObject`，还是需要继承其他 Objective-C 类？
* 互操作类是否需要实现任何 Objective-C 协议？
* 互操作类中需要拥有哪些 `public`/`protected` 构造方法/成员方法？开发者目前只需要知道它们的功能以确定其函数签名，真正的实现则是在后续步骤中通过仓颉编写。

另请参见[互操作类的特性与限制](#互操作类的特性与限制)。

在 Objective-C 源码的视角下，互操作层所提供的 Objective-C 类与普通的 Objective-C 类在外观和使用上不存在任何区别，唯一区别在于后者的 `@implementation` 是用户自己手写的，而前者的 `@implementation` 则是用户手写仓颉 `ObjCImpl class` 后，由 cjc 编译之自动生成的。因此，开发者要做的是用伪代码来描述 Objective-C 侧的 `@interface`，然后使用仓颉来照着这个 `@interface` 来依次实现各个方法，详情请参考[步骤三](#步骤三实现互操作类)。

**支持的形参类型：** 任何被镜像的 Objective-C 类型。

**支持的返回类型：** 任何被镜像的 Objective-C 类型或 `void` 类型。

**支持的继承与实现关系：** 互操作类只能继承 Objective-C 类的镜像类 `@ObjCMirror class`，不能继承其他互操作类，且只能实现 Objective-C 协议的镜像接口 `@ObjCMirror interface`。

**当前存在的使用限制：**

* 不支持拥有变长形参列表（varargs）的方法。

* 互操作类禁止拥有类型形参，互操作类的非 `private` 成员函数禁止拥有类型形参。

* 泛型 Objective-C 类型在镜像生成前，其泛型会被擦除，各个类型变元会被替换为各个类型上界。

**端到端示例：**

假设开发者的 iOS 应用源码中存在一个类 `M`，类中定义有一个无参且返回类型为 `void` 的实例方法 `foo`：

```objectivec
// M.h
#import <Foundation/Foundation.h>

@interface M : NSObject
- (instancetype)init;
- (void)foo;
@end
```

```objectivec
// M.m
#import "M.h"

@implementation M
- (instancetype)init {
    if (self = [super init]) {
        // 一些初始化工作
    }
    return self;
}
- (void) foo {
    printf("Hello from Objective-C M.foo()\n");
}
@end
```

开发者希望在仓颉侧定义一个继承类 `M` 的类 `A`，其中对方法 `foo` 进行重写。

那么，开发者的互操作类的设计应该类似如下：

```objectivec
#import "M.h"

@interface A : M
- (instancetype)init;
- (void)foo;
@end
```

#### 步骤二：生成镜像类型

开发者需要为上一步中设计的互操作类所依赖的所有 Objective-C 类型生成其仓颉侧可用的镜像类型，这包括互操作类的父类型、成员属性类型、形参和返回类型等，如果涉及数组类型，则还包括其元素类型。

不过在此之前，请正常构建 iOS 应用项目，无需做任何修改，确保能够编译构建成功，这样能保证接下来镜像生成器所接收的 Objective-C 头文件是完整且连贯的。

编写[镜像生成器配置文件](#objective-c-镜像生成器配置文件语法)，并运行镜像生成器：

```bash
ObjCInteropGen <config-file>
```

其中 `<config-file>` 是配置文件的路径。

**端到端示例（续）：**

类 `A` 唯一依赖的 Objective-C 类型是其父类 `M`，于是配置文件应如下：

```toml
# A.toml
# 将 M 的镜像及其可能依赖的任何镜像放置在 'objcworld' 包中：
[[packages]]
filters = { include = ["M", "NS.+"] }
package-name = "objcworld"

# 将包含镜像类型定义的输出文件写入当前目录：
[output-roots.default]
path = "."

# 指定输入头文件的路径：
[sources.all]
paths = ["M.h"]

[sources-mixins.default]
sources = [".*"]
arguments-append = [
    # 如遇到 "unknown type name" 错误，请取消下面这行的注释
    # "-DTARGET_OS_IPHONE=1",

    # 请根据系统上 Objective-C 头文件的实际位置修改以下路径：
    "-F", "/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/System/Library/Frameworks",
    "-isystem", "/Library/Developer/CommandLineTools/usr/lib/clang/17/include",
    "-isystem", "/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/include"
]
```

镜像生成器命令行如下：

```bash
ObjCInteropGen A.toml
```

以上命令将在当前目录生成 `objcworld/M.cj`，文件中保存有类 `M` 的镜像类型定义；以及若干 `objcworld/NS*.cj` 文件，分别保存有类 `M` 所依赖的 Foundation 框架中的类和协议的镜像类型定义。

**常见错误配置问题：**

* **镜像生成器无法找到某些标准库的头文件，例如 `stdarg.h` 或 `stdbool.h` 等。**

    典型报错信息：

    ```text
    .../CoreFoundation.h:19:10: error: 'stdarg.h' file not found
    ```

    一般是由于 `[sources-mixins]` 表中 `arguments-append` 数组中的路径设置错误导致的，请仔细检查这些头文件搜索路径下确实存在相应头文件。

* **镜像生成器产生“Unknown type name 'NSUInteger'”或类似报错。**

    典型报错信息：

    ```text
    .../NSObjCRuntime.h:626:74: error: unknown type name 'NSUInteger'
    ```

    在某些系统上，开发者需要给 clang 传入额外的参数 "`-DTARGET_OS_IPHONE=1`"。

    将该额外的参数加入 `[sources-mixins]` 表中的 `arguments-append` 数组，在上述示例中，该配置被注释了：

    ```toml
    # ...
    arguments-append = [
        # 如遇到 "unknown type name" 编译报错，请解注释以下这行：
        # "-DTARGET_OS_IPHONE=1",
        # ...
    ]
    ```

#### 步骤三：实现互操作类

对于开发者在[步骤一](#步骤一设计互操作层) 中设计的互操作层中的各个 Objective-C 类框架，现在需要分别为之编写一个仓颉类（即互操作类）：

* 创建源文件，选择一个合适的包名。
* 导入包 `objc.lang.*`。
* 导入开发者在[步骤二](#步骤二生成镜像类型) 中生成的镜像类型，但请不要将所有生成的依赖镜像类型全部导入，只需要导入实际将用到的那些镜像类型。
* 开始编写互操作类，为互操作类添加 `@ObjCImpl` 注解。
* 根据开发者的设计，让互操作类继承某 `@ObjCMirror open class`；如果并不需要特别继承某父类，则让其继承 `NSObject`。
* 实现互操作类中的构造函数及其他成员，为其中 `public` 的构造函数和成员函数添加 `@ForeignName["foreign-name"]` 注解，其中 `foreign-name` 是开发者希望的 Objective-C 方法名，并遵守以下规定：

    * 重写父类方法的成员函数：请勿添加 `@ForeignName` 注解，否则将导致编译报错。被重写的方法名已由镜像生成器从 Objective-C 侧自动传播至仓颉侧，保存在镜像父类对应成员函数的 `@ForeignName` 注解中（详见[Objective-C 类和协议](#objective-c-类和协议)），cjc 会从中获取原方法名，无需重复指定。

    * 拥有两个及以上形参的构造函数或非重写成员函数：请务必添加 `@ForeignName` 注解，否则将导致编译错误。对于此类函数，cjc 无法自动推导出有效的 Objective-C 方法名，因此必须手动指定。

    * 构成重载的构造函数或成员函数：请务必通过 `@ForeignName` 注解为每个重载赋予不同的 Objective-C 方法名，因为 Objective-C 不支持方法重载。

    * 其余构造函数和成员函数：可选择性添加 `@ForeignName` 注解。若未添加，cjc 会按以下规则自动推导 Objective-C 方法名：无参函数的方法名与原函数名相同（构造函数则为 `init`）；仅有一个形参的函数，方法名为原函数名加上 `:` 后缀（构造函数则为 `init:`）。

> **注意**：
>
> 当前版本的 cjc 并不会全面校验 _`foreign-name`_ 的合法性。特别地，cjc 并不会校验 _`foreign-name`_ 中冒号 `:` 的数量是否与构造函数/成员函数的形参个数一致。

* 请参考以下 Objective-C 类型到仓颉类型的映射关系表（`T'` 是对应的值类型或镜像类型）：

    Objective-C            | 仓颉          | 备注
    ---------------------- | ------------  | ------
    `void`                 | `Unit`        | -
    `BOOL`                 | `Bool`        | -
    `signed char`          | `Int8`        | -
    `short`                | `Int16`       | -
    `int`                  | `Int32`       | -
    `long`                 | `Int64`       | -
    `long long`            | `Int64`       | -
    `unsigned char`        | `UInt8`       | -
    `unsigned short`       | `UInt16`      | -
    `unsigned int`         | `UInt32`      | -
    `unsigned long`        | `UInt64`      | -
    `unsigned long long`   | `UInt64`      | -
    `float`                | `Float32`     | -
    `double`               | `Float64`     | -
    `struct`               | `@C struct'`  | (\*)
    `enum` 且底层类型为 `T` | `T'`          | -
    `id`                   | `ObjCId`      | (†)

    Objective-C  | 仓颉           | `T` 是...                        | 备注
    ------------ | ------------   | ---------------------------------| ------
    `T*`         | `CPointer<T'>` | ... 基本数据类型或结构体类型       | -
    `T*`         | `CPointer<U'>` | ... 枚举类型且 `U` 是其底层类型    | -
    `T*`         | `CFunc<T'>`    | ... 纯 C 函数类型                 | -
    `T*`         | `T'`           | ... Objective-C 类               | (†)

    (\*) Objective-C 结构体禁止包含任何非 `CType` 兼容类型的字段。详情请参见 [Objective-C 结构体](#objective-c-结构体类型)。

    (†) 对于可能持有或接受 `nil` 值的镜像类型或互操作类的形参类型、返回类型及局部变量等，请使用 `Option<T'>`。详情请参见[由 Objective-C 到仓颉的映射关系](#由-objective-c-到仓颉的映射关系) 章节。

**支持的特性：**

互操作类中的成员函数：

* 可以重写其父类中的方法。
* 可以实现其实现的接口中的方法。
* 可以新定义互操作类中独有的成员函数。

互操作类的构造函数和成员函数：

* 在函数体中，如果纯粹是在操作普通仓颉类型的值，那么实际上可以正常使用所有仓颉语言特性。
* 可以使用普通仓颉语法来：
    * 实例化互操作类和 `@ObjCMirror` 类。
    * 调用互操作类和 `@ObjCMirror` 类型的构造函数、实例/静态成员函数，包括通过 `super` 调用其父类的构造函数和成员函数。
    * 访问自己的成员属性，和访问其他互操作类和 `@ObjCMirror` 类型的非 `private` 成员属性。

**使用限制：**

* 互操作类可以实现 `@ObjCMirror` 接口，但禁止实现普通仓颉接口。

* 互操作类禁止声明为 `open` 或 `abstract`，且禁止 `extend`。

* 互操作类中的非 `private` 构造函数和成员函数：

    * 形参类型和返回类型只能为上述表中列举的 Objective-C 类型（由于变长参数要求使用仓颉 `Array<T>`，而该类型并没有 Objective-C 映射类型，故变长参数并不支持）。
    * 禁止拥有命名形参。
    * 禁止拥有类型形参。

* 非 `private` 成员属性的类型只能为前文表中列举的 Objective-C 映射类型。

* 泛型 Objective-C 类型将被镜像为非泛型仓颉类型，详情请参见 [Objective-C 泛型](#objective-c-泛型)。

* 镜像类型和互操作类的实例，即 Objective-C 引用类型的值，禁止逃逸至仓颉全局变量、静态变量，以及任何能够在每次调用之间持久化的数据结构中。

**端到端示例（续）：**

继续上述的例子，开发者可能会如下实现互操作类 `A`：

<!-- compile -->
```cangjie
package objcworld         // 为简洁起见，使用相同的包名

import objc.lang.*  // 始终需要导入

@ObjCImpl
public class A <: M {
    override public open func foo(): Unit {
        println("Hello from overridden A.foo()")
    }
}
```

> **注意**：
>
> `A` 的默认构造函数会调用 `super()`，这在 Objective-C 语义上等价于调用 `[super init]`，尽管假设了它会返回一个合适类的实例。

#### 步骤四：编译互操作类

执行以下命令行以编译互操作类：

```bash
cjc --target=arm64-apple-ios-simulator \
    --sysroot=$(xcrun --show-sdk-path --sdk iphonesimulator) \
    --output-type=dylib \
    --int-overflow=wrapping \
    <source-files> \
    -o <target-file> \
    --link-options "-undefined dynamic_lookup"
```

其中：

`<source-files>` 是互操作类的源文件，以及各镜像类型定义的源文件。

`<target-file>` 是得到的包含互操作类逻辑的动态库的文件名，例如 `libobjcworld.dylib`。

cjc 会同时自动生成 Objective-C 源文件（`.h` 和 `.m` 文件），这些 Objective-C 源文件中包含有 Objective-C 包装类（对应互操作类）。这些源文件默认生成在 `./objc-gen` 子目录中。

需要为生成的动态库 `.dylib` 文件签名：

```bash
xcrun codesign --sign - <dylib-file>
```

**端到端示例（续）：**

首先编译互操作类源文件：

```bash
cd objcworld

cjc --target=arm64-apple-ios-simulator \
    --sysroot=$(xcrun --show-sdk-path --sdk iphonesimulator) \
    --output-type=dylib \
    --int-overflow=wrapping \
    *.cj \
    -o libobjcworld.dylib \
    --link-options "-undefined dynamic_lookup"
```

cjc 将生成三个文件：`./libobjcworld.dylib`、`./objc-gen/A.h` 和 `./objc-gen/A.m`。

然后为动态库签名：

```bash
xcrun codesign --sign - libobjcworld.dylib
```

#### 步骤五：整合所有产物

当前版本的镜像生成器**不会**将原始头文件名传播到镜像类型声明中。因此，上一步生成的包装类可能 `#import` 不存在的头文件而无法编译。临时解决办法如下。

例如，若互操作类的成员函数形参或返回类型为 UIKit 的 `UIDevice`，生成的包装类源文件可能包含：

```objectivec
#import "UIDevice.h"
```

而非：

```objectivec
#import <UIKit/UIKit.h>
```

或

```objectivec
#import <UIKit/UIDevice.h>
```

Foundation 框架是例外——`cjc` 生成的所有 Objective-C 文件顶部均包含：

```objectivec
#import <Foundation/Foundation.h>
#import <stddef.h>
```

临时解决办法：手动创建此类镜像类型对应的 `.h` 文件，写入正确的 `#import` 声明，并将其加入 XCode 工程。此不便将在未来版本中消除。

* 在 XCode 项目中创建一个子目录，将 `$CANGJIE_HOME/runtime/lib/ios_simulator_aarch64_cjnative/` 目录下的所有动态库文件都拷贝一份到该目录下。

* 将这些动态库，以及步骤四中得到的动态库添加进 XCode 项目依赖（“BuildPhases”标签页，添加进“Copy Files”和“Link Binary With Libraries”）。

* 将步骤四中生成的 `.h` 和 `.m` 文件拷贝到项目根目录以参与编译构建。

接着重新构建项目即可。

**端到端示例（续）：**

在 XCode 工程的根目录创建一个子目录，将仓颉 SDK 中用于 iOS 的所有动态库文件全部拷贝到该目录下：

```bash
cd ..
mkdir -p CJRuntimeDylibs
cp $CANGJIE_HOME/runtime/lib/ios_simulator_aarch64_cjnative/*.dylib CJRuntimeDylibs/
```

将这些动态库以及 `./objcworld/libobjcworld.dylib` 作为依赖添加进 XCode 工程，具体操作是，在“BuildPhases”中的“Copy Files”和“Link Binary With Libraries”列表中将它们添加进去。

将所有 cjc 生成的 `.h` 和 `.m` 文件放置到 XCode 工程根目录：

```bash
mv objcworld/objc-gen/*.h ./
mv objcworld/objc-gen/*.m ./
```

然后重新构建 XCode 工程。

### 在仓颉侧调用 Objective-C

按照[上一节](#从零实现互操作层)设计、构建并集成互操作层后，可在互操作类的成员函数中添加使用 Objective-C 类型的代码。类型映射关系与[步骤三](#步骤三实现互操作类)中的表格相同。

### 在 Objective-C 侧调用仓颉

在[上一节](#从零实现互操作层) 中开发者设计、实现并编译了互操作层，最后将互操作层集成进 XCode 工程。现在，开发者就可以在 Objective-C 侧实现对仓颉侧实现的逻辑的调用了。互操作类经 cjc 编译自动得到的 Objective-C 包装类可以直接由 Objective-C 代码调用使用，从而间接调用对应的互操作类的逻辑。

**端到端示例（续）：**

在 Objective-C 侧代码中，实例化类 `A`，然后调用其 `foo` 实例方法，如下：

```objectivec
// ...
#import "M.h"
#import "A.h"
// ...
    M* a = [[A alloc] init];
    [a foo];
// ...
```

重新构建 XCode 工程，然后直接在 iOS 模拟器上运行应用查看效果。

### 后续操作步骤

经过[上述操作流程](#从零实现互操作层) 的讲解，现在应该已经明白了如何往互操作层中新增更多的互操作类，以及如何在互操作类中使用到更多的 Objective-C 类型，以下是对后续操作步骤的总结，开发者会发现其实依然对应着上述的五个操作步骤：

**[步骤一](#步骤一设计互操作层)：** 现在开发者可以根据设计，实现更多的互操作类，或者对现存的互操作类进行增强实现，比如实现新成员函数、成员属性等。

**[步骤二](#步骤二生成镜像类型)：** 如果开发者在修改互操作层的过程中识别到需要用到某些尚未被镜像的 Objective-C 类型，应按以下方式操作：首先重新构建当前的 XCode 工程看看是否能够编译成功，确保代码的连贯性无问题；然后根据需要编辑镜像生成器配置文件，配置好后调用镜像生成器全量重新生成所有镜像。注意，推荐全量重新生成，以确保镜像代码的连贯性。

**[步骤三](#步骤三实现互操作类)：** 获得需要的全部 Objective-C 类型的镜像后，开发者就可以继续根据设计实现新的互操作类，或对现存互操作类进行翻修。

**[步骤四](#步骤四编译互操作类)：** 编译所有互操作类及其他相关源文件。

**[步骤五](#步骤五整合所有产物)：** 将上一步中 cjc 生成的 `.h`、`.m` 和 `.dylib` 文件拷贝到相应的位置，然后重新构建 XCode 工程。

接着，开发者就可以[在 Objective-C 侧使用](#在-objective-c-侧调用仓颉) 刚刚新增的互操作类了。

### 互操作类的特性与限制

* 互操作类必须是一个镜像类的直接子类。

* 互操作类可以实现一到若干个镜像接口，但禁止实现任何普通仓颉接口。反之，普通仓颉类禁止实现镜像接口，普通仓颉接口也禁止继承镜像接口。

* 互操作类禁止被声明为 `open` 或 `abstract`，禁止被 `extend`，且禁止为泛型。

* 互操作类中可以定义新的实例成员变量，且变量类型可以是任何仓颉类型（因为这些成员变量不会被暴露至 Objective-C 侧）。互操作类中的成员函数可以对其父类中的成员方法进行重写。

* 互操作类中可以定义构造函数，构造函数中可以通过 `super()` 调用父类的构造方法，且遵循普通仓颉构造函数中 `super` 和实例成员函数调用的顺序的相关规定，以及同样要求在构造函数中需要对所有无初始化器的实例成员变量初始化。

* 互操作类中的实例成员函数体中，可以调用父类的实例方法，如果父类的实例方法在互操作类中被重写，也同样支持通过 `super.` 来调用。

* 镜像类型和互操作类中定义的构造函数和成员函数的函数签名中所允许使用的类型必须是 (a) 镜像类型或互操作类 (b) 100%与 Objective-C 对应的基本数据类型。详情请参见上述的[步骤三](#步骤三实现互操作类) 和[由 Objective-C 到仓颉的映射关系](#由-objective-c-到仓颉的映射关系) 章节。

## 由 Objective-C 到仓颉的映射关系

当前版本的 Objective-C 镜像生成器遵循以下所描述的 Objective-C 到仓颉的类型映射规格。

### 注意事项

Objective-C 镜像生成器依赖 Clang 解析 Objective-C 源码，调用时带有 `-fobjc-arc` 编译选项。

标记为 `unavailable` 的声明将被忽略。

全局函数、文件作用域函数及变量声明亦将被忽略。

输出仓颉源文件中声明的顺序与输入 Objective-C 源文件中的定义顺序一致，嵌套类型定义除外。

### Objective-C 名称

原 Objective-C 标识符一般情况下都会被原样保留，除了以下这些情况：

* Objective-C 的标识符与仓颉关键字存在冲突，例如 `catch`、`false`、`UInt32` 等。冲突的标识符会在仓颉侧使用反引号 ` `` ` 包裹。

* Objective-C 允许定义一对同名的类和协议，而仓颉禁止同包中存在一对同名的类和接口。当 Objective-C 侧一对同名的类和协议同时被镜像时，由协议镜像得到的仓颉接口的名字，会在原协议的名称的基础上，加上 `Protocol` 后缀。

* Objective-C 允许定义一对同名的实例方法和静态方法，而仓颉禁止定义同名的实例成员函数和静态成员函数。当 Objective-C 侧存在同名的实例方法和静态方法，且它们将被镜像，那么最靠近冲突源的那个方法将会被重命名。如果导致冲突的实例方法和静态方法定义源于同一个类或协议中，静态方法将被重命名。重命名的方法是，对于实例方法，方法名加上 `Instance` 后缀，对于静态方法，方法名加上 `Static` 后缀。例如，在以下例子中：

    ```objectivec
    @interface A
    +(void)foo;
    @end
    
    @interface B : A
    -(void)foo;
    +(void)bar;
    -(void)bar;
    @end
    ```

    上述 Objective-C 类型将被镜像为：

    <!-- compile -->
    ```cangjie
    @ObjCMirror
    public open class A <: ObjCId {
        public static func foo()
    }
    
    @ObjCMirror
    public open class B <: A {
        @ForeignName["foo"]
        public open func fooInstance()

        @ForeignName["bar"]
        public static func barStatic()

        public open func bar()
    }
    ```

* 若多个 `init` 方法的形参个数与类型相同、仅选择器名称不同，则存在冲突，处理方式见 [Objective-C 类](#objective-c-类)。

### Objective-C 类型别名

`typedef` 声明将被映射为仓颉可见性 `public` 的类型别名。​

### Objective-C 基本数据类型

Objective-C 基本数据类型将被映射为对应的仓颉基本数据类型。对于各平台特定大小的 C 类型，将根据​host 平台（即运行镜像生成器本身的平台）上其长度来决定映射到哪个仓颉类型。例如，在 MacOS 上，该映射关系可能为：

|   Objective-C        | 仓颉      |
| -------------------- | --------- |
| `void`               | `Unit`    |
| `BOOL`               | `Bool`    |
| `signed char`        | `Int8`    |
| `short`              | `Int16`   |
| `int`                | `Int32`   |
| `long`               | `Int64`   |
| `long long`          | `Int64`   |
| `unsigned char`      | `UInt8`   |
| `unsigned short`     | `UInt16`  |
| `unsigned int`       | `UInt32`  |
| `unsigned long`      | `UInt64`  |
| `unsigned long long` | `UInt64`  |
| `float`              | `Float32` |
| `double`             | `Float64` |

### 字符串

仓颉 `String` 与 Objective-C `NSString` 在二进制层面不兼容，字符编码亦不同（分别为 UTF-8 与 UTF-16）。

为便于转换，cjc 对 Foundation 框架中 `NSObject` 与 `NSString` 的镜像隐式扩展如下：

* `NSObject` 镜像类隐式定义实例成员函数 `toString()`：

  ```cangjie
  public open func toString(): String
  ```

  该函数调用接收者的 `description` 方法，将结果转换为仓颉 `String` 并返回。

* `NSString` 镜像类隐式定义接受仓颉 `String` 的构造函数：

  ```cangjie
  public init(s: String)
  ```

  它以实参的转码字符数据初始化正在构造的 `NSString` 实例。

  镜像类的仓颉名称无关紧要；`@ObjCMirror` 注解的值必须分别为 `"NSObject"` 与 `"NSString"`，编译器才会插入上述隐式声明：

  ```cangjie
  @ObjCMirror["NSObject"]
  public class ObjC_Object {    // 隐式添加 toString()
      // ...
  }
  ```

  ```cangjie
  @ObjCMirror["NSObjectWrapper"]
  public class NSObject {       // 不会隐式添加 toString()
      // ...
  }
  ```

### Objective-C 结构体类型

如果一个 Objective-C `struct` 中仅包含对于仓颉而言 `CType` 兼容的类型的字段，那么该 Objective-C `struct` 将被映射为仓颉可见性 `public` 的 `@C struct` 类型。如果 Objective-C `struct` 包含非 `CType` 兼容的类型的字段，例如 Objective-C 对象指针，当前不支持镜像。

嵌套的 `struct` 将被镜像为顶级仓颉 `@C struct`，因为仓颉仅支持顶级类型定义。

不完全结构声明（用于前向声明，例如 `struct MyRecord;`）将被镜像为空 `struct`（`struct` 类型定义中无任何成员）。

Objective-C `struct` 的字段将被镜像为可见性 `public` 的成员变量。

仓颉不支持位域，如果原 `struct` 的字段带有位域，这些宽度信息将被忽略，且镜像生成器将告警。

与 Objective-C 不同，仓颉要求 `struct` 中所有成员变量都需要在构造时被初始化。因此，镜像得到的仓颉 `struct` 的所有成员变量都会带有零初始化器。例如：

```objectivec
struct A {
    int x;
    double y;
    BOOL z;
    struct A *w;
};
```

上述类型将被镜像为：

<!-- compile -->
```cangjie
@C
public struct A {
    public var x: Int32 = 0
    public var y: Float64 = 0.0
    public var z: Bool = false
    public var w: CPointer<A> = CPointer<A>()
}
```

### Objective-C 联合体类型

仓颉不支持 C 的 `union` 类型，故其将镜像为仓颉 `struct`，各原联合体中的成员依次被镜像为成员变量，这明显是不符合原联合体的语义的，故对此将输出告警信息。

### Objective-C 枚举类型

C 枚举声明镜像为一系列顶层 `public const` 变量声明：各常量名与枚举常量一致，初始化器为对应值。此外声明一个类型别名，其名为原枚举名（匿名枚举则合成唯一名称），等于镜像底层 C 类型的仓颉值类型别名；枚举的使用处均镜像为该类型别名。

> **注意**：
>
> 枚举名成为整型仓颉类型的别名后，其值集并不限于关联的 `const` 变量集合，原声明的类型安全在镜像过程中丢失。

```objectivec
enum E : char { NONE, ONE, TWO, FIVE = ONE + TWO + TWO };
```

<!-- compile -->
```cangjie
public type M = Int8

public const NONE: M = 0
public const ONE: M = 1
public const TWO: M = 2
public const FIVE: M = 5
```

### `id` 类型

`id` 类型将镜像为内置 `@ObjCMirror interface ObjCId`。

### Objective-C 类和协议

Objective-C 类与协议分别镜像为仓颉类与接口。所有此类镜像类与接口均隐式实现/继承内置 `ObjCId` 镜像接口（见[内置类型](#objective-c-内置类型)）。

**方法**（`init` 方法除外，见下文）镜像为 `public open` 成员函数，形参与返回类型替换为相应镜像类型；返回 `void` 的方法返回类型为 `Unit`；`instancetype` 镜像为当前声明的名称；类方法（`+` 前缀）加 `static` 修饰。

`init` 方法为特殊情况，镜像规则见 [Objective-C 类](#objective-c-类)。

所有方法的镜像均省略函数体，外观类似仓颉抽象成员函数。

声明了可变参数的方法，参数列表中的 `...` 部分被忽略。

完整 Objective-C 方法名（选择器）可含 `:`，不是合法仓颉标识符，按以下规则变换为仓颉函数名：

* 紧跟在 `:` 之后的字母（若有）改为大写；
* 删除所有 `:`。

原选择器保留在 `@ForeignName` 注解中。

**示例：**

```c
@interface A
- (void)foo;
- (void)foo:(int)i;
- (void)foo:(int)i bar:(int) j;
- (void)foo:(int)i bar:(int) j baz:(int) k;
@end
```

镜像结果为：

<!-- compile -->
```cangjie
@ObjCMirror
public open class A {
    public open func foo(): Unit

    @ForeignName["foo:"]
    public open func foo(i: Int32): Unit

    @ForeignName["foo:bar:"]
    public open func fooBar(i: Int32, j: Int32): Unit

    @ForeignName["foo:bar:baz:"]
    public open func fooBarBaz(i: Int32, j: Int32, k: Int32): Unit
}
```

**属性** 通常镜像为相应类型的 `public` 仓颉成员属性，但存在例外与细节：

Objective-C `@property` 本质上是访问方法与后备变量声明的语法糖。类中或父类中可能已声明签名匹配的方法，编译器自动将其与属性关联；指令也可显式指定非标准名称的 getter/setter。这种灵活性意味着即使属性本身未被重写，getter/setter 仍可能被重写。Objective-C 中还常见子类以 `readwrite` 属性覆盖父类 `readonly` 属性的模式。而仓颉要求重写属性与被重写属性可变性一致，且成员属性不得与成员函数同名。上述差异导致以下限制：

* 若属性重写父类属性，则**不**镜像该重写属性（子类实例上仍会经动态派发调用其 getter/setter）。

* 若属性的 getter 方法为标准名称**且**重写了父类方法，则**不**镜像该属性。

* 反之，重写继承属性 getter 的方法亦**不**镜像。

无论名称如何、是否编译期自动生成、属性是否被镜像，getter/setter 方法本身**从不**镜像。属性镜像中省略 getter/setter 的 `{ }` 体。

仓颉属性的 getter/setter 函数名不可任意指定。`@ForeignGetterName` 与 `@ForeignSetterName` 注解保留原 getter/setter 的自定义名称：

```objectivec
@interface FormElement : UIComponent
- (void)setEditable:(BOOL)flag;
- (BOOL)isEditable;
@property(getter=isEditable, setter=setEditable:) BOOL editable;
@end
```

<!-- compile -->
```cangjie
public interface FormElement <: UIComponent {
    @ForeignGetterName["isEditable"]
    public mut prop editable: Bool
}
```

> **注意**：
>
> 无需指定 `@ForeignSetterName["setEditable:"]`，该名称已是该 setter 的标准名称。

#### Objective-C 类

Objective-C `@interface` 类声明镜像为带 `@ObjCMirror` 注解的 `public open` 仓颉类。注解值为字符串，在镜像类名与原 Objective-C 类名不同时保留原类名。

Objective-C 的 `@interface` 的分类（category）和扩展（extension）声明的镜像均将直接融合进相应的仓颉类定义中。

Objective-C 的 `@implementation` 声明将被忽略。

类的前向声明（`@class` 标记）被镜像为空的仓颉类定义（即类中无任何成员）。

依据 Clang Objective-C ARC 文档中的方法族（Method families）被识别为 `init` 方法的方法，镜像为仓颉构造函数，存在以下限制：

* 若某类的两个及以上 `init` 方法仅名称不同、形参个数与类型相同，则无法镜像为重载构造函数，而镜像为带 `@ObjCInit` 注解的 `static` 成员函数（对象工厂），返回被镜像类的实例。工厂函数名由原始 `init` 方法名推导，原名称保留在 `@ForeignName` 中，规则与其他镜像方法相同。

  ```objectivec
  @interface Point2D : NSObject {
  @private
      double _x;
      double _y;
  }
  - (id)init;
  - (id)initWithX:(double)x;
  - (id)initWithY:(double)y;
  - (id)initWithX:(double)x andY:(double)y;
  @end
  ```

  <!-- compile -->
  ```cangjie
  @ObjCMirror
  public open class Point2D {
      public init()

      @ObjCInit
      @ForeignName["initWithX:"]
      public static func initWithX(x: Float64): Point2D

      @ObjCInit
      @ForeignName["initWithY:"]
      public static func initWithY(y: Float64): Point2D

      @ForeignName["initWithX:andY:"]
      public init(x: Float64, y: Float64)
  }
  ```

* 在 Objective-C 中 `init` 方法与其他方法一样可继承，而仓颉构造函数不继承。因此镜像类或互操作类实例化时，无法调用父类 `init` 所镜像的构造函数。上一项中的工厂函数**可**继承，但不得用作 `super` 构造器，因其返回的是已完全初始化的超类实例。

* 与仓颉等语言中的构造函数不同，Objective-C `init` 方法可返回替代对象且有显式返回类型。现代 Objective-C 中返回类型通常为 `instancetype`，表示指向接收者类（或其子类）实例的指针，但仍可能返回 `nil` 以表示初始化失败（且不必抛异常）。当前实现期望 `init` 方法返回**合适类**的实例，不校验返回值。

> **注意**：
>
> 若从仓颉调用的 `init` 返回 `nil` 或指向非接收者类（及其子类）实例的指针，行为未定义。

**实例变量** 镜像为相应镜像类型的实例成员变量。Objective-C 中不存在类变量。

#### Objective-C 协议

Objective-C `@protocol` 镜像为 `@ObjCMirror public` 仓颉接口。

协议的前向声明被镜像为空的仓颉接口定义（即接口中无任何成员）。

Objective-C 协议的方法镜像为相应仓颉接口的 `public open` 成员函数。实例方法为实例成员函数，类方法加 `static`。

Objective-C 协议可包含可选实例方法，仓颉无直接对应。此类方法的镜像成员函数带 `@ObjCOptional` 注解。跨语言调用时，桥接代码先检查接收者是否实现该方法，未实现则抛出 `NotImplementedException`。

```objectivec
@protocol MyDelegate <NSObject>

@required
- (void)requiredMethod;

@optional
- (void)optionalMethod;
@end
```

<!-- compile -->
```cangjie
@ObjCMirror
public interface MyDelegate {
    func requiredMethod(): Unit

    @ObjCOptional
    func optionalMethod(): Unit
}
// ...
    try {
        delegate.optionalMethod()
    } catch (nie: NotImplementedException) { } // 未实现亦无妨
```

#### 镜像类型的继承层次结构

镜像类与镜像接口各自构成独立的子类型层次：

* 镜像类可继承其他镜像类，反映 Objective-C 类继承关系；不得继承普通仓颉类（`std.core.Object` 除外），反之亦然。

* 镜像接口可继承其他镜像接口，反映协议继承关系；不得继承普通仓颉接口，反之亦然。

* 镜像类可实现镜像接口，但不得实现普通仓颉接口（`Any` 为例外，但[支持尚有限制](#尚未实现的特性)）。普通仓颉类不得实现镜像接口。

* Objective-C `id` 的镜像是内置接口 `ObjCId`；所有镜像接口隐式继承 `ObjCId`，所有镜像类隐式实现 `ObjCId`。

* 镜像类不得使用 `extend` 扩展，任何类型亦不得以镜像接口进行接口扩展。

### Objective-C 指针类型

> **注意**：
>
> 当前版本在同时使用 Objective-C 特有类型的上下文中（如 Objective-C 方法或全局函数的形参/返回类型，且该函数还接受/返回指向 Objective-C 类实例或 `id` 的指针），尚不支持 `CPointer<T>`。临时解决办法：内置类型 [`ObjCPointer<T>`](#objective-c-内置类型) 额外支持满足 `CType` 约束的类型变元。

Objective-C 指针类型的 `const`、`volatile` 和 `restrict` 修饰符均将被忽略。

C 基本数据类型 `T` 的指针及其类型别名镜像为 `CPointer<T'>` 或 `ObjCPointer<T'>`（见上文说明），其中 `T'` 为 `T` 的镜像类型。

指向基本类型的 `typedef` 指针别名始终镜像为 `CPointer<T'>` 的类型别名；若在 Objective-C 上下文中使用（见上文说明），该处镜像为 `ObjCPointer<T'>`，原别名名保留在注释中。

底层类型为 `T` 的 C 枚举类型的指针镜像为 `CPointer<T'>` 或 `ObjCPointer<T'>`（见上文说明），其中 `T'` 为 `T` 的镜像类型，原枚举名体现在注释中。

#### Objective-C 结构体指针类型

指向 C 结构体 `T` 的指针镜像为 `CPointer<T'>` 或 `ObjCPointer<T'>`（见 [Objective-C 指针类型](#objective-c-指针类型) 中的说明），其中 `T'` 为 `T` 的镜像类型。

#### Objective-C 函数指针类型

指向 C 函数的指针镜像为内置类型 `ObjCFunc<F>`，其中 `F` 为相应仓颉函数类型，即使形参与返回类型均为 `CType` 兼容类型亦然。`ObjCFunc<F>` 为内置结构体类型，其公开接口仅含单一属性：

<!-- compile -->
```cangjie
public struct ObjCFunc<F> {
    public prop call: F
}
```

编译器对该类型施加以下限制：

* 用作 `ObjCFunc<F>` 类型变元 `F` 的函数类型，其形参与返回类型必须是 [Objective-C 兼容类型](#objective-c-兼容类型)。注意，类型 `F` 本身并非 Objective-C 兼容类型。

* 属性 `call` 仅可在函数调用表达式中使用。

* 无法在仓颉代码中创建 `ObjCFunc<F>` 实例，所有此类实例均来自 Objective-C 代码。

* 当前尚无简便方法检查从 Objective-C 传入的某个 `ObjCFunc<F>` 值是否为 `null`。

#### Objective-C 块指针类型

指向 Objective-C 块的指针镜像为内置结构体类型 `ObjCBlock<F>`，其中 `F` 为相应仓颉函数类型。`ObjCBlock` 类型定义在互操作库中。其公开接口由一个构造函数和一个属性组成：

<!-- compile -->
```cangjie
public struct ObjCBlock<F> {
    public init(f: F)
    public prop call: F
}
```

编译器对该类型施加以下限制：

* 用作 `ObjCBlock<F>` 类型变元 `F` 的函数类型，其形参与返回类型必须是 [Objective-C 兼容类型](#objective-c-兼容类型)。注意，这并不意味着类型 `F` 本身是 Objective-C 兼容类型。

* 属性 `call` 仅可在函数调用表达式中使用。

* 当前尚无简便方法检查从 Objective-C 传入的某个 `ObjCBlock<F>` 值是否为 `null`。

与 `ObjCFunc<F>` 不同，可在仓颉代码中通过 lambda 表达式创建 `ObjCBlock<F>` 实例：

<!-- compile -->
```cangjie
let halve: ObjCBlock<(Double) -> Double> =
    ObjCBlock { it => it / 2.0 }
```

块可在仓颉中调用：

<!-- compile -->
```cangjie
let x = halve.call(2.0)    // x == 1.0d
```

并可作为实参传递给镜像的 Objective-C 方法与函数。

#### Objective-C 对象指针类型

指向 Objective-C 类实例的指针类型将按照以下规则进行镜像：

* Objective-C 类实例的指针（形如 `SomeClass*`）类型将被镜像为该类的镜像类型 `@ObjCMirror class`。

* Objective-C 带有有且只有一个协议约束的 `id` 类型（例如 `id<NSCopying>`）将被镜像为该协议的镜像类型 `@ObjCMirror interface`。

* Objective-C 带有多于一个协议约束的 `id` 类型（例如 `id<NSCopying, NSSecureCoding>`）将被镜像为纯粹的 `ObjCId` 类型，而各协议则将被列举在生成的注释中。`ObjCId` 接口类型定义于互操作库，所有 `@ObjCMirror` 类和接口均实现或继承该接口。

* 如果在泛型模板中使用的一个泛型类型形参指定有单个约束协议，该类型形参在使用处将被替换为协议类型的引用类型，原类型形参名将被留存在注释中。

#### 指向指针的指针

类型 `**T` 的镜像规则如下：

* 若在给定上下文中 `*T` 会镜像为 `CPointer<T'>`，则 `**T` 镜像为 `CPointer<CPointer<T'>>`，其中 `T'` 为 `T` 的镜像类型。

* 否则（`*T` 会镜像为 `@ObjCMirror` 类型或[内置](#objective-c-内置类型) Objective-C 类型镜像），`**T` 镜像为 `ObjCPointer<U'>`，其中 `U'` 为 `*T` 的镜像类型。

### Objective-C 泛型

泛型 Objective-C 类将被镜像为非泛型仓颉类。原 Objective-C 的轻量级泛型的信息，例如“`<T>`”、“`<Foo>`”将被保存在生成的仓颉类旁边的注释中。类定义中的所有泛型使用均将被替换为 `ObjCId`。

**示例：**

```objectivec
@interface G<T> : NSObject
- (void)f:(T)t;
@end
```

以上类型将被镜像为：

<!-- compile -->
```cangjie
@ObjCMirror
public open class G/*<T>*/ <: NSObject {
    @ForeignName["f:"] public open func f(t: ?ObjCId /*T*/): Unit
}
```

注意，在当前版本，泛型约束将被忽略，相关示例：

```objectivec
@interface G<T: SomeType*> : NSObject
- (void)f:(T)t;
@end
```

以上类型的镜像结果将与上一个的镜像结果完全一致。

### 顶层函数

顶层 Objective-C 函数镜像为带 `@ObjCMirror` 注解的 `public` 全局函数声明，形参与返回类型替换为相应镜像类型；返回 `void` 的函数返回类型为 `Unit`。省略函数体。

> **注意**：
>
> 若函数所有形参类型与返回类型均满足 `CType` 约束，生成器产出常规 `foreign func` 声明。

声明了可变参数的函数，参数列表中的 `...` 部分被忽略。

### Objective-C 内置类型

镜像生成器在生成镜像时会假设以下仓颉类型已被定义在互操作库中，生成的镜像中将用到这些类型，用户亦可使用这些类型。当前版本中，部分类型尚未实现完全，且未来版本中这些类型的名称可能会改变。

| Objective-C         | 仓颉 (\*)           | 备注                                                                              |
| ------------------- | ------------------- | -----------------------------------------------------                            |
| `id`                | `ObjCId`            | 所有 `@ObjCMirror` 类和接口均实现该接口，对应 Objective-C 的 `id` |
| `SEL`               | `SEL`?              | 对应 Objective-C 的 `SEL` 的类（尚未完全实现）                                   |
| `Class`             | `ObjCClass`?        | 对应 Objective-C 的 `Class` 的类（尚未完全实现）                                 |
| [指针类型](#objective-c-指针类型) | `ObjCPointer<T>`    | 指向非 `CType` 兼容结构体，或 arity 大于 1 的指针                                 |
| [非 C 函数类型](#objective-c-函数指针类型) | `ObjCFunc<F>`       | 形参/返回类型不全是 `CType` 兼容的 C 函数；`F` 为仓颉函数类型                    |
| [块类型](#objective-c-块指针类型) | `ObjCBlock<F>`      | 实现 Objective-C 块的结构体；`F` 为仓颉函数类型                                  |
|                     | `__builtin_va_list` | `CPointer<Unit>` 的辅助类型别名，未来版本将移除                                  |

(\*) 这些仓颉类型名称均为暂定的，未来版本中可能改变。

## 尚未实现的特性

仓颉 SDK 对 Objective-C 互操作的支持尚在开发中：部分特性尚未实现，部分可能在正式版前变更，部分因两种语言的根本差异而无法（完全）实现。

* 改变数据默认大小、打包、填充和/或对齐的 C 语言特性与 ABI 密切相关。仓颉不支持此类底层特性（尤其是位域），镜像生成器忽略位域宽度说明并发出告警。

* 仓颉不支持 C `union`，故镜像为 `struct` 并告警。

* 含非 `CType` 兼容字段的 `struct` 不支持镜像。

* C 枚举镜像为 `const` 变量序列与类型别名，类型安全在转换中丢失。详见 [Objective-C 枚举类型](#objective-c-枚举类型)。

* 可变参数方法中 `...` 被忽略。

* 与内存管理相关的注解（如 `NS_RETURNS_RETAINED`）被忽略。

* 属性在可能时镜像为属性，否则其 getter/setter 按普通实例方法镜像。详见 [Objective-C 类和协议](#objective-c-类和协议)。

* 重写父类属性的属性不被镜像。

* `const`、`volatile` 修饰符被忽略（`restrict` 在指针小节中说明）。

* Objective-C 类型 `SEL` 与 `Class` 尚不支持。

* 镜像 Objective-C 类实例的构造存在若干细节：通常将 `init` 镜像为构造函数，但若多个 `init` 形参个数与类型完全相同，则改为带 `@ObjCInit` 的静态工厂函数（不可作 `super` 构造器）；仓颉构造函数不继承而 `init` 可继承（工厂函数可继承）；`init` 可能返回 `nil`，当前版本可能导致异常终止，未来版本将抛出 `NoneValueException`。详见 [Objective-C 类](#objective-c-类)。

* 泛型 Objective-C 类镜像为非泛型仓颉类。详见 [Objective-C 泛型](#objective-c-泛型)。

* Objective-C 错误处理仅可通过 `ObjCPointer<Option<NSError'>>` 类型进行，其中 `NSError'` 为 `NSError` 类的镜像名。

## Objective-C 侧 nil 值处理

仓颉无 `nil` 引用的概念，因此对于 Objective-C 的 `nil` 值不存在等价物。对于 Objective-C 侧的类和协议类型镜像为仓颉类和接口类型的实例的指针，从 Objective-C 侧传入到仓颉侧后，如果为 `nil`，则会导致仓颉侧的段错误。反之，仓颉侧也不存在直接往 Objective-C 侧返回 `nil` 值的途径。

因此决定将仓颉 `Option<T>` 枚举用于表示这类 Objective-C 类型，其中 `None` 表示 `nil` 值，而 `Some(r)` 表示一个非空引用值 `r`。假设仓颉类型 `T` 是 `@ObjCMirror` 镜像类型或 `@ObjCImpl` 互操作类，cjc 将把 `Option<T>` 判定为 Objective-C 兼容类型，并对该类型的值进行装包/拆包。

示例如下，以下 `@interface`：

```objectivec
@interface MyContainer: NSObject
// ...
- (void)addItem:(MyItem *)item withUuid:(NSString *)uuid;
- (MyItem *)itemWithUuid:(NSString *)uuid;
- (NSString *)uuidForItem:(MyItem *)item;
@property (copy) NSArray<MyItem *> *allItems;
@end
```

> 说明：下文代码为精简展示，省略了 `@ForeignName` 注解。

镜像生成结果如下：

<!-- compile -->
```cangjie
@ObjCMirror
public open class MyContainer <: NSObject {
// ...
    public open func addItemWithUuid(item: ?MyItem, uuid: ?NSString): Unit
    public open func itemWithUuid(uuid: ?NSString): ?MyItem
    public open func uuidForItem(item: ?MyItem): ?NSString
    public open mut prop allItems: ?NSArray/*<MyItem>*/
}
```

上述 `Option<T>` 装包确保了即便 Objective-C 侧往仓颉侧传入 `nil` 值，仓颉侧不会因此崩溃，但这个解决方法不可避免地带来了部分性能和内存足迹的劣化。解决方法引入的另一个缺点是[型变的丢失](#型变丢失)。不过，[Objective-C 可空性注解](#objective-c-可空性注解) 显著消减了上述由于引用封装所带来的影响。类型测试方面的注意事项见 [外部类型的转换与类型测试](#外部类型的转换与类型测试)。

> **注意**：
>
> 上述问题对于能够被映射为仓颉 `CPointer<T>` 类型的 C 类型并不构成麻烦，因为 `CPointer<T>` 类型实现内部提供有相关的空指针检查功能。

### 型变丢失

为 Objective-C 镜像类型和互操作类进行 [`Option<T>`装包](#objective-c-侧-nil-值处理) 带来了一个显著的限制：像这样装包的类型在所有其他方面均完全遵循仓颉语义规则。具体而言，根据仓颉语义规则，`Option<T>` 对其类型变元 `T` 是不变的，即，对于两个类型 `U` 和 `T`，除非 `U` 和 `T` 是相同的类型，否则即便 `U` 是 `T` 的子类型，`Option<U>` 也与 `Option<T>` 不存在任何子类型关系。这意味着，对于镜像类型中存在重写关系的方法，如果这两个方法的返回类型存在协变的关系，这个协变的关系无法在仓颉侧保留下来，子类中的重写方法的返回类型的镜像必须改为父类中方法的返回类型的镜像。

示例如下，在以下代码片段中，Objective-C 类 `Foo` 是类 `Bar` 的直接父类：

```objectivec
@interface Foo : NSObject
@end

@interface Bar : Foo
@end
```

Objective-C 类 `C` 中声明有方法 `get`，其返回类型为 `Foo`：

```objectivec
@interface C : NSObject
- (Foo*) get;
@end
```

Objective-C 类 `D` 继承 `C`，其中重写了方法 `get`，返回类型换为了更加精确的类型 `Bar`：

```objectivec
@interface D : C
- (Bar*) get;
@end
```

假设不存在 `Option<T>` 装包，上述所有类型将被镜像为：

<!-- compile -->
```cangjie
@ObjCMirror
public open class Foo <: NSObject {}

@ObjCMirror
public open class Bar <: Foo {}

@ObjCMirror
public open class C <: NSObject {
    open func get(): Foo
}

@ObjCMirror
public open class D <: C {
    open func get(): Bar       // 此处存在函数返回类型协变
}
```

但正如前文所说，如果 `get` 方法存在返回 `nil` 的可能性，那么仓颉侧将不可避免地崩溃。

如果进行 `Option<T>` 装包，就可以解决 `nil` 的问题，不过所有重写的成员函数的返回类型就不得不降级为原始的（定义在父类型中的）成员函数的返回类型：

<!-- compile -->
```cangjie
@ObjCMirror
public open class Foo <: NSObject {}

@ObjCMirror
public open class Bar <: Foo {}

@ObjCMirror
public open class C <: NSObject {
    open func get(): Option<Foo>
}

@ObjCMirror
public open class D <: C {
    // open func get(): Option<Bar>  // 将导致编译报错，因为 `Option<T>` 在 `T` 上不具备协变性
    open func get(): Option<Foo>     // 编译通过，但返回类型被降级了
}
```

[Objective-C 可空性注解](#objective-c-可空性注解)一定程度上消减了该问题。

### Objective-C 可空性注解

XCode6.3 开始支持 Objective-C 的可空性注解，其目的是更好地与新 iOS/OSX 开发语言 Swift 集成配合，Swift 本身将调用 Objective-C 所提供的 API。

**Objective-C 可空性标注：**

关键字 `nullable`、`nonnull` 可用于修饰 Objective-C 属性、方法形参类型和返回类型。它们的含义分别是指定的实体可能/不可能持有或接受 `nil` 值。除此之外还有关键字 `null_unspecified`，意思是并不确定指定的实体到底是可能还是不可能持有或接受 `nil` 值，不过该关键字极少被使用。另外，指针类型也可以被 `_Nullable`、`_Nonnull` 注解，与上述各关键字的语义相同。Objective-C 属性也可以被指定为 `null_resettable`，语义是该属性的 getter 不可能返回 `nil` 值，而如果调用 setter 时传入 `nil` 值，该属性将被重置为某默认值。

因此，如果某处对 Objective-C 引用类型的使用被标记为不可为空（例如被 `nonnull` 标记），则该使用处将被免去 `Option<T>` 装包。即镜像生成器仅对未被 `nonnull` 或 `_Nonnull` 注解的成员属性类型、成员函数形参类型及返回类型进行 `Option<T>` 装包。

现在，请重新考虑[上一节中](#objective-c-侧-nil-值处理) 的例子，这次我们对其添加了可空性注解，如下：

```objectivec
@interface MyContainer: NSObject
// ...
- (void)addItem:(nonnull MyItem *)item withUuid:(nonnull NSString *)uuid;
- (nullable MyItem *)itemWithUuid:(nonnull NSString *)uuid;
- (nullable NSString *)uuidForItem:(nonnull MyItem *)item;
@property (copy, nonnull) NSArray<MyItem *> *allItems;
@end
```

镜像生成器将不对 `nonnull` 实体进行 `Option<T>` 装包：（注意，以下代码出于简洁考虑省略了 `@ForeignName` 注解）

<!-- compile -->
```cangjie
@ObjCMirror
public open class MyContainer <: NSObject {
    // ...
    public open func addItemWithUuid(item: MyItem, uuid: NSString): Unit
    public open func itemWithUuid(uuid: NSString): ?MyItem
    public open func uuidForItem(item: MyItem): ?NSString
    public open mut prop allItems: NSArray/*<MyItem*>*/
}
```

> **注意**：
>
> 当前尚不支持正确地将 Objective-C 属性的 `null_resettable` 的语义传播至仓颉成员属性，故该注解将被视作 `nullable` 处理。

如果开发者的 Objective-C 代码尚未采用上述的可空性注解，推荐开发者在开始进行互操作层设计与实现前，事先为互操作层相关的 Objective-C 代码合理添加 `nonnull` 注解。这样做可显著减少镜像类型中的 `Option<T>` 装包，使互操作层更加清晰易读。

### 外部类型的转换与类型测试

仓颉运算符 `is` 和 `as`，以及 `match`、`if-let`、`while-let` 中的类型模式 `v: T`，均支持所有[外部类型](#外部类型)（`if-let` 与 `while-let` 的支持尚有限制，见下段）。

类型测试与转换的语义与 Objective-C 一致，一般借助 Objective-C Runtime 函数完成。

当前版本中，`if-let` 与 `while-let` 的支持有限：`let` 表达式必须构成整个条件表达式，不得与 `&&` 或 `||` 组合。

> **注意**：
>
> 如 [Objective-C 侧 nil 值处理](#objective-c-侧-nil-值处理) 所述，经 `Option<T>` 装包的镜像类型与互操作类值，类型测试前须先做空值检测并拆包。原因是仓颉泛型对类型变元不变：`e is ?T` 仅当 `e` 的类型恰好为 `Option<T>` 时为 `true`，而非 `Option<U>`（`U <: T`）。此外，无论 `e` 为 `Some(v)` 还是 `None`，均不对 `v` 进行类型测试。

## Objective-C 镜像生成器参考

### 准备工作

在使用镜像生成器前请确保已执行仓颉 SDK 中的 `envsetup.sh` 脚本。

开发者需要知道将要为之生成镜像的所有类型所依赖的类型所在的头文件的本地路径。这包括 iOS 标准库头文件，以及所有 XCode 在构建项目时 Objective-C 编译器的所有头文件搜索路径。

### 命令行使用方法

```text
ObjCInteropGen [-v] <config-file>
```

`-v`：输出详细日志。

`<config-file>`：配置文件的路径。

### Objective-C 镜像生成器配置文件语法

对于配置文件中将被视作正则表达式的字符串，必须遵循 ECMAScript 正则表达式语法。

配置文件采用 TOML 语法，指定：

* 输出目录
* 源 Objective-C 头文件（`.h`）路径
* 输出仓颉包名及镜像类型在各包中的分布
* 需特殊处理的类型映射

#### 输出根目录

`[output-roots]`表的每个子表键定义了一个目录标签，这个目录标签对应了本地文件系统中的一个路径，这个路径定义于子表中的`path`配置项。[`[[packages]]`数组](#镜像生成器单包配置) 中的 `output-root` 配置项将被指定一个目标标签，该目录标签对应的本地文件系统路径将被作为根目录，该 `[[packages]]` 相应包下生成的镜像源文件均将相对于该根目录放置。

**示例：**

```toml
[output-roots.lib]
path = "./lib/src"

[output-roots.app]
path = "./main/src"

[[packages]]
package-name = "com.vendor1.lib1"
output-root = "lib"  # 生成的镜像源文件将生成于目录 "./lib/src/com/vendor1/lib1"
filters = ...

[[packages]]
package-name = "com.vendor2.lib2"
output-root = "lib"  # 生成的镜像源文件将生成于目录 "./lib/src/com/vendor2/lib2"
filters = ...

[[packages]]
package-name = "com.mycompany.app"
output-root = "app"  # 生成的镜像源文件将生成于目录 "./main/src/com/mycompany/app"
filters = ...
```

#### 头文件输入

`[sources]` 表的每个子表键定义了一组独立的头文件，镜像生成器将确保将这些头文件作为输入。

**该表支持以下配置项：**

* `paths` (必选)：字符串数组，每个字符串是单个头文件的路径，镜像生成器将确保读取这些头文件。

* `arguments` (可选)：字符串数组，保存有一系列 clang 编译选项，当镜像生成器处理列举在 `paths` 中的源文件时，将作为 clang 的命令行参数传入。

**示例：**

```toml
[sources.all]
paths = ["original-objc/M.h"]
```

#### 额外的 clang 命令行参数

`[sources-mixins]`表中的每个表项是个表，每个表首先通过`sources`属性的正则表达式匹配 [`[sources]`表](#头文件输入) 中的一到若干个表键，匹配到的这些表键所对应的头文件在被 clang 处理时，将额外指定命令行参数。

**支持的属性：**

* `sources` （必选）：正则表达式字符串，用于匹配头文件名。

* `arguments-prepend`、`arguments-append` （可选）：字符串数组，内容将被用于作为额外的 clang 命令行参数。`arguments-prepend` 和 `arguments-append` 中的命令行参数将被分别插入到相应 `[sources]` 表项的 `arguments` 属性的数组的前和后。

**示例：**

```toml
[sources.UIWidgets]
paths = ["objc/UIWidgets.h"]
arguments = [ "-I", "/usr/local/include/share/Widgets" ]

[sources.UIPanels]
paths = ["objc/UIPanels.h"]
arguments = [ "-I", "/usr/local/include/share/Panels" ]

[sources-mixins.UI]
# 这些 clang 编译选项将同时追加至上述 UIWidgets 和 UIPanels 的 clang 编译命令
sources = ["UI.+"]
arguments-append = [
    "-I", "/usr/local/include/Frameworks/AcmeUI"
]
```

#### 镜像生成器单包配置

`[[packages]]` 数组的每个表项指定了一个目标仓颉包名，一组名称过滤器，用于说明哪些 Objective-C 实体将被镜像到该仓颉包中，以及可选的，该包的输出目录。

**支持以下配置项：**

* `package-name` （必选）：字符串，目标仓颉包的名称。

* `output-path` （可选）：字符串，值为文件系统的路径（绝对路径或相对路径均可），该仓颉包的镜像文件均将被输出到该目录下。如果该路径中存在任何不存在的目录，镜像生成器将尝试创建它们。

* `output-root` （可选）：字符串，值为 [`[output-roots]`表](#输出根目录) 中的子表键，也就是一个目录标签。该目录标签对应一个输出根目录，基于这个根目录，目标仓颉包的包名将作为子目录名，包名中的点 `.` 被替换为路径分隔符 `/`，得到的路径将与 `output-path` 配置同等效果。

  如果 `output-root` 未配置，且 `[output-roots]` 中有且只有一个子表键，该子表键对应的输出根目录将被采用；否则镜像生成器将报错。

  **示例：**

  ```toml
  [output-roots.main]
  path="./cj-mirrors"
  
  [[packages]]
  package-name = "objc.foundation"
  output-root = "main"
  ```

​  输出文件将置于 `./cj-mirrors/objc/foundation` 目录下。

* `filters` （必选）：一个表，指定了一组名称过滤器，说明了需要确保源文件中的哪些 Objective-C 实体声明镜像到指定的仓颉包中。

  **示例：**

  ```toml
  # Foundation 框架镜像
  [[packages]]
  package-name = "objc.foundation"
  filters = { include = "NS.+" }
  ```

**名称过滤器：**

一个名称过滤器是一个 TOML 表，表中包含：

* `include`、`exclude`、`union`、`intersect` 和 `not` 其中之一选一个作为属性。
* 可选的 `filter` 属性和/或 `filter-not` 属性。

以下将对各属性进行详细解释：

* `include`：该属性值可以是一个正则表达式字符串，或一个数组中包含若干正则表达式字符串。如果值为单个正则表达式，只有匹配该正则表达式的类型名将被采纳。如果是正则表达式的数组，类型名匹配其中任一正则表达式即可被采纳。

  **示例：**

  ```toml
  # 仅包含名称以 "NS" 开头的实体：
  filters = { include = "NS.+" }
  
  # 仅包含名称以 "Foo" 开头或以 "Bar" 结尾的实体：
  filters = { include = ["Foo.*", ".*Bar"] }
  ```

* `exclude`：与 `include` 相反（见上文）如果一个类型名能够被 `include` 过滤器采纳，那么它就不会被 `exclude` 采纳；反之亦然。

  **示例：**

  ```toml
  # 包含所有实体，但名称以 "INTERNAL_" 开头的除外：
  filters = { exclude = "INTERNAL_.+" }
  ```

* `union`：该属性用于结合两个以上的过滤器。其值为过滤器的数组，类型名只需要被其中任一过滤器采纳，就将被 `union` 过滤器采纳。

  **示例：**

  ```toml
  # 等效于上面第二个 `include` 示例：
  filters = { union = [ { include = "Foo.*" }, { include = ".*Bar" } ] }
  ```

* `intersect`：该属性用于结合两个以上的过滤器。其值为过滤器的数组，类型名必须被其中所有过滤器采纳，才会被 `intersect` 过滤器采纳。

  **示例：**

  ```toml
  # 添加排除过滤器：
  filters = { intersect = [ { include = "NS.+" },
                            { exclude = "NSAccidentalClash" } ] }
  ```

* `not`：该属性用于反转一个过滤器的含义，其值为单个过滤器。

  **示例：**

  ```toml
  # 添加排除过滤器的另一种方式：
  filters = { intersect = [ { include = "NS.+" },
                            { not = { include = "NSAccidentalClash" } } ] }
  ```

* `filter`/`filter-not` （可选）：这两个属性必须与上述其他属性一起使用，即它们不能是 `filters` 表中的唯一属性。该属性值可以是一个正则表达式字符串，或一个数组中包含若干正则表达式字符串，其语义与 `include` 和 `exclude` 属性完全一致。

  `filter` 和 `filter-not` 用于在主过滤器已经过滤得到的所有类型名的基础上，进一步缩减成功匹配的类型名。对于 `filter`，只有成功匹配其中任一正则表达式的类型名将被采纳；对于 `filter-not`，只有不匹配其中任何正则表达式的类型名将被采纳。

  `filter` 和 `filter-not` 本质上是 `intersect` 分别配合 `include` 和 `exclude` 操作的简写形式。

  **示例：**

  ```toml
  # 不使用 filter-not：
  filters = { intersect = [ { include = "NS.+" },
                            { exclude = "NSAccidentalClash" } ] }
  # 使用 filter-not：
  filters = { include = "NS.+", filter-not = "NSAccidentalClash" }
  
  # filter 和 filter-not 可以被同时使用：
  filters = { include    = ".*Fizz.+",
              filter     = ".+Buzz.*",
              filter-not = ".*FizzBuzz.*" }
  ```

#### 类型名映射替换

`[[mappings]]` 是一个表的数组，每个表中表示的类型名映射替换关系，最终会被统一收集为一个映射替换表，用于镜像生成过程中，将表中指定的若干类型名称替换为其他类型名。除了 C 的基本数据类型（例如 `int`），几乎所有其他 Objective-C 类型均可通过此配置实现类型名的替换。

**示例：**

```toml
# 生成镜像时，将所有 Objective-C 的 id 类型镜像替换为 NSObjectProtocol。
[[mappings]]
id = "NSObjectProtocol"
```

#### 导入其他配置文件

`imports` 配置项的值是一个字符串数组，每个字符串是其他配置文件的文件路径，该配置文件中的配置信息将被添加进当前配置文件中。被导入的配置文件中的 `packages` 和 `mappings` 条目配置项中的配置信息将被追加到当前配置文件中。

支持配置文件的嵌套导入，但如果检测到配置文件的循环依赖导入则将导致编译器报错。

**使用示例:**

```toml
imports = ["../common.toml"]
```

## 运行时行为

### 初始化

当控制流首次进入仓颉代码时，所有全局及 `static` 仓颉变量完成初始化，所有仓颉类型的静态初始化器被调用。这发生在[互操作类](#互操作类)首次从 Objective-C 代码被访问时。

> **注意**：
>
> 上述仓颉初始化代码**不得**以任何方式使用镜像类型或互操作类，否则将导致死锁。

### 终结器

#### `dealloc`

互操作类不得重写 `NSObject` 的 `dealloc()` 方法。若尝试这样做，将与互操作支撑代码产生名称冲突并导致编译错误。

#### 仓颉终结器

镜像类声明中不得包含仓颉终结器（`~init()`）。[互操作类](#互操作类)是否可包含终结器**尚待确定**。

### 异常

Objective-C 与仓颉均支持异常。双向互操作场景中，栈上两种语言帧可能交错；异常抛出时栈展开可能跨越语言边界。

> **注意**：
>
> 当前版本中，此类跨边界展开导致**未定义行为**。从仓颉调用的 Objective-C 方法/函数不得遗留未捕获异常，反之亦然。

无法在仓颉代码中 `throw` Objective-C 异常，亦无法在 Objective-C 代码中 `throw` 仓颉异常。

### 内存管理

Objective-C 与仓颉对象分别驻留于各自堆中。互操作库与桥接代码确保：只要另一语言中的可访问变量或数据结构仍持有对某对象的引用，该对象就不会被释放或回收。

跨语言堆一致性机制有三项重大限制：

1. 曾在仓颉代码中使用过的 Objective-C 对象，其释放时机取决于仓颉垃圾回收器；短暂引用未必立即释放。在循环中遍历大型 Objective-C 数组或集合可能暂时抬高内存占用，直至 Cangjie GC 运行。

2. Objective-C ARC 与仓颉垃圾回收器各自仅在其环境内运行，跨语言循环引用可能导致内存泄漏。应避免或主动断开循环引用。

3. 当前版本中，镜像类型与互操作类的值**不得**存入仓颉全局或 `static` 变量，亦不得存入此类变量所引用的数据结构。因此禁止将外部类型值转换为 `Object` 或 `Any`。正在消除此限制。**注意**： `cjc` 尚未完全强制执行上述限制，须严格遵守编程纪律，否则可能导致异常终止。

### 线程

由仓颉 `spawn` 创建的线程可不受限制地使用 Objective-C 镜像类型。

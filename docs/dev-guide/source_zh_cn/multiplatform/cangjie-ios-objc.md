# 仓颉-ObjC 互操作

## 简介

> **注意：**
>
> 仓颉-ObjC 互操作特性为实验性功能，特性还在持续完善中。

仓颉 SDK 支持仓颉与 ObjC(Objective-C) 的互操作，因此在 IOS 应用中可以使用仓颉编程语言开发业务模块。整体调用流程如下：

- 场景一：从 ObjC 调用仓颉，ObjC code → (glue code) → 仓颉 code
- 场景二：从仓颉调用 ObjC，仓颉 code → (glue code) → ObjC code

**涉及的概念：**

1. Mirror Type：镜像类型，ObjC 类型使用仓颉语法形式的表达，供开发者使用仓颉的方式调用 ObjC 方法。
2. CFFI：C Foreign Function Interface，是 Java/Objetive C/仓颉等高级编程语言提供的 C 语言外部接口。
3. 胶水代码：用于弥合不同编程语言差异的中间代码。
4. 互操作代码：ObjC 调用仓颉方法的实现代码，或仓颉调用 ObjC 的实现代码。

**涉及的工具：**

1. 仓颉 SDK：仓颉的开发者工具集。
2. cjc：指代仓颉编译器。
3. ObjCInteropGen: ObjC 镜像生成工具
4. IOS 工具链：IOS 应用开发所需的工具集合。

仓颉与 ObjC 之间的调用，通常需要利用 ObjC 编程语言或仓颉编程语言的 CFFI 编写"胶水代码"。然而，人工书写这种胶水代码（Glue Code）对于开发者来说特别繁琐。

仓颉 SDK 中包含的仓颉编译器（cjc）支持自动生成必要的胶水代码，减轻开发者负担。

cjc 自动生成胶水代码需要获取在跨编程语言调用中涉及的 ObjC 类型（类和接口）的符号信息，该符号信息包含在 Mirror Type 中。

**语法映射：**

|            仓颉类型                            |                    ObjC 类型                       |
|:----------------------------------------------|:---------------------------------------------------|
|`@ObjCMirror public interface A`         |        `@protocol A`                               |
|  `@ObjCMirror public class A`           |        `@interface A`                              |
|    `func fooAndB(a: A, b: B): R`              |        `- (R)foo:(A)a andB:(B)b`                   |
|  `static func fooAndB(a: A, b: B): R`         |        `+ (R)foo:(A)a andB:(B)b`                   |
|    `prop foo: R`                              |        `@property(readonly) R foo`                 |
|    `mut prop foo: R`                          |        `@property R foo`                           |
|     `static prop foo: R`                      |        `@property(readonly, class) R foo`          |
|     `static mut prop foo: R`                  |        `@property(class) R foo`                    |
|    `init()`                                   |        `- (instancetype)init`                      |
|    `let x: R`                                 |        `const R x`                                 |
|     `var x: R`                                |        `R x`                                       |

**类型映射：**

| 仓颉类型                                   |                ObjC 类型      |
|:------------------------------------------|:------------------------------|
|    `Unit`                                 |        `void`                 |
|     `Int8`                                |        `signed char`          |
|    `Int16`                                |        `short`                |
|     `Int32`                               |        `int`                  |
|     `Int64`                               |        `long/NSInteger`       |
|     `Int64`                               |        `long long`            |
|     `UInt8`                               |        `unsigned char`        |
|    `UInt16`                               |        `unsigned short`       |
|    `UInt32`                               |        `unsigned int`         |
|    `UInt64`                               |   `unsigned long/NSUInteger`  |
|    `UInt64`                               |   `unsugned long long`        |
|    `Float32`                              |        `float`                |
|   `Float64`                               |        `double`               |
|  `Bool`                                   |        `bool/BOOL`            |
| `A` where `A` is `class`                  | `A*`                          |
| `ObjCPointer<A>` where `A` is `class`     | `A**`                         |
| `ObjCPointer<A>` where `A` is not `class` | `A*`                          |
| `struct A`                                | `@C struct A`                 |
| `ObjCBlock`                               | `Block`                       |
| `ObjCFunc`                                | `function type`               |
| `ObjCId`                                  | `id`                          |
| `A` where `A` is `interface`              | `id<A>`                       |

注意：

1. ObjC 源码中标记为 `unavailable` 的类型不做映射转化
2. 当前版本不支持 ObjC 中的全局函数与全局变量的转化
3. 匿名 `C enumeration` 的类型不做映射转化
4. `C unions` 的类型不做映射转化
5. `const` `volatile` `restrict`  的类型不做映射转化

以仓颉调用 ObjC 为例，整体开发过程描述如下：

1. 开发者进行接口设计，确定函数调用流程和范围。

2. 根据步骤 1，为被调用的 ObjC 类和接口生成 Mirror Type。

    ```objc
    // original-objc/Base.h
    #import <Foundation/Foundation.h>

    @interface Base : NSObject

    - (void)f;

    @end
    ```

    ```objc
    // original-objc/Base.m
    #import "Base.h"

    @implementation Base

    - (void) f {
        printf("Hello from ObjC Base f()\n");
    }

    @end
    ```

    并创建配置文件，以`example.toml`为例：

    ```toml
    # Place mirrors of the Foundation framework classes in the 'objc.foundation' package:
    [[packages]]
    filters = { include = ["NS.+"] }
    package-name = "objc.foundation"

    # Place the mirror of the class Base in the 'example' package:
    [[packages]]
    filters = { include = "Base" }
    package-name = "example"

    # Write the output files with mirror type definitions to the './mirrors' directory:
    [output-roots.default]
    path = "mirrors"

    # Specify the pathnames of input files
    [sources.all]
    paths = ["original-objc/Base.h"]

    # Extra settings for testing with GNUstep
    [sources-mixins.default]
    sources = [".*"]
    arguments-append = [
        "-F", "/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS.sdk/System/Library/Frameworks",
        "-isystem", "/Library/Developer/CommandLineTools/usr/lib/clang/15.0.0/include",
        "-isystem", "/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS.sdk/usr/include",
        "-DTARGET_OS_IPHONE"
    ]
    ```

    配置文件格式如下:

    ```toml
    # 可选，import其他的toml配置文件，值为字符串数组，每行表示相对于当前工作目录的toml文件路径
    imports = [
        "../path_to_file.toml"
    ]

    # 生成的Mirror Type文件存放路径,当目标文件夹不存在时会自动创建
    [output-roots]
    path = "path_to_mirrors"

    # Mirror Type的输入文件列表，与Clang编译选项
    [sources]
    path = "path_to_ObjC_header_file"
    # 编译选项
    arguments-append = [
        "-F", "/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS.sdk/System/Library/Frameworks",
        "-isystem", "/Library/Developer/CommandLineTools/usr/lib/clang/15.0.0/include",
        "-isystem", "/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS.sdk/usr/include",
        "-DTARGET_OS_IPHONE"
    ]

    # 多个sources属性的集合
    [sources-mixins]
    sources = [".*"]

    # 指定包名
     [[packages]]
    # 如果转换的代码隐式的使用到某些符号，如Uint32/Bool等，需要将该符号一并添加到filters中（如未添加运行 ObjCInteropGen 时会出现Entity Uint32/Bool from `xx.h` is ambiguous between x packages报错信息)
    filters = { include = "Base" }
    package-name = "example"
    ```

    使用 ObjC 的镜像生成工具生成 ObjC 的镜像文件：

    ```bash
    ObjCInteropGen example.toml
    ```

    生成的镜像文件 `Base.cj` 如下：

    <!-- compile -->

    ```cangjie
    // Base.cj
    package example

    import objc.lang.*

    @ObjCMirror
    open class Base {
        public init()
        public open func f(): Unit
    }
    ```

    如果需要基于 `Base` 对部分函数进行重写，示例如下：

    <!-- compile -->
    ```cangjie
    // Interop.cj
    package example

    import objc.lang.*

    @ObjCImpl
    public class Interop <: Base {
        override public func f(): Unit {
            println("Hello from overridden Cangjie Interop.f()")
            Base().f()
        }
    }
    ```

3. 开发互操作代码，使用步骤 2 中生成的 Mirror Type 创建 ObjC 对象、调用 ObjC 方法。

    `互操作代码.cj` + `Mirror Type.cj`→ 实现仓颉调用 ObjC

    例如如下示例，可通过 ObjCImpl 类型继承 Mirror Type 调用 ObjC 类型构造函数：

    <!-- compile -->
    ```cangjie
    // A.cj
    package cjworld

    import objc.lang.*

    @ObjCImpl
    public class A <: M {
        override public func goo(): Unit {
            println("Hello from overridden A goo()")
        }
    }
    ```

4. 使用 cjc 编译互操作代码和 'Mirror Type.cj' 类型。cjc 将生成：

    - 胶水代码。
    - 实际进行互操作的 ObjC 源代码。

    `Mirror Type.cj`\+`互操作代码.cj`→ cjc → `仓颉代码.so`和 `互操作.m`/`互操作.h`

5. 将步骤 4 中生成的文件添加到 macOs/iOS 项目中：

    - `互操作.m/h`：cjc 生成的文件
    - `仓颉代码.so`：cjc 生成的文件
    - 仓颉 SDK 包含的运行时库

    在 ObjC 源代码中插入必要的调用并重新生成该程序。

    macOs/iOS 项目 + `互操作.m/h` + `仓颉代码.so` → macOS/iOS 工具链 → 可执行文件

## 环境准备

**系统要求：**

- **硬件/操作系统**：macOS 12.0 及以上版本(aarch64)运行

**ObjCInteropGen 工具使用环境准备步骤：**

1. 安装对应版本的仓颉 SDK，安装方法请参见[开发指南](https://cangjie-lang.cn/docs?url=%2F1.0.0%2Fuser_manual%2Fsource_zh_cn%2Ffirst_understanding%2Finstall_Community.html)。
2. 执行如下命令安装 LLVM 15。

    ```bash
    brew install llvm@15
    ```

3. 将 `llvm@15/lib/` 子目录添加到 `DYLD_LIBRARY_PATH` 环境变量中。

   ```bash
    export DYLD_LIBRARY_PATH=/opt/homebrew/opt/llvm@15/lib:$DYLD_LIBRARY_PATH
    ```

4. 检测是否安装成功：执行如下命令，如果出现 ObjCInteropGen 使用说明，则证明安装成功。

    ```bash
    ObjCInteropGen --help
    ```

## ObjCMirror 类

ObjCMirror 为 ObjC 类型使用仓颉语法形式的表达，由工具自动生成，供开发者使用仓颉的方式调用 ObjC 方法。

> **注意：**
>
> 本特性尚为实验特性，用户仅可使用文档已说明的特性，若使用了未被提及的特性，将可能出现如编译器崩溃等的未知错误。
> 暂不支持的特性将出现未知错误。

### 构造函数

```objc
// M.h
@interface M : NSObject
// Constructor w/out parameters
- (id)init;
// Constructor w/ scalar parameters
- (id)initWithArg0:(int)arg0 andArg1:(float)arg1;
@end

// M.m
#import "M.h"

@implementation M
// -- implement constructors --
@end
```

<!-- compile -->

```cangjie
// M.cj
package cjworld

import objc.lang.*

@ObjCMirror
open class M {
    public init()
    // 带参数的构造函数，必须使用 @ForeignName 修饰。
    @ForeignName["initWithArg0:andArg1:"]
    public init(arg0: IntNative, arg1: Float32)
}
```

<!-- compile -->

```cangjie
// A.cj
package cjworld

import objc.lang.*

@ObjCImpl
class A <: M {
    @ForeignName["initWithArg2:"]
    public init(arg2: Bool) {
        M()
        println("arg2 is ${arg2}")
    }
}
```

当前具体规格如下：

- ObjCMirror 类 的构造函数可有零个、一个或多个参数。
- 构造函数的参数和返回值类型可为具备与 ObjC 的映射关系的基础类型、Mirror 类型或 Impl 类型。
- 构造函数显式声明时，不为 private/static/const 类型。无显式声明的构造函数时，则无法构建该对象。
- 仅支持单个命名参数。
- 暂不支持默认参数值。
- 暂不支持 this() 调用。

### 成员函数

```objc
// M.h
@interface M : NSObject

// Static method with parameters and return type
+ (int64_t) booWithArg0: (int32_t)arg0 andArg1: (bool)arg1;

// Instance method with parameters and return type
- (double) gooWithArg0: (int16_t)arg0 andArg1: (float)arg1;

@end

// M.m
#import "M.h"

@implementation M
// -- implement methods --
@end
```

<!-- compile -->

```cangjie
// M.cj
package cjworld

import objc.lang.*

@ObjCMirror
open class M {
    public init()
    @ForeignName["booWithArg0:andArg1:"]
    public static func boo(arg0: Int32, arg1: Bool): Int64

    @ForeignName["gooWithArg0:andArg1:"]
    public open func goo(arg0: Int16, arg1: Float32): Float64
}
```

<!-- compile -->

```cangjie
// A.cj
package cjworld

import objc.lang.*

@ObjCImpl
class A <: M {
    @ForeignName["gooWithArg0:andArg1:"]
    public override func goo(arg0: Int16, arg1: Float32): Float64 {
        M.boo(1, true)
        M().goo(arg0, arg1)
    }
}
```

具体规格如下：

- ObjCMirror 类 的成员函数可有零个、一个或多个参数。
- 函数的参数和返回值类型可为具备与 ObjC 的映射关系的基础类型、Mirror 类型或 Impl 类型。
- 仅支持单个命名参数。
- 不支持在 Mirror 类中新增成员函数，运行时将有异常。
- 支持 static/open 类型。
- 不支持 private/const 成员。
- 暂不支持默认参数值。
- 暂不支持重载函数。

### 属性

```objc
// M.h
@interface M : NSObject

@property int f

@end
```

<!-- compile -->

```cangjie
// M.cj
package cjworld

import objc.lang.*

@ObjCMirror
open class M {
    public init()
    protected open mut prop f: IntNative
}
```

<!-- compile -->

```cangjie
// A.cj
package cjworld

import objc.lang.*

@ObjCImpl
class A <: M {
    @ForeignName["gooWithArg0:"]
    public func goo(arg0: IntNative): IntNative{
        M().f + arg0
    }
}
```

具体规格如下：

- 类型可为具备与 ObjC 的映射关系的基础类型、Mirror 类型或 Impl 类型。
- 不支持 private/const 成员。
- 支持 static/open 修饰。
- 暂不支持映射 assign/readonly 等 attribute，仓颉侧映射均按照 readwrite 处理，在 objc 侧处理上述 attribute 的属性时，以 objc 规格为准。
- 若属性被 Impl 子类重载，则不允许在 ObjC 侧的构造函数中调用该属性，将出现运行时崩溃。

### 成员变量

```objc
// M.h
@interface M : NSObject
{
@public
double m;
}

@end
```

<!-- compile -->

```cangjie
// M.cj
package cjworld

import objc.lang.*

@ObjCMirror
open class M {
    public init()
    public var m: Float64
}
```

<!-- compile -->

```cangjie
// A.cj
package cjworld

import objc.lang.*

@ObjCImpl
class A <: M {
    @ForeignName["initWithM:"]
    public init(m: Float64) {
        M()
        this.m = m
    }
    @ForeignName["gooWithArg0:"]
    public func goo(arg0: Float64): Float64 {
        this.m + arg0
    }
}
```

具体规格如下：

- 支持可变变量 var、不可变变量 let。
- 类型可为具备与 ObjC 的映射关系的基础类型、Mirror 类型或 Impl 类型。
- 不支持 static/const/private 类型变量。

### 继承

Mirror 类支持继承 open Mirror 类。
示例如下：

<!-- compile -->

```cangjie
// M1.cj
package cjworld

import objc.lang.*

@ObjCMirror
open class M1 {
    @ForeignName["init"]
    public init()
    @ForeignName["goo"]
    public open func goo(): Int64
}
```

<!-- compile -->

```cangjie
// M.cj
package cjworld

import objc.lang.*

@ObjCMirror
open class M <: M1 {
    @ForeignName["init"]
    public init()
    @ForeignName["goo"]
    public override func goo(): Int64
}
```

<!-- compile -->

```cangjie
// A.cj
package cjworld

import objc.lang.*

@ObjCImpl
public class A <: M {
    public init() {
        println("cj: A.init()")
        let m = M()
    }

    @ForeignName["boo"]
    public func boo(): Bool {
        println("cj: A.boo: M.init()")
        let m = M()
        println("cj: A.boo: M1.init()")
        let m1 = M1()

        m.goo() > m1.goo()
    }
}
```

```objc
// main.m
#import "A.h"
#import <Foundation/Foundation.h>

int main(int argc, char** argv) {
    @autoreleasepool {
        A* a = [[A alloc] init];

        NSLog(@"objc: result on [a boo] %s", [a boo] ? "YES" : "NO");
    }
    return 0;
}
```

### 约束限制

- 暂不支持 String 类型。（暂不支持的特性将出现未知错误）
- 暂不支持类型检查和类型强转。（暂不支持的特性将出现未知错误）
- 暂不支持处理异常。（暂不支持的特性将出现未知错误）
- 不支持普通仓颉类继承 Mirror 类。
- 不支持继承普通仓颉类。
- 当成员函数或构造函数有超过一个的参数时，必须使用 @ForeignName。

> 注意：
>
> 支持在 Impl 类或普通仓颉类中调用 Mirror 类成员，规格一致。

## ObjCMirror 接口

支持映射 ObjC 中的 `protocol` 为接口，`interface` 为 `open class`，具体示例如下：

```objc
// Foo.h
@protocol Foo : <NSObject>
- (void) foo;
@end
```

```objc
// M.h
@interface M : NSObject
- (void) acceptFoo: (id<Foo>)foo;
@end
```

<!-- compile -->

```cangjie
// Foo.cj
@ObjCMirror
public interface Foo {
    // 不支持构造函数。
    func foo(): Unit
}
```

<!-- compile -->

```cangjie
// M.cj
@ObjCMirror
public open class M {
    public init()
    public open func acceptFoo(foo: Foo): Unit 
}
```

<!-- compile -->

```cangjie
// A.cj
@ObjCImpl
class A <: M {
    public init() {}

    public func acceptFoo(foo: Foo) {
        foo.foo()
    }
}
```

具体规格如下:

- 支持成员函数，规则同 [ObjCMirror 类](#成员函数)
- 支持属性，规则同 [ObjCMirror 类](#属性)
- 暂不支持默认成员实现。
- 暂不支持映射 ObjC protocols 的构造函数。
- 暂不支持映射 ObjC protocols 的 @optional 和 @required 成员函数。
- 暂不支持成员变量。
- @ObjCMirror 接口仅支持继承 @ObjCMirror 接口，不支持其他类型。
- 当成员函数或构造函数有超过一个的参数时，必须使用 @ForeignName。

## ObjCImpl 类

ObjCImpl 为仓颉注解，语义为该类的方法与成员可以被 ObjC 函数调用。编译期间编译器会为 `@ObjCImpl` 的仓颉类生成对应的 objc 代码。

> **注意：**
>
> 本特性尚为实验特性，用户仅可使用文档已说明的特性，若使用了未被提及的特性，将可能出现如编译器崩溃等的未知错误。
> 暂不支持的特性将出现未知错误。

### 调用 ObjCImpl 的构造或成员函数

在 ObjC 中调用 ObjCImpl 的构造或成员函数如下：

<!-- compile -->

```cangjie
// A.cj
package cjworld

import objc.lang.*

@ObjCImpl
class A <: M {
    @ForeignName["initWithM:"]
    public init(m: Float64) {
        this.m = m
    }
    @ForeignName["gooWithArg0:"]
    public func goo(arg0: Float64): Float64 {
        this.m + arg0
    }
}
```

```objc
int main(int argc, char** argv) {
    @autoreleasepool {
        M* a = [[A alloc] init];
        [a goo];
        M* b = [[A alloc] init];
        [b goo];
    }
    return 0;
}
```

具体规格如下：

- ObjCImpl 类 的成员函数可有零个、一个或多个参数。
- 函数的参数和返回值类型可为具备与 ObjC 的映射关系的基础类型、Mirror 类型或 Impl 类型。
- 构造函数必须显式声明。
- 仅支持单个命名参数。
- 暂不支持默认参数值。
- 成员函数支持 static/open 修饰。
- 仅有 public 函数可在 ObjC 侧被调用。

### 属性

<!-- compile -->

```cangjie
package cjworld

import objc.lang.*

@ObjCMirror
public class M1 {
    public prop answer: Float64

    @ForeignName["initWithAnswer:"]
    public init(answer: Float64)
}

@ObjCMirror
open public class M {
    public mut prop bar: M1
    @ForeignName["init"]
    public init()
}

@ObjCImpl
public class A <: M {
    private var _b: M1
    public mut prop b: M1 {
        get() {
            _b
        }
        set(value) {
            _b = value
        }
    }
    public init() {
        println("cj: A.init()")
        _b = M1(123.123)
        bar = M1(_b.answer)
    }
}
```

```objc
#import "A.h"
#import <Foundation/Foundation.h>

int main(int argc, char** argv) {
    @autoreleasepool {
        A* a = [[A alloc] init];
        NSLog(@"objc: value of bar.answer: %lf", a.bar.answer);
        NSLog(@"objc: value of b.answer: %lf", a.b.answer);
    }
    return 0;
}
```

具体规格如下：

- 类型可为具备与 ObjC 的映射关系的基础类型、Mirror 类型或 Impl 类型。
- 支持 static/open 修饰。
- 仅 public 类型可在 ObjC 侧被调用。

### 成员变量

<!-- compile -->

```cangjie
// M.cj
package cjworld

import objc.lang.*

@ObjCMirror
open public class M {
    @ForeignName["init"]
    public init()
}

@ObjCMirror
open public class M1 {
    @ForeignName["init"]
    public init()
    @ForeignName["foo"]
    public func foo(): Float64
}

@ObjCImpl
public class A <: M {
    public let m1 : M1
    public var num: Int64

    public init() {
        super()

        m1 = M1()
        num = 42
    }
}
```

```objc
// main.m
#import "A.h"
#import <Foundation/Foundation.h>

int main(int argc, char** argv) {
    @autoreleasepool {
        A* a = [[A alloc] init];

        // NOTE: Cangjie fields are translated to props, NOT instance variables
        NSLog(@"objc: value of [a.m1 foo]: %lf", [a.m1 foo]);
        NSLog(@"objc: value of a.num %ld", a.num);
    }
    return 0;
}
```

具体规格如下：

- 支持可变变量 var、不可变变量 let。
- 类型可为具备与 ObjC 的映射关系的基础类型、Mirror 类型或 Impl 类型。
- 仅 public 类型可在 ObjC 侧被调用。

### 约束限制

- 暂不支持 String 类型。
- 暂不支持类型检查和类型强转。
- 暂不支持在 ObjC 侧继承 Impl 类。
- 暂不支持处理异常。
- 当成员函数或构造函数有超过一个参数时，必须使用 @ForeignName。（当该函数为重载函数时则不需要）
- 不支持普通仓颉类继承 Impl 类。
- 不支持继承普通仓颉类。
- 不支持继承 Impl 类。
- Impl 类必须继承 Mirror 类。
- Impl 可以实现零个或多个 Mirror 接口。

> 注意：
>
> 暂不支持在仓颉中调用 Impl 类的任意成员。

## objc.lang

是与互操作库一起提供的包，包含用于对 Objective-C 中的其他类型完成映射的支持类型。

### ObjCPointer

`ObjCPointer` 类型定义在 `objc.lang` 包中，用于映射 Objective-C 中定义的原始指针。其签名如下：

<!-- compile -->

```cangjie
struct ObjCPointer<T> {
    /* 从 C Pointer构造对象 */
    public init(ptr: CPointer<Unit>)

    public func isNull(): Bool

    public func read(): T

    public func write(value: T): Unit
}
```

`ObjCPointer` 方法的实现均在编译器中。

具体规则如下：

- 只有与 Objective-C 兼容的明确的类型才能用于实例化参数 `T`，包括其他 `ObjCPointer` 类型，例如：`ObjCPointer<Class>`、`ObjCPointer<Int64>`、`ObjCPointer<ObjCPointer<Bool>>`。如下类型不合法：`ObjCPointer<U>`，其中 `U` 为类型参数，`ObjCPointer<String>`。
- `ObjCPointer<T>` 当 `T` 是一个明确的与 Objective C 兼容的类型时, 该类型为 Objective-C 兼容类型。
- 由于仓颉类类型 `A` 在 Objective-C 中已经对应了指针类型 `A*`，因此指向类 `ObjCPointer<A>` 的指针对应着指向指针的指针 `A**`。这是在仓颉中模拟 Objective-C 指向指针的唯一方法。

当前约束如下：

- 由于 Objective-C ARC 的限制，`ObjCPointer<class>` 类型**不能**用作任何仓颉方法或属性的返回类型，包括 `@ObjCMirror` 和 `@ObjCImpl` 声明的方法和属性

### ObjCBlock

`ObjCBlock` 类型定义在 `objc.lang` 包中，用于映射 Objective-C 的 `Block` 类型。其签名如下：

<!-- compile -->

```cangjie
public class ObjCBlock<F> {

    public ObjCBlock(let ptr: CPointer) 

    public prop call: F 

    public unsafe func unsafeGetNativeABIPointer(): CPointer

    public unsafe func unsafeGetFunctionPointer(): CPointer 
}
```

`ObjCBlock` 方法的实现均在编译器中。

### ObjCFunc

`ObjCFunc` 类型定义在 `objc.lang` 包中，用于映射 Objective-C 的函数。其签名如下：

<!-- compile -->

```cangjie
public struct ObjCFunc<F> {

    public ObjCFunc(let ptr: CPointer)

    public prop call: F

    public unsafe func unsafeGetFunctionPointer(): CPointer
}
```

`ObjCFunc` 方法的实现均在编译器中。

示例如下：

<!-- code_no_check -->
```cangjie
let f: ObjCFunc<(Int64) -> Int64> = mirrorFuncCreator() // 对象必须从 ObjC 侧创建，通过 Mirror 类型的返回值或参数传递到仓颉侧。
f.call(123)
let ff = f.call // 报错：不允许值类型赋值。
```

具体规格如下：

- ObjCFunc<F> 中的 F 必须为合法的仓颉函数类型。
- F 的返回值和参数必须为 ObjC 兼容类型。
- ObjCFunc 中的 call 属性仅允许被直接调用，禁止用于其他场景（如赋值给变量、作为函数参数等）。
- 不允许在仓颉侧构造 ObjCFunc<F> 类型对象。

### ObjCId

`ObjCId` 类型定义在 `objc.lang` 包中，用作所有 Mirror 类型的父类型。它是 ObjC 在仓颉世界中的 `id` 类型代表。其签名如下：

<!-- code_no_check -->
```cangjie
@ObjCMirror
public interface ObjCId {}
```

具体规格如下：

- 任意 @ObjCMirror 或 @ObjCImpl 均可以实现该接口。
- 默认所有 Mirror 类型均继承该接口，因此需导入 `import objc.lang.ObjCId` 。

## @C structs

使用 `@C` 注解的结构体，在 `ObjCPointer<T>` 内部使用时，可以用于 `@ObjCMirror` 和 `@ObjcImpl` 的声明参数、返回类型、字段和属性。仓颉代码中此类注解的结构体 `X` 对应于 Objective C 代码中的 `struct X` 类型，前述类型需同时在仓颉侧和 ObjC 侧均存在。

约束如下：

- 结构体可以包含基础类型、CPointer 指针和其他带有 @C 注解的结构体。
- 对于在仓颉或 Objective C 中定义的每个结构体，在对应的另一个语言中都应该有相同的声明。字段及其类型的差异可能会导致运行时错误。
- 结构体应与 `ObjCPointer<T>` 一起使用。例如，`ObjCPointer<MyStruct>`。通过值传递的结构体暂不支持。
- struct 的类型别名 typedef 暂不支持。

示例如下：

```objc
struct X {
    long a;
    float b;
};
```

<!-- compile -->

```cangjie
@C
public struct X {
    var a: Int64
    var b: Float32
}
```

## @optional 方法

支持在仓颉侧调用 ObjC 的 @optional 方法。例如：

```objc
@protocol K
@optional
- (void) unimplemented;
- (void) implemented;
@end
```

在 Objective-C 中，@optional 方法不要求必须实现。当仓颉为包含此类方法的类生成 Mirror 对象并在运行时从 Cangjie 调用这些方法时，若方法未被实际实现，将导致运行时错误。

为避免此问题，仓颉引入了 @ObjCOptional 注解：当调用被该注解标记的方法时，若其实现不存在，则抛出 ObjCOptionalMethodUnimplementedException 异常,若存在，则正常执行调用。

<!-- compile -->

```cangjie
@ObjCMirror
open class M {
    @ObjCOptional
    @ForeignName["foo"]
    public func foo(): Unit
}
```

在调用处可增加 `try-catch` 捕获异常：

<!-- code_no_check -->
```cangjie
try {
    m.foo()
} catch (e: ObjCOptionalMethodUnimplementedException) {
    println("cj: caught ObjCOptionalMethodUnimplementedException!")
}
```

## @property 属性

支持通过 `@ForeignGetterName` 和 `@ForeignSetterName` 映射 ObjC 的 @property 语法。
在 Mirror 类中，具有 `@ForeignGetterName` 和 `@ForeignSetterName` 注解的属性将改变从 Cangjie 端访问属性时使用的方法。默认生成的选择器（来自属性名或 `@ForeignName` 注解）会被指定的选择器覆盖。在下面的示例中，Objective-C 端重新定义了属性的 getter 和 setter 名称：

```objc
@interface Component
@property (getter=isShared, setter=applyShared:) BOOL shared;
/*...*/
@end
```

生成的 ObjMirror 使用上述注解指示了 ObjC 侧正确的名称：

<!-- compile -->

```cangjie
@ObjCMirror
class Component {
    @ForeignGetterName["isShared"]
    @ForeignSetterName["applyShared:"]
    public mut prop shared: Bool
}
```

支持在 ObjCImpl 类中使用该注解，在 ObjC 侧调用属性方法时，需使用注解所指示的名称。

<!-- code_no_check -->

```cangjie
@ObjCImpl
class ComponentChild <: Component {

    @ForeignName["_count"]
    @ForeignSetterName["putCount:"]
    public mut prop count: Int64 {
        get() { 42 }
        set(value) {}
    }
}
```

具体约束如下：

- 注解仅支持一个字符串类型的参数。
- 不支持修饰重载的属性。
- `@ForeignSetterName` 不支持修饰可变类型。

## 同参数类型构造函数

```objc
@interface M
- (id) initWithA: (int) a andB: (float) b;
- (id) initWithC: (int) c andD: (float) d;
@end
```

ObjC 不认为同参数类型但不同参数名的构造函数属于冲突，但在仓颉中，参数类型一致即认为声明冲突。

<!-- compile-error -->

```cangjie
@ObjCMirror
class M {
    @ForeignName["initWithA:andB:"]
    public init(a: Int32, b: Float32)
    @ForeignName["initWithC:andD:"]
    public init(c: Int32, b: Float32) // cjc error, duplicated init type signature
}
```

因此，新增 `@ObjCInit` 注解，通过不同名的静态函数映射 ObjC 中的同类型构造函数。

```cangjie
@ObjCMirror
class M {
    @ObjCInit["initWithA:andB:"]
    public static func initWithAandB(a: Int32, b: Float32): M // M.initWithAandB(...)
    @ObjCInit["initWithC:andD:"]
    public static func initWithCandD(c: Int32, b: Float32): M // M.initWithCandD(...)
}
```

具体规格如下：

- 只支持修饰 `@ObjCMirror` 类中的静态函数。
- `@ObjCInit` 支持零个或一个字符串常量作为注解的参数。
- `@ObjCInit` 修饰的类 `T` 中的静态函数，返回值类型必须为 `T`。
- 暂不支持在 `@ObjCInit` 修饰的函数内使用 `super(x)`。
- `@ObjCInit` 修饰的静态函数其他规则同 `@ObjCMirror` 类的静态函数。

## ObjC 使用 Cangjie 规格

### 新增实验编译选项 `--experimental --enable-interop-cjmapping=<Target Languages>`

该选项启用前端（FE）对非 C 语言的 Cangjie 互操作支持。目前支持的值为 Java 和 ObjC。

### ObjC 使用 Cangjie 接口

为实现 Cangjie 与 Objective-C 的互操作，需将 Cangjie 的接口类型映射为 ObjC 的协议（@protocol）。映射后，用户可：

- 将 Cangjie 接口映射为 ObjC 协议
- 在 ObjC 函数中使用该协议作为参数类型
- 在 ObjC 类中采纳该协议，并将其实例作为函数参数传递

#### 示例

Cangjie 源码：

<!-- compile -->

```cangjie
// cangjie code
package cjworld

public interface A {
    public func foo() : Unit
    public func goo(a: Int32, b:Int32) : Unit
    public func koo() : Int32
    public func hoo(a: Int32) : Int32
}
```

生成的 ObjC 头文件（A.h）：

```ObjC
// A.h
#import <Foundation/Foundation.h>
#import <stddef.h>
@protocol A
- (void)foo;
- (void)goo:(int32_t)a :(int32_t)b;
- (int32_t)koo;
- (int32_t)hoo:(int32_t)a;
@end

```

#### 规格约束

由于与其他语言特性的集成仍在开发中，以下场景暂不支持：

- Cangjie 接口不得继承其他接口
- 不支持泛型接口
- 不支持泛型成员函数
- 不支持成员属性、操作符重载函数
- 成员函数不得为 static
- 成员函数不得包含默认实现
- 不支持函数重载
- 仅允许使用基础数据类型：
    - 数值类型
    - Bool 类型
    - Unit 类型
- 已支持的互操作特性不得使用 interface 作为函数返回值类型

### ObjC 使用 Cangjie 枚举

为实现 Cangjie 与 Objective-C 的互操作，需将 Cangjie 的枚举类型映射为 ObjC 的类。映射后，用户可：

- 调用枚举类型的构造函数以创建对应的枚举对象
- 在 Cangjie 与 ObjC 之间无缝传递枚举对象
- 调用枚举中定义的静态方法和实例方法
- 支持枚举属性的访问

#### 示例

<!-- compile -->

```cangjie
// Cangjie
// =============================================
// Enum Definition: Basic Enum (TimeUnit)
// =============================================
public enum TimeUnit {
    | Year(Int64)
    | Month(Int64)
    | Year
    | Month

    // The public method to Calculate how many months.
    public func CalcMonth(): Unit {
        let s = match (this) {
            case Year(n) => "x has ${n * 12} months"
            case Year => "x has 12 months"
            case TimeUnit.Month(n) => "x has ${n} months"
            case Month => "x has 1 month"
        }
        println(s)
    }

    // The static method to return ten years.
    public static func TenYear(): TimeUnit {
        Year(10)
    }
}
```

生成的 ObjC 头文件与源文件：

```ObjC
// TimeUnit.h
#import <Foundation/Foundation.h>
#import <stddef.h>
__attribute__((objc_subclassing_restricted))
@interface TimeUnit : NSObject
- (id)initWithRegistryId:(int64_t)registryId;
+ (TimeUnit*)Year:(int64_t)p1;
+ (TimeUnit*)Month:(int64_t)p1;
+ (TimeUnit*)Year;
+ (TimeUnit*)Month;
+ (void)initialize;
@property (readwrite) int64_t $registryId;
- (void)CalcMonth;
+ (TimeUnit*)TenYear;
- (void)deleteCJObject;
- (void)dealloc;
@end

// TimeUnit.m
#import "TimeUnit.h"
#import "Cangjie.h"
#import <dlfcn.h>
#import <stdlib.h>
static int64_t (*CJImpl_ObjC_default_TimeUnit_Year_l)(int64_t) = NULL;
static int64_t (*CJImpl_ObjC_default_TimeUnit_Month_l)(int64_t) = NULL;
static int64_t (*CJImpl_ObjC_TimeUnit_Year)() = NULL;
static int64_t (*CJImpl_ObjC_TimeUnit_Month)() = NULL;
static void (*CJImpl_ObjC_default_TimeUnit_deleteCJObject)(int64_t) = NULL;
static void (*CJImpl_ObjC_default_TimeUnit_CalcMonth)(int64_t) = NULL;
static int64_t (*CJImpl_ObjC_default_TimeUnit_TenYear)() = NULL;
static void* CJWorldDLHandle = NULL;
static struct RuntimeParam defaultCJRuntimeParams = {0};
@implementation TimeUnit
- (id)initWithRegistryId:(int64_t)registryId {
    if (self = [super init]) {
        self.$registryId = registryId;
    }
    return self;
}
+ (TimeUnit*)Year:(int64_t)p1 {
    int64_t regId = CJImpl_ObjC_default_TimeUnit_Year_l(p1);
    return [[TimeUnit alloc]initWithRegistryId: regId];
}
+ (TimeUnit*)Month:(int64_t)p1 {
    int64_t regId = CJImpl_ObjC_default_TimeUnit_Month_l(p1);
    return [[TimeUnit alloc]initWithRegistryId: regId];
}
+ (TimeUnit*)Year {
    int64_t regId = CJImpl_ObjC_TimeUnit_Year();
    return [[TimeUnit alloc]initWithRegistryId: regId];
}
+ (TimeUnit*)Month {
    int64_t regId = CJImpl_ObjC_TimeUnit_Month();
    return [[TimeUnit alloc]initWithRegistryId: regId];
}
+ (void)initialize {
    if (self == [TimeUnit class]) {
        defaultCJRuntimeParams.logParam.logLevel = RTLOG_ERROR;
        if (InitCJRuntime(&defaultCJRuntimeParams) != E_OK) {
            NSLog(@"ERROR: Failed to initialize Cangjie runtime");
            exit(1);
        }
        if (LoadCJLibraryWithInit("libdefault.dylib") != E_OK) {
            NSLog(@"ERROR: Failed to init cjworld library ");
            exit(1);
        }
        if ((CJWorldDLHandle = dlopen("libdefault.dylib", RTLD_LAZY)) == NULL) {
            NSLog(@"ERROR: Failed to open cjworld library ");
            NSLog(@"%s", dlerror());
            exit(1);
        }
        if ((CJImpl_ObjC_default_TimeUnit_Year_l =
                dlsym(CJWorldDLHandle, "CJImpl_ObjC_default_TimeUnit_Year_l")) == NULL) {
            NSLog(@"ERROR: Failed to find CJImpl_ObjC_default_TimeUnit_Year_l symbol in cjworld");
            exit(1);
        }
        if ((CJImpl_ObjC_default_TimeUnit_Month_l =
                dlsym(CJWorldDLHandle, "CJImpl_ObjC_default_TimeUnit_Month_l")) == NULL) {
            NSLog(@"ERROR: Failed to find CJImpl_ObjC_default_TimeUnit_Month_l symbol in cjworld");
            exit(1);
        }
        if ((CJImpl_ObjC_TimeUnit_Year = dlsym(CJWorldDLHandle, "CJImpl_ObjC_TimeUnit_Year")) == NULL) {
            NSLog(@"ERROR: Failed to find CJImpl_ObjC_TimeUnit_Year symbol in cjworld");
            exit(1);
        }
        if ((CJImpl_ObjC_TimeUnit_Month = dlsym(CJWorldDLHandle, "CJImpl_ObjC_TimeUnit_Month")) == NULL) {
            NSLog(@"ERROR: Failed to find CJImpl_ObjC_TimeUnit_Month symbol in cjworld");
            exit(1);
        }
        if ((CJImpl_ObjC_default_TimeUnit_deleteCJObject =
                dlsym(CJWorldDLHandle, "CJImpl_ObjC_default_TimeUnit_deleteCJObject")) == NULL) {
            NSLog(@"ERROR: Failed to find CJImpl_ObjC_default_TimeUnit_deleteCJObject symbol in cjworld");
            exit(1);
        }
        if ((CJImpl_ObjC_default_TimeUnit_CalcMonth =
                dlsym(CJWorldDLHandle, "CJImpl_ObjC_default_TimeUnit_CalcMonth")) == NULL) {
            NSLog(@"ERROR: Failed to find CJImpl_ObjC_default_TimeUnit_CalcMonth symbol in cjworld");
            exit(1);
        }
        if ((CJImpl_ObjC_default_TimeUnit_TenYear =
                dlsym(CJWorldDLHandle, "CJImpl_ObjC_default_TimeUnit_TenYear")) == NULL) {
            NSLog(@"ERROR: Failed to find CJImpl_ObjC_default_TimeUnit_TenYear symbol in cjworld");
            exit(1);
        }
    }
}
- (void)CalcMonth {
     CJImpl_ObjC_default_TimeUnit_CalcMonth(self.$registryId);
}
+ (TimeUnit*)TenYear {
    return [[TimeUnit alloc]initWithRegistryId: CJImpl_ObjC_default_TimeUnit_TenYear()];
}
- (void)deleteCJObject {
     CJImpl_ObjC_default_TimeUnit_deleteCJObject(self.$registryId);
}
- (void)dealloc {
    [self deleteCJObject];
}
@end

```

#### 规格约束

由于与其他语言特性的集成仍在开发中，以下场景暂不支持：

- Cangjie enum 不得实现其他接口
- 不支持泛型 enum
- 不支持泛型成员函数
- 成员函数中不使用 Lambda
- 不支持操作符重载
- 不支持函数重载
- 不支持命名参数
- 仅允许使用基础数据类型：
    - 数值类型
    - Bool 类型
    - Unit 类型
- 不支持通过 extend 对 enum 进行扩展

### ObjC 使用 Cangjie 结构体

为实现 Cangjie 与 Objective-C 的互操作，需将 Cangjie 的 struct 类型映射为 ObjC 的类。映射后，用户可：

- 在 ObjC 中调用 Cangjie 侧 public struct 的 public 实例方法、静态方法
- 在 ObjC 中访问 Cangjie 侧 public struct 的 public 静态/非静态成员变量
- 在 ObjC 中访问 Cangjie 侧 public struct 的 public 成员属性
- 在 ObjC 函数中使用 Cangjie 侧 public struct 作为参数类型、返回值类型

#### 示例

Cangjie 源码：

<!-- compile -->

```cangjie
// cangjie code
package cjworld

public struct Vector {
    public let x: Int32
    public let y: Int32

    public init(x: Int32, y: Int32) {
        this.x = x
        this.y = y
    }

    public func dot(v: Vector): Int64 {
        let res: Int64 = Int64(x * v.x + y * v.y)
        return res
    }

    public static func processPrimitive(intArg: Int32, floatArg: Float64, boolArg: Bool): Unit {
        println("Hello from static processPrimitive: ${intArg * 2}, ${floatArg + 1.0} + ${!boolArg}")
    }
}
```

生成的 ObjC 头文件与源文件：

```ObjC
// Vector.h
#import <Foundation/Foundation.h>
#import <stddef.h>
__attribute__((objc_subclassing_restricted))
@interface Vector : NSObject
- (id)init:(int32_t)x :(int32_t)y;
- (id)initWithRegistryId:(int64_t)registryId;
+ (void)initialize;
@property (readwrite) int64_t $registryId;
@property (readonly, getter=x) int32_t x;
- (int32_t)x;
@property (readonly, getter=y) int32_t y;
- (int32_t)y;
- (int64_t)dot:(Vector*)v;
+ (void)processPrimitive:(int32_t)intArg :(double)floatArg :(BOOL)boolArg;
- (void)deleteCJObject;
- (void)dealloc;
@end

// Vector.m
#import "Vector.h"
#import "Cangjie.h"
#import <dlfcn.h>
#import <stdlib.h>
static int64_t (*CJImpl_ObjC_cjworld_Vector_init__ii)(int32_t,int32_t) = NULL;
static void (*CJImpl_ObjC_cjworld_Vector_deleteCJObject)(int64_t) = NULL;
static int32_t (*CJImpl_ObjC_cjworld_Vector_x_get)(int64_t) = NULL;
static int32_t (*CJImpl_ObjC_cjworld_Vector_y_get)(int64_t) = NULL;
static int64_t (*CJImpl_ObjC_cjworld_Vector_dot_RN7cjworld6VectorE)(int64_t,int64_t) = NULL;
static void (*CJImpl_ObjC_cjworld_Vector_processPrimitive___idb)(int32_t,double,BOOL) = NULL;
static void* CJWorldDLHandle = NULL;
static struct RuntimeParam defaultCJRuntimeParams = {0};
@implementation Vector
- (id)init:(int32_t)x :(int32_t)y {
    if (self = [super init]) {
        self.$registryId = CJImpl_ObjC_cjworld_Vector_init__ii(x, y);
    }
    return self;
}
- (id)initWithRegistryId:(int64_t)registryId {
    if (self = [super init]) {
        self.$registryId = registryId;
    }
    return self;
}
+ (void)initialize {
    if (self == [Vector class]) {
        defaultCJRuntimeParams.logParam.logLevel = RTLOG_ERROR;
        if (InitCJRuntime(&defaultCJRuntimeParams) != E_OK) {
            NSLog(@"ERROR: Failed to initialize Cangjie runtime");
            exit(1);
        }
        if (LoadCJLibraryWithInit("libcjworld.dylib") != E_OK) {
            NSLog(@"ERROR: Failed to init cjworld library ");
            exit(1);
        }
        if ((CJWorldDLHandle = dlopen("libcjworld.dylib", RTLD_LAZY)) == NULL) {
            NSLog(@"ERROR: Failed to open cjworld library ");
            NSLog(@"%s", dlerror());
            exit(1);
        }
        if ((CJImpl_ObjC_cjworld_Vector_init__ii = dlsym(CJWorldDLHandle, "CJImpl_ObjC_cjworld_Vector_init__ii")) == NULL) {
            NSLog(@"ERROR: Failed to find CJImpl_ObjC_cjworld_Vector_init__ii symbol in cjworld");
            exit(1);
        }
        if ((CJImpl_ObjC_cjworld_Vector_deleteCJObject = dlsym(CJWorldDLHandle, "CJImpl_ObjC_cjworld_Vector_deleteCJObject")) == NULL) {
            NSLog(@"ERROR: Failed to find CJImpl_ObjC_cjworld_Vector_deleteCJObject symbol in cjworld");
            exit(1);
        }
        if ((CJImpl_ObjC_cjworld_Vector_x_get = dlsym(CJWorldDLHandle, "CJImpl_ObjC_cjworld_Vector_x_get")) == NULL) {
            NSLog(@"ERROR: Failed to find CJImpl_ObjC_cjworld_Vector_x_get symbol in cjworld");
            exit(1);
        }
        if ((CJImpl_ObjC_cjworld_Vector_y_get = dlsym(CJWorldDLHandle, "CJImpl_ObjC_cjworld_Vector_y_get")) == NULL) {
            NSLog(@"ERROR: Failed to find CJImpl_ObjC_cjworld_Vector_y_get symbol in cjworld");
            exit(1);
        }
        if ((CJImpl_ObjC_cjworld_Vector_dot_RN7cjworld6VectorE = dlsym(CJWorldDLHandle, "CJImpl_ObjC_cjworld_Vector_dot_RN7cjworld6VectorE")) == NULL) {
            NSLog(@"ERROR: Failed to find CJImpl_ObjC_cjworld_Vector_dot_RN7cjworld6VectorE symbol in cjworld");
            exit(1);
        }
        if ((CJImpl_ObjC_cjworld_Vector_processPrimitive___idb = dlsym(CJWorldDLHandle, "CJImpl_ObjC_cjworld_Vector_processPrimitive___idb")) == NULL) {
            NSLog(@"ERROR: Failed to find CJImpl_ObjC_cjworld_Vector_processPrimitive___idb symbol in cjworld");
            exit(1);
        }
    }
}
- (int32_t)x {
    return CJImpl_ObjC_cjworld_Vector_x_get(self.$registryId);
}
- (int32_t)y {
    return CJImpl_ObjC_cjworld_Vector_y_get(self.$registryId);
}
- (int64_t)dot:(Vector*)v {
    return CJImpl_ObjC_cjworld_Vector_dot_RN7cjworld6VectorE(self.$registryId, v.$registryId);
}
+ (void)processPrimitive:(int32_t)intArg :(double)floatArg :(BOOL)boolArg {
     CJImpl_ObjC_cjworld_Vector_processPrimitive___idb(intArg, floatArg, boolArg);
}
- (void)deleteCJObject {
     CJImpl_ObjC_cjworld_Vector_deleteCJObject(self.$registryId);
}
- (void)dealloc {
    [self deleteCJObject];
}
@end

```

#### 规格约束

由于与其他语言特性的集成仍在开发中，以下场景暂不支持：

- Cangjie 结构体不得实现其他接口
- 不支持泛型结构体
- 不支持泛型成员函数
- 不支持操作符重载函数
- 不支持函数重载
- 不支持 mut 函数
- 不支持 ObjC 端对成员变量、成员属性赋值
- 成员变量仅允许使用基础数据类型：
    - 数值类型
    - Bool 类型
    - Unit 类型
- 不支持通过 extend 对 struct 进行扩展

### ObjC 使用 Cangjie 类

为实现 Cangjie 与 Objective-C 的互操作，需将 Cangjie 的非 open 类映射为 ObjC 的类。映射后，用户可在 ObjC 代码中，直接调用 Cangjie 侧公开类（public class）的公共实例方法与静态方法。

#### 示例

<!-- compile -->

Cangjie 源码：

```cangjie
// cangjie code
package cjworld

public class Vector {
    let x: Int32
    let y: Int32

    public init(x: Int32, y: Int32) {
        this.x = x
        this.y = y
    }

    public func dot(v: Vector): Int64 {
        let res: Int64 = Int64(x * v.x + y * v.y)
        return res
    }

    public static func processPrimitive(intArg: Int32, floatArg: Float64, boolArg: Bool): Unit {
        println("Hello from static processPrimitive: ${intArg * 2}, ${floatArg + 1.0} + ${!boolArg}")
    }
}

```

生成的 ObjC 头文件与源文件：

```ObjC
// Vector.h
#import <Foundation/Foundation.h>
#import <stddef.h>
__attribute__((objc_subclassing_restricted))
@interface Vector : NSObject
- (id)init:(int32_t)x :(int32_t)y;
- (id)initWithRegistryId:(int64_t)registryId;
+ (void)initialize;
@property (readwrite) int64_t $registryId;
- (int64_t)dot:(Vector*)v;
+ (void)processPrimitive:(int32_t)intArg :(double)floatArg :(BOOL)boolArg;
- (void)deleteCJObject;
- (void)dealloc;
@end

// Vector.m
#import "Vector.h"
#import "Cangjie.h"
#import <dlfcn.h>
#import <stdlib.h>
static int64_t (*CJImpl_ObjC_cjworld_Vector_init__ii)(int32_t,int32_t) = NULL;
static void (*CJImpl_ObjC_cjworld_Vector_deleteCJObject)(int64_t) = NULL;
static int64_t (*CJImpl_ObjC_cjworld_Vector_dot_CN7cjworld6VectorE)(int64_t,int64_t) = NULL;
static void (*CJImpl_ObjC_cjworld_Vector_processPrimitive___idb)(int32_t,double,BOOL) = NULL;
static void* CJWorldDLHandle = NULL;
static struct RuntimeParam defaultCJRuntimeParams = {0};
@implementation Vector
- (id)init:(int32_t)x :(int32_t)y {
    if (self = [super init]) {
        self.$registryId = CJImpl_ObjC_cjworld_Vector_init__ii(x, y);
    }
    return self;
}
- (id)initWithRegistryId:(int64_t)registryId {
    if (self = [super init]) {
        self.$registryId = registryId;
    }
    return self;
}
+ (void)initialize {
    if (self == [Vector class]) {
        defaultCJRuntimeParams.logParam.logLevel = RTLOG_ERROR;
        if (InitCJRuntime(&defaultCJRuntimeParams) != E_OK) {
            NSLog(@"ERROR: Failed to initialize Cangjie runtime");
            exit(1);
        }
        if (LoadCJLibraryWithInit("libcjworld.dylib") != E_OK) {
            NSLog(@"ERROR: Failed to init cjworld library ");
            exit(1);
        }
        if ((CJWorldDLHandle = dlopen("libcjworld.dylib", RTLD_LAZY)) == NULL) {
            NSLog(@"ERROR: Failed to open cjworld library ");
            NSLog(@"%s", dlerror());
            exit(1);
        }
        if ((CJImpl_ObjC_cjworld_Vector_init__ii = dlsym(CJWorldDLHandle, "CJImpl_ObjC_cjworld_Vector_init__ii")) == NULL) {
            NSLog(@"ERROR: Failed to find CJImpl_ObjC_cjworld_Vector_init__ii symbol in cjworld");
            exit(1);
        }
        if ((CJImpl_ObjC_cjworld_Vector_deleteCJObject = dlsym(CJWorldDLHandle, "CJImpl_ObjC_cjworld_Vector_deleteCJObject")) == NULL) {
            NSLog(@"ERROR: Failed to find CJImpl_ObjC_cjworld_Vector_deleteCJObject symbol in cjworld");
            exit(1);
        }
        if ((CJImpl_ObjC_cjworld_Vector_dot_CN7cjworld6VectorE = dlsym(CJWorldDLHandle, "CJImpl_ObjC_cjworld_Vector_dot_CN7cjworld6VectorE")) == NULL) {
            NSLog(@"ERROR: Failed to find CJImpl_ObjC_cjworld_Vector_dot_CN7cjworld6VectorE symbol in cjworld");
            exit(1);
        }
        if ((CJImpl_ObjC_cjworld_Vector_processPrimitive___idb = dlsym(CJWorldDLHandle, "CJImpl_ObjC_cjworld_Vector_processPrimitive___idb")) == NULL) {
            NSLog(@"ERROR: Failed to find CJImpl_ObjC_cjworld_Vector_processPrimitive___idb symbol in cjworld");
            exit(1);
        }
    }
}
- (int64_t)dot:(Vector*)v {
    return CJImpl_ObjC_cjworld_Vector_dot_CN7cjworld6VectorE(self.$registryId, v.$registryId);
}
+ (void)processPrimitive:(int32_t)intArg :(double)floatArg :(BOOL)boolArg {
     CJImpl_ObjC_cjworld_Vector_processPrimitive___idb(intArg, floatArg, boolArg);
}
- (void)deleteCJObject {
     CJImpl_ObjC_cjworld_Vector_deleteCJObject(self.$registryId);
}
- (void)dealloc {
    [self deleteCJObject];
}
@end

```

#### 规格约束

由于与其他语言特性的集成仍在开发中，以下场景暂不支持：

- 不支持 open class
- Cangjie class 不得实现其他接口
- 仅支持 public 成员函数映射，其它成员不会报错，不会映射到 ObjC 端
- 不支持泛型 class
- 不支持泛型成员函数
- 不支持函数重载
- 成员函数参数和返回值允许使用基础数据类型（数值类型、Bool 类型和 Unit 类型）和本包定义的非 open public class 类型
- 不支持通过 extend 对 class 进行扩展

## 版本约束限制

1. 当前版本的 ObjCInteropGen 功能存在如下约束限制：

    - 不支持 ObjC 类/接口中的非静态成员变量转换
    - 不支持 ObjC 类/接口中的指针属性转换
    - 不支持对构造函数 init 方法的转换
    - 不支持同时转换多个 .h 头文件
    - 不支持 Bit fields 转换

2. 当前版本的 ObjC 互操作方案存在如下约束限制：
    - 不支持 ObjC Mirror 和 Impl 类的实例逃逸出线程范围，即不能作为全局变量、静态变量，或作为这些变量的字段成员
    - ObjC Mirror 和 Impl 类的实例不能作为其他 ObjC Mirror 或 Impl 对象的字段成员
    - ObjC Mirror 和 Impl 类的实例不能被 Lambda 表达式块或 spawn 线程捕获

3. 版本使用过程中需额外下载依赖文件 `Cangjie.h` (下载地址： <https://gitcode.com/Cangjie/cangjie_runtime/blob/dev/runtime/src/Cangjie.h>),并集成至项目中。
4. 在仓颉中依赖了 Foundation 中的类型时，例如 NSObject 等，由于 Foundation 实际已被导入，但是 NSObject.h 头文件未被显式指定，因此当前可通过创建同名空头文件，保证编译正常。
5. 当前 ObjCImpl 的构造函数实现使用 `[self doesNotRecognizeSelecor:_cmd];` 特性，每次均抛出异常，无返回值，因此需关闭 `-Werror=return-type` 的编译期检查能力，保证编译正常。
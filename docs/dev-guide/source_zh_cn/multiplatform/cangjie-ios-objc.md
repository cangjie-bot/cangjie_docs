# 仓颉-ObjC 互操作

## 简介

> **注意：**
>
> 仓颉-ObjC 互操作特性为实验性功能，特性还在持续完善中。

仓颉 SDK 支持仓颉与 ObjC(Objective-C) 的互操作，因此在 IOS 应用中可以使用仓颉编程语言开发业务模块。整体调用流程如下：

- 场景一：从 ObjC 调用仓颉，ObjC code → (glue code) → 仓颉 code
- 场景二：从仓颉调用 ObjC，仓颉 code → (glue code) → ObjC code

**涉及的概念：**

1. Mirror Type：镜像类型，Java 类型使用仓颉语法形式的表达，供开发者使用仓颉的方式调用 ObjC 方法。
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

    ```cangjie
    // Base.cj
    package example

    import interoplib.objc.*

    @ObjCMirror
    open class Base {
        public init()
        public open func f(): Unit
    }
    ```

    如果需要基于 `Base` 对部分函数进行重写，示例如下：

    ```cangjie
    // Interop.cj
    package example

    import interoplib.objc.*

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

    ```cangjie
    // A.cj
    package cjworld

    import interoplib.objc.*

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

## ObjCMirror 规格

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

import interoplib.objc.*

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

import interoplib.objc.*

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
- 构造函数的参数和返回值类型可为具备与 Objc 的映射关系的基础类型、Mirror 类型或 Impl 类型。
- 构造函数显式声明时，不为 private/static/const 类型。无显式声明的构造函数时，则无法构建该对象。
- 仅支持单个命名参数。
- 暂不支持默认参数值。
- 暂不支持 super()/this() 调用。

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

import interoplib.objc.*

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

import interoplib.objc.*

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
- 函数的参数和返回值类型可为具备与 Objc 的映射关系的基础类型、Mirror 类型或 Impl 类型。
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

import interoplib.objc.*

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

import interoplib.objc.*

@ObjCImpl
class A <: M {
    @ForeignName["gooWithArg0:"]
    public func goo(arg0: IntNative): IntNative{
        M().f + arg0
    }
}
```

具体规格如下：

- 类型可为具备与 Objc 的映射关系的基础类型、Mirror 类型或 Impl 类型。
- 不支持 private/const 成员。
- 支持 static/open 修饰。
- 暂不支持重载。

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

import interoplib.objc.*

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

import interoplib.objc.*

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
- 类型可为具备与 Objc 的映射关系的基础类型、Mirror 类型或 Impl 类型。
- 不支持 static/const 类型变量。

### 继承

Mirror 类支持继承 open Mirror 类。
示例如下：

<!-- compile -->

```cangjie
// M1.cj
package cjworld

import interoplib.objc.*

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

import interoplib.objc.*

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

import interoplib.objc.*

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

- 暂不支持主构造函数。（暂不支持的特性将出现未知错误）
- 暂不支持 String 类型。（暂不支持的特性将出现未知错误）
- 暂不支持类型检查和类型强转。（暂不支持的特性将出现未知错误）
- 暂不支持处理异常。（暂不支持的特性将出现未知错误）
- 不支持普通仓颉类继承 Mirror 类。
- 不支持继承普通仓颉类。
- 当成员函数或构造函数有参数时，必须使用 @ForeignName。

> 注意：
>
> 支持在 Impl 类或普通仓颉类中调用 Mirror 类成员，规格一致。

## ObjCImpl 规格

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

import interoplib.objc.*

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

> 注意：
>
> 暂不支持在仓颉中调用 Impl 类的任意成员。

具体规格如下：

- ObjCImpl 类 的成员函数可有零个、一个或多个参数。
- 函数的参数和返回值类型可为具备与 Objc 的映射关系的基础类型、Mirror 类型或 Impl 类型。
- 构造函数必须显式声明。
- 仅支持单个命名参数。
- 暂不支持默认参数值。
- 暂不支持重载函数。
- 成员函数支持 static/open 修饰。
- 仅有 public 函数可在 Objc 侧被调用。
- 暂不支持 super()/this() 调用。

### 属性

<!-- compile -->

```cangjie
package cjworld

import interoplib.objc.*

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

- 类型可为具备与 Objc 的映射关系的基础类型、Mirror 类型或 Impl 类型。
- 支持 static/open 修饰。
- 暂不支持重载。
- 仅 public 类型可在 Objc 侧被调用。

### 成员变量

<!-- compile -->

```cangjie
// M.cj
package cjworld

import interoplib.objc.*

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
- 类型可为具备与 Objc 的映射关系的基础类型、Mirror 类型或 Impl 类型。
- 仅 public 类型可在 Objc 侧被调用。

### 约束限制

- 暂不支持主构造函数。
- 暂不支持 String 类型。
- 暂不支持类型检查和类型强转。
- 暂不支持在 ObjC 测继承 Impl 类。
- 暂不支持处理异常。
- 当成员函数或构造函数有参数时，必须使用 @ForeignName。（当该函数为重载函数时则不需要，但当前暂不支持重载，因此该规则暂不应用。）
- 不支持普通仓颉类继承 Impl 类。
- 不支持继承普通仓颉类。
- 不支持继承 Impl 类。

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

### @C structs

使用 `@C` 注解的结构体，在 `ObjCPointer<T>` 内部使用时，可以用于 `@ObjCMirror` 和 `@ObjcImpl` 的声明参数、返回类型、
字段和属性。仓颉代码中此类注解的结构体 `X` 对应于 Objective C 代码中的 `struct X` 类型。

约束如下：

- 结构体可以包含原始类型、指针和其他带有 @C 注解的结构体。
- 对于在仓颉或 Objective C 中定义的每个结构体，在对应的另一个语言中都应该有相同的声明。字段及其类型的差异可能会导致运行时错误。
- 结构体应与 `ObjCPointer<T>` 一起使用。例如，`ObjCPointer<MyStruct>`。通过值传递的结构体现在不稳定。
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

### ObjCId

`ObjCId` 类型定义在 `objc.lang` 包中，用于映射 Objective-C 的 `id` 类型。其签名如下：

<!-- compile -->

```cangjie
public interface ObjCId 
```

`ObjCId` 方法的实现在编译器中。

## 约束限制

1. 当前版本的 ObjCInteropGen 功能存在如下约束限制：

    - 不支持 ObjC 类/接口中的非静态成员变量转换
    - 不支持 ObjC 类/接口中的指针属性转换
    - 不支持对构造函数 init 方法的转换
    - 不支持同时转换多个 .h 头文件
    - 不支持 Bit fields 转换

2.  当前版本的 ObjC 互操作方案存在如下约束限制：
     - 不支持 ObjC Mirror 和 Impl 类的实例逃逸出线程范围，包括成为 global variable、static varialble 或 global variable、static varialble 的 filed variable
     - 不支持 ObjC Mirror 和 Impl 类的实例成为 ObjC Mirror 或 ObjC Impl 对象的 filed variable
     - 不支持 ObjC Mirror 和 Impl 类的实例被 lambda 块或 spawn 线程捕获

3. 版本使用过程中需额外下载依赖文件 `Cangjie.h` (下载地址： <https://gitcode.com/Cangjie/cangjie_runtime/blob/dev/runtime/src/Cangjie.h>),并集成至项目中。

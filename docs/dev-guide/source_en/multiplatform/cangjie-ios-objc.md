# Cangjie-ObjC Interoperability

## Introduction

> **Note:**
>
> The Cangjie-ObjC interoperability feature is experimental and still under active development.

The Cangjie SDK supports interoperability between Cangjie and ObjC (Objective-C), enabling the development of business modules in Cangjie programming language for iOS applications. The overall calling process is as follows:

- Scenario 1: Calling Cangjie from ObjC: ObjC code → (glue code) → Cangjie code
- Scenario 2: Calling ObjC from Cangjie: Cangjie code → (glue code) → ObjC code

**Key Concepts:**

1. Mirror Type: A representation of Java types using Cangjie syntax, allowing developers to call ObjC methods in a Cangjie-style manner.
2. CFFI: C Foreign Function Interface, providing C language external interfaces for high-level programming languages like Java/Objective-C/Cangjie.
3. Glue Code: Intermediate code that bridges differences between programming languages.
4. Interoperation Code: Implementation code for ObjC calling Cangjie methods or vice versa.

**Involved Tools:**

1. Cangjie SDK: A collection of developer tools for Cangjie.
2. cjc: Refers to the Cangjie compiler.
3. ObjCInteropGen: ObjC mirror generation tool.
4. iOS Toolchain: The set of tools required for iOS application development.

Interoperation between Cangjie and ObjC typically requires writing "glue code" using the CFFI of either ObjC or Cangjie programming languages. However, manually writing this glue code can be particularly tedious for developers.

The Cangjie compiler (cjc) included in the Cangjie SDK supports automatic generation of necessary glue code to reduce developer burden.

To automatically generate glue code, cjc requires symbol information about the ObjC types (classes and interfaces) involved in cross-language calls. This symbol information is contained within Mirror Types.

**Syntax Mapping:**

|            Cangjie Type                            |                    ObjC Type                       |
|:--------------------------------------------------|:---------------------------------------------------|
|`@ObjCMirror public interface A <: id`             |        `@protocol A`                               |
|  `@ObjCMirror public class A <: id`               |        `@interface A`                              |
|     `@ObjCMirror struct S`                        |        `struct S`                                  |
|    `func fooAndB(a: A, b: B): R`                  |        `- (R)foo:(A)a andB:(B)b`                   |
|  `static func fooAndB(a: A, b: B): R`             |        `+ (R)foo:(A)a andB:(B)b`                   |
|    `prop foo: R`                                  |        `@property(readonly) R foo`                 |
|    `mut prop foo: R`                              |        `@property R foo`                           |
|     `static prop foo: R`                          |        `@property(readonly, class) R foo`          |
|     `static mut prop foo: R`                      |        `@property(class) R foo`                    |
|    `init()`                                       |        `- (instancetype)init`                      |
|    `let x: R`                                     |        `const R x`                                 |
|     `var x: R`                                    |        `R x`                                       |
|`@ObjCMirror public interface test <: NSObject`    |`@protocol test <NSObject>`                         |

**Type Mapping:**

| Cangjie Type         |                ObjC Type      |
|:---------------------|:------------------------------|
|    `Unit`            |        `void`                 |
|     `Int8`           |        `signed char`          |
|    `Int16`           |        `short`                |
|     `Int32`          |        `int`                  |
|     `Int64`          |        `long/NSInteger`       |
|     `Int64`          |        `long long`            |
|     `UInt8`          |        `unsigned char`        |
|    `UInt16`          |        `unsigned short`       |
|    `UInt32`          |        `unsigned int`         |
|    `UInt64`          |   `unsigned long/NSUInteger`  |
|    `UInt64`          |   `unsugned long long`        |
|    `Float32`         |        `float`                |
|   `Float64`          |        `double`               |
|  `Bool`              |        `bool/BOOL`            |

Notes:

1. Types marked as `unavailable` in ObjC source code are not mapped.
2. The current version does not support conversion of global functions and variables in ObjC.
3. Anonymous `C enumeration` types are not mapped.
4. `C unions` types are not mapped.
5. Types with `const`, `volatile`, or `restrict` qualifiers are not mapped.

Taking Cangjie calling ObjC as an example, the overall development process is described as follows:

1. Developers design interfaces, determining function call flows and scopes.

2. Based on step 1, generate Mirror Types for the called ObjC classes and interfaces.

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

    Create a configuration file, such as `example.toml`:

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

    Configuration file format:

    ```toml
    # Optional: Import other toml configuration files. Value is a string array, each line representing a toml file path relative to the current working directory.
    imports = [
        "../path_to_file.toml"
    ]

    # Path for storing generated Mirror Type files. The target directory will be created if it does not exist.
    [output-roots]
    path = "path_to_mirrors"

    # Input file list for Mirror Types, along with Clang compilation options.
    [sources]
    path = "path_to_ObjC_header_file"
    # Compilation options
    arguments-append = [
        "-F", "/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS.sdk/System/Library/Frameworks",
        "-isystem", "/Library/Developer/CommandLineTools/usr/lib/clang/15.0.0/include",
        "-isystem", "/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS.sdk/usr/include",
        "-DTARGET_OS_IPHONE"
    ]

    # Collection of multiple sources properties.
    [sources-mixins]
    sources = [".*"]

    # Specify package names
     [[packages]]
    # If the converted code implicitly uses certain symbols (e.g., Uint32/Bool), these symbols must be added to the filters (otherwise, running ObjCInteropGen may result in errors like "Entity Uint32/Bool from `xx.h` is ambiguous between x packages").
    filters = { include = "Base" }
    package-name = "example"
    ```

    Use the ObjC mirror generation tool to generate ObjC mirror files:

    ```bash
    ObjCInteropGen example.toml
    ```

    The generated mirror file `Base.cj` is as follows:

    ```cangjie
    // Base.cj
    package example

    import interoplib.objc.*

    @ObjCMirror
    open class Base {
        public open func f(): Unit
    }
    ```

    If partial function overrides are needed based on `Base`, the example is as follows:

    ```cangjie
    // Interop.cj
    package example

    import interoplib.objc.*

    @ObjCImpl
    public open class Interop <: Base {
        override public func f(): Unit {
            println("Hello from overridden Cangjie Interop.f()")
            super.f()
        }
    }
    ```

3. Develop interoperation code, using the Mirror Types generated in step 2 to create ObjC objects and call ObjC methods.

    `Interoperation code.cj` + `Mirror Type.cj` → Implement Cangjie calling ObjC.

    For example, the following sample demonstrates calling ObjC type constructors through ObjCImpl type inheritance of Mirror Types:

    ```cangjie
    // A.cj
    package cjworld

    import interoplib.objc.*

    @ObjCImpl
    public class A <: M {
        override public func goo(): Unit {
            println("Hello from overridden A goo()")
            super.goo()
        }
    }
    ```

4. Use cjc to compile the interoperation code and 'Mirror Type.cj'. cjc will generate:

    - Glue code.
    - Actual interoperation ObjC source code.

    `Mirror Type.cj` + `Interoperation code.cj` → cjc → `Cangjie code.so` and `Interoperation.m`/`Interoperation.h`

5. Add the files generated in step 4 to the macOS/iOS project:

    - `Interoperation.m/h`: Files generated by cjc.
    - `Cangjie code.so`: Files generated by cjc.
    - Runtime libraries included in the Cangjie SDK.

    Insert necessary calls in ObjC source code and regenerate the program.

    macOS/iOS project + `Interoperation.m/h` + `Cangjie code.so` → macOS/iOS toolchain → Executable file.

## Environment Preparation

**System Requirements:**

- **Hardware/OS**: macOS 12.0 or later (aarch64).

**Steps to Prepare the ObjCInteropGen Tool Environment:**

1. Install the corresponding version of the Cangjie SDK. Refer to the [Development Guide](https://cangjie-lang.cn/en/docs?url=%2F0.53.13%2Fuser_manual%2Fsource_en%2Ffirst_understanding%2Finstall_Community.html) for installation instructions.
2. Execute the following command to install LLVM 15.

    ```bash
    brew install llvm@15
    ```

3. Add the `llvm@15/lib/` subdirectory to the `DYLD_LIBRARY_PATH` environment variable.

   ```bash
    export DYLD_LIBRARY_PATH=/opt/homebrew/opt/llvm@15/lib:$DYLD_LIBRARY_PATH
    ```

4. Verify successful installation: Execute the following command. If the ObjCInteropGen usage instructions appear, the installation was successful.

    ```bash
    ObjCInteropGen --help
    ```

## ObjCMirror Specifications

ObjCMirror provides a Cangjie syntax representation of ObjC types, automatically generated by tools to allow developers to call ObjC methods in a Cangjie-style manner.

> **Note:**
>
> This feature is still experimental. Users should only use documented features. Using undocumented features may result in unknown errors such as compiler crashes.
> Unsupported features will cause unknown errors.

### Constructors

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
    // Constructors with parameters must be decorated with @ForeignName.
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
        super(0, 0.0) // or super()
        println("arg2 is ${arg2}")
    }
}
```

Current specifications:

- ObjCMirror class constructors can have zero, one, or multiple parameters.
- Constructor parameter and return types can be basic types with ObjC mapping relationships, Mirror types, or Impl types.
- When explicitly declared, constructors cannot be private/static/const types. If no constructor is explicitly declared, the object cannot be constructed.
- Only single named parameters are supported.
- Default parameter values are not currently supported.

### Member Functions

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
    public override goo(arg0: Int16, arg1: Float32): Float64 {
        super.goo(arg0, arg1) + 1
        // we could also call M.boo(...)
    }
}
```

Specifications:

- ObjCMirror class member functions can have zero, one, or multiple parameters.
- Function parameter and return types can be basic types with ObjC mapping relationships, Mirror types, or Impl types.
- Only single named parameters are supported.
- Adding new member functions to Mirror classes is not supported; runtime exceptions will occur.
- static/open types are supported.
- private/const members are not supported.
- Default parameter values are not currently supported.
- Function overloading is not currently supported.

### Properties

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
    // we could override f
    // protected override mut prop f: IntNative { get() { super.f + 1 } }
    @ForeignName["gooWithArg0:"]
    public goo(arg0: IntNative): IntNative{
        this.f + arg0
    }
}
```

Specifications:

- Types can be basic types with ObjC mapping relationships, Mirror types, or Impl types.
- private/const members are not supported.
- static/open modifiers are supported.
- Overloading is not currently supported.

### Member Variables

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
        super()
        this.m = m
    }
    @ForeignName["gooWithArg0:"]
    public goo(arg0: Float64): Float64 {
        this.m + arg0
    }
}
```

Specifications:

- Mutable variables (var) and immutable variables (let) are supported.
- Types can be basic types with ObjC mapping relationships, Mirror types, or Impl types.
- static/const type variables are not supported.

### Inheritance

Mirror classes support inheriting from open Mirror classes.
Example:

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
    public func goo(): Int64
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
        super()

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

### Constraints and Limitations

- Primary constructors are not currently supported. (Unsupported features may cause undefined errors)
- The `String` type is not currently supported. (Unsupported features may cause undefined errors)
- Type checking and type casting are not currently supported. (Unsupported features may cause undefined errors)
- Exception handling is not currently supported. (Unsupported features may cause undefined errors)
- Regular Cangjie classes cannot inherit from Mirror classes.
- Regular Cangjie classes cannot be inherited.
- When member functions or constructors have parameters, the `@ForeignName` annotation must be used.

> Note:
>
> Calling Mirror class members from Impl classes or regular Cangjie classes is supported, with consistent specifications.

## ObjCImpl Specification

ObjCImpl is a Cangjie annotation, indicating that the methods and members of the class can be invoked by ObjC functions. During compilation, the compiler generates corresponding ObjC code for Cangjie classes marked with `@ObjCImpl`.

> **Note:**
>
> This feature is currently experimental. Users should only use features explicitly documented. Utilizing undocumented features may lead to unknown errors such as compiler crashes.
> Unsupported features will result in undefined behavior.

### Invoking Constructors or Member Functions of ObjCImpl

To invoke constructors or member functions of ObjCImpl in ObjC:

<!-- compile -->

```cangjie
// A.cj
package cjworld

import interoplib.objc.*

@ObjCImpl
class A <: M {
    @ForeignName["initWithM:"]
    public init(m: Float64) {
        super()
        this.m = m
    }
    @ForeignName["gooWithArg0:"]
    public goo(arg0: Float64): Float64 {
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

> Note:
>
> Currently, invoking any member of an Impl class from a regular Cangjie class is not supported.

Specific specifications:

- Member functions of an ObjCImpl class may have zero, one, or multiple parameters.
- Parameter and return types can be primitive types with ObjC mapping, Mirror types, or Impl types.
- Constructors must be explicitly declared.
- Only single named parameters are supported.
- Default parameter values are not yet supported.
- Function overloading is not yet supported.
- Member functions can be marked with `static` or `open`.
- Only `public` functions can be invoked from the ObjC side.

### Properties

<!-- compile -->

```cangjie
package cjworld

import interoplib.objc.*

@ObjCMirror
class M1 {
    public prop answer: Float64

    @ForeignName["initWithAnswer:"]
    public init(answer: Float64)
}

@ObjCMirror
open class M {
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
        super()
        println("cj: A.init()")
        _b = M1(123.123)
        bar = M1(d)
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

Specific specifications:

- Types can be primitive types with ObjC mapping, Mirror types, or Impl types.
- Supports `static` and `open` modifiers.
- Overloading is not yet supported.
- Only `public` properties can be accessed from the ObjC side.

### Member Variables

<!-- compile -->

```cangjie
// M.cj
package cjworld

import interoplib.objc.*

@ObjCMirror
open class M {
    @ForeignName["init"]
    public init()
}
```

<!-- compile -->

```cangjie
// M1.cj
package cjworld

import interoplib.objc.*

@ObjCMirror
open class M1 {
    @ForeignName["init"]
    public init()
    @ForeignName["foo"]
    public foo(): Float64
}
```

<!-- compile -->

```cangjie
// A.cj
package cjworld

import interoplib.objc.*

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

Specific specifications:

- Supports mutable variables (`var`) and immutable variables (`let`).
- Types can be primitive types with ObjC mapping, Mirror types, or Impl types.
- Only `public` member variables can be accessed from the ObjC side.

### Constraints and Limitations

- Primary constructors are not yet supported.
- `String` type is not yet supported.
- Type checking and type casting are not yet supported.
- Inheriting from Impl classes in ObjC is not yet supported.
- Exception handling is not yet supported.
- When member functions or constructors have parameters, `@ForeignName` must be used. (This rule does not apply to overloaded functions, but overloading is currently unsupported.)
- Regular Cangjie classes cannot inherit from Impl classes.
- Inheriting from regular Cangjie classes is not supported.
- Inheriting from Impl classes is not supported.

## Constraints and Limitations

1. The current version of ObjCInteropGen has the following constraints:

    - Non-static member variables in ObjC classes/interfaces cannot be converted.
    - Pointer properties in ObjC classes/interfaces cannot be converted.
    - Constructor `init` methods cannot be converted.
    - Converting multiple `.h` header files simultaneously is not supported.
    - Bit fields cannot be converted.

2. Additional dependency file `cangjie.h` must be downloaded (available at <https://gitcode.com/Cangjie/cangjie_runtime/blob/dev/runtime/src/Cangjie.h>) and integrated into the project.

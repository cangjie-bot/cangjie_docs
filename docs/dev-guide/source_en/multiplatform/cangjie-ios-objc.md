# Cangjie Interoperability with Objective-C

**NOTE:** The Objective-C interoperability feature is experimental and still
under continuous improvement.


## Introduction

The **Cangjie Multiplatform** technology enables Cangjie developers
to incorporate their code into new Android/iOS applications and/or gradually
replace the Java/Objective-C parts of existing applications with Cangjie code.
The central mechanism that enables the cross-language, cross-runtime
interoperability is _[mirror types](#mirror-types)_, which expose the types
of one language to the other.

On the Cangjie side, mirror types enable the inheritance of Java/Objective-C
classes with method overriding, as well as the implementation of Java
interfaces and Objective-C protocols, all while using conventional Cangjie
syntax. And those Cangjie classes that extend and implement the mirrored classes
and interfaces/protocols are exposed back to Java/Objective-C as if they were
classes of the respective platform-native language. All in all, this enables
seamless transitions between the Java/Objective-C and Cangjie parts
of the application, which includes usage of the target O/S APIs in Cangjie code.


### The Challenge and the Solution

Java, Objective-C and Cangjie are object-oriented languages that support
inheritance and polymorphism. However, the differences in their semantics,
object models, and execution models preclude direct usage of Cangjie
objects in Java or Objective-C code and vice versa.

Then, all three languages have managed runtimes that support automatic memory
management, threading, exception handling and other low-level features,
but again they do that differently. Making two separately designed complex
language runtimes aware of each other might push the complexity of the entire
system beyond human comprehension.

Instead, the interoperability between Cangjie and Objective-C is achieved
by disguising them both as the lower-level C language. The Objective-C Runtime
module API is designed specifically to enable the creation of bridge layers
between Objective-C and other languages. It provides a rich, but low-level
C API, and writing the bridge code manually is known to be cumbersome.
Fortunately, CJMP automates all that complexity away.

Similarly, on Android the Cangjie and Java parts of the application see each
other through the lens of the Java Native Interface (JNI), originally designed
to facilitate the development of Java `native` methods in languages such
as C and C++. And just like with the Objective-C Runtime, CJMP takes care
about the cumbersome parts.


## Key Concepts

### Mirror Types

Considering Cangjie and Objective-C as a pair of interoperating languages,
a _mirror type_ _`T'`_ defined in Cangjie is a type that represents an existing
Objective-C type _`T`_, enabling the code written in the first language to use
that type, possibly with some limitations.

The Boolean types and numeric types that are essentially the same in both
languages naturally mirror each other: the Cangjie mirror type for the
Objective-C type `int` is `Int32` and so on. Mirroring of numeric types
that Cangjie does not naturally support is not currently possible,
so e.g. the Objective-C type `int128_t` cannot be mirrored to a Cangjie
primitive type.

For a user-defined type such as a class or protocol, the mirror type would be
its closest equivalent in the other language. For instance, Objective-C
protocols are quite similar to Cangjie interfaces.

A mirror type exposes the members and constructors of the original user-defined
type that are accessible _and_ can be used in the other language, so again
an Objective-C method that returns a value of type `int128_t` cannot be
mirrored to Cangjie.

Normally you would obtain mirror type definitions for Objective-C types that you
want to use in Cangjie automatically using the standalone
[mirror generator](#objective-c-mirror-generator-reference) included in the
Cangjie SDK for iOS.


#### Mirroring Objective-C Types to Cangjie

The `cjc` compiler replaces any uses of Objective-C mirror types with the
appropriate bridge code, so only the names of types themselves and names and
types of their accessible members matter. Therefore, mirrors of Objective-C
types only contain member declarations, not definitions: constructors and
member functions/properties have no bodies and member variables have no
initializers. For the same reason, `@private` members are omitted.

For such an extension of the regular Cangjie syntax to work, each mirror type
must be marked with an `@ObjCMirror` annotation. It helps the compiler
distinguish between mirror type declarations and normal Cangjie type
definitions.

For example, a mirror class for the following Objective-C class:

```objectivec
@interface Node : NSObject {
}
- (id)initWith:(int)x;
- (int)getX;
@end
```

might look like this:

```cangjie
@ObjCMirror
public open class Node <: NSObject {
    @ForeignName["initWith:"]
    public init(x: Int32)
    public open func getX(): Int32
}
```


### Mirror Functions

Both Objective-C and Cangjie support top-level, global functions that are not
members of any other type, so such Objective-C functions are exposed to Cangjie
as _mirror functions_, which are essentially automatically generated pieces
of bridge code that pass control and data through the inter-language barrier.


### Interop Classes

An _interop class_ is essentially a Cangjie class that is derived from one
or more [mirror types](#mirror-types) and is usable from Objective-C. All its
constructors and non-inherited `public` member functions are exposed
to Objective-C code via a _wrapper class_, automatically generated by the `cjc`
compiler. The wrapper class itself defines no other user-callable methods
or constructors, but any methods it may have inherited from its supertypes
may be called from both Objective-C and Cangjie code.

For example, when compiling the following Objective-C interop class:

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

the `cjc` compiler will also yield a pair of Objective-C source code files
similar to the following:

```objectivec
// BooleanNode.h
@interface BooleanNode : Node
/* glue code */
- (id)initWithX:(int32_t)x AndFlag:(BOOL)flag;
- (BOOL)isFlagged;
/* more glue code */
@end
```

```objectivec
// BooleanNode.m
@implementation BooleanNode : Node
/* glue code */
- (id)initWithX:(int32_t)x AndFlag:(BOOL)flag {
    /* Bridge code constructing a Cangjie BooleanNode(x, flag) instance and
     * associating it with the Objective-C instance being constructed,
     * i.e. 'self'.
     */
}
- (BOOL)isFlagged {
    /* Bridge code invoking the 'isFlagged' member function of the Cangjie
     * BooleanNode instance associated with 'self' and returning the result.
     */
}
/* more glue code */
@end
```

Now both Objective-C and Cangjie parts of the application code can instantiate
the interop class `BooleanNode`, call its `public` instance member functions
`getX()` (inherited from the Objective-C class `Node`) and `isFlagged()`,
convert its instances respectively to the type `Node` and its mirror, etc.


### Foreign Types

The [mirror types](#mirror-types) and [interop classes](#interop-classes) are
not perfectly native to the language in which they are defined. Extending the
analogy, their status is more like temporary visitors and work visa holders
respectively, so they are collectively called _foreign types_ throughout
this document.


### Objective-C-Compatible Types

The following Cangjie types are called _Objective-C-compatible_:

* Value types that have direct equivalents among Objective-C primitive types
  (`Int16` is included, but `Float16` is not);
* [Foreign types](#foreign-types);
* Types of the form `Option<`_`T`_`>` where _`T`_ is a [foreign type](#foreign-types)
  (see _[`nil` Handling](#nil-handling)_ for reasoning);
* [Built-in `ObjC*`-types](#built-in-types)
  `ObjCPointer<T>`, `ObjCFunc<F>`, `ObjCBlock<F>`;
* `@C` structs;
* `CFunc<F>` function types;
* `CString`; and
* `Unit` (as a function return value type only).

**IMPORTANT:** As of the current version, the types `CPointer<T>`
and `VArray<T,$N>` are _not_ Objective-C-compatible types themselves, but that
does not impose any restrictions on their use in `@C` structs or `CFunc<F>`
function types.


## Using Objective-C in Cangjie

To enable interoperation between the Objective-C and Cangjie parts of your iOS
application, you need to take the following steps:

1. Design the interoperability layer in terms of Objective-C classes
   and methods.

    developer → `interop layer design` (Objective-C _pseudo-code_)

2. Generate mirror declarations for any Objective-C classes and protocols used
   in the design of the interoperability layer.

    `.h` → mirror generator → `.cj` (mirrors)

3. Write the classes constituting the interoperability layer _in Cangjie_.
   Use the mirrored Objective-C types as needed: create instances, call member
   functions, etc.

    `interop layer design` + `.cj` (mirrors) → developer → `.cj` (interop layer)

4. Compile the interoperability layer classes (_interop classes_ for short)
   and mirror types together with `cjc`. `cjc` will generate:

    * the necessary glue code for all uses of the mirror types.
    * the actual Objective-C source for the Objective-C part
      of the interoperability layer
      (so called Objective-C _wrappers_ for interop classes)

   `.cj` (mirrors + interop layer) → `cjc`  → `.dylib` + `.h`/`.m` (interop layer)

5. Add to your iOS project:

    * `.h` and `.m` files generated by `cjc`
    * `.dylib` file generated by `cjc`
    * runtime libraries (`.dylib`) from the Cangjie SDK

    iOS project + `.h`, `.m`, `.dylib` → iOS toolchain → iOS application

6. Now you may start using the interop class wrappers in the Objective-C code
   of your application, effectively calling Cangjie code from Objective-C.


### Initial Interop Class Creation Workflow

#### Step 1: Design the Interoperability Layer {#step-1}

On this step, you design the API of one or more interop classes
_from the Objective-C code perspective_, i.e. you need to decide, for each
interop class:

* What Objective-C class it will extend, e.g. `NSObject`;
* Which Objective-C protocols it will implement, if any; and
* What `public`/`protected` methods and/or constructors it will have.
  At this step, you only need to know the types of parameters and, for methods,
  their names and return value types; the implementations you will write
  in Cangjie later.

See also
[Features and Limitations of Interop Classes](#features-and-limitations-of-interop-classes).

For the Objective-C part of the application, the interoperability layer is
indistinguishable from a set of regular Objective-C classes. So you can design
it in the form of Objective-C pseudo-code. Write down the specifications
and `@interface` definitions of one or more Objective-C classes. **NOTE:**
You do not need to write the respective `@implementation` parts as you would
re-write them in Cangjie at a [later step](#step-3) anyway.

**Supported parameter types:** any [mapped](#objective-c-to-cangjie-mapping)
Objective-C types.

**Supported return types:** any mapped Objective-C type or `void`.

**Supported inheritance:** interop classes may only extend mirrored Objective-C
classes (_not_ other interop classes!), and may only implement interfaces that
mirror Objective-C protocols.

**Limitations:**

* Variable arity (varargs) methods and constructors are not supported.

* Neither an interop class nor its individual non-private member functions
  may be generic.

* Generic Objective-C types are erased before mirror generation;
  any type variables are replaced with the respective upper bounds.

**End-to-end Example:**

For the sake of simplicity, suppose that in your iOS application there is
already a class `M` with a single parameterless `void` instance method
`foo()`:

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
        // Some initialization work
    }
    return self;
}
- (void) foo {
    printf("Hello from Objective-C M.foo()\n");
}
@end
```

and you want to override that method `foo()` with a Cangjie implementation
in a subclass of `M` called `A`.

Your interop class design would then look like this:

```objectivec
#import "M.h"
@interface A : M
- (instancetype)init;
- (void)foo;
@end
```


#### Step 2: Generate Mirror Type Declarations {#step-2}

Now, you need [mirror type](#mirror-types) declarations for all Objective-C
types on which your interop classes depend: their supertypes, types of member
properties and local variables, types of method parameters/return values,
and, if any of the foregoing are array types, their element types.

Before you proceed further, build your iOS application normally without
changing anything in it, to ensure that the mirror generator takes
a complete and consistent set of Objective-C headers as input.

Then write an appropriate
[mirror generator configuration file](#configuration-file-syntax) and
run the mirror generator:

```bash
ObjCInteropGen <config-file>
```

where `<config-file>` is the pathname of the configuration file.

**End-to-end Example (continued):**

The only immediate dependency of your class `A` is its superclass `M`,
so the configuration file is pretty simple:

```toml
# A.toml
# Place the mirror of M and any dependencies it may have in the 'objcworld' package:
[[packages]]
filters = { include = ["M", "NS.+"] }
package-name = "objcworld"

# Write the output files with mirror type definitions to the current directory:
[output-roots.default]
path = "."

# Specify the pathname of the input header:
[sources.all]
paths = ["M.h"]

[sources-mixins.default]
sources = [".*"]
arguments-append = [
    # Uncomment the following line if you get "unknown type name" errors
    # "-DTARGET_OS_IPHONE=1",

    # Edit the pathnames below to match the locations of Objective-C headers on your system:
    "-F", "/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/System/Library/Frameworks",
    "-isystem", "/Library/Developer/CommandLineTools/usr/lib/clang/17/include",
    "-isystem", "/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/include"
]
```

Mirror generator command line:

```bash
ObjCInteropGen A.toml
```

The above command will generate files `objcworld/M.cj` with a mirror type
declaration for the class `M` and a bunch of `objcworld/NS*.cj` files
with mirror type declarations for the Foundation framework classes
and protocols on which `M` depends.


##### Common Misconfiguration Issues

**The mirror generator is unable to find the standard header files, such as
`stdarg.h` or `stdbool.h`.**

Example error message:

```text
..../CoreFoundation.h:19:10: error: 'stdarg.h' file not found
```

This usually means that the pathnames in `arguments-append` array of the
`[sources-mixins]` table are not correct. Double check that they match the
locations of the header files on your system.

**The mirror generator reports "Unknown type name 'NSUInteger'" errors or similar.**

Example error message:

```text
.../NSObjCRuntime.h:626:74: error: unknown type name 'NSUInteger'
```

On some systems, you need to pass "`-DTARGET_OS_IPHONE=1`" as an additional
argument to Clang.

Add it to the `arguments-append` array of the `[sources-mixins]` table.
In the above example, it is present but commented out:

```toml
#   .  .  .
arguments-append = [
    # Uncomment the following line if you get "unknown type name" errors
    # "-DTARGET_OS_IPHONE=1",
#   .  .  .
```


#### Step 3: Write the Interop Classes {#step-3}

For each Objective-C class skeleton that you included in your interop layer
design on [Step 1](#step-1), write a matching Cangjie class as follows:

* Use the appropriate package and class names (the Objective-C wrapper class
  that `cjc` will generate for you will have the same fully qualified name).
* Import `objc.lang.*`.
* Import the mirror types, if you've generated any on [Step 2](#step-2).
  Do not import all generated dependencies, only the types you actually need.
* Annotate the interop class with `@ObjCImpl`.
* Make the interop class inherit the respective `open` mirror class,
  or `NSObject` if it has no explicit supertype in your design.
* Annotate `public` class members and constructors
  with `@ForeignName["`_`foreign-name`_`"]`, where _`foreign-name`_ is the
  desired Objective-C name for that member or constructor, according
  to the following rules:

  1. _Do not_ annotate member functions that override member functions of
     mirror supertypes. The _presence_ of a `@ForeignName` annotation
     on such a member function of an interop class triggers a compile-time
     error.

     > The Objective-C names of such overriding functions must match
     > the names of the overridden Objective-C methods. The latter are
     > propagated _from_ Objective-C to Cangjie automatically
     > via `@ForeignName` annotations in mirror supertype declarations
     > (see [Classes and Protocols](#classes-and-protocols) for details).
     > The `cjc` compiler picks those annotations up.

  2. _Do_ annotate constructors and non-overriding member functions that have
     _two or more_ parameters. The _absence_ of a `@ForeignName` annotation
     on such a member function or constructor triggers a compile-time error.

     > The compiler is unable to morph the original Cangjie name
     > of a member function or constructor with two or more parameters
     > into a valid Objective-C name automatically. Such names essentially
     > consist of the respective number of fragments separated with the colon
     > character `:`.

  3. Overloaded member functions and constructors _must_ be given distinctive
     Objective-C names using `@ForeignName`, as there is no method overloading
     in Objective-C. Exactly one of several overloaded entities can retain
     its original Cangjie name, provided it has at most one parameter.

  4. Annotating the remaining members and constructors with `@ForeignName`
     is optional. In the absence of a `@ForeignName` annotation,
     the original Cangjie name of such a member is used as the
     Objective-C name of the corresponding entity, with colon `:` appended
     to names of single-parameter methods, whereas the `init`-method
     matching the sole `public` constructor with one or zero parameters
     is called respectively `init:` or `init`.

  **NOTE:** The current version of the `cjc` compiler  does _not_ fully
  validate _`foreign-name`_. In particular, it does not check that the number
  of colon `:` separators matches the number of member function or constructor
  parameters.

* Use the following Objective-C-to-Cangjie type mapping (`T'` is either the
  equivalent value type or the respective mirror type):

    Objective-C            | Cangjie       | Remark
    ---------------------- | ------------  | ------
    `void`                 | `Unit`        |
    `BOOL`                 | `Bool`        |
    `signed char`          | `Int8`        |
    `short`                | `Int16`       |
    `int`                  | `Int32`       |
    `long`                 | `Int64`       |
    `long long`            | `Int64`       |
    `unsigned char`        | `UInt8`       |
    `unsigned short`       | `UInt16`      |
    `unsigned int`         | `UInt32`      |
    `unsigned long`        | `UInt64`      |
    `unsigned long long`   | `UInt64`      |
    `float`                | `Float32`     |
    `double`               | `Float64`     |
    `struct`               | `@C struct'`  | (\*)
    `enum` w/base type `T` | `T'`          |
    `id`                   | `ObjCId`      | (†)

    Objective-C  | Cangjie        | If `T` is...                              | Remark
    ------------ | ------------   | ----------------------------------------- | ------
    `T*`         | `CPointer<T'>` | ... a primitive or structure type         |
    `T*`         | `CPointer<U'>` | ... an enum type and `U` is its base type |
    `T*`         | `CFunc<T'>`    | ... a pure C function type                |
    `T*`         | `T'`           | ... a class type                          | (†)

    (\*) The Objective-C structure must not contain fields of types that
    are not `CType`-compatible. See [Structs](#structs) for details.

    (†) Use `?<T'>` (`Option<T'>`) types for parameters, return values,
    and local variables of mirror types and interop class types that may
    receive/hold the Objective-C `nil` value.

    See [Objective-C to Cangjie Mapping](#objective-c-to-cangjie-mapping)
    for details.

**Supported features:**

Member functions of interop classes:

* Can override/redefine `open` instance/static member functions
  of their mirrored superclasses, implement member functions
  of their mirrored superinterfaces, and be newly introduced member functions.

Constructors and member functions of interop classes:

* Can use any Cangjie language features in the body when working solely
  with (values of) regular Cangjie types.

* Can use conventional Cangjie syntax to:

    - Instantiate objects of interop classes and mirrored Objective-C classes;
    - Call static and instance member functions of interop classes and mirrored
      Objective-C types (that includes using `super` to call Objective-C
      superconstructors and superclass methods); and
    - Access their own member properties and non-`private` member properties
      of other interop classes and mirrored Objective-C types

**Limitations:**

* Interop classes may implement interfaces that are mirrored Objective-C
  protocols, but not regular Cangjie interfaces.

* Interop classes may not be declared as `open` or `abstract` and may not be
  extended using `extend`.

* Non-`private` member functions and constructors of interop classes:

    - May only use the Objective-C-mapped types listed above as parameter
      and return value types. (As a consequence, variable-length arguments
      are not supported as that requires using the Cangjie `Array<T>` type,
      which has no Objective-C mapping.)

    - May not have named parameters.

    - May not have type parameters, i.e. be generic.

* Non-`private` member properties may only have the Objective-C-mapped types
  listed above.


* Generic Objective-C types are erased; mirror types have upper bounds in place
  of the respective type parameters.

* **IMPORTANT:** Instances of mirror types and interop classes
  (in other words, values of Objective-C reference types) must not escape
  to global or static Cangjie variables or any data structures that persist
  between calls.

**End-to-end Example (continued):**

To continue the above example, the interop class `A` would look
like this:

```cangjie
package objcworld         // Same package name for simplicity

import objc.lang.*        // Always required

@ObjCImpl
public class A <: M {
    override public open func foo(): Unit {
        println("Hello from overridden A.foo()")
    }
}
```

**NOTE:** The default constructor of `A` calls `super()`, which is equivalent
to calling `[super init]` in Objective-C terms, albeit with the assumption that
it will return an instance of a suitable class.


#### Step 4: Compile the Interop Classes {#step-4}

Command line:

```bash
cjc --target=arm64-apple-ios-simulator \
    --sysroot=$(xcrun --show-sdk-path --sdk iphonesimulator) \
    --output-type=dylib \
    --int-overflow=wrapping \
    <source-files> \
    -o <target-file> \
    --link-options "-undefined dynamic_lookup"
```

where

`<source-files>` are the source code files of the interop classes and
mirror type declarations.

`<target-file>` is the desired name of the output `.dylib` file with
compiled Cangjie code for the interop classes, such as `libobjcworld.dylib`.

The compiler will also generate Objective-C source files (`.h` and `.m`)
for the Objective-C wrappers of the interop classes, placing them in the
`./objc-gen/` subdirectory.

The `.dylib` file has to be signed:

```bash
xcrun codesign --sign - <dylib-file>
```

**End-to-end example (continued):**

Compile the interop class:

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
The compiler will produce three files: `./libobjcworld.dylib`,
`./objc-gen/A.h` and `./objc-gen/A.m`.

Sign the generated dynamic library:

```bash
xcrun codesign --sign - libobjcworld.dylib
```


#### Step 5: Put It All Together {#step-5}

**IMPORTANT: The current version of the mirror generator does _not_ propagate
names of the original header files into mirror type declarations. As a result,
the wrapper classes generated on the previous step will import non-existing
headers and won't compile. A temporary workaround is given out below.**

For example, if any member functions of your interop class accept parameters
and/or return values of the type `UIDevice` from UIKit, the source code file
containing the generated wrapper class will include a line

```objectivec
#import "UIDevice.h"
```

instead of

```objectivec
#import <UIKit/UIKit.h>
```
or

```objectivec
#import <UIKit/UIDevice.h>
```

**NOTE:** The Foundation framework is an exception, as all Objective-C files
generated by `cjc` always contain these two lines at the top:

```objectivec
#import <Foundation/Foundation.h>
#import <stddef.h>
```

As a temporary workaround, you need to manually create the `.h` files for
such mirror types, putting the right `#import` declarations into them, and
add those files to your Xcode project before proceeding.

This inconvenience will be eliminated in future versions.

Now do the following:

* Create another subdirectory in your project and copy all dynamic libraries
  from the `$CANGJIE_HOME/runtime/lib/ios_simulator_aarch64_cjnative/`
  directory into it.

* Add all those dynamic libraries _and_ the `.dylib` file generated
  on the [previous step](#step-4) to your Xcode project as dependencies
  (BuildPhases - in both “Copy Files” and “Link Binary With Libraries” lists).

* Move the `.h` and `.m` files generated on the [previous step](#step-4)
  to project root.

Then re-build the project.


**End-to-end Example (continued):**

Create a new subdirectory in the root directory of your project and copy
the dynamic libraries constituting the Cangjie Runtime for iOS into it:

```bash
cd ..
mkdir -p CJRuntimeDylibs
cp $CANGJIE_HOME/runtime/lib/ios_simulator_aarch64_cjnative/*.dylib CJRuntimeDylibs/
```

Add those dynamic libraries _and_ `./objcworld/libobjcworld.dylib` to your Xcode
project as dependencies (BuildPhases - in both “Copy Files” and “Link Binary
With Libraries” lists).

Move the `.h` and `.m` files generated by `cjc` to project root:

```shell
mv objcworld/objc-gen/*.h ./
mv objcworld/objc-gen/*.m ./
```

Rebuild the Xcode project.


### Calling Objective-C from Cangjie

Once you have designed, built and integrated the interoperability layer
as described
in the [previous section](#initial-interop-class-creation-workflow),
you can add code that uses Objective-C types to the member functions
of your interop classes. The type mapping is the same.

Add the code invoking a Cangjie function from Objective-C via the respective
method of an interop class wrapper and re-build your iOS project again.

**End-to-end Example (continued):**

Insert an instantiation of `A` and a call of the method `foo()` of that
instance into your Objective-C code:

```objectivec
       .  .  .
#import "M.h"
#import "A.h"
       .  .  .
    M* a = [[A alloc] init];
    [a foo];
       .  .  .
```

Rebuild the Xcode project and run your app on the iOS simulator.


### Subsequent Enhancements

Following [the above process](#initial-interop-class-creation-workflow) should
have helped you develop a good understanding of how to add more classes
to the interoperability layer and/or enable the use of more Objective-C
classes in its Cangjie code. Below please find a brief summary with links
to the respective subsections:

**[Step 1](#step-1):** Add more interop classes to your design and/or enhance
the existing ones with new methods and/or properties as you wish.

**[Step 2](#step-2):** If any additional, not yet mirrored Objective-C
reference types will now get involved, build the _current_ version
of the application to ensure consistency, edit the mirror generator
configuration file appropriately, and then (re-)generate all mirror
type declarations.

**[Step 3](#step-3):** Implement the newly added interop classes, if any,
and/or change the code of existing ones, using the mirrored
Objective-C types as necessary.

**[Step 4](#step-4):** Compile all interop classes together.

**[Step 5](#step-5):** Copy the `.h`, `.m` and `.dylib` files that `cjc` has
generated on the previous step to their respective locations in the project
and re-build it.

Now you can start [using](#calling-objective-c-from-cangjie) the newly added interop classes and/or methods
in the Objective-C part of your application.


### Features and Limitations of Interop Classes

1. An interop class _must_ be a direct subclass of a mirror class.

2. An interop class _may_ implement one or more mirror interfaces, but never
   a conventional Cangjie interface. Conversely, a conventional Cangjie type
   may not implement or inherit an interface mirroring an Objective-C protocol.

3. An interop class may not be declared as `open` or `abstract` and may not be
   extended using `extend`, and may not be generic.

4. An interop class may introduce new instance fields _of any Cangjie type_
   (as they are not exposed to Objective-C)
   and override the member functions of its mirrored superclass.


5. The constructors of an interop class may call superclass constructors
   using `super()`, but have the same limitation on instance member function
   calls as the conventional Cangjie constructors, as well as the requirement
   to initialize all newly introduced member variables.

6. The instance member functions of an interop class may call instance
   member functions which that interop class has inherited from its mirrored
   superclass, and/or use `super.` to call the member functions of that
   superclass that the interop class overrides.

7. The signatures of constructors and member functions of all mirror types
   and interop classes can only use
   [Objective-C compatible types](#objective-c-compatible-types)
   See the Type Mapping table in [Step 3](#step-3) above and the Chapter
   [Objective-C to Cangjie Mapping](#objective-c-to-cangjie-mapping).


## Objective-C to Cangjie Mapping

The current version of the mirror generator does the following Objective-C
→ Cangjie conversions.


### General Considerations

The Objective-C mirror generator relies on Clang for parsing Objective-C
sources, invoking it with the `-fobjc-arc` option.

Declarations marked with the `unavailable` attribute are ignored.

Global, file-level function and variable declarations are also ignored.

The order of declarations in the output Cangjie source code is the same
as the order of the respective definitions in the input Objective-C source
code, with the exception of nested type definitions.


### Names

The original Objective-C identifiers are preserved, with the following
exceptions:

* Objective-C identifiers that clash with Cangjie keywords, such as `catch`,
  `false`, or `UInt32`, are enclosed in backticks ` `` `.

* Name conflicts between Objective-C classes and protocols are resolved
  by adding the suffix `Protocol` to the name of the protocol.

* Name conflicts between instance and static methods are resolved,
  if possible, by adding either `Instance` or `Static` suffix to the highest
  (closest to the root) class/protocol where the conflict is encountered.
  If the conflicting instance/static methods are first introduced
  in the same class/protocol, the static method is renamed. For example:

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

    will be mirrored into

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

* Conflicts between `init` methods that have the same number and types
  of parameters are resolved, but with a limitation. See [Classes](#classes)
  for details.


### Type Aliases

C typedef declarations are mirrored into public Cangjie type aliases.


### Primitive Types

Objective-C primitive types map to the respective Cangjie value types.
C types with platform-specific sizes map to Cangjie in accordance with their
actual sizes on the _host_ platform, i.e. the platform on which the mirror
generator runs. For example, on macOS it can be the following:

| Objective-C          | Cangjie   |
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


### Strings

The Cangjie `String` type and the Objective-C `NSString` type are not binary
compatible. They even encode character data differently (in UTF-8 and UTF-16
respectively).

To facilitate string conversions, the Cangjie compiler implicitly extends
mirrors of the Foundation Framework classes `NSObject` and `NSString`
as follows:

* A mirror of the `NSObject` class has an implicitly defined instance member
  function `toString()`:

  ```cangjie
      public open func toString(): String
  ```

  That function calls the `description` method of its receiver, converts the
  result into a Cangjie `String` and returns it.

* A mirror of the `NSString` class has an implicitly defined constructor taking
  a Cangjie `String` as an argument:

  ```cangjie
      public init(s: String)
  ```

  It initializes the `NSString` instance being constructed with transcoded
  character data of its argument.

**NOTE:** It does not matter how those mirror classes themselves are called.
The values of their `@ObjCMirror` annotations must be respectively `"NSObject"`
and `"NSString"` for the compiler to insert the implicit declarations described
above:

  ```cangjie
  @ObjCMirror["NSObject"]
  public class ObjC_Object {    // toString() added implicitly
      //   .  .  .
  }
  ```

  ```cangjie
  @ObjCMirror["NSObjectWrapper"]
  public class NSObject {       // toString() NOT added implicitly
      //   .  .  .
  }
  ```


### Structs

A C struct is mirrored into a public Cangjie struct marked as `@C` if it is
`CType`-compatible, that is, contains fields of `CType`-compatible types only.
Other scenarios, e.g. the original Objective-C structure containing pointers
to objects, are not currently supported.


Nested structures are mirrored into top-level Cangjie structs.

Incomplete structure declarations are mirrored into empty structs
(without any members).

Fields of Objective-C structures are mirrored into public member variables
of the respective Cangjie structs. The `static` modifier is maintained.

Bit fields are not supported by the Cangjie language,
so their widths are ignored and a warning is issued.

The Cangjie language, unlike Objective-C, requires all member variables
of structs (even `@C` ones) to be initialized. Therefore, mirrored structures
contain explicit zero initializers for all fields. For example:

```c
struct A {
    int x;
    double y;
    BOOL z;
    struct A *w;
};
```

is mirrored to

```cangjie
@C
public struct A {
    public var x: Int32 = 0
    public var y: Float64 = 0.0
    public var z: Bool = false
    public var w: CPointer<A> = CPointer<A>()
}
```


### Enumerations

C enumeration declarations are mirrored into series of top-level `public`
`const` variable declarations. The name of each `const` variable matches the
name of an enum constant, and its initializer is the respective value.
Additionally, a type alias is declared for the Cangjie value type that mirrors
the underlying type of the original enumeration, whether the latter is explicitly
specified or implied. The alias equals the name of the original enumeration,
unless the latter is anonymous, in which case a unique name is synthesized.
Any _usages_ of the enum are then mirrored into that type alias.


**WARNING:** As the enum name becomes an alias for an integral Cangjie type,
its set of values is not actually limited to the associated set of `const`
variables, so the type safety provided by the original declarations gets lost.

```objectivec
enum E : char { NONE, ONE, TWO, FIVE = ONE + TWO + TWO };
```

```cangjie
public type M = Int8

public const NONE: M = 0
public const ONE: M = 1
public const TWO: M = 2
public const FIVE: M = 5
```


### Unions

The Cangjie language does not support C union types.
Unions are mirrored into structs with sequential fields and a warning is issued.


### `id`

The type `id` is mirrored into a built-in mirror interface `ObjCId`.


### Classes and Protocols

Objective-C classes and protocols are mirrored respectively into Cangjie
classes and interfaces. All such mirror classes and interfaces implicitly
implement/inherit the built-in `ObjCId` mirror interface
(see [Built-in Types](#built-in-types)).

**Methods** (except `init`-methods, see below) are mirrored into `public open`
member functions with the respective mirror types substituted for parameter
types and return value type. Mirrors of `void` methods have the `Unit` return
type. `instancetype` is mirrored into the name of the current declaration.
Mirrors of class methods (those with the "`+`" prefix) are modified
with `static`.

`init`-methods are a special case. They are mirrored into constructors,
as described in [Classes](#classes).

Bodies are omitted in the mirrors of _all_ methods, so they all look like
abstract member functions in regular Cangjie code.

In methods with a variable number of parameters, `, ...` is ignored.

Full Objective-C method names (selectors) decorated with `:` are not
valid Cangjie identifiers. Such names are mangled as follows:

* Each letter that immediately follows a `:` character, if any,
  is capitalized.
* All `:` characters are removed.

The original selector is retained in the `@ForeignName` annotation.

**Example:**

```c
@interface A
- (void)foo;
- (void)foo:(int)i;
- (void)foo:(int)i bar:(int) j;
- (void)foo:(int)i bar:(int) j baz:(int) k;
@end
```

is mirrored to

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

**Properties** are normally mirrored into `public` Cangjie member properties
of the respective mirror types, with some exceptions and nuances.

> The Objective-C `@property` directive is essentially syntactic sugar
> for access methods and backing variable declarations and enabling field
> access-like syntax for the calls of those methods. However, methods with the
> right signatures may be already declared in the class or inherited
> from a superclass. Such methods are automatically associated
> with the property. The directive may also explicitly designate a method
> with a non-standard name to serve as the getter or setter of the property.
> All this flexibility means that overriding of the getter and setter methods
> may occur even if the property itself is not overridden.
>
> It is also a common Objective-C pattern to override a `readonly` property
> with a `readwrite` one in a subclass.
>
> In contrast, the overridden and overriding properties must have the same
> mutability in Cangjie, and a member property may not have the same name
> as a member function.
>
> The above language discrepancies lead to the following restrictions:

* If a property overrides a superclass property, the overriding property is
  not mirrored. **NOTE:** The getter/setter methods of the overriding property
  would still get called on subclass instances thanks to dynamic dispatch.

* If the getter method of a property has a standard name _and_ overrides
  a superclass method, the property is not mirrored.

* Conversely, methods that override getter methods of inherited properties are
  not mirrored.

Getter and setter methods themselves are _never_ mirrored, regardless of their
names, of whether they are automatically generated at compile-time or not, and
of whether the property is mirrored.

Bodies (`{ }` blocks containing the getter and setter) are omitted in the
mirrors of properties.


The getter/setter functions of Cangjie properties may not be named arbitrarily.
The annotations `@ForeignGetterName` and `@ForeignSetterName` retain custom
names, if any, of the original getter and setter:

```objectivec
@interface FormElement : UIComponent
- (void)setEditable:(BOOL)flag;
- (BOOL)isEditable;
@property(getter=isEditable, setter=setEditable:) BOOL editable;
//   .  .  .
@end
```

```cangjie
public interface FormElement <: UIComponent {
    @ForeignGetterName["isEditable"]
    public mut prop editable: Bool
//   .  .  .
}
```

**NOTE:** Specifying `@ForeignSetterName["setEditable:"]` is unnecessary,
as that is the standard name for that setter method.


#### Classes

Objective-C `@interface` class declarations are mirrored into `public open`
Cangjie classes annotated with `@ObjCMirror`. The value of that annotation is
a string that retains the original name of the Objective-C class, if different
from the mirror class name.

Objective-C `@interface` category and extension declarations are merged
with the respective Cangjie class declarations.


Objective-C `@implementation` declarations are ignored.

Forward class declarations (`@class` directives) are mirrored into empty
classes (without any members).


**Methods identified as `init` methods** in accordance with the
[Method families section of the Clang documentation on Objective-C ARC](https://clang.llvm.org/docs/AutomaticReferenceCounting.html#method-families)
are mirrored into Cangjie constructors, with three limitations:

* If two or more `init` methods of a class differ only by name, i.e. have
  the same number and types of parameters, they cannot be mirrored into
  overloaded constructors. Instead, they are mirrored into `static`
  member functions annotated with `@ObjCInit`, which return an instance
  of the class being mirrored --- essentially, object factories:

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

  The names of those factory functions are derived from the original Objective-C
  names of the respective `init` methods, and the latter are retained
  in `@ForeignName` annotations, in the exact same way as it is done for any
  other mirrored methods.
  See [Classes and Protocols](#classes-and-protocols) for details.

* In Objective-C, `init` methods are inherited just like any other methods,
  whereas constructors in Cangjie are not inherited. The constructors that
  mirror the `init` methods of superclasses are therefore unavailable at the
  point of a mirror class or interop class instantiation. In contrast, the
  factory functions mirroring certain `init` methods as per the previous item
  _are_ inherited, but may not be used as superconstructors, as they return
  completely initialized superclass instances.


* Unlike a Cangjie/Java/etc. constructor that always initializes its receiver
  object, an Objective-C `init` method can return a substitute object and has
  an explicit return type. That type used to be `id`, so technically an `init`
  method could even return an instance of a totally different class.
  In [modern Objective-C code](https://developer.apple.com/library/archive/releasenotes/ObjectiveC/ModernizationObjC/AdoptingModernObjective-C/AdoptingModernObjective-C.html),
  however, the return type of an `init` method is normally `instancetype`,
  a special keyword meaning "pointer to an instance of (a subclass of) the
  receiver class". (The Xcode compiler actually promotes `id` to `instancetype`
  for known methods such as `init` and `alloc`, but not for, say, object
  factories.) The use of `instancetype` ensures that the returned object
  at least belongs to the right class, but does not preclude the return
  of a `nil` value. In fact, returning `nil` from an `init` method is the
  recommended way of signaling to the caller that the object could not be
  initialized for a reason that does not warrant raising an exception.

  The current implementation expects `init` methods to return instances
  of suitable classes and does not validate the returned value.
  **WARNING: If an `init` method called from Cangjie returns `nil` or a pointer
  to an instance of a class that is not (a subclass of) the receiver class,
  the behavior is undefined.**

**Instance variables** are mirrored into instance member variables of the
respective mirror types. There are no class variables in Objective-C.


#### Protocols

Objective-C `@protocol` directives are mirrored into Cangjie interfaces
marked as `@ObjCMirror public`.

Forward protocol declarations are mirrored into empty interfaces (without any
members).


**Methods** of Objective-C protocols are mirrored into member functions of the
respective Cangjie interfaces. Mirrors of class methods (those with the "`+`"
prefix) are modified with `static`.

Objective-C protocols may contain optional instance methods, which have no
direct equivalent in Cangjie. The member functions mirroring such methods are
annotated with `@ObjCOptional`. The bridge code implementing a cross-language
call of an `@ObjCOptional` method first checks if the receiver implements it at
all, throwing `NotImplementedException` if it does not.

```objectivec
@protocol MyDelegate <NSObject>

@required
- (void)requiredMethod;

@optional
- (void)optionalMethod;
@end
```

```cangjie
@ObjCMirror
public interface MyDelegate {
    func requiredMethod(): Unit

    @ObjCOptional
    func optionalMethod(): Unit
}
//   .  .  .
    try {
        delegate.optionalMethod()
    } catch (nie: NotImplementedException) { } // OK, not a big deal
```


#### Mirror Type Inheritance

Mirror classes and interfaces form separate subtype hierarchies, which
means that:

* Mirror classes may inherit other mirror classes, reflecting
  the inheritance relationships between the original Objective-C classes.
  Mirror classes may not inherit regular Cangjie classes other
  than `std.core.Object`, and vice versa.

* Mirror interfaces may inherit other mirror interfaces, reflecting
  the inheritance relationships between the original Objective-C protocols.
  Mirror interfaces may not inherit regular Cangjie interfaces and
  vice versa.

* Mirror classes may implement mirror interfaces, but not regular
  Cangjie interfaces. The interface `Any` is an exception, but its support is
  [currently limited](#not-yet-implemented-features). Regular Cangjie classes
  may not implement mirror interfaces.

* The mirror for the Objective-C `id` type is the built-in interface `ObjCId`.
  All mirror interfaces implicitly inherit `ObjCId`, and all mirror classes
  implicitly implement `ObjCId`.

* Neither mirror classes may be extended using `extend`, nor any other type
  may be interface-extended with a mirror interface.


### Pointers

**IMPORTANT: The current version does not yet support the type `CPointer<T>`
in contexts where types specific to Objective-C are also used,
such as the type of a parameter or return value of an Objective-C method
or a global function that also accepts or returns pointers to Objective-C class
instances or `id`. As a temporary workaround,
the [built-in type](#built-in-types) `ObjCPointer<T>` additionally supports type
arguments that meet the CType constraint.**

The modifiers `const`, `volatile`, and `restrict` are ignored.

Pointers to [primitive C types](#primitive-types) are mirrored
into `CPointer<`_`T`_`>` or `ObjCPointer<`_`T`_`>` (see notice above),
where _`T`_ is the matching Cangjie type.

C type aliases (`typedef`) for pointers to primitive types are always mirrored
into type aliases for `CPointer<`_`T`_`>` where _`T`_ is the matching Cangjie
type. If such an alias is used in an Objective-C context (see notice above),
that particular use is mirrored into `ObjCPointer<`_`T`_`>`, with the type alias
name preserved in a comment.

A pointer to a C enumeration is mirrored into `CPointer<`_`E`_`>`
or `ObjCPointer<`_`E`_`>` (see notice above), where
_`E`_ is the respective type alias. See [Enumerations](#enumerations)
for details.


#### Pointers to Structures

A pointer to a C structure `T` is mirrored into `CPointer<T'>`
or `ObjCPointer<`_`T'`_`>` (see notice in [Pointers](#pointers)), where `T'` is
the mirror type for `T`.



#### Pointers to Functions

Pointers to C functions are mirrored into a built-in type `ObjCFunc<F>`, where
`F` is the respective Cangjie function type, even if function
parameter types and return value are all `CType`-compatible. `ObjCFunc<F>` is
a built-in struct type, the public interface of which consists of a single
property:

```cangjie
public struct ObjCFunc<F> {
    public prop call: F
}
```

The compiler enforces a handful of restrictions on that type:

* The types of parameters and return value of the function type used as the
  type argument _`F`_ of `ObjCFunc<`_`F`_`>` must be
  [Objective-C-compatible](#objective-c-compatible-types).
  **NOTE:** The type _`F`_ itself is not Objective-C-compatible.

* The property `call` may only be used in function call expressions.

* There is no way to create an instance of `ObjCFunc<F>` in Cangjie code,
  all such instances originate from Objective-C code.

* There is currently no straightforward way to check whether the value
  of a certain `ObjCFunc<F>` passed over from Objective-C is `null`.


#### Pointers to Blocks

Pointers to Objective-C blocks are mirrored into the built-in struct type
`ObjCBlock<`_F_`>`, where _`F`_ is the respective Cangjie function type.
The public interface of `ObjCBlock<F>` consists of a single constructor and
a single property:

```cangjie
public struct ObjCBlock<F> {
    public init(f: F)
    public prop call: F
}
```

The compiler enforces a handful of restrictions on that type:

* The types of parameters and return value of the function type used as the
  type argument _`F`_ of `ObjCBlock<`_`F`_`>` must be
  [Objective-C-compatible](#objective-c-compatible-types).
  **NOTE:** That does not make the type _`F`_ itself Objective-C-compatible.

* The property `call` may only be used in function call expressions.

* There is currently no straightforward way to check whether the value
  of a certain `ObjCBlock<F>` passed over from Objective-C is `null`.

Unlike `ObjCFunc<F>`, instances of `ObjCBlock<F>` may be created in Cangjie
code from lambda expressions:

```cangjie
let halve: ObjCBlock<(Double) -> Double> =
    ObjCBlock { it => it / 2.0 }
```

A block may be called from Cangjie:

```cangjie
let x = halve.call(2.0)    // x == 1.0d
```

and passed over to mirrored Objective-C methods and functions as an argument.


#### Pointers to Class Instances

Pointers to Objective-C class instances are mirrored as follows:

* Pointers to class instances (`SomeClass*`) are mirrored into the
  respective mirror class types.

* The mirror of the Objective-C type `id` narrowed with a single protocol
  (for example, `id<NSCopying>`), is the mirror interface for that protocol.

* `id` narrowed with several protocols, e.g. `id<NSCopying, NSSecureCoding>`,
  is mirrored into the built-in type `ObjCId` (an interface that all
  `@ObjCMirror` classes and interfaces respectively implement and inherit),
  with the actual protocol list specified in a comment.

* If a generic type parameter is used inside a generic template
  with a narrowing protocol specified, it is mirrored into a reference to that
  protocol, with the name of the type parameter specified in a comment.

When used as the type of a parameter, return value, or member variable, such
mirror type additionally gets wrapped in `Option<T>`, _unless_ the said entity
is annotated as non-nullable. That enables the safe passage of `nil` values
between Objective-C and Cangjie.
See [`nil` Handling](#nil-handling) for details.


#### Pointers to Pointers

A pointer to a pointer to a type _`T`_ (`**`_`T`_) is mirrored as follows:

* If `*`_`T`_ would have been mirrored into a `CPointer<`_`T'`_`>` in the given
  context, `**T` is mirrored to `CPointer<CPointer<`_`T'`_`>>`.
  _`T'`_ is the mirror type for _`T`_.

* Otherwise (`*`_`T`_ would have been mirrored into an `@ObjCMirror` type or
  one of the [built-in](#built-in-types) Objective-C type mirrors), `**T` is
  mirrored into `ObjCPointer<`_`U'`_`>`, where _`U'`_ is the mirror type
  for `*`_`T`_.


### Generics

Parameterized Objective-C classes are mirrored into regular, non-generic
Cangjie class declarations. The original lightweight generics syntax
such as "`<T>`" or "`<Foo>`" is retained in comments; type parameter
usages are replaced with `ObjCId`.

**Example:**

```objectivec
@interface G<T> : NSObject
- (void)f:(T)t;
@end
```

is mirrored to

```cangjie
@ObjCMirror
public open class G/*<T>*/ <: NSObject {
    @ForeignName["f:"] public open func f(t: ?ObjCId /*T*/): Unit
}
```

However, type constraints are lost as of the current version, i.e.

```objectivec
@interface G<T: SomeType*> : NSObject
- (void)f:(T)t;
@end
```

is now mirrored exactly as the first sample above.


### Top-Level Functions

Top-level Objective-C functions are mirrored into global `public` function
declarations annotated with `@ObjCMirror`, with the respective mirror types
substituted for parameter types and return value type. Mirrors of `void`
functions have the `Unit` return type. Function bodies are omitted.

**NOTE:** If all parameter types and the return type of the function meet the
`CType` constraint, the generator produces a conventional `foreign func`
declaration instead.

In functions with a variable number of parameters, `, ...` is ignored.


### Built-in Types

The mirror generator assumes that the following user-available Cangjie types
are implemented in the interop library. As of the current version, however,
some of them (marked with "?") are not yet implemented, and their names may
change.

| Objective-C                                   | Cangjie (\*)        | Description                                           |
| --------------------------------------------- | ------------------- | ----------------------------------------------------- |
| `id`                                          | `ObjCId`            | Interface implicitly implemented by all `@ObjCMirror` classes and interfaces. Bound to Objective-C `id`. |
| `SEL`                                         | `SEL`?              | Class bound to Objective-C `SEL`.                     |
| `Class`                                       | `ObjCClass`?        | Class bound to Objective-C `Class`.                   |
| [pointer type](#pointers)                     | `ObjCPointer<T>`    | Structure bound to Objective-C pointers of two kinds: with arity more than one and when `T` is not a `CType`-compatible structure. |
| [non-C function type](#pointers-to-functions) | `ObjCFunc<F>`       | Class implementing `CFunc` for functions that are not `CType`-compatible. `F` is a Cangjie function type. |
| [block type](#pointers-to-blocks)             | `ObjCBlock<F>`      | Structure implementing an Objective-C block. `F` is a Cangjie function type. |

(\*) Names of Cangjie types marked with "?"" are provisional.


### Not (Yet) Implemented Features

Objective-C interoperability support in the Cangjie SDK is still in development.
Certain features are not implemented yet, others may change in the first
production release, and some cannot be (fully) implemented due to principal
differences between the two languages.

* The C language features that alter the default size, packing, padding,
  and/or alignment of data are very ABI-specific. The Cangjie language
  does not support such low-level features, and bit fields in particular.
  The mirror generator therefore ignores bit field width specifiers
  and issues warnings.

* The Cangjie language does not provide support for C union types,
  so they are mirrored as structs and warnings are issued.

* Structures with fields of types that do not meet the CType constraint are
  not supported.

* C enums are mirrored into series of `const` variables and type alias
  declarations, so type safety gets lost in translation.
  See [Enumerations](#enumerations) for details.

* In methods with a variable number of parameters, `...` is ignored.

* Annotations related to memory management, such as `NS_RETURNS_RETAINED`,
  are ignored.

* Properties are mirrored as properties if possible, otherwise their getter
  and setter methods are mirrored as if they were regular instance methods.
  See [Classes and Protocols](#classes-and-protocols) for details.

* Properties that override superclass properties are not mirrored.

* The modifiers `const` and `volatile` are ignored.

* The Objective-C types `SEL` and `Class` are not supported.

* There is a handful of nuances related to the construction of instances
  of mirrored Objective-C classes:

  - The `init` methods of an Objective-C class are normally mirrored
    into Cangjie constructors, except when two or more of those methods have
    exactly the same number and types of parameters. Such `init` methods are
    instead mirrored into `static` factory member functions annotated
    with `@ObjCInit`. However, such functions may not be used
    as superconstructors.

  - Constructors in Cangjie are not inherited, unlike `init` methods
    (but the factory functions are).

  - An `init` method may return `nil`, which Cangjie does not expect in the
    current version, leading to abnormal application termination.
    Future versions will throw `NoneValueException` in such a case.

  See [Classes](#classes) for details.

* Parameterized Objective-C classes are mirrored into regular, non-generic
  Cangjie class declarations. See [Generics](#generics) for details.

* Objective-C error handling is only possible using the
  `ObjCPointer<Option<`_`NSError'`_`>>` type, where _`NSError'`_ is the name
  of the mirror of the `NSError` class.

### `nil` Handling

Cangjie has no concept of null references and hence no equivalent for the
Objective-C `nil` value. If pointers to Objective-C classes and protocols were
mirrored into Cangjie classes and interfaces, any such value passed over
from Objective-C to Cangjie could lead to a segmentation fault. That would also
happen if Cangjie code accessed a member variable of such type mirroring a field
containing `nil`. Conversely, there would be no way to pass a `nil` value
from Cangjie to Objective-C either.

The `Option<T>` enum is therefore generally used to represent the values
of those Objective-C types, with `None` standing for the `nil` value,
and `Some(r)` representing a (non-null) reference value `r`. The `cjc` compiler
recognizes `Option<T>` as an Objective-C compatible type if `T` is a mirror type
or interop class and maps its values accordingly.

For instance, the following Objective-C `@interface` directive:

```objectivec
@interface MyContainer: NSObject
//   .  .  .
- (void)addItem:(MyItem *)item withUuid:(NSString *)uuid;
- (MyItem *)itemWithUuid:(NSString *)uuid;
- (NSString *)uuidForItem:(MyItem *)item;
@property (copy) NSArray<MyItem *> *allItems;
@end
```

will be mirrored into (`@ForeignName` annotations omitted for brevity):

```cangjie
@ObjCMirror
public open class MyContainer <: NSObject {
//       .  .  .
    public open func addItemWithUuid(item: ?MyItem, uuid: ?NSString): Unit
    public open func itemWithUuid(uuid: ?NSString): ?MyItem
    public open func uuidForItem(item: ?MyItem): ?NSString
    public open mut prop allItems: ?NSArray/*<MyItem>*/
}
```

`Option<T>` wrapping ensures that the code won't break if a `nil` value sneaks
into the Cangjie world from the Objective-C one, but it does that at the cost
of performance and memory footprint. The other disadvantage of this approach is
the [loss of variance](#loss-of-variance). That being said,
[support for nullability annotations](#nullability-annotations) considerably
reduces the impact of reference wrapping, at least when it comes to the use
of iOS APIs in Cangjie code.

**NOTE:** This problem does not exist for the low-level C types that get
mirrored to the conventional `CPointer<T>` type, as the latter provides
explicit null check member functions.


#### Loss of Variance

One limitation imposed by the [`Option<T>` wrapping](#nil-handling)
of Objective-C mirror types and interop classes is that such wrapped types
follow the semantics of Cangjie in all other respects. In particular,
`Option<T>` is _invariant by its type parameter `T`_: `Option<U>` is not
a subtype of `Option<T>` if `U` is a subtype of `T` unless `T` and `U` are
exactly the same type. For mirror types that means that any overriding
Objective-C method that relies on return type covariance may not be mirrored
that way with `Option<T>` wrapping. The return value type of the mirror of such
a method has to be propagated from the method that it overrides.

**Example:**

Suppose an Objective-C class `Foo` is a direct superclass of the class `Bar`:

```objectivec
@interface Foo : NSObject
@end

@interface Bar : Foo
@end
```

and the class `C` declares a method `get` that returns an instance of `Foo`:

```objectivec
@interface C : NSObject
- (Foo*) get;
@end
```

A subclass of `C` may then override `get` with a more precise return type,
`Bar`:

```objectivec
@interface D : C
- (Bar*) get;
@end
```

Without `Option<T>` wrapping, all those classes could be mirrored to:

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
    open func get(): Bar       // Return type covariance in action
}
```

but if `get()` can possibly return `nil`, an application crash is inevitable.

`Option<T>` wrapping makes it safe, but the return types of all overriding
methods have to be lowered to the return type of the original method:

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
    // open func get(): Option<Bar>  // Error, `Option<T>` is not covariant by T
    open func get(): Option<Foo>     // OK, but the return type is lowered
}
```

[Nullability annotations](#nullability-annotations) alleviate the problem
partially.


#### Nullability Annotations

Nullability keywords were introduced to Objective-C with the release
of XCode 6.3 back in the day, for better integration with the new iOS/OS X
development language, Swift. The definitions of all iOS APIs were enhanced
with nullability annotations to reduce the use of optionals in Swift code
utilizing those APIs.

> **Objective-C Nullability Annotations**
>
> The keywords `nullable` and `nonnull` can be used for Objective-C properties,
> method result types and method parameter types. They mean that the given entity
> respectively may or may not hold or accept the `nil` value. The (very rare)
> `null_unspecified` annotation denotes the entities about which it is unknown
> whether their value may be `nil`.
>
> In addition, any pointer type may be annotated with `__nullable`, `__nonnull`
> or `__null_unspecified` with the same semantics.
>
> Finally, a property may also be annotated with `null_resettable`, meaning that
> its getter will never return `nil`, but passing `nil` to its setter would
> reset the property to some default value.
>
> For details, see
> [Designating Nullability in Objective-C APIs](https://developer.apple.com/documentation/swift/designating-nullability-in-objective-c-apis)
> on the Apple Developer Web site.

All uses of Objective-C reference types annotated as non-nullable are therefore
exempt from `Option<T>` wrapping. In other words, the Objective-C mirror
generator emits wrapped types only for properties, method result types and
method parameter types that are _not_ annotated with either `nonnull`
or `_Nonnull` in the original Objective-C code.

Suppose the example from the
[introduction to this section](#nil-handling)
was enhanced with nullability annotations as follows:

```objectivec
@interface MyContainer: NSObject
//   .  .  .
- (void)addItem:(nonnull MyItem *)item withUuid:(nonnull NSString *)uuid;
- (nullable MyItem *)itemWithUuid:(nonnull NSString *)uuid;
- (nullable NSString *)uuidForItem:(nonnull MyItem *)item;
@property (copy, nonnull) NSArray<MyItem *> *allItems;
@end
```

The mirror generator would then bypass `Option<T>` wrapping for the
`nonnull` entities (`@ForeignName` annotations omitted for brevity):

```cangjie
@ObjCMirror
public open class MyContainer <: NSObject {
//       .  .  .
    public open func addItemWithUuid(item: MyItem, uuid: NSString): Unit
    public open func itemWithUuid(uuid: NSString): ?MyItem
    public open func uuidForItem(item: MyItem): ?NSString
    public open mut prop allItems: NSArray/*<MyItem*>*/
}
```

**NOTE:** It is currently not possible to mirror `null_resettable` properties
into Cangjie properties while propagating the semantics of that annotation,
hence the latter is treated exactly as `nullable`.

**If your Objective-C code that you want to access from Cangjie is not taking
advantage of the nullability annotations, you may want to add at least the
`nonnull` annotations as appropriate before you begin using the Cangjie SDK
interoperability features described here.** That may considerably reduce
the use of `Option<T>` wrapping in the respective mirror types, making the
code of interop classes cleaner and easier to read.


### Foreign Types Conversion And Testing

The Cangjie operators `is` and `as` are supported for all
[foreign types](#foreign-types), as well as the type pattern _`v`_`: `_`T`_
in match, if-let, and while-let expressions (see the next paragraph for
applicable limitations). Note, however, that the semantics of type testing
and conversion for those types matches that of Objective-C, and those operations
are conducted with the help of the Objective-C Runtime functions
in the general case.

As of the current version, if-let and while-let support is limited: the
`let` expression must constitute the entire conditional expression, i.e.
it cannot be combined with another one using a logical binary operator `&&`
or `||`.


**IMPORTANT:** Nullable values of mirror types and interop classes, represented
using `Option<T>` wrapping as described
in [`nil` Handling](#nil-handling), need to be null-tested and unwrapped
before type testing. The reason is that Cangjie generics are invariant
with respect to their type arguments: _`e`_` is ?`_`T`_ evaluates to `true` only
if the type of _`e`_ is `Option<`_`T`_`>` specifically, not some
`Option<`_`U`_`>` where _`U`_` <: `_`T`_. Moreover, it does not even matter
whether the value _`e`_ is `Some(`_`v`_`)` or `None`; _`v`_ is not type-tested
at all.



## Objective-C Mirror Generator Reference

### Prerequisites

Make sure to run the `envsetup.sh` script from the Cangjie SDK before using
the mirror generator.

You need to know the pathnames of the header files of the frameworks and libraries
in which all dependencies of the classes you are going to mirror are defined.
This includes the locations of the iOS standard library headers and generally
all directories in which the Objective-C compiler looks up header files when it
builds your project.


### Command-line Syntax

`ObjCInteropGen [-v] _`config-file`_`]`

`-v`

Produce verbose output.

_`config-file`_

The pathname of the configuration file.


### Configuration File Syntax

An Objective-C mirror generator configuration file is a plain text file
in the [TOML](https://toml.io) syntax. It specifies:

* Output directories
* Names of the source Objective-C headers (`.h`-files)
* Names of the output Cangjie packages and the distribution of mirror types
  across them
* Mappings for types that you want to be handled in a special way

String values that are interpreted as regular expressions must follow the
[ECMAScript regular expression syntax](https://262.ecma-international.org/#sec-regular-expressions).


#### Output Directories Roots

Each entry in the `[output-roots]` table of tables defines a symbolic name
for the pathname of a directory in the local file system. The mirror generator
will use that pathname as the common root of one or more package-specific
output directories, set using the `output-root` property of the respective
[`[[packages]]` array](#packages) elements.

**Example:**

```toml
[output-roots.lib]
path = "./lib/src"

[output-roots.app]
path = "./main/src"

[[packages]]
package-name = "com.vendor1.lib1"
output-root = "lib"  # Output to "./lib/src/com/vendor1/lib1"
filters = ...

[[packages]]
package-name = "com.vendor2.lib2"
output-root = "lib"  # Output to "./lib/src/com/vendor2/lib2"
filters = ...

[[packages]]
package-name = "com.mycompany.app"
output-root = "app"  # Output to "./main/src/com/mycompany/app"
filters = ...
```


#### Source Files

Each entry in the `[sources]` table of tables specifies a set of individual
header files that the mirror generator must take as input.

**Properties:**

`paths` (mandatory)

An array of strings that contain pathnames of individual header
files that the mirror generator must read.

`arguments` (optional)

Array of strings containing Clang options that the mirror generator will
pass over when processing the source files listed in `paths`.

**Example:**

```toml
[sources.all]
paths = ["original-objc/M.h"]
```


#### Additional Clang Arguments

Each entry in the `[sources-mixins]` table of tables specifies a regular
expression matching one or more keys of the [`[sources]` table](#source-files)
and additional arguments that must be passed to Clang when processing
the header files from its respective entries.

**Properties:**

`sources`   _(mandatory)_

A string containing a regular expression.

`arguments-prepend`\
`arguments-append`   _(optional)_

Arrays of strings that the mirror generator will pass over to Clang
as options when processing the source files listed in any `[sources]`
table entry which key matches the regular expression specified in
the `sources` property.

The options listed in `arguments-prepend` and `arguments-append`
will be passed respectively before and after the options specified
in the `arguments` property of each matching `[sources]` entry,
if any.

**Example:**

```toml
[sources.UIWidgets]
paths = ["objc/UIWidgets.h"]
arguments = [ "-I", "/usr/local/include/share/Widgets" ]

[sources.UIPanels]
paths = ["objc/UIPanels.h"]
arguments = [ "-I", "/usr/local/include/share/Panels" ]

[sources-mixins.UI]
# Add these Clang arguments for both UIWidgets and UIPanels
sources = ["UI.+"]
arguments-append = [
    "-I", "/usr/local/include/Frameworks/AcmeUI"
]
```


#### Packages

Each entry in the `[[packages]]` array specifies a target Cangjie package
name, a set of name filters that define which Objective-C entities will
be mirrored to that package, and, optionally, the output directory
specific for that package.

**Properties:**

`package-name`   _(mandatory)_

A string containing the target Cangjie package name.

`output-path`   _(optional)_

A string containing the _exact_ pathname of the directory into which
the mirror generator shall place the output files for the given package.
If any directories in that pathname do not exist, the mirror generator
will attempt to create them.

`output-root`   _(optional)_

A string containing a key from the [`output-roots` table](#output-directories-roots)
The name of the target Cangjie package will be appended to the value
of the respective `output-roots` table entry, dots `.` replaced with
the file separator `/`, and the resulting pathname will be used
as if it was the value of `output-path`.

If `output-root` is not set, and if there is only one element
in `[output-roots]`, the pathname from that entry is used.
Otherwise, an error is shown.

**Example:**

```toml
[output-roots.main]
path="./cj-mirrors"

[[packages]]
package-name = "objc.foundation"
output-root = "main"
```

The output files will be placed in `./cj-mirrors/objc/foundation`.

`filters`   _(mandatory)_

A table defining a set of name filters that select only those
Objective-C entity declarations from the source files that
must be mirrored to the given Cangjie package. See
[Name Filters](#name-filters) for details.

**Example:**

```toml
# The Foundation framework
[[packages]]
package-name = "objc.foundation"
filters = { include = "NS.+" }
```


##### Name Filters

Each name filter is, in turn, a TOML table that contains:

* Exactly one of the properties `include`, `exclude`, `union`, `intersect`
  and `not`, and
* Optionally, the `filter` property _and/or_ the `filter-not` property.

Descriptions of the properties follow.

`include`

The value can be a regular expression or an array thereof.
If the value is a single regex, only names that match it
pass the filter. If the value is an array, a name only needs to
match any single regex from that array to pass.

**Examples:**

```toml
# Only include entities that names of which start with "NS":
filters = { include = "NS.+" }

# Only include entities with names that either start with "Foo"
# or end with "Bar":
filters = { include = ["Foo.*", ".*Bar"] }
```

`exclude`

The opposite of `include` (see above). A name that would pass
an `include` filter with the given value does not pass an `exclude`
filter with the same value and vice versa.

**Example:**

```toml
# Include everything but entities the names of which start with "INTERNAL_":
filters = { exclude = "INTERNAL_.+" }
```

`union`

An operator combining two or more filters. Its value must be an
array of filters. A name that passes any single one of those
filters passes the entire `union` filter.

**Example:**

```toml
# An equivalent of the second `include` example above:
filters = { union = [ { include = "Foo.*" }, { include = ".*Bar" } ] }
```

`intersect`

An operator combining two or more filters. The value must be an
array of filters. A name must pass all of them to pass the
entire `intersect` filter.

**Example:**

```toml
# Adding a negative filter:
filters = { intersect = [ { include = "NS.+" },
                          { exclude = "NSAccidentalClash" } ] }
```

`not`

An operator that reverses the meaning of a filter. The value must
be a single filter:

**Example:**

```toml
# Another way to add a negative filter:
filters = { intersect = [ { include = "NS.+" },
                          { not = { include = "NSAccidentalClash" } } ] }
```

`filter`\
`filter-not`   _(optional)_

These properties must be mixed with other properties, i.e.
they cannot be the only properties of a `filters` table.
Their values can be regular expressions or arrays thereof,
just as those of the `include` and `exclude` properties.

`filter` reduces the set that passed the main filter to include
only names that match the given single regular expression or any
regex from the array.

`filter-not` reduces the set that passed the main filter to
include only names that do not match the given single regular
expression or any regex from the array.

`filter` and `filter-not` are actually shorthands for `intersect`
operations with an `include` and `exclude` filter respectively.
They can be specified together.

**Examples:**

```toml
# Without filter-not:
filters = { intersect = [ { include = "NS.+" },
                          { exclude = "NSAccidentalClash" } ] }
# With filter-not:
filters = { include = "NS.+", filter-not = "NSAccidentalClash" }

# 'filter' and 'filter-not' can be used together:
filters = { include    = ".*Fizz.+",
            filter     = ".+Buzz.*",
            filter-not = ".*FizzBuzz.*" }
```


#### Type Substitutions

`[[mappings]]` is an array of tables, each adding to the list of substitutions
of one Objective-C type for another. All mentions of almost any type
(except for C primitive types such as `int`) can be replaced with mentions
of another type.

**Example:**

```toml
# Replace the type id, the root of the Objective-C type hierarchy,
# with NSObjectProtocol everywhere.
[[mappings]]
id = "NSObjectProtocol"
```


#### Configuration File Import

`imports` is an array of strings each containing the pathname of another
configuration file, the settings from which will be added to the current
configuration. The entries of `packages` and `mappings` arrays found
in the imported file are appended to those present in the importing file.

Nested imports are supported, circular import is detected and results
in an error.

**Example:**

```toml
imports = ["../common.toml"]
```



## Execution

### Initialization

When control first reaches Cangjie code, all global and `static` Cangjie
variables are initialized, and the static initializers of all Cangjie types are
called. That happens when any [interop class](#interop-classes) is accessed
from Objective-C code for the first time.

**IMPORTANT: All that Cangjie initialization code must _not_ use any mirror
types or interop classes in any manner whatsoever. A deadlock is imminent if
this rule is violated.**



### Finalization

#### `dealloc`

Interop classes may not override the `dealloc()` method of `NSObject`.
An attempt to do so would result in a name clash and compile-time error,
because the interop-enabling code defines a `dealloc` method in the wrapper
object for its own purposes.



#### Cangjie Finalizers

A mirror class declaration cannot contain a Cangjie finalizer (`~init()`).

It is **TBD** whether an [interop class](#interop-classes) can contain
a finalizer.


### Exceptions

Both Objective-C and Cangjie feature exceptions - events that interrupt the
normal control flow of the program, usually to signal that a non-fatal execution
error has occurred and needs to be handled.

In the bidirectional interop scenario supported by CJMP, control may pass
between methods/functions written in different languages multiple times
in a chain of nested calls. As a result, there may be a number of Objective-C
and Cangjie frames interspersed on the thread stack when an exception gets
thrown in either Objective-C or Cangjie code. That in turn means that the stack
unwinding process may cross a language boundary. **IMPORTANT: In the current
version, such crossing leads to undefined behavior**, which means that
Objective-C methods and functions called from Cangjie must not leave any
exceptions uncaught and vice versa.


There is no way to `throw` an Objective-C exception in Cangjie code or vice
versa.


### Memory Management

Objective-C and Cangjie objects reside in separate heaps. The respective
language runtime manages each of the two heaps. The interop library and bridge
code ensure that objects in one language heap do not get released/garbage
collected if references to them still exist in accessible variables and data
structures of the other language.

**IMPORTANT: The mechanism that ensures cross-language heap consistency has
three significant limitations that must be understood and remembered at all
times when using the interop facilities:**

1. The efficiency and timing of release of Objective-C objects that had been
   used in Cangjie code and then became inaccessible from either language
   depends on the Cangjie garbage collector. Even if a reference is short-lived
   from the application developer perspective, that does not mean the object
   gets released immediately when e.g. the last variable holding it gets
   assigned a new value. Traversing a large Objective-C array or collection
   in Cangjie code in a loop could create thousands of such temporarily pinned
   objects. The application memory footprint would therefore inflate until the
   Cangjie garbage collector is invoked.


2. As Objective-C ARC and Cangjie garbage collector each operate solely within
   their respective managed environments, inter-language circular references may
   lead to memory leaks. The creation of such references must be either avoided
   altogether, or accompanied with logic that would break such cycles before the
   objects forming the cycles become unreachable from both Objective-C
   and Cangjie parts of the application code.

3. As of the current version, values of mirror types and interop classes must
   not be stored in global or static Cangjie variables, nor in any data
   structures to which such variables refer. For that reason, converting values
   of foreign types to `Object` and `Any` is prohibited.

   Work is underway on the removal of this limitation.

   **WARNING:** The `cjc` compiler does not fully enforce the above
   restrictions, so strict programming discipline is required. Failure to comply
   with that discipline may lead to an abnormal program termination.


### Threads

Threads created by Cangjie `spawn` may use Objective-C mirror types
without any restrictions.



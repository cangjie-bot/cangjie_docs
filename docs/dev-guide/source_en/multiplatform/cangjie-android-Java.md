# Cangjie Interoperability With Java

**NOTE:** The Java interoperability feature is experimental and still
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

Instead, the interoperability between Cangjie and Java is achieved
by disguising them both as low-level languages. Cangjie and Java parts
of the application see each other through the lens of the Java Native Interface
(JNI), originally designed to facilitate the development of Java `native`
methods in languages such as C and C++. JNI is a rich, but low-level API,
and writing `native` Java methods is known to be cumbersome. Fortunately,
CJMP automates away the complexity of JNI.

The Objective-C Runtime module API serves a similar role on iOS. It is also
designed specifically to enable the creation of bridge layers between
Objective-C and other languages. And just like with JNI, CJMP takes care
about the cumbersome parts.


## Android Versions Support

**IMPORTANT: The Cangjie SDK for Android generally supports Android 6 and above
(API level 23 and up). However, all Java 8+ language features and APIs used
in Cangjie code must be _directly_ supported at the _minimum_ Android API level
specified in the application build script. Details below.**

The Android toolchain supports new Java language features and APIs on legacy
versions of Android via bytecode transformations called _desugaring_. The D8
compiler performs those transformations when converting Java class files
into DEX code that runs on Android devices.
See [https://developer.android.com/studio/write/java8-support](https://developer.android.com/studio/write/java8-support)
for details.
However, those newer features and APIs are _not_ available to Cangjie code
at run time if they are desugared.

In particular, `static` and `default` methods in interfaces are not supported
on Android 6 (API level 23). Therefore, Cangjie code must not use such methods
if `minSdk` is set to `23` in the main Gradle build script of the application.

Similarly, the newer Java platform APIs are unavailable unless you target only
those versions of Android that support them out-of-the-box, without desugaring.
An attempt to use a desugared API from Cangjie code will result in a run-time
error _on any version of Android_.

For instance, the Cangjie SDK for Android does not support Java 8+ platform APIs
such as the Stream API and `java.util.Optional` on Android 6 to 8.1 (API levels
23-27). If your Cangjie code uses those APIs, `minSdk` must be set to `28`
or above in your Gradle build script.

Refer to the official Android developer documentation for detailed information
about the availability of specific Java 8+ features and APIs in the versions
of Android that you are targeting.


## Key Concepts

### Mirror Types

Considering Cangjie and Java as a pair of interoperating languages,
a _mirror type_ _`T'`_ defined in Cangjie is a type that represents an existing
Java type _`T`_. It enables Cangjie developers to use the type _`T`_, possibly
with some limitations.

The Boolean types and numeric types that are essentially the same in both
languages naturally mirror each other: the Cangjie mirror type for the Java
type `int` is `Int32` and so on.

For a user-defined type such as a class, interface, or array, the mirror type
would be its closest equivalent in the other language. For instance, Cangjie
enums are way too different from Java enum classes, so the latter are
best [approximated with Cangjie classes](#enum-classes).

A mirror type exposes the members and constructors of the original user-defined
type that are accessible _and_ can be used in the other language. Constructors
of a Java enum class, for example, are only used during its initialization,
so it makes no sense to expose them to Cangjie.

Normally you would obtain mirror type definitions for a Java type _and_ its
required dependencies automatically, using the standalone
[mirror generator](#java-mirror-generator-reference) included in the Cangjie
SDK for Android.


#### Mirroring Java Types to Cangjie

The `cjc` compiler replaces any uses of Java mirror types with the appropriate
glue code, so only the names of types themselves and names and types of their
accessible members matter. Therefore, mirrors of Java types only contain member
declarations, not definitions: constructors and member functions/properties
have no bodies and member variables have no initializers.
For the same reason, `private` and package-private members are omitted.

For such an extension of the regular Cangjie syntax to work, each mirror type
must be marked with a `@JavaMirror` annotation. It helps the compiler
distinguish between mirror type declarations and normal Cangjie type
definitions.

For example, a mirror class for the following Java class:

```java
public class Node {
    public static final int A = 0xDeadBeef;
    private int _id;
    public Node(int id) { _id = id; }
    public int id() { return _id; }
}
```

might look like this:

```cangjie
@JavaMirror
public open class Node {
    public static let A: Int32
    public init(id: Int32)
    public open func id(): Int32
}
```

A few foundational Java types, namely `java.lang.Object`, `java.lang.String`
and arrays, have [predefined mirrors](#interop-library-api-reference)
in the interoperability library.



### Interop Classes

An _interop class_ is essentially a Cangjie class that is derived from one
or more [mirror types](#mirror-types) and is usable from both Cangjie and Java.
All its constructors and non-inherited `public` member functions are exposed
to Java code via a _wrapper class_ automatically generated by the `cjc`
compiler. The wrapper class itself defines no other user-callable
methods or constructors, but any methods it may have inherited from its Java
supertypes may be called from both Java and Cangjie code.

For example, when compiling the following Java interop class:

```cangjie
@JavaImpl
public class BooleanNode <: Node {
    private let flag: Bool
    public init(id: Int32, flag: Bool) {
        super(id)
        this.flag = flag
    }
    public func isFlagged(): Bool {
        flag
    }
}
```

the `cjc` compiler will also yield a Java source code file similar
to the following:

```java
public class BooleanNode extends Node {
    /* glue code */
    public BooleanNode(int id, boolean flag) {
        /* Glue code constructing a Cangjie BooleanNode instance and associating
         * it with the Java BooleanNode instance being constructed, i.e. 'this'.
         */
    }
    public boolean isFlagged() {
        /* Glue code invoking the 'isFlagged' member function of the associated
           Cangjie BooleanNode instance and returning the result.
         */
    }
    /* more glue code */
}
```

Now both Java and Cangjie parts of the application code can instantiate the
interop class `BooleanNode`, call its `public` instance member functions `id()`
(inherited from the Java class `Node`) and `isFlagged()`, convert its instances
respectively to the type `Node` and its mirror, etc.


### Foreign Types

The [mirror types](#mirror-types) and [interop classes](#interop-classes) are
not perfectly native to the language in which they are defined. Extending the
analogy, their status is more like temporary visitors and work visa holders
respectively, so they are collectively called _foreign types_ throughout
this document.


### Java-Compatible Types

The following Cangjie types are called _Java-compatible_:

* Value types that have direct equivalents among Java primitive types
  (`Int16` is included, but `UInt8` is not)
* [Foreign types](#foreign-types)
* Types of the form `Option<`_`T`_`>` where _`T`_ is a [foreign type](#foreign-types)
  (see _[`null` Handling](#null-handling)_ for reasoning)

The special generic mirror type [`JArray<T>`](#java.lang.jarrayt) included
in the [interop library](#interop-library-api-reference)
represents Java arrays. Its type variable `T` must be a Java-compatible type.

For obvious reasons, the parameters and return values of `public` member
functions of a foreign type may only have Java-compatible types, and so do
the parameters of its `public` constructors.

The member variables of an interop class may have any types, as they are
not exposed to Java even if they are `public`. `public` member properties are
exposed, though:

```cangjie
@JavaImpl
class WeightedNode <: Node {
    public let weight: Float64      // Not exposed to Java
    public init(weight: Float64) {
        this.weight = weight
    }
}
```

```cangjie
@JavaImpl
class ColoredNode <: Node {
    private let _color: Int32
    public prop color: Int32 {     // Exposed to Java as method 'int getColor()'
        get() { _color }
    }
    public init(color: Int32) {
        _color = color
    }
}
```

For convenience, the Cangjie type `String` has limited compatibility with Java.
See [Automatic String Conversion](#automatic-string-conversion) for details.



## Using Java in Cangjie

To enable interoperation between the Java and Cangjie parts of your Android
application, you need to take the following steps:

1. Design the interoperability layer API in terms of _Java_ classes and methods.

   developer → `interop layer design` (Java _pseudo-code_)

2. Generate mirror declarations for any existing Java classes and interfaces
   used in the design of the interoperability layer.

   `.class` → mirror generator → `.cj` (mirrors)

3. Write the classes constituting the interoperability layer _in Cangjie_.
   Use the mirrored Java types as needed: create instances, call methods, etc.

   `interop layer design` + `.cj` (mirrors) → developer → `.cj` (interop layer)

4. Compile the interop classes and mirror type declarations together with `cjc`.
   `cjc` will generate:

   * the necessary glue code for all uses of the mirror types.
   * the actual Java source for the Java part of the interoperability layer.

   `.cj` (mirrors + interop layer) → `cjc`  → `.so` + `.java` (interop layer)

5. Add to your Android project:

    * `.java` files generated by `cjc`
    * `.so` file generated by `cjc`
    * runtime libraries (`.so` and `.jar`) from the Cangjie SDK

    then insert the necessary uses of the interop layer in the Java source
    and rebuild the project.

    Android project + `.java`, `.jar`, `.so` → Android toolchain → `.apk`

The sections
[Initial Interop Class Creation Workflow](#initial-interop-class-creation-workflow)
and [Calling Java From Cangjie](#calling-java-from-cangjie)
describe the above process in detail with an end-to-end example.


### Initial Interop Class Creation Workflow

#### Step 1: Design the interoperability layer

On this step, you design the API of one or more interop classes
_from the Java code perspective_, i.e. you need to decide, for each interop
class:

* To which Java package it will belong;
* What Java class it will extend, if not the default `java.lang.Object`;
* Which Java interface(s) it will implement, if any; and
* What `public` methods and/or constructors it will have.
  You only need to know the types of parameters and, for methods, their names
  and return value types; the implementations you will write in Cangjie.

See also
[Features and Limitations of Interop Classes](#features-and-limitations-of-interop-classes).

**Example:**

Suppose we want to pass control from Java to Cangjie by invoking a certain
`static` method `m()` accepting parameters of types `com.example.a.A`,
`java.lang.String`, and `int`, and returning a value of type `com.example.b.B`.
We want that method to belong to a class called `Interop` that extends `Object`
and belongs to the package `cjworld`.

In other words, we have determined that the Java source code file that we
want to be generated on step 4 should have the following content:

```java
package cjworld

import com.example.a.A;
import com.example.b.B;

public class Interop {
    /* glue code */
    public static B m(A a, String s, int i) {
        /* glue code invoking the Cangjie implementation of m() */
    }
    /* more glue code */
}
```


#### Step 2: Generate mirror type declarations

Now, you need [mirror type](#mirror-types) declarations for all Java reference
types on which your interop classes depend: their supertypes, types of their
method and constructor parameters, etc., and, possibly, the dependencies of
those types.

**NOTE:** Mirror classes for `java.lang.Object` and `java.lang.String`,
as well as a generic mirror class representing Java arrays, are provided
by the [interop library](#interop-library-api-reference). Skip this step
if the methods and constructors of your interop classes only pass to/receive
from Cangjie values of primitive Java types, `java.lang.Object`,
`java.lang.String`, and/or arrays of the foregoing.

Use the [Java mirror generator](#java-mirror-generator-reference) included
in the Cangjie SDK to produce those mirror type declarations as follows:

**Command line:**

```bash
java -Dpackage.mode=true -Dpackage.name=package-name \
    -jar ${CANGJIE_HOME}/tools/bin/java-mirror-gen.jar \
    --boot-class-path path-to-android-jar \
    --class-path full-application-classpath \
    -d output-directory \
    names-of-mirrored-types
```

or

```bash
java -Dpackage.mode=true -Dpackage.name=package-name -Djar.mode=true \
    -jar ${CANGJIE_HOME}/tools/bin/java-mirror-gen.jar \
    --boot-class-path path-to-android-jar \
    --class-path full-application-classpath \
    -d output-directory \
    jar-file
```


* `package-name` is the desired name of the Cangjie package for all
  mirrored types.

* `path-to-android-jar` is the pathname of the `android.jar` file in the
  Android SDK that you are using,
  e.g. `${ANDROID_SDK}/platforms/android-35/android.jar`.

* `full-application-classpath` is the complete classpath of the application
  that is used during its build, _including `android.jar`_ again.

* `output directory` is the pathname of the directory where the Cangjie source
  files containing mirror type declarations will be placed, such as `./src/cj`.

* `names-of-mirrored-types` are space-separated _fully qualified names_ of the
  Java reference types on which your interop class design depends,
  other than `java.lang.Object`, `java.lang.String` or arrays.

* `jar-file` is the pathname of a single jar file. All `public` classes and
  interfaces, the `.class` files of which that jar contains, are mirrored, along
  with any dependencies found along the `full-application-classpath`.

**Example:**

Continuing the above example, we notice that our class `cjworld.Interop` depends
on:

* `java.lang.Object` as a superclass;
* `com.example.a.A`, `java.lang.String`, and `int` as parameter types; and
* `com.example.b.B` as a return value.

We do not need to mirror primitive types such as `int`, and the classes
`java.lang.Object` and `java.lang.String` are pre-mirrored. This leaves us
with just the two `com.example` classes to mirror. Suppose we want them
to appear in the package `javaworld`.

Mirror generator command line with a fictional application classpath:

```bash
java -Dpackage.mode=true -Dpackage.name=javaworld \
    -jar ${CANGJIE_HOME}/tools/bin/java-mirror-gen.jar \
    --boot-class-path ${ANDROID_SDK}/platforms/android-35/android.jar \
    --class-path ${ANDROID_SDK}/platforms/android-35/android.jar:./App.jar \
    -d ./src/cj \
    com.example.a.A com.example.b.B
```

This will generate files `src/cj/javaworld/src/A.cj`
and `src/cj/javaworld/src/B.cj`, _plus_ mirror type declarations for any
dependencies those two Java types may have.


#### Step 3: Write interop classes

For each Java class skeleton you designed on step 1, write a new Cangjie class
as follows:

1. Use the appropriate package and class names (the Java wrapper class will
   have the same fully qualified name).
2. Import `java.lang.*`.
3. Import the necessary mirror types, if you generated any on Step 2. Do not
   import the dependencies for now.
4. Annotate the class with `@JavaImpl`.
5. Make the class inherit the mirror of the desired Java class (by default,
   it will inherit the built-in mirror of `java.lang.Object`).
6. Use `JObject`, `JString` and `JArray<T>` in place of `java.lang.Object`,
   `java.lang.String` and Java array types respectively.

Type mapping (`T'` is either the matching Cangjie value type or the respective
mirror type):

Java Type (`T`)  | Cangjie Type (`T'`)
---------------- | --------------------------------------------------
`boolean`        | `Bool`
`byte`           | `Int8`
`short`          | `Int16`
`char`           | `UInt16`
`int`            | `Int32`
`long`           | `Int64`
`float`          | `Float32`
`double`         | `Float64`
`Object`         | `JObject` or `?JObject`^\*^
`String`         | `JString`, `?JString`, `String` or `?String`^\*†^
`class C`        | `C'` or `?C'`^\*^
`interface I`    | `I'` or `?I'`^\*^
`T[]`            | `JArray<T'>` or `?JArray<T'>`^\*^

^\*^ Use `?<T'>` (`Option<T'>`) types for parameters, return values, and local
variables of mirror and interop types that may receive/hold Java `null`.

^†^ See [Automatic String Conversion](#automatic-string-conversion) for the
details of using the Cangjie type `String` in the definitions of interop class
members and constructors exposed to Java.

Use `Unit` as a return type to represent a `void` Java method.

See also
[Features and Limitations of Interop Classes](#features-and-limitations-of-interop-classes).

**Example:**

To continue the above example, the interop class for the function `m()` would
look like this:

```cangjie
package cjworld

import java.lang.*
import javaworld.*

@JavaImpl
public class Interop {
    public static func m(a: ?A, s: ?JString, i: Int32): ?B {
        /* Just a dummy implementation for now */
        B()    // Assuming that B has an accessible parameterless constructor
    }
}
```

If you don't need `m()` to ever return `null` to its Java caller, remove
`?` from its return value type.


#### Step 4: Compile interop classes

**NOTE:** Make sure you have the
[Android NDK](https://developer.android.com/ndk) installed and the
`ANDROID_NDK_HOME` environment variable points to it.

Compile the mirror type declarations first, then the interop classes
(assuming they are in different packages, e.g. `javaworld` and `cjworld`).

Command line:

```bash
cjc --output-type=dylib \
    --target=aarch64-linux-android31 \
    -p source-directory \
    -ljava.lang -ljava.internal \
    --output-javagen-dir=java-output-directory \
    --sysroot=${ANDROID_NDK_HOME}/toolchains/llvm/prebuilt/linux-x86_64/sysroot \
    -B ${ANDROID_NDK_HOME}/toolchains/llvm/prebuilt/linux-x86_64/bin
```

where

`source-directory` is the pathname of the directory containing the source code
of interop classes and mirror type declarations.

`java-output-directory` is the pathname of the directory to which the generated
Java source files are placed.

The output is a `.so` file with compiled Cangjie code for the interop classes
and Java source files (`.java`) for their Java wrappers.

**Example:**

```bash
cjc --output-type=dylib \
    --target=aarch64-linux-android31 \
    -p src/cjworld \
    -ljava.lang -ljava.internal \
    --output-javagen-dir=src/java
    --sysroot=${ANDROID_NDK_HOME}/toolchains/llvm/prebuilt/linux-x86_64/sysroot \
    -B ${ANDROID_NDK_HOME}/toolchains/llvm/prebuilt/linux-x86_64/bin
```

The compiler will produce two files: `libcjworld.so`
and `src/java/cjworld/Interop.java`.


#### Step 5: Put it all together

1. Add the following files to your Android project:

    * The Java source files generated on the previous step - to the source tree
      under `src/main` in accordance with the package names.

    * The `.so` file generated on the previous step - to `src/main/jniLibs/arm64-v8a`
      (create those subdirectories if they don't exist).

    * All `.so` files from the appropriate subdirectory of `$CANGJIE_HOME/runtime/lib/`,
      such as `$CANGJIE_HOME/runtime/lib/linux_android31_aarch64_cjnative/` - also
      to `src/main/jniLibs/arm64-v8a`.

    * The file `libc++_shared.so` from the Android NDK - also
      to `src/main/jniLibs/arm64-v8a`.
      (That file is located
      in `toolchains/llvm/prebuilt/`_`host`_`/sysroot/usr/lib/aarch64-linux-android/`
      under the Android NDK root, where _`host`_ is a string corresponding
      to the O/S and hardware of the system on which you conduct the build,
      such as "`linux-x86_64`".

    * The file `$CANGJIE_HOME/lib/library-loader.jar` as a dependency.

2. **IMPORTANT: Enforce the legacy convention of compressing all `.so` files
   in the APK, or the application will crash trying to load Cangjie libraries.**

   In the Gradle build script of your Android application, usually called
   `build.gradle.kts`, locate the `android {}` block and insert the following
   option setting into it, if it is not already present:

   ```gradle
      .  .  .
   android {
          .  .  .
       packaging {
           jniLibs {
               useLegacyPackaging = true
           }
       }
   }
      .  .  .
   ```

3. Re-build the project to ensure everything is correct so far. You may
   also try deploying it.

4. Add the Java code working with interop classes and re-build your Android
   project again.

**Example:**

Now we can add an invocation of `Interop.m()` to our Java code, for control
to be passed to its Cangjie implementation in our interop class:

```java
       .  .  .
    B b = Interop.m(new A(), "Test", 0);
       .  .  .
```


### Calling Java from Cangjie

Once you have designed, built and integrated the interoperability layer
as described
in the [previous section](#initial-interop-class-creation-workflow),
you can add code that uses Java types to the member functions of
your interop classes. The type mapping is the same:

Cangjie Type (`T'`)           | Java Type (`T`)
----------------------------- | ---------------
`Bool`                        | `boolean`
`Int8`                        | `byte`
`Int16`                       | `short`
`UInt16`                      | `char`
`Int32`                       | `int`
`Int64`                       | `long`
`Float32`                     | `float`
`Float64`                     | `double`
`JObject` or `?JObject`       | `Object`
`JString` or `?JString`       | `String`
`T'` or `?T'`                 | `T`^\*^
`JArray<T'>` or `?JArray<T'>` | `T[]`^†^

^\*^ `T'` must be either a mirror type for the Java type `T`
or an interop class, for which the source code of its Java wrapper
class `T` was generated automatically by `cjc`.

^†^ `T'` must be either a mirror type, an interop class,
or one of the value types listed above, e.g. `Int32`.

**Limitations:**

* A variable arity parameter that has the Java type `T` is mirrored into a
  regular parameter of the type `?JArray<T'>`. In order to pass any arguments
  to such a parameter Cangjie code must explicitly create and initialize
  an instance of `JArray<T'>` before calling such a method or constructor.


#### Step 0: Build your Android application normally

#### Step 1: Generate mirror type declarations

Skip this step if you only want to call member functions of interop classes,
their superclasses and/or types for which you have already generated mirror
declarations, such as the types of interop class parameters or return values.

**Command line:**

```bash
java -Dpackage.mode=true -Dpackage.name=package-name \
    -jar ${CANGJIE_HOME}/tools/bin/java-mirror-gen.jar \
    --boot-class-path path-to-android-jar \
    --class-path full-application-classpath \
    -d output-directory \
    names-of-mirrored-types
```

or

```bash
java -Dpackage.mode=true -Dpackage.name=package-name -Djar.mode=true \
    -jar ${CANGJIE_HOME}/tools/bin/java-mirror-gen.jar \
    --boot-class-path path-to-android-jar \
    --class-path full-application-classpath \
    -d output-directory \
    jar-file
```

where

* `<package-name>` is the desired name of the Cangjie package for all
  mirrored types.

* `path-to-android-jar` is the pathname of the `android.jar` file in the
  Android SDK that you are using,
  e.g. `${ANDROID_SDK}/platforms/android-35/android.jar`.

* `<full-application-classpath>` is the complete classpath of the application
  that is used during its build, including `android.jar`.

* `<output directory>` is the pathname of the directory where the Cangjie source
  files containing mirror type declarations will be placed, such as `./src/cj`.

* `<names-of-mirrored-types>` are space-separated _fully qualified names_
  of _all_ Java reference types that want to use in the code of your interop
  classes, besides `java.lang.Object`, `java.lang.String` and array types.

* `<jar-file>` is the pathname of a single jar file. All `public` classes and
  interfaces, the `.class` files of which it contains, are mirrored, along
  with any dependencies found along the `<full-application-classpath>`.

**Example:**

Continuing the example from the
[previous section](#initial-interop-class-creation-workflow),
suppose we want our function `Interop.m()` to call the method
`String g(A a, int i)` of a class `com.example.c.C`, where `A`
is the type `com.example.a.A` that we already mirrored before:

```java
package com.example.c;

import com.example.a.A;

public class C {
    public static String g(A a, int i) {
        /* Some Java code returning a string */
    }
}
```

We re-run the mirror generator, passing `com.example.c.C` as an extra
(not sole!) type name to mirror:

```bash
java -Dpackage.mode=true -Dpackage.name=javaworld \
    -jar ${CANGJIE_HOME}/tools/bin/java-mirror-gen.jar \
    --boot-class-path /home/user/Android/Sdk/platforms/android-35/android.jar \
    --class-path /home/user/Android/Sdk/platforms/android-35/android.jar:App.jar \
    --d ./src/cj \
    com.example.a.A com.example.b.B com.example.c.C
```

This command will generate the same files with mirror type declaration
plus the file `src/javaworld/src/C.cj` and files with mirror type declarations
for any dependencies that type may have that were not previously mirrored.

The file `src/cj/javaworld/src/C.cj` would look like this:

```cangjie
package javaworld

import java.lang.*

@JavaMirror["com.example.c.C"]
public class C {
    public static func g(a: ?A, i: Int32): ?JString
}
```


#### Step 2: Import the mirror type(s) and write code

Make sure that all mirror type(s) involved are imported, then implement
the constructors and member functions of your interop classes, manipulating
Java types as if they were Cangjie types.

**Example:**

Continuing the example from the previous section, we can implement
the function `cjworld.Interop.m()` using the mirror type `javaworld.C`:

```cangjie
package cjworld

import java.lang.*
import javaworld.A
import javaworld.B
import javaworld.C

@JavaImpl
public class Interop {
    public static func m(a: ?A, s: ?JString, i: Int32): ?B {
        let s1: JString = match (a) {
            case Some(aa) => C.g(aa, i) ?? JString("")
            case None => JString("")
        }
        B(s1)  // Assuming there is a B(String) constructor
    }
}
```

#### Step 3: Recompile the Cangjie part

See [Step 4](#step-4-compile-interop-classes) of the interop class
creation workflow for details and example.

As long as you have not changed the public interface of the interop class(es)
on the previous steps, you only need to update the `.so` file.
The generated Java sources should be identical to previously generated ones.


#### Step 4: Update and re-build the Android project

Copy the `.so` file and, if applicable, `.java` files updated/generated
on the previous step over to your Android project and re-build it.

See [Step 5](#step-5-put-it-all-together) of the interop class
creation workflow for details and example.


### Features And Limitations of Interop Classes

1. An interop class _must_ be a direct subclass of a mirror class.
   By default, the interop class will inherit the mirror class
   [`java.lang.JObject`](#java.lang.jobject), not `std.core.Object`.

2. An interop class _may_ implement one or more mirror interfaces, but never
   a conventional Cangjie interface. Conversely, a conventional Cangjie type may
   not implement or inherit a mirrored Java interface.

3. An interop class may not be declared as `open` or `abstract`, may not be
   extended using `extend`, and may not be generic.

4. An interop class may introduce new instance fields _of any Cangjie type_,
   as they are not exposed to Java, and override the member functions of its
   mirrored superclass.


5. The constructors of an interop class may call superclass constructors
   using `super()`, but have the same limitation on instance member function
   calls as the conventional Cangjie constructors, as well as the requirement
   to initialize all newly introduced member variables.

6. The instance member functions of an interop class may call instance
   member functions which that interop class has inherited from its mirrored
   superclass, and/or use `super.` to call the member functions of that
   superclass that the interop class overrides.

7. The signatures of `public` constructors and member functions of interop
   classes can only use types that are (a) mirror types or interop classes
   themselves, (b) are 100% analogous to Java primitive types, or (c) the type
   `String`. This limitation also applies to the types of `public` properties.
   For details, see the Type Mapping table
   in [Calling Java from Cangjie](#calling-java-from-cangjie) and
   the [Automatic String Conversion](#automatic-string-conversion) subsection
   below.

The following two limitations are Android/JVM specific:

8. The Java counterparts of all mirror types and interop classes must all be
   loaded by the same classloader.

9. Unlike Java and other JVM languages, Cangjie does not permit circular import
   dependencies to exist between packages. That poses a challenge to the
   mirror generation process. Refer to the
   [Circular Import Dependencies Handling](#circular-import-dependencies-handling)
   subsection for details.


#### Automatic String Conversion

Java and Cangjie string types, `java.lang.String` and `std.core.String`, are
not binary compatible. A copying conversion is therefore required to enable the
code written in one language to work with string data originating
from the other. The built-in mirror type
[`java.lang.JString`](#java.lang.jstring) facilitates such conversions
by providing a special constructor that accepts a Cangjie string and uses its
UTF-8 character data to construct a Java string containing the same characters
in the UTF-16 encoding, and a member function `toString()` that does the reverse
and returns a Cangjie string:

```java
public class J {
    public static String s2s(String s) { ... }
}
```

```cangjie
@JavaMirror
public class J {
    public static func s2s(s: ?JString): ?JString
}

   .  .  .

    let s: String = J.s2s(JString("Cangjie string")).getOrThrow().toString()
```

Any Cangjie code that passes Java string values over to regular Cangjie
functions for further processing and/or transfers Cangjie strings to Java
would certainly look cleaner and more compact without the calls of the form
_`Java-string`_`.toString()` and `JString(`_`Cangjie-string`_`)`. However,
converting _all_ Java string values automatically as they cross the
inter-language border may lead to substantial CPU and memory overheads
in certain scenarios. If Cangjie code does not actually manipulate a Java
string, but merely passes it back to Java, or accesses it only in rare edge
cases, or does not use it at all, the conversion is a waste of computational
resources, which is undesirable in mobile applications.

Instead, interop class writers may enable such automatic conversion selectively
for specific constructor/member function parameters, member function return
values and member properties that need to have the Java type `String` when they
are exposed to Java. That is accomplished by permitting those entities to be
declared as having the Cangjie type `String` instead of `JString`. The only
limitation is that such member functions must not override methods of their
mirrored Java supertypes (see note below). The generated Java wrappers use the
Java `String` type in the respective places, and bridge code automatically
converts string values between the two types when the methods and constructors
of the wrapper class are invoked from Java.

As usual, if support for receiving/returning `null` values is required, the type
`?String` must be used instead of `String`.
See [Null Handling](#null-handling) for details.


**NOTES:**

1. `String` and `JString` are distinct types that are not in a subtype
   relationship. This means that member functions of interop classes that
   override methods of their mirrored Java supertypes should use the same
   parameter and return value type, `?JString` <!-- !!! and `?JArray<?JString>`, -->
   for the `override` modifier to work as expected:

   ```cangjie
   @JavaMirror
   public class J {
       public func f(s: ?JString): Unit
   }

   @JavaImpl
   public class CJ <: J {
       override public func f(s: ?String): Unit {}   // Error
       public func f(s: ?String): Unit {}            // Overloads, not overrides
       override public func f(s: ?JString): Unit {}  // OK
   }
   ```

   Although the manual replacement of `JString` with `String` in mirror type
   declarations is also supported, that practice is discouraged for the above
   reason, as the lack of subtyping may introduce unexpected, hard-to-diagnose
   override/overload ambiguities if such replacement is not done 100%
   consistently across all subtypes.

2. `String` is not supported as the type argument
   of [`java.lang.JArray<T>`](#java.lang.jarrayt),
   so Java string arrays have to be mapped to `JArray<JString>`, or, more
   likely, to `?JArray<?JString>`.



## Java to Cangjie Mapping

The current version of the mirror generator does the following Java
→ Cangjie conversions.


### General Considerations {#java-general-considerations}

The mirror generator takes Java _class_ files as input. Any information that
the Java compiler does not propagate from Java source code to class files
in the first place, or was removed afterwards, is therefore not available
to the mirror generator. Most notably, this concerns [generics](#java-generics)
and names of method parameters.


### Names {#java-names}

The original names of Java types, fields and methods are preserved to the
maximum possible extent. If the original name could not be preserved for
one of the reasons outlined below, it is propagated to the Cangjie compiler
via an annotation.

* Java identifiers that clash with Cangjie keywords, such as "`func`",
  "`main`"  or `Int32` are enclosed in backticks ` `` ` to form raw Cangjie
  identifiers:

  ```java
      public static final long Int32 = 0xffff_ffff;
  ```

  ```cangjie
      public static let `Int32`: Int64
  ```

* Java identifiers may contain characters that are not permitted in
  Cangjie identifiers, most notably the dollar sign `$`, which is used
  in binary names of nested Java types. Such characters are replaced
  with underscores `_`:

  ```java
  public class Outer {
      public class Inner {}
      public Inner getInner() { return new Inner(); }
  }
  ```

  ```cangjie
  @JavaMirror["Outer"]
  public open class Outer {
      public init()

      public open func getInner(): ?Outer_Inner
  }

  @JavaMirror["Outer$Inner"]
  public open class Outer_Inner {
      public init(p0: ?Outer)
  }
  ```

* Although it is considered bad practice, a Java type may have a field,
  a member type, and a method share the same simple name (identifier).
  This works, because it is always possible to determine the meaning of a name
  either by syntax (only method names can be followed by "`(`") or from context.
  In addition, instance and static methods of a type may have the same names
  in Java as long as their signatures are different.

  In Cangjie, however, all member variables and non-overloaded functions
  declared in a given scope must have distinct names. (There are no member
  types in Cangjie, so name clashes with member types simply never happen,
  as they are [mirrored into top-level types](#java-classes-and-interfaces).
  In particular, an instance member function and a static member function
  cannot have the same name.

  The mirror generator therefore appends "`_`_`type-name`_" to the names
  of instance variables and "`Static`" to the names of  `static` methods
  if their original Java names clash with other field/method names.
  It retains the original name as the value of the `@ForeignName` annotation:

  ```java
  public class Node {
      public int id;
      public Node(int id) { this.id = id; }
      public static int id(long x) { return (int)x; }
      public static int id(short x) { return x; }
      public int id() { return id; }
      public void id(int newId) { this.id = newId; }
  }
  ```

  translates to

  ```cangjie
  public open class Node {
      @ForeignName["id"]
      public var id_Node: Int32

      public init(arg0: Int32)

      @ForeignName["id"]
      public static func idStatic(arg0: Int64): Int32

      @ForeignName["id"]
      public static func idStatic(arg0: Int16): Int32

      public open func id(): Int32

      public open func id(arg0: Int32): Unit
  }
  ```

* Names of Java packages cannot be preserved in practice as circular import
  dependencies between packages, omnipresent in Java, are forbidden
  in Cangjie. See
  [Circular Import Dependencies Handling](#circular-import-dependencies-handling)
  for details.


### Primitive Types

Java primitive types are mirrored to the respective Cangjie value types:

Java Type        | Cangjie Type
---------------- | ------------
`boolean`        | `Bool`
`byte`           | `Int8`
`short`          | `Int16`
`char`           | `UInt16`
`int`            | `Int32`
`long`           | `Int64`
`float`          | `Float32`
`double`         | `Float64`


### Classes and Interfaces {#java-classes-and-interfaces}

Java class and interface _definitions_ are mirrored respectively into Cangjie
class and interface declarations annotated with `@JavaMirror`.
The value of that annotation is a string that retains the original
fully qualified name of the mirrored Java type. Simple names of mirrored
types are preserved unless they contain characters that are not permitted
in Cangjie identifiers. Such characters, most notably the dollar sign `$`,
the presence of which in a Java class file name is usually an indication
of the respective type being a nested type, are replaced with underscores `_`.

_Uses_ of Java class and interface types as types of mirrored fields,
and/or as types of parameters and return values of mirrored methods and
constructors, get wrapped in `Option<T>`
(see [Null Handling](#null-handling) for details.)

`@JavaMirror`-annotated declarations differ from conventional
Cangjie class/interface definitions in a few aspects:

* The root of the Java mirror classes hierarchy is not `std.core.Object`,
  but a built-in mirror class [`java.lang.JObject`](#java.lang.jobject),
  which is an immediate subclass of `std.core.Object`.

* The mirror for `java.lang.String` is also built-in, its name
  is [`java.lang.JString`](#java.lang.jstring).

* Only the symbolic information is mirrored. Variable initializers
  and function/constructor bodies are omitted.

**Example:**

For example, a mirror class for the following Java class:

```java
public class Node {
    public static final int A = 0xDeadBeef;
    private int id;
    public Node(int id) { this.id = id; }
    public int id() { return id; }
}
```

might look like this:

```cangjie
@JavaMirror["Node"]
public open class Node {
    public static let A: Int32
    public init(id: Int32)
    public func id(): Int32
}
```

Only `public` Java classes and interfaces are mirrored. Mirrors of non-`final`
classes are modified with `open`. The modifiers `sealed` and `non-sealed` are
ignored, as well as the legacy modifier `strictfp`.

`private` and package-private members and constructors, as well as static
and instance initializers, are not mirrored.

If the names of a Java type member and its mirror differ for one of the
reasons outlined in [Names](#java-names) the original Java member name
is propagated to Cangjie via the value of the `@ForeignName` annotation
on the mirror member:

```java
    CurrencyAmount priceInUS$Per(WeightUnit wu) { ... }
```

```cangjie
    @ForeignName["priceInUS$Per"]
    public open priceInUS_Per(arg0: WeightUnit): CurrencyAmount
```

> **NOTE:** The meaning of the access modifier `protected` is different
> in Java and Cangjie.
>
> In Java, access to a `protected` member or constructor of a class
> is permitted from anywhere within the same package and from subclasses
> defined in other packages.
>
> In Cangjie, that modifier also permits access from subpackages and
> from any package within the same module.
>
> Normally, this should not pose a problem.

**Fields** are mirrored into member variables of the respective mirror types.
Field names are preserved as long as they don't [clash](#java-names) with names
of other members. The modifiers `public`, `protected` and `static` are
preserved. The modifiers `transient` and `volatile` are ignored. `final` Java
fields are mirrored into `let` member variables, non-`final` — into `var`
member variables. Variable initializers are omitted.

**Methods** are mirrored into member functions with the respective mirror
types substituted for parameter types and return value type. Mirrors of `void`
methods have the `Unit` return type. Instance method names are preserved.
Static method names are preserved as long as they don't [clash](#java-names)
with instance method names. The modifiers `public`, `protected` and `static`
are preserved. The modifier `default` is replaced with the annotation
`@JavaHasDefault` (see below). The modifiers `native` and `synchronized` are
ignored, as well as the legacy modifier `strictfp`. Mirrors of non-`final`
methods are modified with `open`. The last parameter _`T`_`...`_`name`_
of a variable arity method is mirrored as if its type was _`T`_`[]`.

**NOTE:** `default` and `static` interface methods are not supported when
target O/S versions include Android 6 (`minSdk` set to 23).
See [Android Versions Support](#android-versions-support) for details.

**Constructors** are mirrored into `init` constructors with the respective
mirror types substituted for parameter types. _This includes the default
constructor that might have been implicitly declared._ The modifiers `public`
and `protected` are preserved. The last parameter _`T`_`...`_`name`_
of a variable arity constructor is mirrored as if its type was _`T`_`[]`.

**NOTES:**

1. Mirror classes may not have primary constructors.

2. Mirror classes that contain no `init` constructors do _not_ get a default
   one defined implicitly, unlike the conventional Cangjie classes, and thus
   cannot be instantiated by calling a constructor. For automatically
   generated mirror classes, the absence of a constructor usually means that
   the mirrored Java class declares only `private` or package-private
   constructors so as to prevent its instantiation by arbitrary code.
   See also [Enum Classes](#enum-classes).

3. The Java mirror generator takes _class files_ as input, and the
   names of method/constructor parameters are often absent in class files.
   In that case, the mirror generator will assign them names `arg0`, `arg1`,
   and so on. The `javac` compiler option `-parameters` forces parameter names
   retention, but only for classes, not interfaces. Same goes for the debug
   information generation option `-g` / `-g:vars`.

**Member types** are mirrored into the respective top-level types, as Cangjie
does not support type nesting. The name of a member type mirror is derived
from the _binary_ name of the original member type, which consists of the
binary name of its immediately enclosing type, followed by a dollar sign `$`,
followed by the simple name of the member type itself. However, the dollar sign
is not permitted in Cangjie identifiers, so it is replaced with an underscore
`_` (see also [Names](#java-names)). The modifiers `public` and `protected` are
preserved. The modifier `static` is ignored. `init` constructors of mirrored
inner classes have an extra parameter for passing the enclosing instance
(that parameter is implicit in Java):

```java
public class Outer {
    public class Inner {}
    public static class Static {}
    public Inner getInner() { return new Inner(); }
    public Static getStatic() { return new Static(); }
}
```

```cangjie
@JavaMirror["Outer"]
public open class Outer {
    public init()

    public open func getInner(): ?Outer_Inner
    public open func getStatic(): ?Outer_Static
}

@JavaMirror["Outer$Inner"]        // Original binary name is retained
public open class Outer_Inner {   // '$' is replaced with '_'
    public init(p0: ?Outer)       // Extra parameter for the enclosing instance
}

@JavaMirror["Outer$Static"]       // Original binary name is retained
public open class Outer_Static {  // '$' is replaced with '_'
    public init()
}
```

Bodies are omitted in the mirrors of _all_ methods and constructors, so they
all look like abstract member functions in regular Cangjie code. That
imposes the following syntax alterations:

* The modifier `abstract` on _class_ methods is preserved, as otherwise
  there would have been no way to tell apart the mirrors of abstract and
  concrete Java methods:

  ```java
  public abstract class A {
      public void c() {}
      public abstract void a();
  }
  ```

  ```cangjie
  @JavaMirror["A"]
  public abstract class A {
      public init()

      public open func c(): Unit

      public open abstract func a(): Unit
  }
  ```

* For the same reason, mirrors of default interface methods are annotated
  with `@JavaHasDefault`.

  ```java
  public interface I {
      default void c() {}
      void a();
  }
  ```

  ```cangjie
  @JavaMirror["I"]
  public interface I {
      @JavaHasDefault
      func c(): Unit

      func a(): Unit
  }
  ```

In methods and constructors with a variable number of parameters, `, ...` is
ignored.


#### Mirror Type Inheritance

Mirror classes and interfaces form separate subtype hierarchies, which
means that:

* The root of the Java mirror class hierarchy is not `std.core.Object`,
  but a built-in mirror class [`java.lang.JObject`](#java.lang.jobject).

* Mirror interfaces may inherit other mirror interfaces, reflecting
  the inheritance relationships between the original Java interfaces.
  Mirror interfaces may not inherit regular Cangjie interfaces other than
  the implicitly inherited `Any` and vice versa.

* Mirror classes may inherit other mirror classes, reflecting
  the inheritance relationships between the original Java classes.
  Mirror classes may not inherit regular Cangjie classes and
  vice versa.

* Mirror classes may implement mirror interfaces, but not regular
  Cangjie interfaces. The interface `Any` is an exception; all mirror
  classes implement it implicitly. Regular Cangjie classes may not implement
  mirror interfaces.

* Neither mirror classes may be extended using `extend`, nor any other type
  may be interface-extended with a mirror interface.

* In Java, all interfaces are subtypes of the `java.lang.Object` class. That
  works because only classes may implement interfaces in Java. The latter is not
  the case in Cangjie, where all interfaces are subtypes of the built-in `Any`
  interface, which is _not_ a subtype of `std.core.Object`. Hence, mirror
  interfaces are _not_ subtypes of [`java.lang.JObject`](#java.lang.jobject)
  and the below method `test()` may not be re-written in Cangjie:

  ```java
  public interface I {}

  public class C {
      public static void accept(Object o) {}
      static void test(I i) { accept(i); }     // Works in Java
  }
  ```


#### Generics {#java-generics}

Java generics are erased during compilation, and type arguments are not always
present in class files anyway, so the automatically generated mirror types are
non-parameterized, as if the raw types were mirrored. In particular, type
variables are replaced with mirrors of their leftmost bounds.

It is not possible to declare a parameterized mirror type manually either.
The built-in type `JArray<T>` (see [Arrays](#arrays)) is a special exception.

Mirror types may not be used as type arguments of conventional generic Cangjie
types [other than `Option<T>`](#null-handling).


### Arrays

The special built-in type `java.lang.JArray<T>` is used to represent
mirrors of Java arrays.

A Java array of type _`T`_ (_`T`_`[]`), is generally mirrored into:

* `?JArray<`_`T'`_`>`, if _`T`_ is a primitive type

* `?JArray<`_`?T'`_`>`, if _`T`_ is a reference type

where _`T'`_ is the mirror of _`T`_.
Refer to [Null Handling](#null-handling) for the reasoning
behind `Option<T>` wrapping.

**NOTE:** Java arrays are covariant, whereas Cangjie generics are
invariant, and the class `JArray<T>` is no exception.


### Enum Classes

A Java enum class _`E`_ is mirrored into a Cangjie mirror class _`E'`_ that
inherits the mirror of `java.lang.Enum` and is neither `open` nor `sealed`
itself, thus cannot be extended.

The class _`E'`_ contains:

* Mirrors of the enum constants of _`E`_, in the form of public static
  member `let`-variables of type _`E'`_.

* No constructors, for the instantiation of the mirror class
  to be impossible. (Mirror classes do not have default constructors.)

* Mirrors of the implicitly defined methods
  `public static `_`E`_`[] values()`
   and `public static `_`E`_` valueOf(String name)`

* Mirrors of any `public`/`protected` fields and methods of the original
  enum class as if they were fields and methods of an [ordinary class](#java-classes-and-interfaces)


### Record Classes

Java record classes are essentially syntactic sugar and are mirrored just
like the equivalent regular classes
(see [Classes And Interfaces](#java-classes-and-interfaces)).

A Java record class definition effectively defines a regular Java class that:

* Is `final`
* Is not `abstract`
* Is a direct subclass of `java.lang.Record`
* Contains one or more component fields: `private` instance fields, each with
  a matching accessor method
* May contain `static` fields, but not instance fields other than component
  fields
* Contains a canonical constructor with parameters corresponding to component
  fields
* May contain other constructors
* Overrides `java.lang.Object` methods `equals()`, `hashCode()` and `toString()`
* Does not override other `java.lang.Object` methods

**NOTE**: The methods `hashCode()` and `toString()` are renamed in the built-in
mirror of `java.lang.Object`. See  [`java.lang.JObject`](#java.lang.jobject)
for details.

**Example:**

```java
public record Node (int value, Node next) {}
```

```cangjie
@JavaMirror["Node"]
public class Node <: Record {
    public init(value: Int32, next: ?Node)

    public func toJString(): JString
    public func hashCode32(): Int32
    public func equals(o: ?JObject): Bool

    public func value(): Int32
    public func next(): ?Node
}
```

### `null` Handling

Cangjie has no concept of null references and hence no equivalent for the Java
null type. If Java reference types were mirrored direct to the respective
mirror types, any `null` value returned to Cangjie from a Java method would
result in a `NoneValueException`. That would also happen if Cangjie code
accessed a member variable of such type mirroring a field containing `null`.
Conversely, if Java called an [interop class](#interop-classes) method and
passed `null` as one of the parameters, the Java `NullPointerException` would
get thrown.

Finally, there also has to be a way to pass a `null` value from Cangjie
to a Java method or constructor designed to accept it.

To enable safe passing of Java `null` values across the inter-language border
and within Cangjie code, the mirroring rules for Java reference types are
extended as follows:

If the original Java type of a field, array element, method/constructor
parameter or method return value is a reference type _`R`_, the automatically
generated mirror of that _entity_ has the type `Option<`_`R'`_`>`, where _`R'`_
is the mirror of the _type_ _`R`_. On the Cangjie side, `None` stands for the
`null` value, and `Some(`_`r'`_`)` represents a (non-null) reference value
_`r'`_ of the type _`R'`_. The `cjc` compiler recognizes `Option<T>`
as a [Java-compatible type](#java-compatible-types) if `T` is a mirror type
and wraps/unwraps the values of `T` accordingly.

For instance, consider the following Java interface:

```java
public interface I {
    Object f(Object[] xs);
}
```

The `xs` parameter itself may be `null`, each element of the `xs` array may be
`null` and the `f` method may return `null`. Therefore, the _safe_ way
to mirror that interface is as follows:

```cangjie
@JavaMirror
public interface I {
    func f(xs: ?JArray<?JObject>): ?JObject
}
```

`Option<T>` wrapping also fully applies to [interop classes](#interop-classes),
as values of such types may be passed to/from Java. Furthermore, its application
to the types of parameters of interop class member functions and constructors
exposed to Java and to the types of return values of such member functions is
recommended.

And for the same reason, you should use `Option<T>` wrapping when declaring
a local variable of a foreign type in the code of an interop class, unless you
are 100% sure that it won't be assigned `null`.

```cangjie
    // Suppose M is a Java mirror type
    let m: M = M()     // If M() returns successfully, it is guaranteed
                       // to have returned a new instance of M
```

All in all, `Option<T>` wrapping ensures that the code won't break if a `null`
value sneaks into the Cangjie world from the Java one, but it does that at the
cost of performance and memory footprint. The other disadvantages of this
approach are the [loss of variance](#loss-of-variance) and more cumbersome
[type testing and conversion](#null-aware-type-testing-and-conversion).



#### Loss of Variance

One limitation imposed by the [`Option<T>` wrapping](#null-handling)
of Java mirror types and interop classes is that such wrapped types follow
the semantics of Cangjie in all other respects. In particular, `Option<T>` is
_invariant by its type parameter `T`_: `Option<U>` is not a subtype
of `Option<T>` if `U` is a subtype of `T`, unless `U` and `T` are the same
type. For mirror types that means that any overriding Java method that relies
on return type covariance may not be mirrored that way with `Option<T>`
wrapping. Its return type has to be propagated from the superclass method.

**Example:**

Suppose a Java class `Foo` is a direct superclass of the class `Bar`:

```java
public class Foo {}

public class Bar extends Foo {}
```

and the interface `C` declares a method `get` that returns an instance of `Foo`:

```java
public interface C {
    public Foo get();
}
```

A subinterface of `C` may then override `get` with a more precise return type,
`Bar`:

```java
public interface D extends C {
    @Override
    public Bar get();
}
```

Without `Option<T>` wrapping, all those types could be mirrored to:

```cangjie
@JavaMirror
public open class Foo {}

@JavaMirror
public open class Bar <: Foo {}

@JavaMirror
public interface C {
    public open func get(): Foo
}

@JavaMirror
public interface D <: C {
    public override open func get(): Bar       // Return type covariance in action
}
```

but if `get()` can possibly return `null`, an exception is inevitable.

`Option<T>` wrapping makes it safe, but the return types of all overriding
methods have to be lowered to the return type of the original method:

```cangjie
@JavaMirror
public open class Foo {}

@JavaMirror
public open class Bar <: Foo {}

@JavaMirror
public open interface C {
    public open func get(): ?Foo
}

@JavaMirror
public open interface D <: C {
    // public open func get(): ?Bar    // Error, `Option<T>` is not covariant by T
    public open func get(): ?Foo       // OK, but the return type is lowered
}
```

Nullability annotations could partially alleviate the problem, but the current
version of the Cangjie SDK does not support them.


#### `null`-aware Type Testing And Conversion

**IMPORTANT:** Nullable values of foreign types, represented using `Option<T>`
wrapping, need to be null-tested and unwrapped before type testing and
conversion, for two reasons:

1. Cangjie generics are invariant with respect to their type arguments,
   so _`e`_` is Option<`_`T`_`>` evaluates to `true` only if the type of the
   expression _`e`_  is `Option<`_`T`_`>` specifically,
   not some `Option<`_`U`_`>` where _`U`_` <: `_`T`_.

   For the same reason, you cannot upcast or downcast `Option<T>`-wrapped
   values:

   ```cangjie
   open class Foo {}
   class Bar <: Foo {}
      .  .  .
       let bar: Bar = Bar()
       let foo: Foo = bar                         // OK
       let bar2: Bar = (foo as Bar).getOrThrow()  // OK


       let maybeBar: ?Bar = Some(Bar())
       let maybeFoo: ?Foo = maybeBar              // Error: mismatched types
       let maybeBar2: ?Bar = (maybeFoo as Option<Bar>).getOrThrow()
                                                  // Throws NoneValueException
   ```

   > **HINT:** Use the `Option` type member functions `.map()` and `.flatMap()`
   > to write shorthands for upcasts and downcasts respectively:
   >
   > ```cangjie
   >     // Upcast (from subtype to supertype):
   >     // - Lambda return type is inferred from variable type, 'as' not needed
   >     let maybeFoo: ?Foo = maybeBar.map{ bar => bar }
   >
   >     // Downcast (from supertype to subtype):
   >     // - Lambda return type is inferred from lambda body
   >     // - flatMap() removes excessive Option wrappers, unlike map()
   >     let maybeBar2 = maybeFoo.flatMap{ foo => foo as Bar }
   > ```

2. In Java, `null instanceof `_`T`_ evaluates to `false` for any reference
   type _`T`_. In Cangjie, _`e`_` is Option<`_`T`_`>` evaluates to `true`
   if _`e`_ evaluates to `Option<`_`T`_`>.None`.

   > **HINT:** The semantics of the Java type comparison operator `instanceof`
   > may be replicated in Cangjie as follows:
   >
   > ```java
   > // Java:
   > void f(o: Object) {
   >     if (o instanceof T) { ... }
   > }
   > ```
   >
   > ```cangjie
   > // Cangjie:
   > func f(o: ?JObject): Unit {
   >     if (let Some(t) <- o && t is T) { ... }
   > }
   > ```



### Foreign Types Conversion And Testing

The Cangjie operators `is` and `as` are supported for all
[foreign types](#foreign-types), as well as the type pattern _`v`_`: `_`T`_
in match expressions (but not in if-let and while-let expressions yet).
Note, however, that their semantics matches that of the Java type comparison
operator `instanceof` and cast expressions, and in the general case the actual
type testing and conversion is conducted in the JVM.


**IMPORTANT:** Nullable values of mirror types and interop classes, represented
using `Option<T>` wrapping as described
in [`null` Handling](#null-handling), need to be null-tested and unwrapped
before type testing, for two reasons:

1. Cangjie generics are invariant with respect to their type arguments,
   so _`e`_` is ?`_`T`_ evaluates to `true` only if the type of _`e`_
   is `Option<`_`T`_`>` specifically, not some `Option<`_`U`_`>` where
   _`U`_` <: `_`T`_.

2. In Java, `null instanceof `_`T`_ evaluates to `false` for any reference
   type _`T`_. In Cangjie, _`v`_` is ?`_`T`_ evaluates to `true` if _`v`_
   equals `Option<`_`T`_`>.None`.

> The semantics of the Java type comparison operator `instanceof` may be
> replicated in Cangjie as follows:
>
> ```java
> // Java:
> void f(o: Object) {
>     if (o instanceof T) { ... }
> }
> ```
>
> ```cangjie
> // Cangjie:
> func f(x: ?JObject): Unit {
>     if (let Some(t) <- o && t is T) { ... }
> }
> ```



### Circular Import Dependencies Handling

Circular dependencies between packages are not only possible, but omnipresent
in Java. For instance, the ubiquitous class `java.lang.String` depends on
the class `Charset` from the package `java.nio.charset` and the class `Locale`
from `java.util`. And those two classes of course depend on the class `Object`
from the package `java.lang`, to which `String` belongs!

Compilation to separate `.class` files and late binding effectively circumvent
this problem in Java. For a Cangjie import declaration to compile, however,
the entire package that it imports (from) must be already compiled.
Consequently, all Cangjie source files that belong to a given package must be
compiled together in one session into a single binary. One can say that the
minimum unit of compilation in Cangjie is an entire package, not an individual
file. Hence circular import dependencies between packages are not possible
in Cangjie code.


#### Single-Package Mode

The above [difference](#circular-import-dependencies-handling) between the
two languages precludes the preservation of Java package names during mirror
generation. In practical terms, the mirror generator must _always_ collect all
generated mirror types in a single Cangjie package. The desired name of that
package must be specified with the system property `package.name`.

For instance, if you run the mirror generator with the following system
property:

`    -Dpackage.name=java.world`

it will place all generated mirror type declarations in the `java.world`
package and propagate the fully qualified names of the original Java types
to the `cjc` compiler as the arguments of `@JavaMirror` annotations:

```cangjie
package java.world

import java.lang.*

@JavaMirror["java.lang.Cloneable"]
public interface Cloneable {
}
```


#### Name Clashes

Java types declared in different packages may have the same simple name,
because their fully qualified names are different. For example, there is
a class `Attribute` in the JDK package `javax.management` and an interface
`Attribute` in `javax.naming.directory`. Their mirrors cannot co-exist
in a single Cangjie package. Therefore, the mirror generator detects such
name clashes when operating in the [single-package mode](#single-package-mode)
and mangles _all_ conflicting names. Specifically, it uses the fully qualified
name of each such type in place of its simple name, replacing dots `.`
with underscores `_`:

```cangjie
// src/java/world/src/javax_management_Attribute.cj
package java.world

import java.lang.*

@JavaMirror["javax.management.Attribute"]
public open class javax_management_Attribute <: Serializable {
   .  .  .
```

```cangjie
// src/java/world/src/javax_naming_directory_Attribute.cj
package java.world

import java.lang.*

@JavaMirror["javax.naming.directory.Attribute"]
public interface javax_naming_directory_Attribute <: Cloneable & Serializable {
   .  .  .
```


#### Incremental Mirroring

Putting all Java mirror types into a single Cangjie package such
as `java.world` not only leads to [name clashes](#name-clashes), but
also increases the compile-time overheads and is generally inconvenient.
Breaking that mass of declarations down into several packages is
therefore highly desirable. Fortunately, not all Java packages are
in a single import dependency circle, so at least some breaking down is
possible.

First of all, you may notice that there are three sets of packages in any
Android application that may not have circular dependencies between them:

* Application packages
* Android API packages
* JDK API packages

This means that even in the worst possible case all mirror types could be
segregated into three Cangjie packages, respectively code-named "`app`",
"`android`" and "`java`".

Furthermore, the JDK API has been modularized since Java 9, and circular
import dependencies may _not_ exist between Java modules. Therefore it is
perfectly possible to replicate the structure of JDK API _modules_
during mirroring. For instance, the Cangjie package `java.base` would
then contain mirrors of types defined in the packages exported from the
`java.base` JDK API module: `java.lang` and its subpackages,
`java.io`, `java.math`, and so on.

Finally, a third party library should have no circular import dependencies
with other application components and APIs. If your application contains
any such libraries that you also need to use from Cangjie code, you may
want to mirror each library into its own Cangjie package.

The _incremental mirroring_ process enables such segregation. You run
the mirror generator _once for each target Cangjie package_ (passed over via the
`package.name` system property, as usual), each time specifying two additional
parameters:

1. A _complete_ list of Java packages to mirror into that specific
   target Cangjie package. Those Java packages may have circular
   import dependencies between them, and may also import _previously_
   mirrored packages/types.

2. A cumulative list of fully qualified name mappings for all _previously_
   mirrored Java types. Initially this list is empty.

The mirror generator emits mirrors for all public classes and interfaces
that belong to the packages appearing on the list (1), and any dependencies
they may have that are _not_ already on the list (2). Then it appends all
newly established Java → Cangjie name mappings to the list (2).


##### Command-line Syntax {#incremental-mirroring-command-line-syntax}

Incremental mirroring is only supported in the single-jar mode at present,
which means that the [mirror generator command line](#jmg-command-line-syntax)
must set system property `jar.mode` to `true` and pass the pathname of a
jar file as an argument, not fully qualified type names:

```shell
java -Djar.mode=true \
    system-properties \
    -jar ${CANGJIE_HOME}/tools/bin/java-mirror-gen.jar \
    options \
    jar-file
```

_`system-properties`_ must include the following:

* `-Dpackage.mode=true`

* `-Dpackage.name=`_`target-package-name`_

  _`target-package-name`_ is the name of the target Cangjie package.
  The mirror generator will place _all_ mirror types into that package.

  **IMPORTANT: You must specify a new, previously unused target Cangjie
  package name every time you run the mirror generator in the incremental mode.
  If you mirror some types into, say, `my.java.libs` package in the first
  run of the mirror generator using `-Dpackage.name=my.java.libs` and
  run it again against different input but without changing the value
  of `package.name`, that would completely invalidate the results
  of mirroring, making them inconsistent.**

  > In a certain sense, a target Cangjie package is the minimal unit
  > of mirroring, just like a source Cangjie package is the minimal unit
  > of compilation when you develop in Cangjie: you cannot compile two
  > Cangjie source files belonging to the same package separately.

* `-Djar.mode.packages=`_`pathname`_

  _`pathname`_ must point to a plain text file that contains a list
  of Java package names. Specifically, each line of that file must be
  a fully qualified package name, optionally followed by a wildcard `.*`:

  ```text
  com.example.model
  com.example.ui.*
     .  .  .
  ```

  **NOTICE:** If the wildcard is present, that entry matches all subpackages
  of the named package _and_ that package itself. This is different from
  the semantics of the wildcard in Java and Cangjie import statements.

  The generator will emit mirrors for _all_ non-anonymous non-private types
  defined in the listed packages _and_ all their dependencies _except_
  for the dependencies that are already present in the import mappings file,
  if any. All those mirror types will be placed in a single Cangjie package
  specified using the `package.name` system property (see above).

  This option may only be used in the single-jar mode.

* `-Dimports.config=`_`import-mappings-file`_

  _`import-mappings-file`_ is the pathname of the _import mappings file_,
  a plain text file that accumulates a set of Java to Cangjie fully qualified
  name mappings for all mirror types as they are generated. The mirror generator
  reads that file at startup and does _not_ mirror the Java types for which
  mappings already exist, assuming that they were mirrored on its prior
  invocation(s). If mirroring completes without errors, the mirror generator
  adds the mappings for all mirrors it has just generated to the set and writes
  it out to the `./imports_config.txt` file before termination.

  **NOTE:** If _`import-mappings-file`_ is set to `./imports_config.txt`, that
  file will get overwritten. Make a copy after each incremental run of the mirror
  generator if you want to keep the intermediate sets of mappings,
  e.g. for debugging purposes.


#### Example: JDK API Mirroring

As mentioned [above](#incremental-mirroring), the JDK API is modularized,
so one could look at the
[list of modules](https://docs.oracle.com/en/java/javase/17/docs/api/index.html)
and their dependency graphs, collect lists of exported packages, and run the
mirror generator using the following command line template:

```bash
java \
    -Dpackage.name=java.base \
    -Djar.mode.packages=./java.base.txt \
    -Dimports.config=./imports_config.txt \
    -jar ${CANGJIE_HOME}/tools/bin/java-mirror-gen.jar \
    --boot-class-path /path/to/android.jar \
    -d src/cj \
    --class-path /path/to/android.jar:/path/to/android/SDK/platform/files/directory \
    /path/to/android.jar
```

The file `./java.base.txt` should contain the list of packages exported
from the `java.base` JDK API module _for public use_:

```text
java.io
java.lang
java.lang.annotation
   .  .  .
```

and the file `./imports_config.txt` should be empty or non-existent.

Upon the successful completion of the above command:

* The directory `./src/cj/java/base/src` will contain Cangjie source files
  with mirror type declarations for all public classes and interfaces
  defined in the packages listed in `./java.base.txt`, and

* The file `./imports_config.txt` will contain mappings of fully qualified
  names of all those Java types to the names of their Cangjie mirrors in the
  `java.base` package, for use in subsequent mirror generator invocations.

> **NOTICE:** The _implementations_ of many types exported from the module
> `java.base` depend on classes and interfaces defined in internal JDK
> packages, such as `sun.util.locale.provider.LocaleDataMetaInfo`
> or `jdk.internal.misc.Unsafe`. However, mirror types are _declarations_;
> they do not expose any private members of the respective Java types nor
> the implementation details of their public methods or constructors.
> So the mirror generator will not process the class files that belong
> to such internal packages at all.

Now we can pick a JDK API module that depends only on the `java.base`
module, e.g. `java.xml`, and run the mirror generator again, changing
only the values of two options:

* Change the value of `package.name` to `java.xml`

* Change the value of `jar.mode.packages` to the pathname of a file
  containing the list of packages exported from the `java.xml` JDK API
  module:

  ```text
  javax.xml.*
  org.w3c.dom
     .  .  .
  ```

Note that it is possible to put a wildcard `.*` after a package name to denote
that package _and_ all its subpackages at once. In the above excerpt,
`javax.xml.*` matches the package `javax.xml` itself, as well as
 `javax.xml.catalog`, `javax.xml.transform`, `javax.xml.transform.stream`,
and so on.

```bash
java \
    -Dpackage.name=java.xml \
    -Djar.mode.packages=./java.xml.txt \
    -Dimports.config=./imports_config.txt \
    -jar ${CANGJIE_HOME}/tools/bin/java-mirror-gen.jar \
    --boot-class-path /path/to/android.jar \
    -d src/cj \
    --class-path /path/to/android.jar:/path/to/android/SDK/platform/files/directory \
    /path/to/android.jar
```

`./src/cj/java/xml/src` should now contain mirror declarations for all
`java.xml` module exports, whereas `./imports_config.txt` should now contain
mappings for _both_ `java.base` and `java.xml` Cangjie packages.

Repeat this process for the remaining modules, in any order that ensures
mirroring of all dependencies of each module before mirroring that module
itself.



#### Using Incremental Mirroring

You can use [incremental mirroring](#incremental-mirroring) to segregate
arbitrary components of the application code that need to be used
from Cangjie, as long as there are no circular import dependencies between
those components and/or the rest of the Java code. In particular, the public
classes and interfaces of each third party Java library that Cangjie code
needs to use can be mirrored separately into an appropriately named Cangjie
package.

To accomplish that, first break down the set of packages constituting
the application into subsets that have no circular import dependencies
between them. Then use the incremental mirroring process described above:

1. Pick any subset that has no import dependencies on packages
   from other subsets. Mirror it all at once into an appropriately named
   package.

2. Mirror any subset that only depends on zero or more of the previously
   mirrored subsets.

3. Repeat the previous step until all Java types that you want to use in
   Cangjie code are mirrored, _each time picking a new target Cangjie
   package name_.


### Closure Depth Limiting

In addition to numerous
[circular import dependencies between packages](#circular-import-dependencies-handling),
another property is inherent to the standard Java API and many popular Java
libraries and frameworks: enormous sizes of import closures.

For instance, invoking the mirror generator to produce a mirror for an empty
enum type

```java
public enum E {}
```

makes it also generate mirrors for `java.lang.Enum` _and all its dependencies_
from the standard Java library, about 300 mirror types total:

```text
AbstractInterruptibleChannel.cj
AbstractStringBuilder.cj
AccessControlContext.cj
AccessMode.cj
AccessibleObject.cj
   .  .  .
ZoneOffsetTransitionRule.cj
ZoneRules.cj
ZonedDateTime.cj
```

For all those mirror types, the compiler will generate glue code, most of which
a real program will never use. That may be tolerated in a desktop or server
environment, but not on a mobile platform such as Android.

Asking the developer to supply an exact list of types and members to mirror
would be too much. Fortunately, the simple _limiting of depth_ during the
calculation of dependency closures yields reasonably good results: the number
of generated mirrors can drop by an order of magnitude or more. For instance,
mirroring the above empty enum type `E` with depth limited to 2 produces just
six mirror types in addition to the mirror of the type `E` itself: the enum
base class `java.lang.Enum`, three interfaces that it implements, and the
return types of two of its methods:

```text
Class.cj
Comparable.cj
Constable.cj
E.cj
Enum.cj
Optional.cj
Serializable.cj
```

Here is how depth limiting works exactly:

* The set of mirrored _types_ always includes:

    - Primitive Java types,
      [mirrored into the respective Cangjie value types](#primitive-types);
    - [Class and interface types](#java-classes-and-interfaces) explicitly
      specified on the mirror generator command line;
    - `java.lang.Object` and `java.lang.String`, [pre-mirrored respectively
      into `JObject` and `JString`](#interop-library-api-reference); and
    - [Array types](#arrays), the element types of which are among the above
      listed types, mirrored into `JArray<T>`.

* _Members_ of types included in the set are then filtered as follows:

    - Inherited members are _not_ mirrored as "own" members even if the
      respective supertypes are not included in the set of mirrored types.

    - Fields, the types of which are not included in the set of mirrored
      types, are omitted.

    - Methods, the signatures of which contain types that are not included
      in the set of mirrored types, are omitted.

    - Member types are treated just like top-level types: unless they are
      explicitly specified on the mirror generator command line, they only
      get mirrored if included in the set of mirrored types during the
      limited-depth dependency closure calculation process described here.

* The [command-line option](#jmg-command-line-syntax) `-Dgen.closure.depth`
  sets the depth limit for the explicitly specified class and interface
  types.

* If the depth limit for a given type _`T`_ is zero, _no types on which `T`
  depends get added to the set_. Even the supertypes of _`T`_ are not added.

  **Example:**

  ```java
  // A.java
  public class A {
      public void f(String s) {}
  }

  // B.java
  public class B extends A {
      public void g(String s) {}
  }
  ```

  If the mirror generator is instructed to mirror the class `B` with depth
  limited to 0, it will only generate a mirror for `B` and its method
  `g(String)`, because `JString` is automatically included in the set of
  mirrored types. At the same time, the method `f(String)` that `B` inherits
  from `A` will not be available:

  ```cangjie
  // B.cj
     .  .  .
  @JavaMirror["B"]
  public open class B {
      public init()

      public open func g(s: ?JString): Unit
  }
  ```

* If the depth limit for a given type _`T`_ is any positive _`N`_, the set
  of mirrors _additionally_ includes the following types, with the depth
  limit of _`N`_`-1`:

    - All (that is, recursively collected) supertypes of _`T`_;
    - Types of the non-`private` fields _declared in `T` itself_,
      that is, not the types of inherited fields;
    - Types of parameters of the non-`private` constructors of _`T`_; and
    - Types of parameters and return values of the non-`private` methods
      _declared in `T` itself_. Note again that the inherited methods
      are _not_ scanned.

  If a type is already included in the set, but with a lower depth limit,
  the latter is changed to _`N`_`-1`.

  The process is repeated recursively for all types newly added to the set
  of mirrors.

  **Example:**

  ```java
  // A.java
  public class A {
      public void f(C c) {}
  }

  // B.java
  public class B extends A {
      public void g(D d) { }
  }

  // C.java
  public class C { }

  // D.java
  public class D extends C {}
  ```

  If the mirror generator is instructed to mirror the class `B` with depth
  limited to 1, it will generate mirrors for `B`, its superclass `A` and
  the class `D` that is the type of the sole `B.g(D)` method parameter.
  However, `A` and `D` will be mirrored with depth limit set to 0, so the
  class `C`, and hence the method `A.f(C)` will not be mirrored:

  ```cangjie
  // A.cj
     .  .  .
  @JavaMirror["A"]
  public open class A {
      public init()
  }

  // B.cj
     .  .  .
  @JavaMirror["B"]
  public open class B <: A {
      public init()

      public open func g(d: ?D): Unit
  }

  // D.cj
     .  .  .
  @JavaMirror["D"]
  public open class D {
      public init()
  }
  ```

  Finally, setting the depth limit to 2 will make the mirror generator
  include mirrors of the class `C` and the method `A.f(C)` in its output.



## Java Mirror Generator Reference

### Prerequisites

The Java mirror generator requires a copy of JDK 17 to run.
If you do not have JDK 17 installed and added to your executable search `PATH`
install and/or add it before using the Java mirror generator.


You also need to know the _local_ pathnames of the jar files and directories
that contain all dependencies of the classes you are going to mirror. This
includes the Android standard library jar as well as the jars that will appear
on the classpath when your application launches.


### Command-line Syntax {#jmg-command-line-syntax}

The Java mirror generator can be run in two ways:

`java [ `_`system-properties`_` ] \`\
`    -jar ${CANGJIE_HOME}/tools/bin/java-mirror-gen.jar \`\
`    [ `_`options`_` ] `_`type-names`_`                                  `_(default mode)_

to generate mirrors for one or more Java classes and/or interfaces _and_ all
their dependencies, _or_

`java -Djar.mode=true [ `_`system-properties`_` ] \`\
`    -jar ${CANGJIE_HOME}/tools/bin/java-mirror-gen.jar \`\
`    [ `_`options`_` ] `_`jar-file`_`                                    `_(single-jar mode)_

to generate mirrors for all types the `.class` files of which the given
_`jar-file`_ contains _and_ all their dependencies residing elsewhere
on the supplied class path, if any,

where

* _`system-properties`_ are certain Java system properties that the
  mirror generator recognizes, set using the `-D` option of the `java`
  launcher.

* _`options`_ are the [command-line options](#command-line-options)
  of the mirror generator.

* _`type-names`_ are the _fully qualified names_ of Java classes and
  interfaces that need to be mirrored (that is, _not_ pathnames
  of their `.class` files).

* _`jar-file`_ is the pathname of a single jar file.

**NOTE:** ([Incremental mirroring](#incremental-mirroring) and/or
[closure depth limiting](#closure-depth-limiting) may restrict the
set of mirrored types and/or their members during each invocation
of the mirror generator.


### Command-line Options

* `--boot-class-path `_`pathname`_   _(mandatory)_

    _`pathname`_ must point to the `android.jar` file from the Android SDK
    used to build the Java part of the application. This option must always
    be present and point to an existing file.

* `-d `_`directory`_

    _`directory`_ is the pathname of the directory into which the mirror
    generator shall place its output Cangjie source code files containing
    the generated mirror type declarations, in a hierarchy of subdirectories
    conforming to CJPM requirements.
    If this option is not specified, the current directory is assumed.

* `-cp `_`path`_, `--class-path `_`path`_    _(mandatory)_

    _`path`_ is a list of pathnames of directories, jar files and zip files,
    separated with semicolons `;` on Windows and colons `:` on all other
    platforms. The mirror generator shall go through those directories and
    archives when looking for the class files of the Java types specified
    on the command line (_`type-names`_) _and_ their dependencies.
    _`path`_ must begin with the pathname of the `android.jar` file.

* `-h`, `-?`, `--help`

    Prints a help message with a brief description of the command-line syntax
    and all options and exits.


### System Properties

* `-Dpackage.mode=true`   _(mandatory)_

    Must always be specified in the current version.

* `-Dpackage.name=`_`name`_   _(mandatory)_

    _`name`_ is the name of the Cangjie package into which all generated
    mirror types will be placed.
    See [Single-Package Mode](#single-package-mode) for details.

* `-Dgen.closure.depth=`_`number`_

    _`number`_ is a non-negative decimal integer limiting the depth
    of dependency graph scanning when determining the set of types
    and their members that will be mirrored.

    See [Closure Depth Limiting](#closure-depth-limiting) for details.

* `-Djar.mode=true`

    Single-jar mode of operation
    (see [Command-Line Syntax](#jmg-command-line-syntax) above).

* `-Djar.mode.packages=`_`pathname`_

    _`pathname`_ is the pathname of a plain text file containing a list
    of package names. Only the class files that belong to the listed
    packages _and_ their dependencies will be mirrored. Requires single-jar mode.
    See [Command-Line Syntax](#incremental-mirroring-command-line-syntax)
    in the [Incremental Mirroring](#incremental-mirroring) section for details.

* `-Dimports.config=`_`pathname`_

    _`pathname`_ is the pathname of a plain text file containing
    mirror type mappings accumulated during previous mirror generator
    runs. Requires single-jar mode and `-Djar.mode.packages`.
    See [Command-Line Syntax](#incremental-mirroring-command-line-syntax)
    in the [Incremental Mirroring](#incremental-mirroring) section for details.

### Examples

```shell
java -Dpackage.mode=true -Dpackage.name=com.example \
     -jar ${CANGJIE_HOME}/tools/bin/java-mirror-gen.jar \
    --boot-class-path ${ANDROID_SDK}/platforms/android-35/android.jar \
    --class-path ${ANDROID_SDK}/platforms/android-35/android.jar:App.jar \
    -d ./mirrors \
    com.example.subpkg1.A com.example.subpkg2.B
```

```shell
java -Djar.mode=true -Dpackage.mode=true -Dpackage.name=com.example \
     -jar ${CANGJIE_HOME}/tools/bin/java-mirror-gen.jar \
    --boot-class-path ${ANDROID_SDK}/platforms/android-35/android.jar \
    --class-path ${ANDROID_SDK}/platforms/android-35/android.jar:./Lib.jar \
    -d ./mirrors \
    App.jar
```

For a more complex example, see
[Example: JDK API Mirroring](#example-jdk-api-mirroring).


## Interop Library API Reference

The Android interop library includes enhanced versions of mirror types
for two fundamental Java classes, `java.lang.Object` and `java.lang.String`,
and a generic mirror type for representing Java arrays.

To enable the use of unqualified names, those mirror types are called
respectively `JObject`, `JString`, and `JArray<T>` so as to avoid name
clashes with Cangjie types from `std.core`.

They also have some methods renamed and new methods added to enhance
interoperability.


### `java.lang.JObject`

`java.lang.JObject` is the root of the hierarchy of Java mirror classes
and interop classes. It is itself a mirror of the `java.lang.Object`
class, but has some unsupported methods removed and some methods
renamed and added for better alignment with the Cangjie standard library.

**NOTE:** The methods removed from `JObject` are not available
in any other mirror classes either. Those are `clone()`, `finalize()`,
and `getClass`.

```cangjie
package java.lang

@JavaMirror["java.lang.Object"]
open class JObject {
    public open func equals(obj: ?JObject): Bool

    public func hashCode(): Int64
    @ForeignName["hashCode"]
    public open func hashCode32(): Int32

    public func toString(): String
    @ForeignName["toString"]
    public open func toJString(): JString

    public func wait(timeoutMillis: Int64): Unit
    public func wait(timeoutMillis: Int64, nanos: Int32): Unit
    public func wait(): Unit
    public func notifyAll(): Unit
    public func notify(): Unit
}
```

The instance member function `equals` and all `wait/notify` functions
are mirrors of the respective Java methods.


```cangjie
public func hashCode(): Int64
```

Calls the original Java method `hashCode` and casts the returned 32-bit `int`
value to `Int64`. Introduced to better meet the expectations of Cangjie
developers.

```cangjie
@ForeignName["hashCode"]
public open func hashCode32(): Int32
```

The original Java method `hashCode`, renamed to `hashCode32` to avoid a name
clash.

```cangjie
public func toString(): String
```

Calls the original Java method `toString` and converts its result to a
Cangjie `String`. In the _very_ unlikely event of receiving `null`
from the underlying Java method, throws an exception.

Introduced to better meet the expectations of Cangjie developers.

**NOTE:** This function returns a Cangjie `String`, breaking the requirement
to use only Java-compatible types for parameters and return values of public
member functions and constructors of mirror types and interop classes.
This is only possible because the `cjc` compiler has dedicated support for this
member function.

```cangjie
@ForeignName["toString"]
public open func toJString(): JString
```

The original Java method `toString`, renamed to `toJString` to avoid a name
clash.

An implementation of Java `toString` is very unlikely to return `null`, hence
the return type is `JString`, not `?JString`.


### `java.lang.JString`

```cangjie
package java.lang

@JavaMirror["java.lang.String"]
open class JString {
       .  .  .
    public init(cjString: String)
       .  .  .
}
```

```cangjie
public init(cjString: String)
```

Effectively converts the given Cangjie `String` to a Java string (`JString`).

**NOTE:** This constructor accepts a parameter of the Cangjie type `String`,
breaking the requirement to use only Java-compatible types for parameters
and return values of non-private member functions and constructors of mirror
types and interop classes. This is only possible because the `cjc` compiler
has dedicated support for `JString`.

Members inherited from [`JObject`](#java.lang.jobject):
`equals`, `hashCode`, `hashCode32`, `toString`, `toJString`, `wait/notify`
methods.


### `java.lang.JArray<T>`

`JArray<T>` is a special built-in generic mirror type that mirrors all Java array
types. `T` must be either a value type mapped to a primitive Java type, such as
`Int32` or `Bool`, a mirror type, or an interop class.

**Limitations:**

As of the current version:

* Java arrays with nullable elements are not supported. In other words, if `T`
  is a mirror type or an interop class, the type `JArray<T>` is supported,
  but the type `JArray<?T>` is not.

* Variables and parameters of Java array types may not be marked as
  nullable using the `Option` enum. That is, even if `JArray<T>` is a valid
  type, `?JArray<T>` is not recognized as such.

The API of Java arrays is limited compared to that of the Cangjie `Array<T>`
struct type; the only operations besides construction are length query,
element access, and the inherited methods of `java.lang.Object`.

```cangjie
public init(length: Int32)
```

Constructs a new Java array of the given `length`.

```cangjie
public prop length: Int32
```

The number of elements in the array.

```cangjie
public operator func [](index: Int32): T
public operator func [](index: Int32, value!: T): Unit
```

The element access operator `[]`.

Members inherited from [`JObject`](#java.lang.jobject):
`equals`, `hashCode`, `hashCode32`, `toString`, `toJString`, `wait/notify`
methods.


## Execution

### Java Class Loaders

The Java counterparts of all mirror types _and_ interop classes must be
loaded by the same class loader.


### Initialization

When control first reaches Cangjie code, all global and `static` Cangjie
variables are initialized, and the static initializers of all Cangjie types are
called. That happens when the very first [interop class](#interop-classes)
wrapper gets initialized.

> Java classes and interfaces are initialized on first use, which can be one of:
> class instantiation, `static` method invocation, assignment to a `static`
> field, retrieval of the value of a `static` field that is not a constant
> variable, subclass initialization, implementing class initialization (only if
> the interface declares default methods), or the invocation of certain
> reflective methods.

**IMPORTANT: All that Cangjie initialization code must _not_ use any mirror
types or interop classes in any manner whatsoever. A deadlock or crash is
imminent if this rule is violated.**



### Finalization

#### Java Finalizers

Interop classes may not implement the `finalize()` method of `java.lang.Object`.
The built-in mirror of that class, [`java.lang.JObject`](#java.lang.jobject),
does not even declare that method.

> An attempt to define a Java finalizer in an interop class:
>
> ```cangjie
> @JavaImpl
> public class C {
>        .  .  .
>     public func finalize(): Unit {
>            .  .  .
>     }
>
> }
> ```
>
> would actually result in a name clash and compile-time error, because the
> interop-enabling code uses Java finalizers for its own purposes.


#### Cangjie Finalizers

A mirror class declaration cannot contain a Cangjie finalizer (`~init()`).

It is **TBD** whether an [interop class](#interop-classes) can contain
a finalizer.


### Exceptions

Both Java and Cangjie feature exceptions - events that interrupt the normal
control flow of the program, usually to signal that a non-fatal execution error
has occurred and needs to be handled.

In the bidirectional interop scenario supported by CJMP, control may pass
between methods/functions written in different languages multiple times
in a chain of nested calls. As a result, there may be a number of Java and
Cangjie frames interspersed on the thread stack when an exception gets thrown
in either Java or Cangjie code. That in turn means that the stack unwinding
process may need to cross a language boundary. Here is how that works:

If the call of a Java constructor of method from Cangjie code completes abruptly
due to an uncaught Java exception, a Cangjie exception with the following
characteristics is thrown immediately:

* If the Java exception is an `Error` (i.e. its type is `java.lang.Error`
  or its subclass), the Cangjie exception is also an `Error`, otherwise it is
  an `Exception`.

* The value of the `message` property _contains_ the message returned by the
 `getMessage()` method of the Java exception.

* The elements of the array returned by `getStackTrace()` are **TBD**.

If that new Cangjie exception in turn does not get caught before the stack
unwinding process reaches a Java frame again, the original Java exception
gets re-thrown and the whole process is repeated. (There may be no such Java
frame if the thread was created by Cangjie `spawn`, in which case **TBD**.)

If an exception gets thrown somewhere within Cangjie code, that is, _not_ as
a result of an abrupt completion of a cross-language call from Cangjie to Java
as described in the preceding paragraphs, and the stack unwinding process
in turn reaches a Java frame, a Java exception (`java.lang.Exception`) with the
following characteristics gets thrown:

* The string that its `getMessage()` method returns contains the value of the
  `message` property of the Cangjie exception.

* The elements of the array returned by `getStackTrace()` are **TBD**.

That Java exception does _not_ retain any information about the original
Cangjie exception other than the message associated with the latter, and is
processed further as if it was a Java exception in the first place
(see above for details).

There is no way to `throw` a Java exception in Cangjie code or vice versa.


### Memory Management

Java and Cangjie objects reside in separate heaps. The respective language
runtime manages each of the two heaps. The interop library and bridge code
ensure that objects in one language heap do not get garbage collected
if references to them still exist in variables and data structures of the other
language.

**IMPORTANT: The mechanism that ensures the consistency of both heaps in the
presence of cross-language references has two significant limitations that must
be understood and remembered at all times when using the interop facilities:**

1. The Android VM imposes hard limits on the number of Java heap objects
   references to which may exist simultaneously _outside_ the VM, in particular
   in Cangjie variables and data structures. It is therefore recommended
   to avoid storing such references in global Cangjie variables and long-lived
   data structures as much as practically possible. **NOTE:** This includes
   instances of both mirror types and interop classes.

   Moreover, the efficiency and timing of release of those references when
   they are no longer needed on the Cangjie side depends entirely on the Cangjie
   garbage collector. Even if a reference is short-lived from the application
   developer perspective, that does not mean it gets released immediately when
   e.g. the last Cangjie variable holding it gets assigned a new value.
   Traversing a large Java array or collection in Cangjie code in a loop could
   create thousands of such references. That may cause the application to hit
   the aforementioned hard limit of the Android VM before the Cangjie garbage
   collector is invoked. The application would then terminate abnormally as if
   it has run out of memory.


2. As Java and Cangjie garbage collectors each operate solely within their
   respective managed environments, inter-language circular references may lead
   to memory leaks. The creation of such references must be either avoided
   altogether, or accompanied with logic that would break such cycles before the
   objects forming the cycles become unreachable from both Java and Cangjie
   parts of the application code.


### Threads

A thread created by Cangjie `spawn` gets attached to the JVM automatically
as a daemon thread when it first crosses the border between two
languages: instantiates a mirror/interop class, calls its `static` method that
is not (re)defined in Cangjie, etc.

**IMPORTANT:** In the current version, any interop uses that may occur
in threads created _not_ by the Java VM must be wrapped in `exclusiveScope<T>`.
For example, the code

```
let x = SomeJavaClass()
return x.foo().bar() + 1    // Suppose bar() returns an Int32
```

must be rewritten as

```
return exclusiveScope<Int32> {
   let x = SomeJavaClass()
   return x.foo().bar() + 1
}
```

if it may execute in a non-Java thread.



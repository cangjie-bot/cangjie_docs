# Cangjie-C Interoperability

To ensure compatibility with existing ecosystems, Cangjie supports calling C language functions and also allows C language to call Cangjie functions.

## Calling C Functions from Cangjie

To call C functions in Cangjie, you need to declare the function using the `@C` and `foreign` keywords in Cangjie, but `@C` can be omitted when modifying a `foreign` declaration.

For example, suppose you want to call the C `rand` and `printf` functions with the following signatures:

```c
// stdlib.h
int rand();

// stdio.h
int printf(const char *fmt, ...);
```

Then the way to call these two functions in Cangjie is as follows:

<!-- run -->

```cangjie
// declare the function by `foreign` keyword, and omit `@C`
foreign func rand(): Int32
foreign func printf(fmt: CString, ...): Int32

main() {
    // call this function by `unsafe` block
    let r = unsafe { rand() }
    println("random number ${r}")
    unsafe {
        var fmt = LibC.mallocCString("Hello, No.%d\n")
        printf(fmt, 1)
        LibC.free(fmt)
    }
}
```

Points to note:

1. The `foreign` modifier is used for function declarations, indicating that the function is an external function. Functions modified by `foreign` can only have declarations, not implementations.
2. For functions declared with `foreign`, the parameters and return types must conform to the mapping relationship between C and Cangjie data types. For details, refer to [Type Mapping](./cangjie-c.md#type-mapping).
3. Since C-side functions are likely to perform unsafe operations, calling functions modified by `foreign` must be wrapped in an `unsafe` block; otherwise, a compilation error will occur.
4. The `@C` modifier with the `foreign` keyword can only be used to modify function declarations, not other declarations, otherwise a compilation error will occur.
5. `@C` only supports modifying `foreign` functions, non-generic functions in the top-level scope, and `struct` types.
6. `foreign` functions do not support named parameters and default parameter values. `foreign` functions allow variable-length parameters, expressed using `...`, which can only be used at the end of the parameter list. Variable-length parameters must all satisfy the `CType` constraint, but do not have to be of the same type.
7. Although Cangjie (CJNative backend) provides stack expansion capabilities, since Cangjie cannot perceive the actual stack size used by C-side functions, there is still a risk of stack overflow after FFI calls enter C functions (which may cause program runtime crashes or unpredictable behavior). Developers need to modify the `cjStackSize` configuration according to actual conditions.

Examples of some illegal `foreign` declarations are as follows:

<!-- compile.error -->

```cangjie
foreign func rand(): Int32 { // compiler error
    return 0
}
@C
foreign var a: Int32 = 0 // compiler error
@C
foreign class A{} // compiler error
@C
foreign interface B{} // compiler error
```

## CFunc

`CFunc` in Cangjie refers to functions that can be called by C language code, and there are three forms:

1. `foreign` functions modified by `@C`
2. Cangjie functions modified by `@C`
3. `lambda` expressions of type `CFunc`; unlike ordinary lambda expressions, `CFunc lambda` cannot capture variables.

<!-- run -->

```cangjie
// Case 1
foreign func free(ptr: CPointer<Int8>): Unit

// Case 2
@C
func callableInC(ptr: CPointer<Int8>) {
    print("This function is defined in Cangjie.")
}

// Case 3
let f1: CFunc<(CPointer<Int8>) -> Unit> = { ptr =>
    print("This function is defined with CFunc lambda.")
}
```

The types of functions declared/defined in the above three forms are all `CFunc<(CPointer<Int8>) -> Unit>`. `CFunc` corresponds to the function pointer type in C language. This type is a generic type, and its generic parameter indicates the input parameter and return value type of the `CFunc`, used as follows:

<!-- run -->

```cangjie
foreign func atexit(cb: CFunc<() -> Unit>): Int32
```

Like `foreign` functions, the parameters and return types of other forms of `CFunc` must satisfy the `CType` constraint, and do not support named parameters and default parameter values.

When `CFunc` is called in Cangjie code, it needs to be in an `unsafe` context.

Cangjie language supports converting a variable of type `CPointer<T>` to a specific `CFunc`, where the generic parameter `T` of `CPointer` can be any type that satisfies the `CType` constraint, used as follows:

<!-- compile -->

```cangjie
main() {
    var ptr = CPointer<Int8>()
    var f = CFunc<() -> Unit>(ptr)
    unsafe { f() } // core dumped when running, because the pointer is nullptr.
}
```

> **Note:**
>
> Forcing a pointer to be converted to `CFunc` and making a function call is a dangerous behavior. Users need to ensure that the pointer points to a valid function address; otherwise, a runtime error will occur.

## inout Parameters

When calling `CFunc` in Cangjie, its actual parameters can be modified with the `inout` keyword to form a reference pass-by-value expression. At this time, the parameter is passed by reference. The type of the reference pass-by-value expression is `CPointer<T>`, where `T` is the type of the expression modified by `inout`.

Reference pass-by-value expressions have the following constraints:

- Can only be used at the call site of `CFunc`.
- The type of the object it modifies must satisfy the `CType` constraint, but cannot be `CString`.
- The object it modifies cannot be defined with `let`, and cannot be temporary variables such as literals, input parameters, or values of other expressions.
- The pointer passed to the C side through the reference pass-by-value expression on the Cangjie side is only guaranteed to be valid during the function call, that is, the C side should not save the pointer for later use in such scenarios.

Variables modified by `inout` can be variables defined in the top-level scope, local variables, or member variables in `struct`, but cannot directly or indirectly come from instance member variables of `class`.

Here is an example:

<!-- compile.error -->

```cangjie
foreign func foo1(ptr: CPointer<Int32>): Unit

@C
func foo2(ptr: CPointer<Int32>): Unit {
    let n = unsafe { ptr.read() }
    println("*ptr = ${n}")
}

let foo3: CFunc<(CPointer<Int32>) -> Unit> = { ptr =>
    let n = unsafe { ptr.read() }
    println("*ptr = ${n}")
}

struct Data {
    var n: Int32 = 0
}

class A {
    var data = Data()
}

main() {
    var n: Int32 = 0
    unsafe {
        foo1(inout n)  // OK
        foo2(inout n)  // OK
        foo3(inout n)  // OK
    }
    var data = Data()
    var a = A()
    unsafe {
        foo1(inout data.n)   // OK
        foo1(inout a.data.n) // Error, n is derived indirectly from instance member variables of class A
    }
}
```

> **Note:**
>
> When using the macro expansion feature, the `inout` parameter feature cannot be used temporarily in the macro definition.

## unsafe

In the process of introducing interoperability with C language, many unsafe factors of C are also introduced. Therefore, the `unsafe` keyword is used in Cangjie to identify unsafe behaviors of cross-C calls.

Regarding the unsafe keyword, the following points are explained:

- `unsafe` can modify functions, expressions, or a section of scope.
- Functions modified by `@C` need to be in an `unsafe` context at the call site.
- When calling `CFunc`, the use site needs to be in an `unsafe` context.
- When calling `foreign` functions in Cangjie, the call site needs to be in an `unsafe` context.
- When the called function is modified by `unsafe`, the call site needs to be in an `unsafe` context.

Usage example:

<!-- run -->

```cangjie
foreign func rand(): Int32

@C
func foo(): Unit {
    println("foo")
}

var foo1: CFunc<() -> Unit> = { =>
    println("foo1")
}

main(): Int64 {
    unsafe {
        rand()           // Call foreign func.
        foo()            // Call @C func.
        foo1()           // Call CFunc var.
    }
    0
}
```

It should be noted that ordinary `lambda` cannot pass the `unsafe` attribute. When an `unsafe` `lambda` escapes, it can be directly called without an `unsafe` context without any compilation errors. When you need to call an `unsafe` function in `lambda`, it is recommended to call it in an `unsafe` block, refer to the following use case:

<!-- run -->

```cangjie
unsafe func A(){}
unsafe func B(){
    var f = { =>
        unsafe { A() } // Avoid calling A() directly without unsafe in a normal lambda.
    }
    return f
}
main() {
    var f = unsafe{ B() }
    f()
    println("Hello World")
}
```

## Calling Conventions

Function calling conventions describe how the caller and callee perform function calls (such as how parameters are passed, who cleans up the stack, etc.). Both the function caller and callee must use the same calling convention to run normally. The Cangjie programming language uses `@CallingConv` to represent various calling conventions, and the supported calling conventions are as follows:

- **CDECL**: `CDECL` indicates the default calling convention used by clang's C compiler on different platforms.
- **STDCALL**: `STDCALL` indicates the calling convention used by the Win32 API.

For C functions called through the C language interoperability mechanism, the default `CDECL` calling convention will be adopted if no calling convention is specified. Example of calling the C standard library function `rand`:

<!-- run -->

```cangjie
@CallingConv[CDECL]   // Can be omitted in default.
foreign func rand(): Int32

main() {
    println(unsafe { rand() })
}
```

`@CallingConv` can only be used to modify `foreign` blocks, individual `foreign` functions, and `CFunc` functions in the top-level scope. When `@CallingConv` modifies a `foreign` block, it will add the same `@CallingConv` modifier to each function in the `foreign` block respectively.

## Usage Instructions

- Operating system thread-local variable usage constraints

  When Cangjie interoperates with C language, there are risks in using thread-local variables of the operating system, explained as follows:

  1. Thread-local variables include variables defined by `thread_local` provided by C language and variables created using `pthread_key_create`.
  2. Cangjie has Cangjie thread scheduling capabilities, supporting the switching and recovery of Cangjie threads. The operating system thread to which a Cangjie thread is scheduled is random, so there is a risk in calling thread-local variables of other languages on a Cangjie thread.

  In the following example, there is a risk when Cangjie calls C language thread-local variables:

  ```c
  // C language logic using thread_local
  static thread_local int64_t count = 0;
  int64_t getCount() {
      count++;
      return count;
  }
  ```
  <!-- code_check_manual -->

  ```cangjie
  foreign func getCount(): Int64
  // Cangjie invokes the preceding C language logic
  spawn {
      let r1 = unsafe { getCount() }  // r1 equals 1
      sleep(Duration.second * 10)
      let r2 = unsafe { getCount() }  // r2 may not be equal to 2
  }
  ```

- Thread binding usage constraints

  When Cangjie calls C language to execute interoperability logic, the operating system thread to which the Cangjie thread is scheduled is random. It is not recommended to use behaviors bound to threads such as thread priority and thread affinity.

- Synchronization primitive usage instructions

  When Cangjie calls C language to execute interoperability logic, the current Cangjie thread will wait for the interoperability logic to execute. It is not recommended to have blocking behaviors that may cause long-term waiting in other languages.

- Support description for process fork scenarios

  When Cangjie calls C language to execute interoperability logic, if a child process is created in C language using the `fork()` method, Cangjie logic is not supported in the child process. Other operating system threads in the same process are not affected.

- Instructions when the process exits

  When Cangjie calls C language to execute interoperability logic, if the process exits in C language, the shared resources in the process have been released, which may cause errors such as illegal access.

## Type Mapping

### Basic Types

Cangjie and C language support the mapping of basic data types, with the overall principles:

1. Cangjie's types do not include reference types pointing to managed memory;
2. Cangjie's types and C's types have the same memory layout.

For example, some basic type mapping relationships are as follows:

| Cangjie Type |   C Type   |    Size (byte)     |
|:------------:|:----------:|:------------------:|
|    `Unit`    |   `void`   |         0          |
|    `Bool`    |   `bool`   |         1          |
|   `UInt8`    |   `char`   |         1          |
|    `Int8`    |  `int8_t`  |         1          |
|   `UInt8`    | `uint8_t`  |         1          |
|   `Int16`    | `int16_t`  |         2          |
|   `UInt16`   | `uint16_t` |         2          |
|   `Int32`    | `int32_t`  |         4          |
|   `UInt32`   | `uint32_t` |         4          |
|   `Int64`    | `int64_t`  |         8          |
|   `UInt64`   | `uint64_t` |         8          |
| `IntNative`  | `ssize_t`  | platform dependent |
| `UIntNative` |  `size_t`  | platform dependent |
|  `Float32`   |  `float`   |         4          |
|  `Float64`   |  `double`  |         8          |

> **Explanation:**
>
> Due to the uncertainty of types such as `int` and `long` on different platforms, programmers need to specify the corresponding Cangjie programming language types themselves. In C interoperability scenarios, similar to C language, the `Unit` type can only be used as the return type in `CFunc` and the generic parameter of `CPointer`.

Cangjie also supports mapping with C language's struct and pointer types.

### Structs

For struct types, Cangjie uses `struct` modified by `@C` to correspond. For example, there is such a struct in C language:

```c
typedef struct {
    long long x;
    long long y;
    long long z;
} Point3D;
```

Then its corresponding Cangjie type can be defined like this:

<!-- run -example00-->

```cangjie
@C
struct Point3D {
    var x: Int64 = 0
    var y: Int64 = 0
    var z: Int64 = 0
}
```

If there is such a function in C language:

```c
Point3D addPoint(Point3D p1, Point3D p2);
```

Then correspondingly, this function can be declared like this in Cangjie:

<!-- run -example00-->

```cangjie
foreign func addPoint(p1: Point3D, p2: Point3D): Point3D
```

`struct` modified by `@C` must satisfy the following restrictions:

- The type of member variables must satisfy the `CType` constraint
- Cannot implement or extend `interfaces`
- Cannot be used as the associated value type of `enum`
- Not allowed to be captured by closures
- Cannot have generic parameters

`struct` modified by `@C` automatically satisfies the `CType` constraint.

### Pointers

For pointer types, Cangjie provides the `CPointer<T>` type to correspond to the pointer type on the C side, and its generic parameter `T` needs to satisfy the `CType` constraint. For example, for the malloc function, its signature in C is:

```c
void* malloc(size_t size);
```

Then in Cangjie, it can be declared as:

<!-- run -->

```cangjie
foreign func malloc(size: UIntNative): CPointer<Unit>
```

`CPointer` can perform read/write, offset calculation, null judgment, and conversion to the integer form of the pointer, etc. For detailed APIs, refer to *Cangjie Programming Language Library API*. Among them, read/write and offset calculation are unsafe behaviors. When an illegal pointer calls these functions, undefined behaviors may occur, and these unsafe functions need to be called in an `unsafe` block.

An example of using `CPointer` is as follows:

<!-- run -->

```cangjie
foreign func malloc(size: UIntNative): CPointer<Unit>
foreign func free(ptr: CPointer<Unit>): Unit

@C
struct Point3D {
    var x: Int64
    var y: Int64
    var z: Int64

    init(x: Int64, y: Int64, z: Int64) {
        this.x = x
        this.y = y
        this.z = z
    }
}

main() {
    let p1 = CPointer<Point3D>() // create a CPointer with null value
    if (p1.isNull()) {  // check if the pointer is null
        print("p1 is a null pointer")
    }

    let sizeofPoint3D: UIntNative = 24
    var p2 = unsafe { malloc(sizeofPoint3D) }    // malloc a Point3D in heap
    var p3 = unsafe { CPointer<Point3D>(p2) }    // pointer type cast

    unsafe { p3.write(Point3D(1, 2, 3)) } // write data through pointer

    let p4: Point3D = unsafe { p3.read() } // read data through pointer

    let p5: CPointer<Point3D> = unsafe { p3 + 1 } // offset of pointer

    unsafe { free(p2) }
}
```

Cangjie language supports forced type conversion between `CPointer`s. The generic parameters `T` of the `CPointer`s before and after conversion all need to satisfy the constraint of `CType`, used as follows:

<!-- run -->

```cangjie
main() {
    var pInt8 = CPointer<Int8>()
    var pUInt8 = CPointer<UInt8>(pInt8) // CPointer<Int8> convert to CPointer<UInt8>
}
```

Cangjie language supports converting a variable of type `CFunc` to a specific `CPointer`, where the generic parameter `T` of `CPointer` can be any type that satisfies the `CType` constraint, used as follows:

<!-- run -->

```cangjie
foreign func rand(): Int32
main() {
    var ptr = CPointer<Int8>(rand)
}
```

> **Note:**
>
> Forcibly converting a `CFunc` to a pointer is usually safe, but you should not perform any `read` or `write` operations on the converted pointer, which may lead to runtime errors.

### Arrays

Cangjie uses the `VArray` type to map with C's array type. `VArray` can be used as function parameters and `@C struct` members. When the element type `T` in `VArray<T, $N>` satisfies the `CType` constraint, the `VArray<T, $N>` type also satisfies the `CType` constraint.

**As function parameter types:**

When `VArray` is used as a parameter of `CFunc`, the function signature of `CFunc` can only be of type `CPointer<T>` or `VArray<T, $N>`. When the parameter type in the function signature is `VArray<T, $N>`, the passed parameter is still passed in the form of `CPointer<T>`.

An example of using `VArray` as a parameter is as follows:

<!-- code_check_manual -->

```cangjie
foreign func cfoo1(a: CPointer<Int32>): Unit
foreign func cfoo2(a: VArray<Int32, $3>): Unit
```

The corresponding C-side function definition can be:

```c
void cfoo1(int *a) { ... }
void cfoo2(int a[3]) { ... }
```

When calling `CFunc`, you need to modify the `VArray` type variable through `inout`:

<!-- code_check_manual -->

```cangjie
var a: VArray<Int32, $3> = [1, 2, 3]
unsafe {
    cfoo1(inout a)
    cfoo2(inout a)
}
```

`VArray` is not allowed to be used as the return type of `CFunc`.

**As @C struct members:**

When `VArray` is used as an `@C struct` member, its memory layout is consistent with the arrangement of the struct on the C side. It is necessary to ensure that the declared length and type on the Cangjie side are completely consistent with those on the C side:

```c
struct S {
    int a[2];
    int b[0];
}
```

In Cangjie, it can be declared as the following struct to correspond to the C code:

<!-- run -->

```cangjie
@C
struct S {
    var a = VArray<Int32, $2>(repeat: 0)
    var b = VArray<Int32, $0>(repeat: 0)
}
```

> **Note:**
>
> C language allows the last field of a struct to be an array type with an unspecified length, which is called a flexible array. Cangjie does not support mapping structs containing flexible arrays.

### Strings

In particular, for string types in C language, Cangjie has designed a `CString` type to correspond. To simplify operations on C language strings, `CString` provides the following member functions:

- `init(p: CPointer<UInt8>)`  Construct a CString through CPointer
- `func getChars()` Get the address of the string, type is `CPointer<UInt8>`
- `func size(): Int64`  Calculate the length of the string
- `func isEmpty(): Bool`  Determine whether the length of the string is 0, return true if the string pointer is null
- `func isNotEmpty(): Bool`  Determine whether the length of the string is not 0, return false if the string pointer is null
- `func isNull(): Bool`  Determine whether the string pointer is null
- `func startsWith(str: CString): Bool`  Determine whether the string starts with str
- `func endsWith(str: CString): Bool`  Determine whether the string ends with str
- `func equals(rhs: CString): Bool`  Determine whether the string is equal to rhs
- `func equalsLower(rhs: CString): Bool`  Determine whether the string is equal to rhs, ignoring case
- `func subCString(start: UInt64): CString`  Extract a substring starting from start, the returned substring is stored in newly allocated space
- `func subCString(start: UInt64, len: UInt64): CString`  Extract a substring of length len starting from start, the returned substring is stored in newly allocated space
- `func compare(str: CString): Int32`  Compare the string with str, the return result is the same as `strcmp(this, str)` in C language
- `func toString(): String`  Construct a new String object with the string
- `func asResource(): CStringResource` Get the Resource type of CString

In addition, to convert a `String` type to a `CString` type, you can call the `mallocCString` interface in LibC, and you need to release the `CString` after use.

An example of using `CString` is as follows:

<!-- run -->

```cangjie
foreign func strlen(s: CString): UIntNative

main() {
    var s1 = unsafe { LibC.mallocCString("hello") }
    var s2 = unsafe { LibC.mallocCString("world") }

    let t1: Int64 = s1.size()
    let t2: Bool = s2.isEmpty()
    let t3: Bool = s1.equals(s2)
    let t4: Bool = s1.startsWith(s2)
    let t5: Int32 = s1.compare(s2)

    let length = unsafe { strlen(s1) }

    unsafe {
        LibC.free(s1)
        LibC.free(s2)
    }
}
```

### sizeOf/alignOf

Cangjie also provides two functions, `sizeOf` and `alignOf`, to obtain the memory occupation and memory alignment values (unit: bytes) of the above C interoperability types. The function declarations are as follows:

<!-- code_no_check -->

```cangjie
public func sizeOf<T>(): UIntNative where T <: CType
public func alignOf<T>(): UIntNative where T <: CType
```

Usage example:

<!-- verify -->

```cangjie
@C
struct Data {
    var a: Int64 = 0
    var b: Float32 = 0.0
}

main() {
    println(sizeOf<Data>())
    println(alignOf<Data>())
}
```

Running on a 64-bit machine will output:

```text
16
8
```

## CType

In addition to the types mapped to C-side types provided in the Type Mapping section, Cangjie also provides a `CType` interface. The interface itself does not contain any methods, and it can be used as the parent type of all types supported by C interoperability, which is convenient for use in generic constraints.

It should be noted that:

1. The `CType` interface is an interface type in Cangjie, and it does not satisfy the `CType` constraint itself;
2. The `CType` interface is not allowed to be inherited or extended;
3. The `CType` interface will not break the usage restrictions of subtypes.

An example of using `CType` is as follows:

<!-- verify -->

```cangjie
func foo<T>(x: T): Unit where T <: CType {
    match (x) {
        case i32: Int32 => println(i32)
        case ptr: CPointer<Int8> => println(ptr.isNull())
        case f: CFunc<() -> Unit> => unsafe { f() }
        case _ => println("match failed")
    }
}

main() {
    var i32: Int32 = 1
    var ptr = CPointer<Int8>()
    var f: CFunc<() -> Unit> = { => println("Hello") }
    var f64 = 1.0
    foo(i32)
    foo(ptr)
    foo(f)
    foo(f64)
}
```

The execution result is as follows:

```text
1
true
Hello
match failed
```

## Calling Cangjie Functions from C

Cangjie provides the `CFunc` type to correspond to the function pointer type on the C side. Function pointers on the C side can be passed to Cangjie, and Cangjie can also construct variables corresponding to C's function pointers to pass to the C side.

Assume a C library API is as follows:

```c
typedef void (*callback)(int);
void set_callback(callback cb);
```

Correspondingly, this function can be declared in Cangjie as:

<!-- code_check_manual -->

```cangjie
foreign func set_callback(cb: CFunc<(Int32) -> Unit>): Unit
```

Variables of type CFunc can be passed from the C side or constructed on the Cangjie side. There are two ways to construct CFunc types on the Cangjie side: one is a function modified by `@C`, and the other is a closure marked as CFunc type.

A function modified by `@C` indicates that its function signature complies with C's calling rules, and the definition is still written in Cangjie. The function definition modified by `foreign` is on the C side.

> **Note:**
>
> For functions modified by `foreign` and functions modified by `@C`, it is not recommended to use `CJ_` (case-insensitive) as the prefix for the naming of these `CFunc`s, otherwise conflicts may occur with internal symbols of the standard library and runtime, leading to undefined behaviors.

Example:

<!-- code_check_manual -->

```cangjie
@C
func myCallback(s: Int32): Unit {
    println("handle ${s} in callback")
}

main() {
    // the argument is a function qualified by `@C`
    unsafe { set_callback(myCallback) }

    // the argument is a lambda with `CFunc` type
    let f: CFunc<(Int32) -> Unit> = { i => println("handle ${i} in callback") }
    unsafe { set_callback(f) }
}
```

Assuming the library compiled from the C function is "libmyfunc.so", you need to use the compilation command `cjc -L. -lmyfunc test.cj -o test.out` to make the Cangjie compiler link this library. Finally, the desired executable program can be generated.

In addition, when compiling C code, please enable the `-fstack-protector-all/-fstack-protector-strong` stack protection options. Cangjie-side code has overflow checking and stack protection functions by default. After introducing C code, it is necessary to synchronously ensure the safety of overflows in unsafe blocks.

## Compilation Options

Using C interoperability usually requires manually linking C libraries, and the Cangjie compiler provides corresponding compilation options.

- `--library-path <value>`, `-L <value>`, `-L<value>`: Specify the directory where the library files to be linked are located.

  The path specified by `--library-path <value>` will be added to the linker's library file search path. In addition, the paths specified in the environment variable `LIBRARY_PATH` will also be added to the linker's library file search path, and the paths specified by `--library-path` will have higher priority than the paths in `LIBRARY_PATH`.

- `--library <value>`, `-l <value>`, `-l<value>`: Specify the library file to be linked.

  The given library file will be directly passed to the linker, and the format of the library file name should be `lib[arg].[extension]`.

For all compilation options supported by the Cangjie compiler, refer to "Appendix > cjc Compilation Options" for details.

## Example

This demonstrates how to use C interoperability and the `write/read` interface to assign values to a struct and read values from it.

C code is as follows:

```c
// draw.c
#include<stdio.h>
#include<stdint.h>

typedef struct {
    int64_t x;
    int64_t y;
} Point;

typedef struct {
    float x;
    float y;
    float z;
} Cube;

void drawPicture(Point* point, Cube* cube) {
    point->x = 1;
    point->y = 2;
    printf("Draw Point finished.\n");

    printf("Before draw cube\n");
    printf("%f\n", cube->x);
    printf("%f\n", cube->y);
    printf("%f\n", cube->z);
    cube->x = 4.4;
    cube->y = 5.5;
    cube->z = 6.6;
    printf("Draw Cube finished.\n");
}
```

Cangjie code is as follows:

<!-- code_check_manual -->

```cangjie
// main.cj
@C
struct Point {
    var x: Int64 = 0
    var y: Int64 = 0
}

@C
struct Cube {
    var x: Float32 = 0.0
    var y: Float32 = 0.0
    var z: Float32 = 0.0

    init(x: Float32, y: Float32, z: Float32) {
        this.x = x
        this.y = y
        this.z = z
    }
}

foreign func drawPicture(point: CPointer<Point>, cube: CPointer<Cube>): Int32

main() {
    let pPoint = unsafe { LibC.malloc<Point>() }
    let pCube = unsafe { LibC.malloc<Cube>() }

    var cube = Cube(1.1, 2.2, 3.3)
    unsafe {
        pCube.write(cube)
        drawPicture(pPoint, pCube)   // in which x, y will be changed

        println(pPoint.read().x)
        println(pPoint.read().y)
        println(pCube.read().x)
        println(pCube.read().y)
        println(pCube.read().z)

        LibC.free(pPoint)
        LibC.free(pCube)
    }
}
```

The command to compile Cangjie code is as follows (taking CJNative backend as an example):

```shell
cjc -L . -l draw ./main.cj
```

In the compilation command, `-L .` means to search from the current directory when linking the library (assuming `libdraw.so` exists in the current directory), `-l draw` means the name of the library to link. After successful compilation, the binary file `main` is generated by default. The command to execute the binary file is as follows:

```shell
LD_LIBRARY_PATH=.:$LD_LIBRARY_PATH ./main
```

The running result is as follows:

```shell
Draw Point finished.
Before draw cube
1.100000
2.200000
3.300000
Draw Cube finished.
1
2
4.400000
5.500000
6.600000
```
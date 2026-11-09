# Modal Types Overview

## Why Local Mode

By default, Cangjie objects are allocated on the GC heap and reclaimed by the garbage collector at an unspecified time. A function often creates temporary objects that are used only inside that function, such as strings read from a file or the network, intermediate structures produced by parsing, or temporary arrays used for computation. These objects are no longer needed after the function returns, but they keep occupying heap memory until the next garbage collection. When such functions are called frequently or process large amounts of data, this raises the peak memory of the program.

Local mode uses types to express that "this object does not leave the current function". The compiler checks that such objects indeed do not escape, and the runtime allocates them in a memory area dedicated to the current function, called a local region. When the function returns, the whole local region is released at once, without waiting for garbage collection.

> **Note:**
>
> In this chapter, a local region refers to a memory area set aside for local objects. It is not the same concept as the regions inside the GC heap.

## Enabling Local Mode

Local mode is currently an experimental feature and is disabled by default. To use it, add both the `--experimental` and `--enable-local-mode` compilation options, for example:

```shell
cjc --experimental --enable-local-mode main.cj -o main
```

When building with cjpm, add the two options to the `compile-option` field in `cjpm.toml`. `compile-option` applies only to the current module, so each dependency module that uses local mode must enable it in its own `cjpm.toml`. You can also enable it for all modules in the `override-compile-option` field of the entry module, but then `exclave` and `demode` become keywords in all dependency modules. For details, see [cjpm Introduction](../compile_and_build/cjpm_usage.md). All examples in this chapter are written with local mode enabled.

Local mode is enabled only through the experimental cjc compilation options; cjpm has no separate configuration item for it. The local semantic rules and the `Copyable` restrictions described in this chapter take effect and are checked only when local mode is enabled.

When local mode is not enabled:

- `exclave` and `demode` are ordinary identifiers, and existing variables, functions, and other declarations with these names are not affected.
- Using mode annotations such as `@ local!`, `@ local?`, and `@ ~local` reports the error `local mode is disabled`, with a hint on how to enable it.
- Calling a function whose parameters or return value themselves have `@ local!` or `@ local?`, calling a `this @ local!` member function or constructor, or accessing a property of type `@ local!` requires local mode to be enabled.
- An upstream package may also export variables or constants of function types. If the parameters or return value of such a function type have `@ local!` or `@ local?`, using these variables or constants also requires local mode to be enabled.
- Importing a package that contains such declarations is not affected, and the functions and variables in the package that do not involve local mode can be used as usual.

For these modes and how they are used in constructors, member functions, and properties, see [Modes and Modal Types](#modes-and-modal-types) below and [Rules for Local Mode](./local_mode_rules.md).

> **Note:**
>
> - Local mode is still experimental. Its syntax, rules, and compilation options may change in future versions, and compatibility is not guaranteed.
> - Compilation artifacts involving local mode are not guaranteed to be compatible across SDK versions. If a package uses local mode, compile it and the packages that depend on it with the same SDK version; after upgrading the SDK, recompile these packages with the new version.
> - The compilation options only determine whether local mode can be used in source code. Some standard library APIs use local mode in their internal implementations, and these implementations take effect regardless of whether local mode is enabled.
> - The local feature is disabled by default in the toolchain. Even if it is enabled through the experimental options, tools such as cjfmt, cjlint, LSP, and cjdb are not guaranteed to recognize or fully support local mode.

## Modes and Modal Types

Adding `@` and a mode after a type gives a modal type. There are three modes related to local:

| Syntax | Meaning |
|------|------|
| `T @ ~local` | Non-local. The object can be freely stored, returned, and passed, exactly like an ordinary Cangjie object |
| `T @ local!` | Definitely local. The lifetime of the object is limited to the local region of the current function (or of the caller), and the object cannot escape |
| `T @ local?` | Maybe local. The value may or may not be local, and it is restricted as a local value when used |

Omitting the mode is equivalent to writing `@ ~local`, so the meaning of types in existing code does not change after local mode is enabled. For the keyword changes after enabling it, see [Keywords](../Appendix/keyword.md).

Both `T @ ~local` and `T @ local!` are subtypes of `T @ local?`: a `@ local?` variable or parameter can receive both local and non-local values. There is no subtype relationship between `@ ~local` and `@ local!`; for types that do not implement `Copyable`, values cannot be assigned between these two modes. A local value cannot be used as a non-local value, because it is released when the function returns. Conversely, a non-local value cannot be used as `@ local!` either, because a program can store local objects in the members or elements of a `@ local!` value (see [exclave Expressions](./exclave.md)); once an ordinary object references a local object, it is left with a dangling reference after that object is released. When you need a local version, construct a local copy; see [Local Forms of Common Standard Library Types](#local-forms-of-common-standard-library-types). `Copyable` types can convert between modes by copying the value; for details, see [Copyable and demode](./copyable_and_demode.md).

Note the following when writing modes:

- Spaces are allowed between `@` and the mode, but the mode itself is a single unit: no spaces are allowed inside `~local`, `local!`, or `local?`, so `@local !` and `@~ local` are both invalid. This manual recommends writing one space between `@` and the mode, as in `@ local!`.
- `local` is not a keyword. It denotes a mode only when it follows `@`, and can still be used as an ordinary identifier elsewhere.

## Where Modes Can Be Written

Modes can be written in the following common positions:

- Types of local variables
- Parameter types and return types of functions
- Parameter types and return types in function types, such as `(String @ local?) -> Int64`
- An entire function type, to specify the mode of the function object itself, such as `(() -> Int64) @ local?`; see [Closures and Nested Functions](./local_mode_rules.md#closures-and-nested-functions)
- After a constructor call, to specify the mode of the constructed instance, such as `C() @ local!`
- Types of instance member properties. The mode of a property type is also the mode of `this` in its getter and setter. For example, `prop name: String @ local!` can be accessed only through a `@ local!` instance; see [Quick Reference for Other Restrictions](./local_mode_rules.md#quick-reference-for-other-restrictions)
- Types of static member variables, which can only use the `@ ~local` mode

In a function type, a mode on the right side of `->` belongs to the return type: `() -> String @ local?` is a function that returns `String @ local?`. To specify the mode of the function object itself, parenthesize the entire function type and write `(() -> String) @ local?`.

A nested function is a declaration and has no place to write its type, so you write `@ local?` before `func` (a function mode specifier) to specify its mode as a function object. For example, the type of `@ local? func length(): Int64` is `(() -> Int64) @ local?`. This allows it to capture outer `@ local?` variables of non-`Copyable` types; see [Closures and Nested Functions](./local_mode_rules.md#closures-and-nested-functions).

Modes cannot be written in the following positions:

- Instance member variables of classes and structs: the mode of a member is determined by the instance it belongs to. For details, see [Modes of Members and Instances](./local_mode_rules.md#modes-of-members-and-instances).
- Generic type arguments cannot carry modes themselves, so `Array<String @ local!>` is invalid. A local array of strings should be written as `Array<String> @ local!`: `String` specifies the element type, and `@ local!` specifies the mode of the whole array. Strings read from this array are also subject to local restrictions. For details, see [Modes of Members and Instances](./local_mode_rules.md#modes-of-members-and-instances).
- Likewise, the element types of a tuple and the `T` in `?T` cannot carry modes on their own, so `(Int64, String @ local!)` and `?(String @ local?)` are both invalid. Write the mode on the whole type instead, such as `(Int64, String) @ local!` and `?String @ local?` (that is, `Option<String> @ local?`).

## A First Example

The following example creates a temporary array in `sumOfSquares`, annotates it as `@ local!`, and passes it to the read-only helper function `total` to compute the sum. The parameter of `total` is annotated as `@ local?`, so it can receive both local arrays and ordinary arrays. `getAllocatedLocalObjectSize()` returns the memory occupied by the currently active local regions, which is used here to observe when the local array is reclaimed:

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
import std.runtime.getAllocatedLocalObjectSize

// Reads the array only, so both local and ordinary arrays can be passed in
func total(xs: Array<Int64> @ local?): Int64 {
    var sum = 0
    for (i in 0..xs.size) {
        sum += xs[i]
    }
    sum
}

func sumOfSquares(n: Int64): Int64 {
    // Used only in this function; released with the local region on return
    let squares = Array<Int64>(n, {i => i * i}) @ local!
    println("in sumOfSquares, local memory in use: ${getAllocatedLocalObjectSize() > 0}")
    total(squares)
}

main() {
    let before = getAllocatedLocalObjectSize()
    let result = sumOfSquares(1000)
    let after = getAllocatedLocalObjectSize()
    println("result: ${result}")
    println("local memory released: ${after == before}")
    println(total([1, 2, 3]))
}
```

The output is:

```text
in sumOfSquares, local memory in use: true
result: 332833500
local memory released: true
6
```

A local object cannot leave the function that creates it. If you try to return `squares` from the above example, the compiler reports an error:

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func squaresOf(n: Int64): Array<Int64> {
    let squares = Array<Int64>(n, {i => i * i}) @ local!
    squares // Error, a @ local! value cannot be returned as @ ~local
}
```

## Local Forms of Common Standard Library Types

Common types in the standard library provide overloads that receive or return local values, but not every API supports every mode. The following table lists forms commonly used when getting started. `ArrayList` and `HashMap` require importing the `std.collection` package.

| Operation | Syntax | Description |
|---|---|---|
| Construct a local copy of an ordinary string | `String(text) @ local!` | Constructs a local copy directly from a string |
| Split a local string | `localText.split(",")` | When `localText` is `@ local!`, the result is `Array<String> @ local!` |
| Construct a local array | `Array<Int64>(3, {i => i * 10}) @ local!` | Constructs with an element initialization function |
| Construct a local dynamic array | `ArrayList<Int64>() @ local!` | Constructs an empty list, then adds elements with `add` |
| Construct a local hash map | `HashMap<Int64, Int64>() @ local!` | Constructs an empty hash map, then adds key-value pairs with `add` |

For non-`Copyable` types, adding `@ local!` to a variable does not automatically turn an existing ordinary object into a local object. When you need a local copy, use the corresponding local constructor. The following example constructs a local string from an ordinary string and then calls the local version of `split`:

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func splitLocal(text: String) {
    let localText = String(text) @ local!
    let parts: Array<String> @ local! = localText.split(",")
    println(parts.size)
    println(parts[0])
    println(parts[1])
}

main() {
    splitLocal("a,b")
}
```

The output is:

```text
2
a
b
```

This creates a new local string, and the original `text` remains an ordinary string. `split` on an ordinary string returns an ordinary array, which cannot be assigned directly to a local array variable. `String @ local?` has no matching `split` overload either:

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func splitPlain(text: String) {
    let parts: Array<String> @ local! = text.split(",") // Error, the result is @ ~local
}

func splitView(text: String @ local?): Int64 {
    text.split(",").size // Error, split has no overload for a @ local? string
}
```

The following collections are all constructed as local objects in the current function. Elements can be added and read, and the collections are released with the local region when the function returns:

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
import std.collection.*

func useCollections() {
    let values = Array<Int64>(3, {i => i * 10}) @ local!
    println(values[2])
    let list = ArrayList<Int64>() @ local!
    list.add(7)
    println(list[0])
    let map = HashMap<Int64, Int64>() @ local!
    map.add(1, 9)
    println(map[1])
}

main() {
    useCollections()
}
```

The output is:

```text
20
7
9
```

Check the corresponding overload signatures to find out which modes a standard library API supports.

## exclave and the Caller's Local Region

When you need to return a local object created in a function to the caller, or store it in a local object passed in by the caller, you can use `exclave { ... }` so that the new object is allocated in the caller's local region. Local constructors must also use exclave. For usage, execution steps, and restrictions, see [exclave Expressions](./exclave.md).

## When to Use Local Mode

The benefit of local mode is in peak memory, not throughput: it releases temporary objects immediately when the function returns, which lowers peak memory usage. It suits temporary objects that are created and used up within a function, and that are large or numerous.

When evaluating a rewrite, compare the peak memory before and after the rewrite under the same input and load. After the rewrite, you can also observe the usage of local regions:

- Use `getAllocatedLocalObjectSize()` in the `std.runtime` package to get the memory occupied by the currently active local regions. It counts the capacity of the memory blocks allocated to local regions (including the unused part of those blocks). Local objects are not counted in the result of `getAllocatedHeapSize()`.
- Calling `getAllocatedLocalObjectSize()` pauses all Cangjie threads, so it should not be called frequently on performance-sensitive paths. For details, see the API documentation of the standard library `std.runtime` package.

The size of a single local region is limited. When the limit is exceeded, a `LocalRegionOutOfMemoryError` is thrown. The limit can be configured with the environment variable `cjLocalRegionMaxSize`; see [Runtime Environment Variables](../Appendix/runtime_env.md#cjlocalregionmaxsize).

## In This Chapter

- [Rules for Local Mode](./local_mode_rules.md): assignment and escape, member modes, constructing local instances, overloading, closures, and usage constraints.
- [exclave Expressions](./exclave.md): how to modify local parameters and how to return local objects to the caller.
- [Copyable and demode](./copyable_and_demode.md): value types that are not subject to local restrictions, and how to keep member variables from inheriting the mode of the instance.

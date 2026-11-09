# exclave Expressions

> **Note:**
>
> Local mode is an experimental feature. All examples in this section are written with local mode enabled and must be compiled with the `--experimental --enable-local-mode` compilation options. See [Enabling Local Mode](./modal_overview.md#enabling-local-mode).

## Why exclave Is Needed

Each function has its own local region, and local objects created in a function are released when the function returns. This causes two problems:

- A function cannot hand a local object it creates to its caller, because the object is released when the function returns.
- A function cannot make a `@ local!` parameter passed in by the caller reference a local object created inside the function, because the parameter object belongs to the caller's local region and lives longer than the objects inside the function.

In the following example, `normalize` tries to replace the first path in the array passed in by the caller with a newly created string, and the compiler reports an error:

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func normalize(paths: Array<String> @ local!) {
    let fixed: String @ local! = "/index.html"
    paths[0] = fixed // Error, fixed belongs to this function's region
}
```

`fixed` belongs to the local region of `normalize`, while `paths` belongs to the caller's local region. The caller's array cannot be allowed to reference a string that is released when `normalize` returns.

exclave expressions solve this kind of problem.

## Meaning of exclave

An exclave expression has the form `exclave { block }` and is executed as follows:

1. The local region of the current function ends first, and the local objects in it are released.
2. The code in the block is then executed, and local objects newly created in the block are allocated in the caller's local region instead.
3. After the block finishes, the value of the block is returned to the caller, as with `return`.

Because execution leaves the function after the exclave finishes, the type of the exclave expression itself is `Nothing`, and the type of the last expression in the block must match the return type of the function.

In an exclave, `@ local!` parameters and the local objects newly created in the block are in the same local region (the caller's local region), so they can be assigned to each other, and local objects newly created in the block can also be returned.

### Modifying @ local! Parameters

Change the above `normalize` to create and assign the string in an exclave:

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func normalize(paths: Array<String> @ local!) {
    exclave {
        let fixed: String @ local! = "/index.html"
        paths[0] = fixed // OK, both are in the caller's region
    }
}

main() {
    let paths: Array<String> @ local! = ["/"]
    normalize(paths)
    println(paths[0])
}
```

The output is:

```text
/index.html
```

### The exclave Modifier

Modifying a function or constructor with `exclave` is equivalent to wrapping its entire body in an `exclave { ... }` block. The block can contain multiple statements.

For example, `exclave func f() { ... }` expands to `func f() { exclave { ... } }`, and `exclave init` expands in the same way. The following example uses the modifier form:

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
exclave func redirect(paths: Array<String> @ local!, target: String @ local!): Unit {
    paths[0] = target
}

main() {
    let paths: Array<String> @ local! = ["/old"]
    redirect(paths, "/new")
    println(paths[0])
}
```

The output is:

```text
/new
```

`this @ local!` and `this @ local?` constructors must be written as exclave; see [Constructing Local Instances](./local_mode_rules.md#constructing-local-instances).

## Returning Local Objects

Local objects constructed in an exclave can be returned to the caller. The object the caller receives is located in the caller's own local region and is released only when the caller returns:

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func makeSquares(n: Int64): Array<Int64> @ local! {
    exclave {
        Array<Int64>(n, {i => i * i}) @ local!
    }
}

func sumOfSquares(n: Int64): Int64 {
    let squares = makeSquares(n) // squares lives in the region of sumOfSquares
    var sum = 0
    for (i in 0..squares.size) {
        sum += squares[i]
    }
    sum
}

main() {
    println(sumOfSquares(10))
}
```

The output is:

```text
285
```

Without exclave, local objects created in a function cannot be returned:

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func makeSquares(n: Int64): Array<Int64> @ local! {
    let squares = Array<Int64>(n, {i => i * i}) @ local!
    squares // Error, a @ local! value created in this function cannot be returned
}
```

To put a `@ local!` parameter passed in into a new array or tuple and return it, also construct it in an exclave. The following example puts a string parameter into a new array; the array and its elements are all in the caller's local region:

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func duplicate(s: String @ local!): Array<String> @ local! {
    exclave {
        [s, s] // OK, the new array is in the caller's region
    }
}

main() {
    let s: String @ local! = "hello"
    let copies = duplicate(s)
    println(copies[0])
    println(copies[1])
}
```

The output is:

```text
hello
hello
```

## Usage Conditions

exclave can be used only in the following functions (including lambdas and constructors):

- Functions with `@ local!` parameters of non-`Copyable` types (including `this @ local!`)
- Functions whose return type is `@ local!` or `@ local?` of a non-`Copyable` type
- `this @ local!` or `this @ local?` constructors

Using exclave in other functions reports an error. `abstract` functions have no body and cannot be modified with `exclave` either:

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func compute(): Int64 {
    exclave { 1 } // Error, the function has no @ local! parameter or local return type
}

abstract class Shape {
    public exclave func reset(this @ local!): Unit // Error, abstract function cannot be exclave
}
```

## Usage Restrictions

exclave has the following restrictions:

- exclave expressions in the same function cannot be nested: the outer exclave has already ended the local region of the function, and the inner one cannot end it again.
- An exclave block cannot access `@ local!` or `@ local?` local variables of non-`Copyable` types declared outside the block, because the local region of the current function has already ended when the exclave is entered, and the objects these variables reference may have been released. Variables of `Copyable` types are not subject to this restriction.
- exclave cannot be used directly in spawn blocks, the `main` function, finalizers, or static initializers, and cannot be used in a try or catch block to end the function containing that block.

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
class Request {
    var path: String
    init(path: String) {
        this.path = path
    }
    exclave init(this @ local!, path: String @ local!) {
        this.path = path
    }
}

func twice(req: Request @ local!) {
    exclave {
        exclave { () } // Error, exclave cannot be nested
    }
}

func reuse(req: Request @ local!) {
    let prefix: String @ local! = "/api"
    let depth: Int64 @ local! = 1
    exclave {
        let d = depth // OK, Int64 is Copyable
        req.path = prefix // Error, prefix is declared outside exclave
    }
}
```

The following functions all have `@ local!` parameters of non-`Copyable` types and meet the usage conditions of exclave, but still cannot use exclave in a try block or a catch block:

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func inTry(paths: Array<String> @ local!) {
    try {
        exclave { () } // Error, not allowed in a try block
    } catch (e: Exception) {}
}

func inCatch(paths: Array<String> @ local!) {
    try {
        ()
    } catch (e: Exception) {
        exclave { () } // Error, not allowed in a catch block
    }
}
```

exclave cannot be used directly in spawn blocks, `main`, finalizers, or static initializers:

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
class Resource {
    static let count: Int64 = 0
    static init() {
        exclave {()} // Error, exclave is not allowed here
    }
    init() {}
    ~init() {
        exclave {()} // Error, exclave is not allowed here
    }
}

func inSpawn(): Unit {
    spawn {
        exclave {()} // Error, exclave is not allowed here
    }
}

main() {
    exclave {()} // Error, exclave is not allowed here
}
```

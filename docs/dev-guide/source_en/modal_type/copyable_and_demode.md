# Copyable and demode

> **Note:**
>
> Local mode is an experimental feature. All examples in this section are written with local mode enabled and must be compiled with the `--experimental --enable-local-mode` compilation options. See [Enabling Local Mode](./modal_overview.md#enabling-local-mode).

## The Copyable Interface

Values of some types contain no references, and assigning them copies the value itself, such as `Int64` and `Bool`. For such types, "whether an object escapes the function" is meaningless, so the local mode restrictions do not need to apply to them. The standard library uses the `Copyable` interface to identify these types.

`Copyable` is defined in the `std.core` package and is an empty interface. The following types implement `Copyable` automatically:

- All numeric types, as well as `Rune`, `Bool`, `Unit`, and `Nothing`
- All types that implement `CType`, such as `CPointer<T>` and structs modified with `@C`
- Tuple types whose element types all implement `Copyable`
- `VArray<T, $N>` whose element type `T` implements `Copyable`

Values of types that implement `Copyable` can be converted directly between `@ local!`, `@ local?`, and `@ ~local`. The conversion is just a copy by value and requires no additional runtime operation.

When all element types of a tuple implement `Copyable`, a `@ local?` tuple can be returned by value as an ordinary tuple:

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func toPlain(t: (Int64, Bool) @ local?): (Int64, Bool) {
    t // OK, the tuple type is Copyable
}

main() {
    let t: (Int64, Bool) @ local! = (1, true)
    let plain: (Int64, Bool) = toPlain(t)
    println("${plain[0]} ${plain[1]}")
}
```

The output is:

```text
1 true
```

If the tuple contains a type that does not implement `Copyable`, such as `String`, the same conversion is not allowed:

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func toPlain(t: (Int64, String) @ local?): (Int64, String) {
    t // Error, cannot convert @ local? to @ ~local
}
```

Similarly, a `VArray` whose element type is `Copyable` can be copied from `@ local!` to `@ ~local`:

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
main() {
    let local: VArray<Int64, $2> @ local! = [1, 2]
    let plain: VArray<Int64, $2> = local
    println(plain[1])
}
```

The output is:

```text
2
```

Outside an exclave, a `Copyable` value of `@ local!` or `@ local?` cannot be returned directly as an interface type; copy it to a `@ ~local` variable first and then return it. The following functions receive values of the two modes respectively, copy them to ordinary `Int64` values, and return them as the `ToString` interface:

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func fromLocal(n: Int64 @ local!): ToString {
    let plain: Int64 = n
    plain
}

func fromMaybe(n: Int64 @ local?): ToString {
    let plain: Int64 = n
    plain
}

main() {
    let n: Int64 @ local! = 42
    println(fromLocal(n).toString())
    println(fromMaybe(n).toString())
}
```

The output is:

```text
42
42
```

## Implementing Copyable for a struct

If the types of all instance member variables of a struct implement `Copyable`, the struct can be declared to implement `Copyable`. A generic struct can meet this condition through generic constraints. After that, values produced by ordinary constructors can also be copied by value into variables of local modes, without additional local constructors; see [Constructing Local Instances](./local_mode_rules.md#constructing-local-instances):

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
struct Point <: Copyable {
    var x: Int64
    var y: Int64
    init(x: Int64, y: Int64) {
        this.x = x
        this.y = y
    }
}

struct Pair<A, B> <: Copyable where A <: Copyable, B <: Copyable {
    var first: A
    var second: B
    init(first: A, second: B) {
        this.first = first
        this.second = second
    }
}

func keep(p: Point @ local?): Point {
    p // OK, Point is Copyable
}

main() {
    let p: Point @ local! = Point(1, 2)
    let r: Point = keep(p) // OK, copy a local Point as a non-local value
    let pr: Pair<Int64, Bool> @ local! = Pair<Int64, Bool>(3, true)
    let pr2: Pair<Int64, Bool> = pr
    println("${r.x} ${r.y} ${pr2.first}")
}
```

The output is:

```text
1 2 3
```

It is not enough for the member types to meet the condition; the struct must explicitly declare that it implements `Copyable`. The following `Plain` has only one `Int64` member but does not implement `Copyable`, so the above conversion is not allowed:

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
struct Plain {
    var x: Int64
    init(x: Int64) {
        this.x = x
    }
}

func keep(p: Plain @ local?): Plain {
    p // Error, Plain does not implement Copyable
}
```

You can also implement `Copyable` for an existing struct through an extension, with the same rules as implementing it at the declaration. For example, for an existing struct `Plain` that does not yet implement `Copyable` and whose instance member variables all have types that implement `Copyable`, you can write `extend Plain <: Copyable {}`.

`Copyable` cannot be implemented in the following cases:

- Any instance member variable of the struct has a type that does not implement `Copyable`.
- class and enum types, even if they contain no members of reference types.

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
struct Named <: Copyable {
    var name: String = "" // Error, String is not Copyable
}

class Box <: Copyable {} // Error, a class cannot implement Copyable

enum Color <: Copyable { // Error, an enum cannot implement Copyable
    | Red | Green
}
```

## Where Copyable Can Be Used

`Copyable` can be used only in the superinterface list of a type declaration or extension declaration, or as the upper bound of a generic parameter (such as `where T <: Copyable`). It cannot be used as the type of variables, parameters, and so on:

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func f(c: Copyable): Unit {} // Error, Copyable cannot be used as a parameter type

func g() {
    let c: Copyable = 1 // Error, Copyable cannot be used as a variable type
}
```

As a generic constraint, it allows a generic function to return a non-local result for arguments of any mode:

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func plain<T>(x: T @ local?): T where T <: Copyable {
    x
}

main() {
    let n: Int64 @ local! = 42
    let m: Int64 = plain(n)
    println(m)
}
```

The output is:

```text
42
```

## demode Member Variables

The mode of an instance member variable is usually the same as that of its instance (see [Modes of Members and Instances](./local_mode_rules.md#modes-of-members-and-instances)). If a local object needs to hold a reference to an ordinary object, such as global configuration or a cache, that member cannot become local along with the instance. In this case, modify the member variable with `demode`:

- The mode of a `demode` member variable is always `@ ~local`, regardless of the mode of the instance.
- Therefore, `@ ~local` values can be assigned to it, and the values read from it are also `@ ~local` and can be stored and returned freely.
- Accordingly, `@ local!` or `@ local?` values of non-`Copyable` types cannot be assigned to it; `Copyable` values can be copied by value.

The following example creates a local object through a `this @ local!` constructor and then observes the mode of its `demode` member. This constructor must use `exclave`; see [Constructing Local Instances](./local_mode_rules.md#constructing-local-instances).

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
var defaultTag: String = "default"

class Node {
    demode var tag: String
    var label: String
    exclave init(this @ local!, label: String @ local!) {
        this.tag = ""
        this.label = label
    }
}

func tagOf(): String {
    let n = Node("temp") @ local!
    n.tag = defaultTag // OK, n.tag is always @ ~local
    n.tag // OK, can be returned
}

main() {
    println(tagOf())
}
```

The output is:

```text
default
```

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
var defaultTag: String = "default"

class Node {
    demode var tag: String
    var label: String
    exclave init(this @ local!, label: String @ local!) {
        this.tag = ""
        this.label = label
    }
}

func f() {
    let n = Node("temp") @ local!
    n.label = defaultTag // Error, n.label is @ local!
    let s: String @ local! = "s"
    n.tag = s // Error, n.tag is @ ~local
}
```

## Local Arrays with Non-Local Elements

The elements of `Array<T> @ local!` are also local. If you want a temporary array that is released with the function but stores ordinary objects, use a wrapper struct with a `demode` member. The following `Demode<T>` is a wrapper type defined in this example and used as the array element:

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
struct Demode<T> {
    demode var value: T
    init(v: T) {
        this.value = v
    }
    exclave init(this @ local!, v: T) {
        this.value = v
    }
}

func pickSecond(a: String, b: String): String {
    let arr: Array<Demode<String>> @ local! = [Demode(a) @ local!, Demode(b) @ local!]
    arr[1].value // OK, value is @ ~local
}

main() {
    println(pickSecond("x", "y"))
}
```

The output is:

```text
y
```

In this way, the array `Array<Demode<String>> @ local!` is released with the local region when the function returns, while the `value` of each element references an ordinary object that is not affected and can be returned or stored.

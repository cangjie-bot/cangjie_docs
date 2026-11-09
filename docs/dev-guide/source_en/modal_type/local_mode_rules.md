# Rules for Local Mode

> **Note:**
>
> Local mode is an experimental feature. All examples in this section are written with local mode enabled and must be compiled with the `--experimental --enable-local-mode` compilation options. See [Enabling Local Mode](./modal_overview.md#enabling-local-mode).

This section describes the compile-time constraints of local mode.

## Assignment, Return, and Escape

In an assignment, besides the data types being compatible, the mode and lifetime requirements of the value must also be met.

`Copyable` is an interface that identifies types whose values can be copied by value and converted between modes. Numeric types, `Bool`, `Rune`, and other types already implement this interface. For example, the following `Int64` value can be copied from `@ local!` to `@ ~local` and then returned. For the complete rules, see [Copyable and demode](./copyable_and_demode.md).

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func count(): Int64 {
    let n: Int64 @ local! = 42
    let m: Int64 = n // OK, Int64 is Copyable
    return n + m
}

main() {
    println(count())
}
```

The output is:

```text
84
```

The following assignment rules apply to non-`Copyable` types.

`@ local!` values fall into two categories, external and internal:

- **External local!**: the `@ local!` parameters of a function, and the members, tuple elements, and `VArray` elements accessed through them. They are created by the caller and belong to the caller's local region.
- **Internal local!**: `@ local!` values created in the current function. They belong to the current function's local region and are released when the function returns.

The two are distinguished to prevent objects with a longer lifetime from referencing objects with a shorter lifetime: the caller's objects may still be used after the current function returns, so local objects created in this function cannot be stored in their members.

Outside an exclave, an external local! of a non-`Copyable` type cannot be passed on directly as an argument to another function, even if the parameter is `@ local?`. This is a call constraint in addition to the subtype relationship.

For example, the external local! parameter `p` received by `f` belongs to the caller of `f`. If `f` were allowed to pass `p` directly to a `@ local!` parameter of `g`, `g` would treat it as belonging to the local region of its direct caller `f`. Then, if `g` creates an object in an exclave and stores it in a member of `p`, the new object is allocated in the local region of `f`, while `p` actually belongs to a caller further out. After `f` returns, the member of `p` references a released object.

If you only need to read the value, first assign the external value to a `@ local?` local variable and then pass it to a `@ local?` parameter. Operations through this reference are then subject to `@ local?` restrictions; for example, you cannot assign to its non-`Copyable` members. To operate in the caller's region, use exclave. For a complete example, see [Passing @ local! Parameters](#passing--local-parameters).

### Variable Initialization and Whole Assignment

For `var x: T @ mode`, the rules for the whole assignment `x = value` are as follows. A `let` variable must meet the same mode requirements when it is initialized, but cannot be assigned again afterward.

| Mode of the target variable and where the assignment occurs | Values that can be assigned |
|---|---|
| `@ ~local` | Only `@ ~local` |
| `@ local!` | Only `@ local!`, and the target and the value must both be internal or both be external |
| `@ local?`, initialized or assigned in the function or lambda that defines the variable | `@ ~local`, `@ local!`, or `@ local?` |
| `@ local?`, assigned inside a lambda or nested function that captures the variable | Only `@ ~local` |

For a captured `@ local?` variable of a non-`Copyable` type, only `@ ~local` values can be assigned to it inside the lambda or nested function that captures it. It cannot be assigned a local value created in that function; otherwise the variable might still reference a released object after the closure returns. Variables of `Copyable` types are not subject to this assignment restriction.

A local variable declared as `@ local!` is always an internal local!, so an external local! (except for `Copyable` types) cannot be used to initialize a `@ local!` local variable. To keep a reference to an external value in a local variable, declare the variable as `@ local?`. For an example, see [Passing @ local! Parameters](#passing--local-parameters).

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func keep(s: String @ local!) {
    let alias: String @ local! = s // Error, cannot initialize an internal local! with an external local!
}
```

In the following example, the `@ local?` variable `any` receives a `@ local!` array and then a `@ ~local` array:

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func assign() {
    var plain: Array<Int64> = [1, 2, 3] // @ ~local variable
    var loc: Array<Int64> @ local! = Array<Int64>(2, {i => i * 10}) @ local! // @ local! variable
    var any: Array<Int64> @ local? = plain // @ local? variable

    any = loc // OK, @ local? accepts a @ local! value
    println(any.size)
    any = plain // OK, @ local? also accepts a @ ~local value
    println(any.size)
}

main() {
    assign()
}
```

The output is:

```text
2
3
```

`@ ~local` and `@ local!` variables cannot be assigned to each other:

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func assign() {
    var plain: Array<Int64> = [1, 2, 3]
    var loc: Array<Int64> @ local! = Array<Int64>(2, {i => i * 10}) @ local!
    plain = loc // Error, a @ ~local variable cannot accept a @ local! value
    loc = plain // Error, a @ local! variable cannot accept a @ ~local value
}
```

Local values of non-`Copyable` types cannot be stored in global variables or static member variables, because those can only use the `@ ~local` mode. Nor can a local value be returned directly as a `@ ~local` return value:

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
var cache: String = ""

func process(): String {
    let s: String @ local! = "temporary"
    let t: String = s // Error, @ local! cannot be assigned to @ ~local
    cache = s // Error, @ local! cannot be stored in a global variable
    return s // Error, @ local! cannot be returned as @ ~local
}
```

### Modes of Members and Instances

Instance member variables of classes and structs are declared without a mode; their mode is determined by the instance they belong to. A member accessed through a `@ local!` instance is `@ local!`, and a member accessed through a `@ local?` instance is `@ local?`. Likewise, the elements of `Array<T> @ local!` are also `@ local!`.

The mode of a tuple is determined by its elements together: if the elements include both `@ local!` values and `@ ~local` values, the mode of the tuple is `@ local?`.

### Assignment to Instance Member Variables

For instance member variables of non-`Copyable` types, the assignment rules are as follows:

| Mode of the member | Values that can be assigned |
|---|---|
| `@ ~local` | Only `@ ~local` |
| `@ local!` | Only `@ local!`, and the member and the value must both be internal or both be external |
| `@ local?` | Assignment is generally not allowed |

A `@ local?` instance may actually be non-local, or it may be `@ local!`: the members of the former must not reference local objects, while the members of the latter must be local objects. Whatever the mode of the assigned value, it may violate one of the two cases, so assignment is not allowed.

Constructors, instance member functions, and finalizers can specify the mode of `this` with a `this` mode specifier, `this @ mode`, at the beginning of the parameter list. For the syntax, see [Constructing Local Instances](#constructing-local-instances), [The this Mode of Member Functions](#the-this-mode-of-member-functions), and [Quick Reference for Other Restrictions](#quick-reference-for-other-restrictions). In a `this @ local?` constructor or a finalizer, the members of `this` can be assigned values of compatible data types in any mode.

Members of `Copyable` types are not subject to these mode restrictions. `demode` members always use the `@ ~local` mode; see [demode Member Variables](./copyable_and_demode.md#demode-member-variables).

Array elements follow the same rules: non-`Copyable` elements of a `@ local?` array cannot be assigned, while `Copyable` elements are not subject to this restriction. The following example shows, in order: a member read through a `@ local!` instance is also `@ local!` and cannot be stored in a global variable; non-`Copyable` members and elements of `@ local?` instances and arrays cannot be assigned; `Copyable` members and elements can be assigned:

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
class Order {
    var item: String
    var count: Int64
    init(item: String, count: Int64) {
        this.item = item
        this.count = count
    }
}

var lastItem: String = ""

func record(o: Order @ local!) {
    lastItem = o.item // Error, o.item has type String @ local!
}

func update(o: Order @ local?) {
    o.count = 2 // OK, Int64 is Copyable
    o.item = "pen" // Error, cannot assign to non-Copyable member of a @ local? instance
}

func fill(names: Array<String> @ local?, counts: Array<Int64> @ local?) {
    counts[0] = 0 // OK, Int64 is Copyable
    names[0] = "none" // Error, cannot assign to non-Copyable element of a @ local? array
}
```

### Return Values

Non-`Copyable` local values created inside a function cannot be used after the function returns. External `@ local!` parameters and their members belong to the caller's local region and can be returned to the caller as `@ local!` return values. To return a local object newly created in the function, use [exclave](./exclave.md#returning-local-objects).

A `@ local?` variable or parameter can receive both local and non-local values, so it is often used for function parameters that only read data without storing it:

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func describe(s: String @ local?): String {
    if (s.isEmpty()) {
        "empty"
    } else {
        "size ${s.size}"
    }
}

main() {
    let a: String @ local! = "local"
    let b: String = "not local"
    println(describe(a))
    println(describe(b))
}
```

The output is:

```text
size 5
size 9
```

Because a `@ local?` value may be local, it cannot be returned as a `@ ~local` return value:

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func keep(s: String @ local?): String {
    return s // Error, @ local? cannot be returned as @ ~local
}
```

Outside an exclave, a `@ local?` parameter itself, and the (possibly nested) fields accessed through the parameter, can be returned directly as `@ local?` return values. The return value is still subject to local restrictions and is not converted to `@ ~local`:

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func identity(s: String @ local?): String @ local? {
    s
}

main() {
    let local: String @ local! = "local"
    println(identity(local))
    println(identity("plain"))
}
```

The output is:

```text
local
plain
```

Array elements are not included. A `@ local?` local variable cannot be returned this way either, because it may reference a local object created in this function:

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func firstOf(xs: Array<String> @ local?): String @ local? {
    xs[0] // Error, an array element is not a field of the parameter
}

func viaLocal(s: String @ local?): String @ local? {
    let v: String @ local? = s
    v // Error, a @ local? local variable cannot be returned
}
```

Constructing a new array or tuple differs from accessing a member of an external value. For an array or tuple literal to use the `@ local!` mode, all its elements must be internal values; a literal containing external local! elements is not treated as an external local! as a whole and cannot be returned directly. To combine external local! elements into a local return value, construct and return it in an exclave; see [Returning Local Objects](./exclave.md#returning-local-objects).

## Constructing Local Instances

For non-`Copyable` types, the mode of an instance is determined by the constructor. If you write `this @ local!` or `this @ local?` at the beginning of the parameter list of `init`, that constructor constructs instances of the corresponding mode; omitting it is equivalent to `this @ ~local`. The primary constructor always constructs `@ ~local` instances. `Copyable` types can also convert modes by value after ordinary construction; see the example below.

The body of a `this @ local!` or `this @ local?` constructor must begin with an exclave expression, which is usually expressed by modifying `init` with `exclave` directly (see [exclave Expressions](./exclave.md)). Members can be assigned in such a constructor, and the exception for initializing `this` described in [Assignment to Instance Member Variables](#assignment-to-instance-member-variables) applies.

When constructing an instance, you can explicitly write a mode after the constructor call to select a constructor, such as `Order("pen", 2) @ local!`. If the mode is omitted, the compiler infers it from the context, such as the variable type, the parameter type, or the return type; if it cannot be uniquely inferred, it defaults to `@ ~local`. Once the mode is determined, a constructor is selected from those with the same `this` mode according to the function overloading rules. The following example selects constructors by the variable type:

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
class Order {
    var item: String
    var count: Int64
    init(item: String, count: Int64) {
        this.item = item
        this.count = count
    }
    exclave init(this @ local!, item: String @ local!, count: Int64) {
        this.item = item
        this.count = count
    }
}

main() {
    let a = Order("book", 1) // @ ~local instance
    let b = Order("pen", 2) @ local! // @ local! instance
    let c: Order @ local! = Order("ink", 3) // the this @ local! constructor is selected by the variable type
    println("${a.item} ${b.item} ${c.item}")
}
```

The output is:

```text
book pen ink
```

A `this @ local?` constructor is used when the modes of the arguments are uncertain: its parameters can be declared as `@ local?`, so that it can receive both local and non-local values and store them in members. The constructed instance is `@ local?` and, like a local instance, is released with the local region. It cannot replace a `this @ local!` or `this @ ~local` constructor: if there is only a `this @ local?` constructor, `@ local!` or `@ ~local` instances cannot be constructed.

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
class Tag {
    var name: String
    init(name: String) {
        this.name = name
    }
    exclave init(this @ local?, name: String @ local?) {
        this.name = name
    }
}

func size(t: Tag @ local?): Int64 {
    t.name.size
}

func use(text: String @ local?): Int64 {
    let t = Tag(text) @ local? // text may or may not be local
    size(t)
}

main() {
    let local: String @ local! = "local"
    println(use(local))
    println(use("plain"))
}
```

The output is:

```text
5
5
```

If a non-`Copyable` type has no constructor of the corresponding mode, instances of that mode cannot be constructed. For example, the following `Point` has only a constructor of the default mode:

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
class Point {
    var x: Int64
    init(x: Int64) {
        this.x = x
    }
}

main() {
    let p = Point(1) @ local! // Error, no @ local! constructor
}
```

A `Copyable` type can first use an ordinary constructor and then copy the result by value into a variable of another mode, so no local constructor needs to be added for this conversion. The following `P` defines only an ordinary constructor:

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
struct P <: Copyable {
    var x: Int64
    init(x: Int64) {
        this.x = x
    }
}

main() {
    let p: P @ local! = P(1) // constructed by the ordinary constructor, then copied
    let plain = P(2)
    let copied: P @ local! = plain
    println("${p.x} ${copied.x}")
}
```

The output is:

```text
1 2
```

Explicitly writing `P(1) @ local!` after the constructor call still means selecting a `this @ local!` constructor. If there is no such constructor, an error is reported even though `P` implements `Copyable`. To construct normally and then copy, use the variable type annotation as in the above example:

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
struct P <: Copyable {
    var x: Int64
    init(x: Int64) {
        this.x = x
    }
}

func create() {
    let p = P(1) @ local! // Error, the call explicitly requires a @ local! constructor
}
```

If a `this @ local!` or `this @ local?` constructor is not written as an exclave, the compiler reports an error:

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
class Label {
    var text: String
    init(text: String) {
        this.text = text
    }
    init(this @ local!, text: String @ local!) { // Error, body must be an exclave
        this.text = text
    }
}
```

When a constructor calls another constructor of the current type through `this(...)`, or a superclass constructor through `super(...)`, the called constructor must have the same `this` mode as the current constructor. In a `this @ local!` or `this @ local?` constructor, the call must also be the first expression in the exclave block:

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
open class Base {
    var n: Int64 = 7
    init() {}
    exclave init(this @ local!) {}
}

class Child <: Base {
    exclave init(this @ local!) {
        super() // OK, calls Base's this @ local! constructor
    }
}

main() {
    let child = Child() @ local!
    println(child.n)
}
```

The output is:

```text
7
```

If the superclass has only the default `this @ ~local` constructor, a local constructor cannot call it:

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
open class Base {
    init() {}
}

class Child <: Base {
    exclave init(this @ local!) {
        super() // Error, no constructor with this @ local!
    }
}
```

## The this Mode of Member Functions

The mode of `this` in an instance member function also needs to be determined. Specify it by writing `this @mode` at the beginning of the parameter list, such as `func show(this @ local?): String`; omitting it is equivalent to `this @ ~local`. `this @...` only describes `this` and is not a parameter, so no value needs to be or can be passed for it in a call.

When calling a member function through an instance, the mode of the instance must be a submode of the `this` mode:

| Mode of the instance | Member functions that can be called |
|-----------|------------------|
| `@ ~local` | `this @ ~local` (that is, omitted), `this @ local?` |
| `@ local!` | `this @ local!`, `this @ local?` |
| `@ local?` | `this @ local?` |

Therefore, if you want a read-only member function to be usable on instances of any mode, declare it as `this @ local?`.

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
class Counter {
    var n: Int64 = 0
    init() {}
    exclave init(this @ local!) {}
    func value(): Int64 {
        n
    }
    func reset(this @ local!): Unit {
        n = 0
    }
}

func use(c: Counter @ local?) {
    let a = Counter() @ local!
    a.value() // Error, a @ local! instance cannot call a this @ ~local member function
    c.value() // Error, a @ local? instance cannot call a this @ ~local member function
    Counter().reset() // Error, a @ ~local instance cannot call a this @ local! member function
}
```

The same member function can be overloaded on the `this` mode alone. Among the candidates that pass type checking, matching of the `this` mode takes precedence over parameter types: the version whose `this` mode best matches the mode of the instance is selected first, and parameter types are compared only if that cannot determine the choice:

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
class C {
    init() {}
    exclave init(this @ local!) {}
    func name(this @ ~local): String {
        "this @ ~local"
    }
    func name(this @ local?): String {
        "this @ local?"
    }
    func name(this @ local!): String {
        "this @ local!"
    }
}

func test(p: C @ local?) {
    let a = C()
    let b = C() @ local!
    println(a.name())
    println(b.name())
    println(p.name())
}

main() {
    test(C())
}
```

The output is:

```text
this @ ~local
this @ local!
this @ local?
```

When implementing an interface function or overriding a superclass function, the `this` mode must be exactly the same as that of the function being implemented or overridden:

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
interface Shape {
    func area(this @ local?): Int64
}

class Square <: Shape {
    let side: Int64
    init(side: Int64) {
        this.side = side
    }
    public func area(this @ local?): Int64 { // same this mode as Shape.area
        side * side
    }
}

func areaOf(s: Shape @ local?): Int64 {
    s.area()
}

main() {
    println(areaOf(Square(3)))
}
```

The output is:

```text
9
```

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
interface Shape {
    func area(this @ local?): Int64
}

class Square <: Shape { // Error, area(this @ local?) is not implemented
    public func area(): Int64 {
        1
    }
}

open class Base {
    public open func id(this @ local?): Int64 {
        1
    }
}

class Derived <: Base {
    public override func id(): Int64 { // Error, this mode differs from the overridden function
        2
    }
}
```

## Overloading by Parameter Mode

Functions can be overloaded on parameter modes alone. A common practice in the standard library is to provide both `f(x: T)` and `f(x: T @ local?)`. In a call, the compiler selects the version whose parameter mode best matches the argument mode:

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func f(_: String): String {
    "f(String)"
}

func f(_: String @ local?): String {
    "f(String @ local?)"
}

func g(_: String @ local!): String {
    "g(String @ local!)"
}

func g(_: String @ local?): String {
    "g(String @ local?)"
}

func test(p: String @ local?) {
    let a: String = "a"
    let b: String @ local! = "b"
    println(f(a))
    println(f(b))
    println(f(p))
    println(g(a))
    println(g(b))
    println(g(p))
}

main() {
    test("p")
}
```

The output is:

```text
f(String)
f(String @ local?)
f(String @ local?)
g(String @ local?)
g(String @ local!)
g(String @ local?)
```

That is: for a `@ ~local` argument, the `@ ~local` version is preferred, falling back to the `@ local?` version if there is none; for a `@ local!` argument, the `@ local!` version is preferred, falling back to the `@ local?` version if there is none; for a `@ local?` argument, only the `@ local?` version can be selected.

If no version can receive the mode of the argument, the call reports an error. For example, when only a `@ local!` version is provided, `@ ~local` or `@ local?` arguments cannot be passed:

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func h(s: String @ local!): Int64 {
    s.size
}

func test(p: String @ local?) {
    let a: String = "a"
    h(a) // Error, no h accepts a @ ~local argument
    h(p) // Error, no h accepts a @ local? argument
}
```

## Passing @ local! Parameters

For the difference between external and internal local! and the restrictions on passing arguments, see [Assignment, Return, and Escape](#assignment-return-and-escape). For the restrictions on initializing local variables, see [Variable Initialization and Whole Assignment](#variable-initialization-and-whole-assignment).

Outside an exclave, an external local! (except for `Copyable` types) has two more restrictions:

- It cannot be passed directly as an argument to another function.
- It cannot be used to call a `this @ local!` member function.

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
class Doc {
    var title: String
    init(title: String) {
        this.title = title
    }
    exclave init(this @ local!, title: String @ local!) {
        this.title = title
    }
    exclave func clear(this @ local!): Unit {
        title = ""
    }
}

func size(s: String @ local?): Int64 {
    s.size
}

func process(d: Doc @ local!): Int64 {
    d.clear() // Error, cannot call a this @ local! member function on a @ local! parameter
    size(d.title) // Error, cannot pass a @ local! parameter (or its member) as an argument
}
```

To pass it on, use one of the following two approaches:

- First assign the parameter to a `@ local?` variable, and then pass that variable to a `@ local?` parameter.
- Pass it or make the call in an [exclave](./exclave.md). There, an external local! is treated as an internal local!, so it can be passed to `@ local!` parameters and can call `this @ local!` member functions.

Calling `this @ local?` member functions and accessing members of `Copyable` types through an external local! are not subject to these restrictions.

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func size(s: String @ local?): Int64 {
    s.size
}

func sizeL(s: String @ local!): Int64 {
    s.size
}

func forward1(s: String @ local!): Int64 {
    let t: String @ local? = s
    size(t)
}

exclave func forward2(s: String @ local!): Int64 {
    sizeL(s)
}

main() {
    let s: String @ local! = "abc"
    println(forward1(s))
    println(forward2(s))
}
```

The output is:

```text
3
3
```

## Closures and Nested Functions

Lambda expressions and nested functions capture outer variables, with the following capture rules:

- `@ local!` variables of non-`Copyable` types cannot be captured.
- A lambda that captures `@ local?` variables of non-`Copyable` types has the mode `@ local?` itself, so it cannot escape; for example, it cannot be assigned to a `@ ~local` variable. A lambda that captures only local variables of `Copyable` types still has the mode `@ ~local`.
- `@ local!` function objects cannot be created, that is, the type of a lambda cannot be a `@ local!` function type.

Variables of `Copyable` types are captured by value: the closure holds a copy of the value and does not reference any object in a local region, so these restrictions do not apply. For example, the following lambda captures a local variable of type `Int64`, and its mode is still `@ ~local`, so it can still be used after the function returns:

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func makeCounter(): () -> Int64 {
    let n: Int64 @ local! = 42
    {=> n} // OK, Int64 is Copyable and captured by value
}

main() {
    println(makeCounter()())
}
```

The output is:

```text
42
```

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
var onDone: () -> Int64 = {=> 0}

func run() {
    let text: String @ local! = "abc"
    let view: String @ local? = text
    let byText = {=> text.size} // Error, cannot capture a @ local! variable
    let byView = {=> view.size}
    onDone = byView // Error, a @ local? lambda cannot be assigned to @ ~local
    let fixed: (() -> Int64) @ local! = {=> 1} // Error, a lambda cannot have a @ local! function type
}
```

When capturing variables of non-`Copyable` types, however, the closure references objects in a local region, so the above restrictions apply:

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func makeLabel(): () -> Int64 {
    let s: String @ local! = "abc"
    {=> s.size} // Error, cannot capture a @ local! variable
}
```

A lambda that captures `@ local?` variables can be called normally within the current function. When writing out its type, parenthesize the entire function type and write `(() -> Int64) @ local?`; `() -> Int64 @ local?` specifies the mode of the return value:

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func run() {
    let text: String @ local! = "hello"
    let view: String @ local? = text
    let length: (() -> Int64) @ local? = {=> view.size}
    println(length())
}

main() {
    run()
}
```

The output is:

```text
5
```

Nested functions are `@ ~local` by default and, as in ordinary Cangjie, can be freely returned and stored, so they cannot capture `@ local?` or `@ local!` variables of non-`Copyable` types. To capture `@ local?` variables of non-`Copyable` types, write `@ local?` before `func` (a function mode specifier). This mode specifies the mode of the nested function as a function object, not the mode of its return value: in the following example, the type of `length` is `(() -> Int64) @ local?`, and like a lambda that captures `@ local?` variables, it cannot escape. With or without `@ local?`, nested functions cannot capture `@ local!` variables of non-`Copyable` types; local variables of `Copyable` types are not subject to these capture restrictions:

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func run() {
    let text: String @ local! = "hello"
    let view: String @ local? = text
    @ local? func length(): Int64 {
        view.size
    }
    println(length())
}

main() {
    run()
}
```

The output is:

```text
5
```

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func run() {
    let text: String @ local! = "hello"
    let view: String @ local? = text
    func length1(): Int64 {
        view.size // Error, a @ ~local nested function cannot capture a @ local? variable
    }
    @ local? func length2(): Int64 {
        text.size // Error, cannot capture a @ local! variable
    }
}
```

## Constraints

The following scenarios impose additional restrictions on local values:

- **spawn**: A spawn block cannot use `@ local!` or `@ local?` variables of non-`Copyable` types declared outside the block, and the result of the block can only be a `@ ~local` value. A spawn block can create its own local objects, which are released when the block finishes executing. Exclave expressions cannot be used in a spawn block.
- **Exceptions**: The operand of `throw` can only be a `@ ~local` value. The exception variable bound in `catch` is `@ ~local` and can be stored freely.
- **Generics**: Generic type arguments cannot carry modes (such as `Array<String @ local!>`). Type parameters do not take on the mode of the argument either: when `String @ local!` is passed, `T` is inferred as `String`, and the parameter `x: T` is still `@ ~local`. To receive local values, write it as `x: T @ local?` or a similar form; see the example below.
- **C interoperability**: The parameter types and return types of functions modified with `@C` can only use the `@ ~local` mode.
- **Where exclave can be used**: exclave cannot be used directly in spawn blocks, `main`, finalizers, or static initializers, and cannot be used in a try or catch block to end the function containing that block. For details, see [exclave Expressions](./exclave.md#usage-restrictions).

The following example uses an outer `Copyable` variable in a spawn block and creates its own local array inside the block. The array is released when the spawn block ends, and the sum is an ordinary `Int64` value that can be returned:

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func sumInThread(n: Int64): Int64 {
    let limit: Int64 @ local! = n // Int64 is Copyable, can be used in spawn
    let fut = spawn {
        let squares = Array<Int64>(limit, {i => i * i}) @ local! // local to the spawn block
        var sum = 0
        for (i in 0..squares.size) {
            sum += squares[i]
        }
        sum
    }
    fut.get()
}

main() {
    println(sumInThread(4))
}
```

The output is:

```text
14
```

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func f() {
    let s: String @ local? = "s"
    spawn {
        println(s) // Error, cannot use a @ local? variable declared outside spawn
    }
}

func g(e: Exception @ local?) {
    throw e // Error, the thrown value must be @ ~local
}

func id<T>(x: T): Unit {}

func h() {
    let s: String @ local! = "s"
    id(s) // Error, T is inferred as String, so the parameter x: T is @ ~local
}

@C
func fromC(p: CPointer<Int8> @ local!): Unit {} // Error, @C function cannot use modal types
```

When a generic function needs to receive local values, annotate the parameter as `@ local?`. Elements read from `Array<T> @ local?` have the type `T @ local?` and are also subject to local restrictions; to return an element as `T`, constrain `T <: Copyable`:

<!-- verify -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func count<T>(xs: Array<T> @ local?): Int64 {
    xs.size
}

func first<T>(xs: Array<T> @ local?): T where T <: Copyable {
    xs[0] // OK, the element is copied out because T is Copyable
}

main() {
    let names: Array<String> @ local! = ["a", "b", "c"]
    println(count(names))
    let nums: Array<Int64> @ local! = [7, 8, 9]
    println(first(nums))
}
```

The output is:

```text
3
7
```

Without the `Copyable` constraint, an element cannot be returned as `T`:

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func firstAny<T>(xs: Array<T> @ local?): T {
    xs[0] // Error, the element is T @ local?
}
```

> **Note:**
>
> When an interface function is called through a `@ local?` value, the `this @ local?` version in the interface is called, and its return value may also carry a mode. Some interfaces in the standard library (such as `ToString`) provide default implementations for such functions that throw `UnimplementedException`. For example, when `x.toString()` is called on `x: T @ local?` (`T <: ToString`) and the actual type implements only the ordinary `toString()`, the program compiles but throws an exception at runtime.

## Quick Reference for Other Restrictions

The following restrictions are rarely encountered and are only listed briefly here:

1. The `this` mode of a finalizer must be a supermode of the `this` modes of all constructors. For example, when there are both `this @ ~local` and `this @ local!` constructors, the finalizer must be written as `~init(this @ local?)`.
2. If the constructors of a type can construct instances of different modes, its member variables of non-`Copyable` types cannot have initial values at their declarations, except for `demode` member variables.
3. Instance member properties with the same name can be overloaded by mode, but their data types must be the same.
4. `abstract` functions cannot be modified with `exclave`.
5. Outside an exclave, external `@ local!` parameters of non-`Copyable` types cannot be passed directly as arguments to other functions; see [Passing @ local! Parameters](#passing--local-parameters).
6. When pattern matching on a `@ local!` parameter, the bound variable is `@ local?` rather than `@ local!`.
7. `this` mode specifiers are used for instance member functions, constructors, and finalizers, and cannot be used for global functions or static functions.
8. A mode can only be written after a constructor call, such as `C() @ local!`; writing it after an ordinary function call reports an error.
9. The getters and setters of instance member properties follow the call and overloading rules of the corresponding member functions, and their `this` mode is the same as the mode of the property type. For example, `prop name: String @ local!` can be accessed only through a `@ local!` instance; to make a property readable on instances of any mode, write the property type as `@ local?`.
10. Ordinary parameters of the primary constructor can only use the `@ ~local` mode.
11. The parameters and return values of macros can only use the `@ ~local` mode.
12. The operand of `perform` in effect handlers can only be a `@ ~local` value.

The default value expression of an optional parameter cannot reference other parameters, and cannot capture `@ local!` variables of non-`Copyable` types.

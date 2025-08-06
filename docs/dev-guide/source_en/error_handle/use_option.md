# Using Option

In the [Option Type](../enum_and_pattern_match/option_type.md), the definition of the Option type was introduced. Since the Option type can represent both the presence and absence of a value, and the absence of a value can sometimes be interpreted as an error, the Option type can also be used for error handling.

For example, in the following case, if the parameter value of the function `getOrThrow` equals `Some(v)`, it returns the value `v`; if the parameter value equals `None`, it throws an exception.

<!-- compile -->

```cangjie
func getOrThrow(a: ?Int64) {
    match (a) {
        case Some(v) => v
        case None => throw NoneValueException()
    }
}
```

Because `Option` is a very commonly used type, Cangjie provides multiple ways to destructure it for convenient usage, including: pattern matching, the `getOrThrow` function, the coalescing operator (`??`), and the question mark operator (`?`). These methods will be introduced one by one below.

- **Pattern Matching**: Since the Option type is an enum type, the pattern matching for enums mentioned earlier can be used to destructure `Option` values. For example, in the following function `getString`, which takes a parameter of type `?Int64`, when the parameter is a `Some` value, it returns the string representation of the numerical value; when the parameter is `None`, it returns the string `"none"`.

    <!-- verify -->

    ```cangjie
    func getString(p: ?Int64): String{
        match (p) {
            case Some(x) => "${x}"
            case None => "none"
        }
    }
    main() {
        let a = Some(1)
        let b: ?Int64 = None
        let r1 = getString(a)
        let r2 = getString(b)
        println(r1)
        println(r2)
    }
    ```

   The execution result of the above code is:

    ```text
    1
    none
    ```

- **Coalescing Operator (`??`)**: For an expression `e1` of type `?T`, if you want to return a value `e2` of type `T` when `e1` equals `None`, you can use the `??` operator. For the expression `e1 ?? e2`, when `e1` equals `Some(v)`, it returns the value `v`; otherwise, it returns the value `e2`. Example:

    <!-- verify -->

    ```cangjie
    main() {
        let a = Some(1)
        let b: ?Int64 = None
        let r1: Int64 = a ?? 0
        let r2: Int64 = b ?? 0
        println(r1)
        println(r2)
    }
    ```

   The execution result of the above code is:

    ```text
    1
    0
    ```

- **Question Mark Operator (`?`)**: The `?` operator must be used with `.`, `()`, `[]`, or `{}` (specifically in the context of trailing lambda calls) to support `.`, `()`, `[]`, and `{}` operations for the `Option` type. Taking `.` as an example (the same applies to `()`, `[]`, and `{}`), for an expression `e` of type `?T1`, when `e` equals `Some(v)`, the value of `e?.b` equals `Option<T2>.Some(v.b)`; otherwise, `e?.b` equals `Option<T2>.None`, where `T2` is the type of `v.b`. Example:

    <!-- compile -->

    ```cangjie
    struct R {
        public var a: Int64
        public init(a: Int64) {
            this.a = a
        }
    }

    let r = R(100)
    let x = Some(r)
    let y = Option<R>.None
    let r1 = x?.a   // r1 = Option<Int64>.Some(100)
    let r2 = y?.a   // r2 = Option<Int64>.None
    ```

   The question mark operator (`?`) supports multi-level access. Taking `a?.b.c?.d` as an example (the same applies to `()`, `[]`, and `{}`), the type of expression `a` must be an `Option<T1>`, and `T1` must contain an instance member `b`. The type of `b` must contain an instance member variable `c`, and the type of `c` must be an `Option<T2>`. `T2` must contain an instance member `d`. The type of the expression `a?.b.c?.d` is `Option<T3>`, where `T3` is the type of the instance member `d` of `T2`. When `a` equals `Some(va)` and `va.b.c` equals `Some(vc)`, `a?.b.c?.d` equals `Option<T3>.Some(vc.d)`. When `a` equals `Some(va)` and `va.b.c` equals `None`, `a?.b.c?.d` equals `Option<T3>.None` (`d` will not be evaluated). When `a` equals `None`, `a?.b.c?.d` equals `Option<T3>.None` (`b`, `c`, and `d` will not be evaluated).

    <!-- compile -->

    ```cangjie
    struct A {
        let b: B = B()
    }

    struct B {
        let c: Option<C> = C()
        let c1: Option<C> = Option<C>.None
    }

    struct C {
        let d: Int64 = 100
    }

    let a = Some(A())
    let a1 = a?.b.c?.d // a1 = Option<Int64>.Some(100)
    let a2 = a?.b.c1?.d // a2 = Option<Int64>.None
    ```

- **`getOrThrow` Function**: For an expression `e` of type `?T`, you can destructure it by calling the `getOrThrow` function. When `e` equals `Some(v)`, `getOrThrow()` returns the value `v`; otherwise, it throws an exception. Example:

    <!-- verify -->

    ```cangjie
    main() {
        let a = Some(1)
        let b: ?Int64 = None
        let r1 = a.getOrThrow()
        println(r1)
        try {
            let r2 = b.getOrThrow()
        } catch (e: NoneValueException) {
            println("b is None")
        }
    }
    ```

   The execution result of the above code is:

    ```text
    1
    b is None
    ```
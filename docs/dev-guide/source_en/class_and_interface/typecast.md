# Type Conversion

Cangjie does not support implicit conversion between different types (subtypes are inherently parent types, so conversion from a subtype to a parent type is not implicit type conversion). Type conversion must be performed explicitly. The following sections will introduce the `is` and `as` operators.

For conversions between numeric types (integer and floating-point types), refer to [Numeric Type Conversions](../basic_data_type/integer.md#numeric-type-conversions). For conversions between `Rune` and integer types, refer to [Conversion from `Rune` to `UInt32` and from Integer Types to `Rune`](../basic_data_type/integer.md#conversion-from-rune-to-uint32-and-from-integer-types-to-rune).

## The `is` and `as` Operators

Cangjie supports using the `is` operator to determine whether the type of an expression is the specified type (or its subtype). Specifically, for the expression `e is T` (where `e` can be any expression and `T` can be any type), when the runtime type of `e` is a subtype of `T`, the value of `e is T` is `true`; otherwise, it is `false`.

The following example demonstrates the use of the `is` operator:

<!-- verify -->

```cangjie
open class Base {
    var name: String = "Alice"
}

class Derived <: Base {
    var age: UInt8 = 18
}

main() {
    let a = 1 is Int64
    println("Is the type of 1 'Int64'? ${a}")
    let b = 1 is String
    println("Is the type of 1 'String'? ${b}")

    let b1: Base = Base()
    let b2: Base = Derived()
    var x = b1 is Base
    println("Is the type of b1 'Base'? ${x}")
    x = b1 is Derived
    println("Is the type of b1 'Derived'? ${x}")
    x = b2 is Base
    println("Is the type of b2 'Base'? ${x}")
    x = b2 is Derived
    println("Is the type of b2 'Derived'? ${x}")
}
```

The execution result of the above code is:

```text
Is the type of 1 'Int64'? true
Is the type of 1 'String'? false
Is the type of b1 'Base'? true
Is the type of b1 'Derived'? false
Is the type of b2 'Base'? true
Is the type of b2 'Derived'? true
```

The `as` operator can be used to convert the type of an expression to the specified type. Since type conversion may fail, the `as` operator returns an `Option` type. Specifically, for the expression `e as T` (where `e` can be any expression and `T` can be any type), when the runtime type of `e` is a subtype of `T`, the value of `e as T` is `Option<T>.Some(e)`; otherwise, it is `Option<T>.None`.

The following example demonstrates the use of the `as` operator (comments indicate the results of the `as` operation):

<!-- compile -->

```cangjie
open class Base {
    var name: String = "Alice"
}

class Derived <: Base {
    var age: UInt8 = 18
}

let a = 1 as Int64 // a = Option<Int64>.Some(1)
let b = 1 as String // b = Option<String>.None

let b1: Base = Base()
let b2: Base = Derived()
let d: Derived = Derived()
let r1 = b1 as Base // r1 = Option<Base>.Some(b1)
let r2 = b1 as Derived // r2 = Option<Derived>.None
let r3 = b2 as Base // r3 = Option<Base>.Some(b2)
let r4 = b2 as Derived // r4 = Option<Derived>.Some(b2)
let r5 = d as Base     // r5 = Option<Base>.Some(d)
let r6 = d as Derived  // r6 = Option<Derived>.Some(d)
```

### Type Patterns and Subtype Conversion

Type patterns can also be used for subtype conversion. For an expression `e` and a type pattern `id: T`, when the runtime type of `e` is a subtype of `T`, the match succeeds, and `e` is converted to type `T` and then bound to `id`; otherwise, the match fails and execution proceeds to another branch.

Unlike the `is` operator, which is used for type checking, and the `as` operator, which returns an `Option`, a successful type pattern match directly yields a bound variable of the corresponding subtype. Here, subtype conversion means narrowing a parent-typed value to a subtype based on its runtime type. The variable bound by the pattern is only available in the branch or loop body where the match succeeds.

In the conditions of `if` expressions and `while` expressions, type patterns can be used through "let patterns".

For more information about type patterns, see [Type Patterns](../enum_and_pattern_match/pattern_overview.md#type-pattern). For more information about "let patterns", see [Examples of "Conditions" Involving let Patterns](../basic_programming_concepts/expression.md#examples-of-conditions-involving-let-patterns).

The following example demonstrates subtype conversion with type patterns in the conditions of `if` expressions and `while` expressions:

<!-- verify -->

```cangjie
open class Base {
    var name: String = "Alice"
}
class Derived <: Base {
    var age: UInt8 = 18
}

main() {
    let b1: Base = Base()
    let b2: Base = Derived()
    var b3: Base = Derived()

    if (let d: Derived <- b1) {
        println("The age of b1 is ${d.age}")
    } else {
        println("b1 is not Derived")
    }

    if (let d: Derived <- b2) {
        println("The age of b2 is ${d.age}")
    } else {
        println("b2 is not Derived")
    }

    while (let d: Derived <- b3) {
        println("The age of b3 is ${d.age}")
        b3 = Base()
    }
}
```

The execution result of the above code is:

```text
b1 is not Derived
The age of b2 is 18
The age of b3 is 18
```

In the above result, `b1` does not match successfully, while `b2` and `b3` match successfully and complete type binding.

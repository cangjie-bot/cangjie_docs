# Dynamic Features

This chapter introduces the dynamic features of Cangjie, which enable developers to implement certain functionalities more elegantly. Cangjie's dynamic features primarily include reflection.

## Basic Introduction to Cangjie Reflection

Reflection refers to a mechanism that allows a program to access, inspect, and modify its own state or behavior.

The dynamic feature of reflection offers the following advantages:

- Enhances program flexibility and extensibility.
- Enables programs to determine the types of various objects at runtime and perform operations such as enumeration and invocation of their members.
- Allows the creation of new types at runtime without the need for hardcoding in advance.

However, reflection calls typically exhibit lower performance compared to direct calls. Therefore, the reflection mechanism is mainly applied to system frameworks that require high flexibility and extensibility.

## How to Obtain TypeInfo

For Cangjie's reflection feature, it is essential to understand the `TypeInfo` type. This core type records type information for any given type and defines methods for retrieving type information, setting values, etc. To facilitate user operations, Cangjie also provides a series of information types such as `ClassTypeInfo`, `PrimitiveTypeInfo`, and `ParameterInfo`.

Three static `of` methods can be used to generate `TypeInfo` information classes.

<!-- code_no_check -->

```cangjie
public class TypeInfo {
    public static func of(a: Any): TypeInfo
    public static func of(a: Object): ClassTypeInfo
    public static func of<T>(): TypeInfo
}
```

When the `of` function with parameters of type `Any` or `Object` is used, the output is the runtime type information of the instance. The `of` function with a generic parameter returns the static type information of the passed parameter. Both methods produce identical information, but there is no guarantee that they will correspond to the same object.

For example, reflection can be used to obtain type information for a custom type.

<!-- verify -->

```cangjie
import std.reflect.*

class Foo {}

main() {
    let a: Foo = Foo()
    let info: TypeInfo = TypeInfo.of(a)
    let info2: TypeInfo = TypeInfo.of<Foo>()
    println(info)
    println(info2)
}
```

Compiling and executing the above code will output:

```text
default.Foo
default.Foo
```

Additionally, `TypeInfo` provides a static function `get`, which retrieves `TypeInfo` by passing in a type name.

<!-- code_no_check -->

```cangjie
public class TypeInfo {
    public static func get(qualifiedName: String): TypeInfo
}
```

Note that the input parameter must conform to the fully qualified pattern rule of `module.package.type`. For compiler-preloaded types, including those in the `core` package and built-in compiler types such as `primitive type`, `Option`, `Iterable`, etc., the search string should directly use the type name without the package or module prefix. If the runtime cannot find an instance of the corresponding type, an `InfoNotFoundException` will be thrown.

Example: (will throw an error when running)

<!-- run.error -->

```cangjie
import std.reflect.*

main() {
    let t1: TypeInfo = TypeInfo.get("default.Foo")
}
```

## How to Use Reflection to Access Members

Once the corresponding type information class, i.e., `TypeInfo`, is obtained, its respective interfaces can be used to access instance members and static members of the corresponding class. Additionally, the `ClassTypeInfo` subclass of `TypeInfo` provides interfaces for accessing the class's public constructors, member variables, properties, and functions. Cangjie's reflection is designed to only access public members within a type, meaning members modified as `private` or `protected` are invisible in reflection.

For example, to retrieve and modify an instance member variable of a class at runtime:

<!-- verify -->

```cangjie
import std.reflect.*

public class Foo {
    public static var param1 = 20
    public var param2 = 10
}

main(): Unit {
    let obj = Foo()
    let info = TypeInfo.of(obj)
    let staticVarInfo = info.getStaticVariable("param1")
    let instanceVarInfo = info.getInstanceVariable("param2")
    println("Initial values of member variables")
    print("Static member variable ${staticVarInfo} of Foo = ")
    println((staticVarInfo.getValue() as Int64).getOrThrow())
    print("Instance member variable ${instanceVarInfo} of obj = ")
    println((instanceVarInfo.getValue(obj) as Int64).getOrThrow())
    println("Modifying member variables")
    staticVarInfo.setValue(8)
    instanceVarInfo.setValue(obj, 25)
    print("Static member variable ${staticVarInfo} of Foo = ")
    println((staticVarInfo.getValue() as Int64).getOrThrow())
    print("Instance member variable ${instanceVarInfo} of obj = ")
    println((instanceVarInfo.getValue(obj) as Int64).getOrThrow())
    return
}
```

Compiling and executing the above code will output:

```text
Initial values of member variables
Static member variable static param1: Int64 of Foo = 20
Instance member variable param2: Int64 of obj = 10
Modifying member variables
Static member variable static param1: Int64 of Foo = 8
Instance member variable param2: Int64 of obj = 25
```

Similarly, properties can be inspected and modified via reflection.

<!-- verify -->

```cangjie
import std.reflect.*

public class Foo {
    public let _p1: Int64 = 1
    public prop p1: Int64 {
        get() { _p1 }
    }
    public var _p2: Int64 = 2
    public mut prop p2: Int64 {
        get() { _p2 }
        set(v) { _p2 = v }
    }
}

main(): Unit {
    let obj = Foo()
    let info = TypeInfo.of(obj)
    let instanceProps = info.instanceProperties.toArray()
    println("Instance member properties of obj include ${instanceProps}")
    let PropInfo1 = info.getInstanceProperty("p1")
    let PropInfo2 = info.getInstanceProperty("p2")

    println((PropInfo1.getValue(obj) as Int64).getOrThrow())
    println((PropInfo2.getValue(obj) as Int64).getOrThrow())
    if (PropInfo1.isMutable()) {
        PropInfo1.setValue(obj, 10)
    }
    if (PropInfo2.isMutable()) {
        PropInfo2.setValue(obj, 20)
    }
    println((PropInfo1.getValue(obj) as Int64).getOrThrow())
    println((PropInfo2.getValue(obj) as Int64).getOrThrow())
    return
}
```

Compiling and executing the above code will output:

```text
Instance member properties of obj include [prop p1: Int64, mut prop p2: Int64]
1
2
1
20
```

Function calls can also be made via the reflection mechanism.

<!-- verify -->

```cangjie
import std.reflect.*

public class Foo {
    public static func f1(v0: Int64, v1: Int64): Int64 {
        return v0 + v1
    }
}

main(): Unit {
    var num = 0
    let intInfo = TypeInfo.of<Int64>()
    let funcInfo = TypeInfo.of<Foo>().getStaticFunction("f1", intInfo, intInfo)
    num = (funcInfo.apply(TypeInfo.of<Foo>(), [1, 1]) as Int64).getOrThrow()
    println(num)
}
```

Compiling and executing the above code will output:

```text
2
```
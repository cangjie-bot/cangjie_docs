# Compatibility Rules

## Compatibility Rules for Global Variable Changes

### Addition and Deletion Scenarios

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding a global variable | Compatible | Compatible |
| Deleting a global variable without the public modifier or whose package does not have the public modifier | Compatible | Compatible |
| Deleting a global variable with the public modifier and whose package has the public modifier | Incompatible | Incompatible |

### Modification Scenarios

We call a global variable **visible outside the module** when it satisfies any of the following conditions:

1. The global variable has the public modifier and the package where it resides has the public modifier.
2. The global variable is depended upon by other module-externally-visible declarations, and these declarations are marked with const or @Frozen.

<!-- code_no_check -->

```cangjie
package A

// const variable x is modified with public, visible outside the module, and will participate in the compilation process of downstream modules
public const x: Int64 = 10

// Although const variable y has no public modifier and is not visible outside the module, it also participates in the compilation process of downstream modules. The reason is that y is depended upon by function foo, and the compiler's current compile-time evaluation implementation mechanism
// is to export y and foo together to downstream packages for computation
const y: Int64 = 10
public const func foo(): Int64 {
    return y
}

// Although variable z has no public modifier and is not visible outside the module, it also participates in the compilation process of downstream modules. The reason is that z is depended upon by function bar marked with Frozen,
// and the compiler will export bar's function body completely to downstream package B, causing z to be depended upon by downstream module compilation
let z: Int64 = 10
@Frozen
public func bar(): Int64 {
    return z
}

//==================================//

package B

import A.*

const m: Int64 = x
const n: Int64 = foo()
let k: Int64 = bar()
```

Modifications to any global variable not visible outside the module satisfy API + ABI compatibility; below we only discuss the module-externally-visible scenario.

#### Modifying the Definition Content of a Global Variable

##### Modifying the Compile Marker Part

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding the @Deprecated compile marker | Based on the @Deprecated parameter conditions, when the variable is unavailable in downstream modules, incompatible; otherwise, compatible | Based on the @Deprecated parameter conditions, when the variable is unavailable in downstream modules, incompatible; otherwise, compatible |
| Deleting the @Deprecated compile marker | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions such that the variable changes from available to unavailable in downstream modules | Incompatible | Incompatible |
| Modifying the @Deprecated compile marker parameter conditions such that the variable changes from unavailable to available in downstream modules | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions such that the variable's availability in downstream modules remains unchanged | Compatible | Compatible |

<!-- code_no_check -->

```cangjie
/* Modifying the compile marker part -- @Deprecated */

// Scenario 1: Adding @Deprecated["This API will be deprecated soon"] to variable v1 is API+ABI compatible, because downstream modules can still use it normally, only generating a compilation warning
// Scenario 2: Adding @Deprecated["This API is deprecated", strict: true] to variable v1 is API+ABI incompatible, because downstream modules can no longer use it normally, generating a compilation error
public let v1: Int64 = 1

// Scenario 3: Deleting the @Deprecated marker from variable v2 is API+ABI compatible, because downstream modules can resume usage
@Deprecated["This API is deprecated", strict: true]
public let v2: Int32 = 1

// Scenario 4: Modifying the @Deprecated marker on variable v3 to @Deprecated["This API is deprecated", strict: true] is API+ABI incompatible, because downstream modules can no longer use it normally
@Deprecated["This API will be deprecated soon"]
public let v3: String = "hello"
```

##### Modifying the Modifier Part

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Access modifier changed from public to non-public | Incompatible | Incompatible |
| Access modifier changed from non-public to public | Compatible | Compatible |
| Access modifier changed among the other three non-public cases (protected/internal/private) | Compatible | Compatible |
| Changing the let modifier to var modifier | Compatible | Compatible |
| Changing the var modifier to let modifier | Incompatible | Incompatible |
| Changing the let modifier to const modifier | Compatible | Incompatible |
| Changing the var modifier to const modifier | Incompatible | Incompatible |
| Changing the const modifier to let modifier | Incompatible | Incompatible |
| Changing the const modifier to var modifier | Incompatible | Incompatible |

<!-- code_no_check -->

```cangjie
/* Modifying the modifier part -- const */

package A

// Scenario 1: Modifying variable x's modifier to const is API compatible but ABI incompatible, because after compiling package A, x as a const variable, after compile-time evaluation, will be optimized away and not exist in the final binary
public let x: Int64 = 1

// Scenario 2: Modifying variable x's modifier to const is API+ABI incompatible, because after the change, the write usage of y in foo in downstream package B becomes illegal
public var y: Int64 = 1

// Scenario 3: Modifying variable z's modifier to let or var is API+ABI incompatible, because after the change, the usage of z in k in downstream package B becomes illegal
public const z: Int64 = 1

//==================================//

package B

import A.*

public let m: Int64 = x
public func foo() {
    y = 2
}
public const k: Int64 = z
```

##### Modifying the Variable Name, Type, and Initial Value Part

| Change Behavior | API Compatibility | ABI Compatibility | Note |
|-|-|-|-|
| Modifying the variable name | Incompatible | Incompatible | NA |
| Modifying the variable type from A to B | When B is a subtype of A, compatible; otherwise, incompatible | When B is a subtype of A, and both A and B are class or interface types, compatible; otherwise, incompatible | The reason the ABI compatibility requires "both A and B are class or interface types" is that a value type can be a subtype of an interface type, but their memory layouts differ and cannot be ABI compatible |
| Modifying the initial value of a non-const variable | Compatible | Compatible | NA |
| Modifying the initial value of a const variable | Compatible | Incompatible | Due to the current compiler implementation, non-module-externally-visible const variables that are directly or indirectly depended upon by module-externally-visible const variables and functions also need to be evaluated and used by the downstream code's compiler |

<!-- code_no_check -->

```cangjie
/* Modifying the variable type part */

interface I {}
struct S <: I {}
class C <: I {}

// 1) If variable t's type is modified to S, it satisfies API compatibility but not ABI compatibility. The reason is that although S is a subtype of I, value types and reference types have different memory layouts and cannot be ABI compatible
// 2) If variable t's type is modified to C, it satisfies both API and ABI compatibility
let t: I = ...
```

<!-- code_no_check -->

```cangjie
/* Modifying the initial value part -- const variable */

package A

// If the initial value of const variable x is modified, it satisfies API compatibility but not ABI compatibility. The reason is that the compile-time evaluation result in downstream package B will still use x's old initial value
public const x: Int64 = 10
// If the initial value of const variable y is modified, it also satisfies API compatibility but not ABI compatibility. The reason is that y is depended upon by function foo, and the compiler's current compile-time evaluation implementation mechanism
// is to export y and foo together to downstream packages for computation, which causes the compile-time evaluation result in downstream package B to still use y's old initial value
const y: Int64 = 10
public const func foo(): Int64 {
    return y
}

//==================================//

package B

import A.*

const m: Int64 = x
const n: Int64 = foo()
```

#### Modifying the Definition Location of a Global Variable

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Modifying the file where a global variable is defined (note: including renaming the file and moving to another file) | Compatible | If the global variable has the private modifier, incompatible; otherwise, compatible |
| Modifying the position of a global variable definition within the file | Compatible | Compatible |

## Compatibility Rules for Global Function Changes

### Addition and Deletion Scenarios

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding a global function | Compatible | Compatible |
| Deleting a global function without the public modifier or whose package does not have the public modifier | Compatible | Compatible |
| Deleting a global function with the public modifier and whose package has the public modifier | Incompatible | Incompatible |

### Modification Scenarios

We call a global function **visible outside the module** when it satisfies any of the following conditions:

1. The global function has the public modifier and the package where it resides has the public modifier.
2. The global function is used in the definition of other module-externally-visible global variables or static member variables marked with const.
3. The global function is used in the definition of other module-externally-visible functions marked with const or @Frozen.

<!-- code_no_check -->

```cangjie
package A

// Function f1 is modified with public, visible outside the module, and will participate in the compilation process of downstream modules
public func f1(): Int64 {
    return 1
}

// Although function f2 has no public modifier and is not visible outside the module, it also participates in the compilation process of downstream modules. The reason is that f2 is depended upon by function f3, and the compiler's current compile-time evaluation implementation mechanism
// is to export f3's signature and function body completely to downstream packages for computation, therefore causing f2 to be passively exported to downstream packages
const func f2(): Int64 {
    return 2
}
public const func f3(): Int64 {
    f2()
}

// Although function f4 has no public modifier and is not visible outside the module, it also participates in the compilation process of downstream modules. The reason is that f4 is depended upon by function f5 marked with Frozen,
// and the compiler will export f5's function body completely to downstream package B, causing f4 to be depended upon by downstream module compilation
func f4(): Int64 {
    return 2
}
@Frozen
public const func f5(): Int64 {
    f4()
}

//==================================//

package B

import A.*

let m: Int64 = f1()
const n: Int64 = f3()
let k: Int64 = f5()
```

Modifications to any global function not visible outside the module satisfy API + ABI compatibility; below we only discuss the module-externally-visible scenario.

#### Modifying the Definition Content of a Global Function

##### Modifying the Compile Marker Part

| Change Behavior | API Compatibility | ABI Compatibility | Note |
|-|-|-|-|
| Adding the @Deprecated compile marker | Based on the @Deprecated parameter conditions, when the function is unavailable in downstream modules, incompatible; otherwise, compatible | Based on the @Deprecated parameter conditions, when the function is unavailable in downstream modules, incompatible; otherwise, compatible | NA |
| Deleting the @Deprecated compile marker | Compatible | Compatible | NA |
| Modifying the @Deprecated compile marker parameter conditions, such that the function changes from available to unavailable in downstream modules | Incompatible | Incompatible | NA |
| Modifying the @Deprecated compile marker parameter conditions, such that the function changes from unavailable to available in downstream modules | Compatible | Compatible | NA |
| Modifying the @Deprecated compile marker parameter conditions, such that the function's availability in downstream modules remains unchanged | Compatible | Compatible | NA |
| Adding the @C compile marker | Incompatible | Incompatible | After adding the @C compile marker, the usage sites in downstream modules need to add unsafe, otherwise compilation errors will occur; it also affects the mangle name |
| Deleting the @C compile marker | Compatible | Incompatible | Deleting the @C compile marker affects the function's mangle name |
| Adding, deleting, or modifying the @CallingConv compile marker such that the calling convention of the marked function changes (note: currently two calling conventions are supported: CDECL and STDCALL)| Compatible | Incompatible | When the calling convention changes, the parameter passing mechanism etc. will change, causing ABI incompatibility |
| Adding or deleting the @CallingConv compile marker while the calling convention of the marked function remains unchanged (note: currently two calling conventions are supported: CDECL and STDCALL)| Compatible | Compatible | NA |
| Adding the @Frozen compile marker | Compatible | Compatible | NA |
| Deleting the @Frozen compile marker | Compatible | Incompatible | The reason for ABI incompatibility is to ensure the transitivity that if version A is compatible with B, and B is compatible with C, then A is compatible with C |

<!-- code_no_check -->

```cangjie
/* Modifying the compile marker part -- @C compile marker */

package A

// Scenario 1: Adding the @C compile marker to function foo is API+ABI incompatible. The reason is that the usage site of foo in m in the downstream package would need to add the unsafe marker, otherwise a compilation error occurs
public func foo(): Int64 {
    return 1
}

// Scenario 2: Deleting the @C compile marker from function bar is API compatible but ABI incompatible. The reason is that after bar changes from CFunc to a normal Cangjie function, the mangle name changes, causing the downstream package to be unable to link using the old mangle name
@C
public func bar(): Int64 {
    return 1
}

//==================================//

package B

import A.*

let m: Int64 = foo()
let n: Int64 =  unsafe { bar() }
```

<!-- code_no_check -->

```cangjie
/* Modifying the compile marker part -- @CallingConv compile marker */

// Scenario 1: Adding @CallingConv[STDCALL] to function foo is API compatible but ABI incompatible. The reason is that by default the calling convention of CFunc is CDECL, and it changes after adding the marker
// Scenario 2: Adding @CallingConv[CDECL] to function foo is API+ABI compatible. The reason is that by default the calling convention of CFunc is CDECL, and no substantial change occurs after adding the marker
@C
public func foo(): Int64 {
    return 1
}

// Scenario 3: Deleting the @CallingConv[STDCALL] compile marker from function bar is API compatible but ABI incompatible. The reason is that by default the calling convention of CFunc is CDECL, and it changes after adding the marker
@CallingConv[STDCALL]
@C
public func bar(): Int64 {
    return 1
}
```

<!-- code_no_check -->

```cangjie
/* Modifying the compile marker part -- @Frozen compile marker */

package A

// Scenario 1: Adding the @Frozen compile marker to function foo is API+ABI compatible
public func foo(): Int64 {
    return 1
}

// Scenario 2: Deleting the @Frozen compile marker from function bar is API compatible but ABI incompatible. The reason is to ensure the transitivity that if version A is compatible with B, and B is compatible with C, then A is compatible with C.
// Otherwise, users could make the modification in two steps: first delete @Frozen, then modify the bar function body, thereby producing a substantive ABI break
@Frozen
public func bar(): Int64 {
    return 1
}

//==================================//

package B

import A.*

let m: Int64 = foo()
let n: Int64 = bar()
```

##### Modifying the Modifier Part

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Access modifier changed from public to non-public | Incompatible | Incompatible |
| Access modifier changed from non-public to public | Compatible | Compatible |
| Access modifier changed among the other three non-public cases (protected/internal/private) | Compatible | Compatible |
| Adding the unsafe modifier | Incompatible | Incompatible |
| Deleting the unsafe modifier | Compatible | Compatible |
| Adding the const modifier | Compatible | Compatible |
| Deleting the const modifier | Incompatible | Incompatible |

<!-- code_no_check -->

```cangjie
/* Modifying the compile marker part -- non-access modifiers */

package A

// Scenario 1: Adding the unsafe modifier to function foo is API+ABI incompatible. The reason is that using f1 in downstream package B will cause a compilation error due to the missing unsafe marker
public func f1(): Int64 {
    return 1
}

// Scenario 2: Deleting the unsafe modifier from function f2 is API+ABI compatible
public unsafe func f2(): Int64 {
    return 1
}

//==================================//

package B

import A.*

let m: Int64 = f1()
let n: Int64 = unsafe { f2() }
```

##### Modifying Generic Parameters and Generic Parameter Constraints

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding a generic parameter | Incompatible | Incompatible |
| Deleting a generic parameter | Incompatible | Incompatible |
| Modifying the name of a generic parameter | Compatible | Compatible |
| Modifying the order of generic parameters | Incompatible | Incompatible |
| Adding a constraint upper bound to a generic parameter | Incompatible | Incompatible |
| Deleting a certain upper bound constraint of a generic parameter | Compatible | Incompatible |
| Modifying a certain upper bound constraint A of a generic parameter to B | When B is a parent type of A, compatible; otherwise, incompatible | Incompatible |
| Modifying the order among generic parameter constraints or among constraint upper bounds | Compatible | Compatible |

##### Modifying Function Parameters, Return Type, and Function Body

| Change Behavior | API Compatibility | ABI Compatibility | Note |
|-|-|-|-|
| Adding a function parameter | Incompatible | Incompatible | Even if the added function parameter is at the end and has a default value, it is still API incompatible, because Cangjie supports functions as first-class citizens, and the function type changes |
| Deleting a function parameter | Incompatible | Incompatible | NA |
| Modifying the name of a non-named parameter | Compatible | Compatible | NA |
| Modifying the name of a named parameter | Incompatible | Incompatible | NA |
| Changing a non-named parameter to a named parameter | Incompatible | Incompatible | The call site of a named parameter needs to use the parameter name, so it will cause a compilation error |
| Changing a named parameter to a non-named parameter | Incompatible | Incompatible | The call site of a named parameter needs to use the parameter name, so it will cause a compilation error |
| Modifying the order of non-named parameters | Incompatible | Incompatible | NA |
| Modifying the order of named parameters | Compatible | Incompatible | NA |
| Modifying the type of a parameter from A to B | When B is a parent type of A, compatible; otherwise, incompatible | Incompatible | NA |
| Adding a default value to a parameter | Compatible | Compatible | NA |
| Deleting an existing default value of a parameter | Incompatible | Incompatible | NA |
| Modifying the default value of a parameter of a const function or a function marked with @Frozen | Compatible | Incompatible | NA |
| Modifying the default value of a parameter of a non-const function not marked with @Frozen | Compatible | Compatible | NA |
| Modifying the return type from A to B | When B is a subtype of A, compatible; otherwise, incompatible | When B is a subtype of A, and both A and B are class or interface types, compatible; otherwise, incompatible | NA |
| Modifying the function body of a const function or a function marked with @Frozen | Compatible | Incompatible | NA |
| Modifying the function body of a non-const function not marked with @Frozen | Compatible | Compatible | NA |

#### Modifying the Definition Location of a Global Function

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Modifying the file where a global function is defined (note: including renaming the file and moving to another file) | Compatible | If the global function has the private modifier, incompatible; otherwise, compatible |
| Modifying the position of a global function definition within the file | Compatible | Compatible |

## Compatibility Rules for struct Types

### Addition and Deletion Scenarios

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding a struct type | Compatible | Compatible |
| Deleting a struct type without the public modifier or whose package does not have the public modifier | Compatible | Compatible |
| Deleting a struct type with the public modifier and whose package has the public modifier | Incompatible | Incompatible |

### Modification Scenarios

The definition of a struct type being visible outside the module differs from the global variables and functions described above. We call a struct type **visible outside the module** when it satisfies any of the following conditions:

1. The struct type has the public modifier and the package where it resides has the public modifier.
2. The struct type is used in the type definition of member variables in other module-externally-visible struct/class types.
3. The struct type is depended upon by other module-externally-visible declarations, and these declarations are marked with const or @Frozen.

<!-- code_no_check -->

```cangjie
package A

// struct type S1 is modified with public, visible outside the module, and will participate in the compilation process of downstream modules
public struct S1 {
    let x: Int64 = 0
}

// Although struct type S2 has no public modifier and is not visible outside the module, it also participates in the compilation process of downstream modules. The reason is that S2 is used to define the x member variable type in S3, creating a dependency
struct S2 {
    let x: Int64 = 0
}
public struct S3 {
    let x: S2 = S2()
}

//==================================//

package B

import A.*

let m: S1 = S1()
let n: S3 = S3()
```

Modifications to any struct type not visible outside the module satisfy API + ABI compatibility; below we only discuss the module-externally-visible scenario.

#### Modifying the Definition Content of a struct Type

##### Modifying Annotations

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding, deleting, or modifying annotations | Compatible | Compatible |

##### Modifying the Compile Marker Part

| Change Behavior | API Compatibility | ABI Compatibility | Note |
|-|-|-|-|
| Adding the @Deprecated compile marker | Based on the @Deprecated parameter conditions, when the struct type is unavailable in downstream modules, incompatible; otherwise, compatible | Based on the @Deprecated parameter conditions, when the struct type is unavailable in downstream modules, incompatible; otherwise, compatible | NA |
| Deleting the @Deprecated compile marker | Compatible | Compatible | NA |
| Modifying the @Deprecated compile marker parameter conditions, such that the struct type changes from available to unavailable in downstream modules | Incompatible | Incompatible | NA |
| Modifying the @Deprecated compile marker parameter conditions, such that the struct type changes from unavailable to available in downstream modules | Compatible | Compatible | NA |
| Modifying the @Deprecated compile marker parameter conditions, such that the struct type's availability in downstream modules remains unchanged | Compatible | Compatible | NA |
| Adding the @C compile marker | Compatible | Incompatible | Adding the @C compile marker affects the struct type's mangle name |
| Deleting the @C compile marker | Compatible | Incompatible | NA |

##### Modifying the Modifier Part

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Access modifier changed from public to non-public | Incompatible | Incompatible |
| Access modifier changed from non-public to public | Compatible | Compatible |
| Access modifier changed among the other three non-public cases (protected/internal/private) | Compatible | Compatible |

##### Modifying Generic Parameters and Generic Parameter Constraints

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding a generic parameter | Incompatible | Incompatible |
| Deleting a generic parameter | Incompatible | Incompatible |
| Modifying the name of a generic parameter | Compatible | Compatible |
| Modifying the order of generic parameters | Incompatible | Incompatible |
| Adding a constraint upper bound to a generic parameter | Incompatible | Incompatible |
| Deleting a certain upper bound constraint of a generic parameter | Compatible | Incompatible |
| Modifying a certain upper bound constraint A of a generic parameter to B | When B is a parent type of A, compatible; otherwise, incompatible | Incompatible |
| Modifying the order among generic parameter constraints or among constraint upper bounds | Compatible | Compatible |

##### Modifying Implemented Interfaces

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding an implemented interface | Compatible | Compatible |
| Deleting an implemented interface | Incompatible | Incompatible |
| Modifying an implemented interface from A to B | Incompatible | Incompatible |
| Modifying the order of implemented interfaces | Compatible | Compatible |

##### Modifying Static Member Variables

**Adding and Deleting Static Member Variables**

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding a static member variable | Compatible | Compatible |
| Deleting a static member variable with the public modifier | Incompatible | Incompatible |
| Deleting a static member variable without the public modifier | Compatible | Compatible |

**Modifying the Definition of a Static Member Variable**

For a struct type visible outside the module, we call its static member variable **visible outside the module** when it satisfies any of the following conditions:

1. The static member variable has the public modifier.
2. The static member variable is depended upon by other module-externally-visible declarations, and these declarations are marked with const or @Frozen.

Modifications to any static member variable not visible outside the module satisfy API + ABI compatibility; below we only discuss the module-externally-visible scenario.

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding, deleting, or modifying annotations | Compatible | Compatible |
| Adding the @Deprecated compile marker | Based on the @Deprecated parameter conditions, when the static member variable is unavailable in downstream modules, incompatible; otherwise, compatible | Based on the @Deprecated parameter conditions, when the static member variable is unavailable in downstream modules, incompatible; otherwise, compatible |
| Deleting the @Deprecated compile marker | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the static member variable changes from available to unavailable in downstream modules | Incompatible | Incompatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the static member variable changes from unavailable to available in downstream modules | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the static member variable's availability in downstream modules remains unchanged | Compatible | Compatible |
| Access modifier changed from public to non-public | Incompatible | Incompatible |
| Access modifier changed from non-public to public | Compatible | Compatible |
| Access modifier changed among the other three non-public cases (protected/internal/private) | Compatible | Compatible |
| Changing the let modifier to var modifier | Compatible | Compatible |
| Changing the var modifier to let modifier | Incompatible | Incompatible |
| Changing the let modifier to const modifier | Compatible | Incompatible |
| Changing the var modifier to const modifier | Incompatible | Incompatible |
| Changing the const modifier to let modifier | Incompatible | Incompatible |
| Changing the const modifier to var modifier | Incompatible | Incompatible |
| Deleting the static modifier | Incompatible | Incompatible |
| Modifying the variable name | Incompatible | Incompatible |
| Modifying the variable type from A to B | When B is a subtype of A, compatible; otherwise, incompatible | When B is a subtype of A, and both A and B are class or interface types, compatible; otherwise, incompatible |
| Modifying the initial value of a non-const variable | Compatible | Compatible |
| Modifying the initial value of a const variable | Compatible | Incompatible |

**Modifying the Order of Static Member Variables**

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Modifying the order of static member variables | Compatible | Compatible |

##### Modifying Instance Member Variables

**Adding and Deleting Instance Member Variables**

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding an instance member variable | Compatible | Incompatible |
| Deleting an instance member variable with the public modifier | Incompatible | Incompatible |
| Deleting an instance member variable without the public modifier | Compatible | Incompatible |

**Modifying the Definition of a public Instance Member Variable**

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding, deleting, or modifying annotations | Compatible | Compatible |
| Adding the @Deprecated compile marker | Based on the @Deprecated parameter conditions, when the instance member variable is unavailable in downstream modules, incompatible; otherwise, compatible | Based on the @Deprecated parameter conditions, when the instance member variable is unavailable in downstream modules, incompatible; otherwise, compatible |
| Deleting the @Deprecated compile marker | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the instance member variable changes from available to unavailable in downstream modules | Incompatible | Incompatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the instance member variable changes from unavailable to available in downstream modules | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the instance member variable's availability in downstream modules remains unchanged | Compatible | Compatible |
| Access modifier changed from public to non-public | Incompatible | Incompatible |
| Changing the let modifier to var modifier | Compatible | Compatible |
| Changing the var modifier to let modifier | Incompatible | Incompatible |
| Modifying the variable name | Incompatible | Incompatible |
| Modifying the instance member variable type from A to B | Incompatible | Incompatible |
| Modifying the variable initial value | Compatible | If the struct type contains an init function annotated with const or @Frozen, incompatible; otherwise, compatible |

**Modifying the Definition of a non-public Instance Member Variable**

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding, deleting, or modifying annotations | Compatible | Compatible |
| Adding the @Deprecated compile marker | Compatible | Compatible |
| Deleting the @Deprecated compile marker | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions | Compatible | Compatible |
| Access modifier changed from non-public to public | Compatible | Compatible |
| Access modifier changed among the other three non-public cases (protected/internal/private) | Compatible | Compatible |
| Changing the let modifier to var modifier | Compatible | Compatible |
| Changing the var modifier to let modifier | Compatible | Compatible |
| Modifying the variable name | Compatible | Compatible |
| Modifying the instance member variable type from A to B | Compatible | Incompatible |
| Modifying the variable initial value | Compatible | If the struct type contains an init function annotated with const or @Frozen, incompatible; otherwise, compatible |

**Modifying the Order of Instance Member Variables**

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Modifying the order of instance member variables | Compatible | Incompatible |

##### Modifying Static Member Functions

**Adding and Deleting Static Member Functions**

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding a static member function | Compatible | Adding a static member function with overriding semantics (with the redef modifier or with the redef modifier omitted), incompatible; otherwise, compatible |
| Deleting a static member function with the public modifier | Incompatible | Incompatible |
| Deleting a static member function without the public modifier | Compatible | Compatible |

**Modifying the Definition of a Static Member Function**

For a struct type visible outside the module, we call its static member function **visible outside the module** when it satisfies any of the following conditions:

1. The static member function has the public modifier.
2. The static member function is depended upon by other module-externally-visible declarations, and these declarations are marked with const or @Frozen.

Modifications to any static member function not visible outside the module satisfy API + ABI compatibility; below we only discuss the module-externally-visible scenario.

| Change Behavior | API Compatibility | ABI Compatibility | Note |
|-|-|-|-|
| Adding, deleting, or modifying annotations | Compatible | Compatible | NA |
| Adding the @Deprecated compile marker | Based on the @Deprecated parameter conditions, when the static member function is unavailable in downstream modules, incompatible; otherwise, compatible | Based on the @Deprecated parameter conditions, when the static member function is unavailable in downstream modules, incompatible; otherwise, compatible | NA |
| Deleting the @Deprecated compile marker | Compatible | Compatible | NA |
| Modifying the @Deprecated compile marker parameter conditions, such that the static member function changes from available to unavailable in downstream modules | Incompatible | Incompatible | NA |
| Modifying the @Deprecated compile marker parameter conditions, such that the static member function changes from unavailable to available in downstream modules | Compatible | Compatible | NA |
| Modifying the @Deprecated compile marker parameter conditions, such that the static member function's availability in downstream modules remains unchanged | Compatible | Compatible | NA |
| Adding the @Frozen compile marker | Compatible | Compatible | NA |
| Deleting the @Frozen compile marker | Compatible | Incompatible | NA |
| Access modifier changed from public to non-public | Incompatible | Incompatible | NA |
| Access modifier changed from non-public to public | Compatible | Compatible | NA |
| Access modifier changed among the other three non-public cases (protected/internal/private) | Compatible | Compatible | NA |
| Adding the unsafe modifier | Incompatible | Incompatible | NA |
| Deleting the unsafe modifier | Compatible | Compatible | NA |
| Adding the const modifier | Compatible | Compatible | NA |
| Deleting the const modifier | Incompatible | Incompatible | NA |
| Deleting the static modifier | Incompatible | Incompatible | NA |
| Modifying the function name | Incompatible | Incompatible | NA |
| Adding a generic parameter | Incompatible | Incompatible | NA |
| Deleting a generic parameter | Incompatible | Incompatible | NA |
| Modifying the name of a generic parameter | Compatible | Compatible | NA |
| Modifying the order of generic parameters | Incompatible | Incompatible | NA |
| Adding a constraint upper bound to a generic parameter | Incompatible | Incompatible | NA |
| Deleting a certain upper bound constraint of a generic parameter | Compatible | Incompatible | NA |
| Modifying a certain upper bound constraint A of a generic parameter to B | When B is a parent type of A, compatible; otherwise, incompatible | Incompatible | NA |
| Modifying the order among generic parameter constraints or among constraint upper bounds | Compatible | Compatible | NA |
| Adding a function parameter | Incompatible | Incompatible | Even if the added function parameter is at the end and has a default value, it is still API incompatible, because Cangjie supports functions as first-class citizens, and the function type changes |
| Deleting a function parameter | Incompatible | Incompatible | NA |
| Modifying the name of a non-named parameter | Compatible | Compatible | NA |
| Modifying the name of a named parameter | Incompatible | Incompatible | NA |
| Changing a non-named parameter to a named parameter | Incompatible | Incompatible | The call site of a named parameter needs to use the parameter name, so it will cause a compilation error |
| Changing a named parameter to a non-named parameter | Incompatible | Incompatible | The call site of a named parameter needs to use the parameter name, so it will cause a compilation error |
| Modifying the order of non-named parameters | Incompatible | Incompatible | NA |
| Modifying the order of named parameters | Compatible | Incompatible | NA |
| Modifying the type of a parameter from A to B | When B is a parent type of A, compatible; otherwise, incompatible | Incompatible | NA |
| Adding a default value to a parameter | Compatible | Compatible | NA |
| Deleting an existing default value of a parameter | Incompatible | Incompatible | NA |
| Modifying the default value of a parameter of a const function or a function marked with @Frozen | Compatible | Incompatible | NA |
| Modifying the default value of a parameter of a non-const function not marked with @Frozen | Compatible | Compatible | NA |
| Modifying the return type from A to B | When B is a subtype of A, compatible; otherwise, incompatible | When B is a subtype of A, and both A and B are class or interface types, compatible; otherwise, incompatible | NA |
| Modifying the function body of a const function or a function marked with @Frozen | Compatible | Incompatible | NA |
| Modifying the function body of a non-const function not marked with @Frozen | Compatible | Compatible | NA |

**Modifying the Order of Static Member Functions**

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Modifying the order of static member functions | Compatible | Compatible |

##### Modifying Static Properties

**Adding and Deleting Static Properties**

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding a static property | Compatible | Adding a static property with overriding semantics (with the redef modifier or with the redef modifier omitted), incompatible; otherwise, compatible |
| Deleting a public static property | Incompatible | Incompatible |
| Deleting a non-public static property | Compatible | Compatible |

**Modifying the Definition of a Static Property**

For a struct type visible outside the module, we call its static property **visible outside the module** when it satisfies any of the following conditions:

1. The static property has the public modifier.
2. The static property is depended upon by other module-externally-visible declarations, and these declarations are marked with const or @Frozen.

Modifications to any static property not visible outside the module satisfy API + ABI compatibility; below we only discuss the module-externally-visible scenario.

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding, deleting, or modifying annotations | Compatible | Compatible |
| Adding the @Deprecated compile marker | Based on the @Deprecated parameter conditions, when the static property is unavailable in downstream modules, incompatible; otherwise, compatible | Based on the @Deprecated parameter conditions, when the static property is unavailable in downstream modules, incompatible; otherwise, compatible |
| Deleting the @Deprecated compile marker | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the static property changes from available to unavailable in downstream modules | Incompatible | Incompatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the static property changes from unavailable to available in downstream modules | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the static property's availability in downstream modules remains unchanged | Compatible | Compatible |
| Adding the @Frozen compile marker | Compatible | Compatible |
| Deleting the @Frozen compile marker | Compatible | Incompatible |
| Access modifier changed from public to non-public | Incompatible | Incompatible |
| Access modifier changed from non-public to public | Compatible | Compatible |
| Access modifier changed among the other three non-public cases (protected/internal/private) | Compatible | Compatible |
| Deleting the static modifier | Incompatible | Incompatible |
| Adding the mut modifier and the corresponding setter function | Compatible | Compatible |
| Deleting the mut modifier and the corresponding setter function | Incompatible | Incompatible |
| Modifying the property name | Incompatible | Incompatible |
| Modifying the property type | Incompatible | Incompatible |
| Modifying the getter/setter functions of a property not marked with @Frozen | Compatible | Compatible |
| Modifying the getter/setter functions of a property marked with @Frozen | Compatible | Incompatible |

**Modifying the Order of Static Properties**

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Modifying the order of static properties | Compatible | Compatible |

##### Modifying Instance Member Functions

**Adding and Deleting Instance Member Functions**

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding an instance member function | Compatible | Adding an instance member function with overriding semantics (with the override modifier or with the override modifier omitted), incompatible; otherwise, compatible |
| Deleting a public instance member function | Incompatible | Incompatible |
| Deleting a non-public instance member function | Compatible | Compatible |

**Modifying the Definition of an Instance Member Function**

For a struct type visible outside the module, we call its instance member function **visible outside the module** when it satisfies any of the following conditions:

1. The instance member function has the public modifier.
2. The instance member function is depended upon by other module-externally-visible declarations, and these declarations are marked with const or @Frozen.

Modifications to any instance member function not visible outside the module satisfy API + ABI compatibility; below we only discuss the module-externally-visible scenario.

| Change Behavior | API Compatibility | ABI Compatibility | Note |
|-|-|-|-|
| Adding, deleting, or modifying annotations | Compatible | Compatible | NA |
| Adding the @Deprecated compile marker | Based on the @Deprecated parameter conditions, when the instance member function is unavailable in downstream modules, incompatible; otherwise, compatible | Based on the @Deprecated parameter conditions, when the instance member function is unavailable in downstream modules, incompatible; otherwise, compatible | NA |
| Deleting the @Deprecated compile marker | Compatible | Compatible | NA |
| Modifying the @Deprecated compile marker parameter conditions, such that the instance member function changes from available to unavailable in downstream modules | Incompatible | Incompatible | NA |
| Modifying the @Deprecated compile marker parameter conditions, such that the instance member function changes from unavailable to available in downstream modules | Compatible | Compatible | NA |
| Modifying the @Deprecated compile marker parameter conditions, such that the instance member function's availability in downstream modules remains unchanged | Compatible | Compatible | NA |
| Adding the @Frozen compile marker | Compatible | Compatible | NA |
| Deleting the @Frozen compile marker | Compatible | Incompatible | NA |
| Access modifier changed from public to non-public | Incompatible | Incompatible | NA |
| Access modifier changed from non-public to public | Compatible | Compatible | NA |
| Access modifier changed among the other three non-public cases (protected/internal/private) | Compatible | Compatible | NA |
| Adding the static modifier | Incompatible | Incompatible | NA |
| Adding the unsafe modifier | Incompatible | Incompatible | NA |
| Deleting the unsafe modifier | Compatible | Compatible | NA |
| Adding the const modifier | Compatible | Compatible | NA |
| Deleting the const modifier | Incompatible | Incompatible | NA |
| Adding the mut modifier | Incompatible | Incompatible | NA |
| Deleting the mut modifier | Compatible | Incompatible | NA |
| Modifying the function name | Incompatible | Incompatible | NA |
| Adding a generic parameter | Incompatible | Incompatible | NA |
| Deleting a generic parameter | Incompatible | Incompatible | NA |
| Modifying the name of a generic parameter | Compatible | Compatible | NA |
| Modifying the order of generic parameters | Incompatible | Incompatible | NA |
| Adding a constraint upper bound to a generic parameter | Incompatible | Incompatible | NA |
| Deleting a certain upper bound constraint of a generic parameter | Compatible | Incompatible | NA |
| Modifying a certain upper bound constraint A of a generic parameter to B | When B is a parent type of A, compatible; otherwise, incompatible | Incompatible | NA |
| Modifying the order among generic parameter constraints or among constraint upper bounds | Compatible | Compatible | NA |
| Adding a function parameter | Incompatible | Incompatible | Even if the added function parameter is at the end and has a default value, it is still API incompatible, because Cangjie supports functions as first-class citizens, and the function type changes |
| Deleting a function parameter | Incompatible | Incompatible | NA |
| Modifying the name of a non-named parameter | Compatible | Compatible | NA |
| Modifying the name of a named parameter | Incompatible | Incompatible | NA |
| Changing a non-named parameter to a named parameter | Incompatible | Incompatible | The call site of a named parameter needs to use the parameter name, so it will cause a compilation error |
| Changing a named parameter to a non-named parameter | Incompatible | Incompatible | The call site of a named parameter needs to use the parameter name, so it will cause a compilation error |
| Modifying the order of non-named parameters | Incompatible | Incompatible | NA |
| Modifying the order of named parameters | Compatible | Incompatible | NA |
| Modifying the type of a parameter from A to B | When B is a parent type of A, compatible; otherwise, incompatible | Incompatible | NA |
| Adding a default value to a parameter | Compatible | Compatible | NA |
| Deleting an existing default value of a parameter | Incompatible | Incompatible | NA |
| Modifying the default value of a parameter of a const function or a function marked with @Frozen | Compatible | Incompatible | NA |
| Modifying the default value of a parameter of a non-const function not marked with @Frozen | Compatible | Compatible | NA |
| Modifying the return type from A to B | When B is a subtype of A, compatible; otherwise, incompatible | When B is a subtype of A, and both A and B are class or interface types, compatible; otherwise, incompatible | NA |
| Modifying the function body of a const function or a function marked with @Frozen | Compatible | Incompatible | NA |
| Modifying the function body of a non-const function not marked with @Frozen | Compatible | Compatible | NA |

**Modifying the Order of Instance Member Functions**

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Modifying the order of instance member functions | Compatible | Compatible |

##### Modifying Instance Properties

**Adding and Deleting Instance Properties**

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding an instance property | Compatible | Adding an instance property with overriding semantics (with the override modifier or with the override modifier omitted), incompatible; otherwise, compatible |
| Deleting a public instance property | Incompatible | Incompatible |
| Deleting a non-public instance property | Compatible | Compatible |

**Modifying the Definition of an Instance Property**

For a struct type visible outside the module, we call its instance property **visible outside the module** when it satisfies any of the following conditions:

1. The instance property has the public modifier.
2. The instance property is depended upon by other module-externally-visible declarations, and these declarations are marked with const or @Frozen.

Modifications to any instance property not visible outside the module satisfy API + ABI compatibility; below we only discuss the module-externally-visible scenario.

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding, deleting, or modifying annotations | Compatible | Compatible |
| Adding the @Deprecated compile marker | Based on the @Deprecated parameter conditions, when the instance property is unavailable in downstream modules, incompatible; otherwise, compatible | Based on the @Deprecated parameter conditions, when the instance property is unavailable in downstream modules, incompatible; otherwise, compatible |
| Deleting the @Deprecated compile marker | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the instance property changes from available to unavailable in downstream modules | Incompatible | Incompatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the instance property changes from unavailable to available in downstream modules | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the instance property's availability in downstream modules remains unchanged | Compatible | Compatible |
| Adding the @Frozen compile marker | Compatible | Compatible |
| Deleting the @Frozen compile marker | Compatible | Incompatible |
| Access modifier changed from public to non-public | Incompatible | Incompatible |
| Access modifier changed from non-public to public | Compatible | Compatible |
| Access modifier changed among the other three non-public cases (protected/internal/private) | Compatible | Compatible |
| Adding the static modifier | Incompatible | Incompatible |
| Adding the mut modifier and the corresponding setter function | Incompatible | Incompatible |
| Deleting the mut modifier and the corresponding setter function | Incompatible | Incompatible |
| Modifying the property name | Incompatible | Incompatible |
| Modifying the property type | Incompatible | Incompatible |
| Modifying the getter/setter functions of a property not marked with @Frozen | Compatible | Compatible |
| Modifying the getter/setter functions of a property marked with @Frozen | Compatible | Incompatible |

**Modifying the Order of Instance Properties**

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Modifying the order of instance properties | Compatible | Compatible |

##### Modifying init Constructors

###### Adding and Deleting init Constructors

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding an init constructor | Compatible | Compatible |
| Deleting an init constructor without the public modifier | Compatible | Compatible |
| Deleting an init constructor with the public modifier | Incompatible | Incompatible |

###### Modifying the Definition of an init Constructor

For a struct type visible outside the module, we call its constructor **visible outside the module** when it satisfies any of the following conditions:

1. The init constructor has the public modifier.
2. The init constructor is depended upon by other module-externally-visible declarations, and these declarations are marked with const or @Frozen.

Modifications to any init constructor not visible outside the module satisfy API + ABI compatibility; below we only discuss the module-externally-visible scenario.

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding, deleting, or modifying annotations | Compatible | Compatible |
| Adding the @Deprecated compile marker | Based on the @Deprecated parameter conditions, when the init constructor is unavailable in downstream modules, incompatible; otherwise, compatible | Based on the @Deprecated parameter conditions, when the init constructor is unavailable in downstream modules, incompatible; otherwise, compatible |
| Deleting the @Deprecated compile marker | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the init constructor changes from available to unavailable in downstream modules | Incompatible | Incompatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the init constructor changes from unavailable to available in downstream modules | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the init constructor's availability in downstream modules remains unchanged | Compatible | Compatible |
| Adding the @Frozen compile marker | Compatible | Compatible |
| Deleting the @Frozen compile marker | Compatible | Incompatible |
| Access modifier changed from public to non-public | Incompatible | Incompatible |
| Access modifier changed from non-public to public | Compatible | Compatible |
| Access modifier changed among the other three non-public cases (protected/internal/private) | Compatible | Compatible |
| Adding the const modifier | Compatible | Compatible |
| Deleting the const modifier | Incompatible | Incompatible |
| Adding a function parameter | If the added function parameter is at the end of all function parameters and has a default value, compatible; otherwise, incompatible | Incompatible |
| Deleting a function parameter | Incompatible | Incompatible |
| Modifying the name of a non-named parameter | Compatible | Compatible |
| Modifying the name of a named parameter | Incompatible | Incompatible |
| Changing a non-named parameter to a named parameter | Incompatible | Incompatible |
| Changing a named parameter to a non-named parameter | Incompatible | Incompatible |
| Modifying the order of non-named parameters | Incompatible | Incompatible |
| Modifying the order of named parameters | Compatible | Incompatible |
| Modifying the type of a parameter from A to B | When B is a parent type of A, compatible; otherwise, incompatible | Incompatible |
| Adding a default value to a parameter | Compatible | Compatible |
| Deleting an existing default value of a parameter | Incompatible | Incompatible |
| Modifying the default value of a parameter of a const function or a function marked with @Frozen | Compatible | Incompatible |
| Modifying the default value of a parameter of a non-const function not marked with @Frozen | Compatible | Compatible |
| Modifying the function body of a const function or a function marked with @Frozen | Compatible | Incompatible |
| Modifying the function body of a non-const function not marked with @Frozen | Compatible | Compatible |

###### Modifying the Order of init Constructors

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Modifying the order of init constructors | Compatible | Compatible |

##### Modifying Primary Constructors

###### Adding and Deleting Primary Constructors

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding a primary constructor | Compatible | When the primary constructor has member variable parameters, incompatible; otherwise, compatible |
| Deleting a constructor without the public modifier | When the primary constructor has member variable parameters modified with public/protected/internal, incompatible; otherwise, compatible | When the primary constructor has member variable parameters, incompatible; otherwise, compatible |
| Deleting a constructor with the public modifier | Incompatible | Incompatible |

###### Modifying the Definition of a Primary Constructor

For a struct type visible outside the module, we call its primary constructor **visible outside the module** when it satisfies any of the following conditions:

1. The primary constructor has the public modifier.
2. The primary constructor is depended upon by other module-externally-visible declarations, and these declarations are marked with const or @Frozen.

**Modifying the Definition of a Module-Externally-Visible Primary Constructor**

| Change Behavior | API Compatibility | ABI Compatibility | Note |
|-|-|-|-|
| Adding, deleting, or modifying annotations | Compatible | Compatible | NA |
| Adding the @Deprecated compile marker | Based on the @Deprecated parameter conditions, when the primary constructor is unavailable in downstream modules, incompatible; otherwise, compatible | Based on the @Deprecated parameter conditions, when the primary constructor is unavailable in downstream modules, incompatible; otherwise, compatible | NA |
| Deleting the @Deprecated compile marker | Compatible | Compatible | NA |
| Modifying the @Deprecated compile marker parameter conditions, such that the primary constructor changes from available to unavailable in downstream modules | Incompatible | Incompatible | NA |
| Modifying the @Deprecated compile marker parameter conditions, such that the primary constructor changes from unavailable to available in downstream modules | Compatible | Compatible | NA |
| Modifying the @Deprecated compile marker parameter conditions, such that the primary constructor's availability in downstream modules remains unchanged | Compatible | Compatible | NA |
| Adding the @Frozen compile marker | Compatible | Compatible | NA |
| Deleting the @Frozen compile marker | Compatible | Incompatible | NA |
| Access modifier changed from public to non-public | Incompatible | Incompatible | NA |
| Access modifier changed from non-public to public | Compatible | Compatible | NA |
| Access modifier changed among the other three non-public cases (protected/internal/private) | Compatible | Compatible | NA |
| Adding the const modifier | Compatible | Compatible | NA |
| Deleting the const modifier | Incompatible | Incompatible | NA |
| Adding an ordinary function parameter | If the added ordinary function parameter is at the end of all function parameters and has a default value, compatible; otherwise, incompatible | Incompatible | NA |
| Deleting an ordinary function parameter | Incompatible | Incompatible | NA |
| Modifying the name of a non-named ordinary function parameter | Compatible | Compatible | NA |
| Modifying the name of a named ordinary function parameter | Incompatible | Incompatible | NA |
| Changing a non-named ordinary function parameter to a named ordinary function parameter | Incompatible | Incompatible | The call site of a named parameter needs to use the parameter name, so it will cause a compilation error |
| Changing a named ordinary function parameter to a non-named ordinary function parameter | Incompatible | Incompatible | The call site of a named parameter needs to use the parameter name, so it will cause a compilation error |
| Modifying the order of non-named ordinary function parameters | Incompatible | Incompatible | NA |
| Modifying the order of named ordinary function parameters | Compatible | Incompatible | NA |
| Modifying the type of an ordinary function parameter from A to B | When B is a parent type of A, compatible; otherwise, incompatible | Incompatible | NA |
| Adding a default value to an ordinary function parameter | Compatible | Compatible | NA |
| Deleting an existing default value of an ordinary function parameter | Incompatible | Incompatible | NA |
| Modifying the default value of an ordinary function parameter of a const function or a function marked with @Frozen | Compatible | Incompatible | NA |
| Modifying the default value of an ordinary function parameter of a non-const function not marked with @Frozen | Compatible | Compatible | NA |
| Adding a member variable parameter | If the added member variable parameter is at the end of all function parameters and has a default value, compatible; otherwise, incompatible | Incompatible | NA |
| Deleting a member variable parameter | Incompatible | Incompatible | NA |
| Modifying the name of a non-named member variable parameter | If the non-named member variable parameter has the public modifier, incompatible; otherwise, compatible | If the non-named member variable parameter has the public modifier, incompatible; otherwise, compatible | NA |
| Modifying the name of a named member variable parameter | Incompatible | Incompatible | NA |
| Changing a non-named member variable parameter to a named member variable parameter | Incompatible | Incompatible | NA |
| Changing a named member variable parameter to a non-named member variable parameter | Incompatible | Incompatible | NA |
| Modifying the order of non-named member variable parameters | Incompatible | Incompatible | NA |
| Modifying the order of named member variable parameters | Compatible | Incompatible | NA |
| Modifying the type of a member variable parameter from A to B | Incompatible | Incompatible | NA |
| Adding a default value to a member variable parameter | Compatible | If the struct type contains an init function annotated with const or @Frozen, incompatible; otherwise, compatible | NA |
| Deleting an existing default value of a member variable parameter | Incompatible | Incompatible | NA |
| Modifying the default value of a member variable parameter of a const function or a function marked with @Frozen | Compatible | Incompatible | NA |
| Modifying the default value of a member variable parameter of a non-const function not marked with @Frozen | Compatible | Compatible | NA |
| Modifying the function body of a const function or a function marked with @Frozen | Compatible | Incompatible | NA |
| Modifying the function body of a non-const function not marked with @Frozen | Compatible | Compatible | NA |

##### Modifying static init Functions

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding a static init function | Compatible | Compatible |
| Deleting a static init function | Compatible | Compatible |
| Modifying the definition of a static init function | Compatible | Compatible |

#### Modifying the Definition Location of a struct Type

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Modifying the file where a struct type is defined (note: including renaming the file and moving to another file) | Compatible | If the struct type has the private modifier, incompatible; otherwise, compatible |
| Modifying the position of a struct type definition within the file | Compatible | Compatible |

## Compatibility Rules for enum Types

### Addition and Deletion Scenarios

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding an enum type | Compatible | Compatible |
| Deleting an enum type without the public modifier or whose package does not have the public modifier | Compatible | Compatible |
| Deleting an enum type with the public modifier and whose package has the public modifier | Incompatible | Incompatible |

### Modification Scenarios

Similarly, we call an enum type **visible outside the module** when it satisfies any of the following conditions:

1. The enum type has the public modifier and the package where it resides has the public modifier.
2. The enum type is used in the type definition of member variables in other module-externally-visible struct/class types.
3. The enum type is depended upon by other module-externally-visible declarations, and these declarations are marked with const or @Frozen.

Modifications to any enum type not visible outside the module satisfy API + ABI compatibility; below we only discuss the module-externally-visible scenario.

#### Modifying the Definition Content of an enum Type

##### Modifying Annotations

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding, deleting, or modifying annotations | Compatible | Compatible |

##### Modifying the Compile Marker Part

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding the @Deprecated compile marker | Based on the @Deprecated parameter conditions, when the enum type is unavailable in downstream modules, incompatible; otherwise, compatible | Based on the @Deprecated parameter conditions, when the enum type is unavailable in downstream modules, incompatible; otherwise, compatible |
| Deleting the @Deprecated compile marker | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the enum type changes from available to unavailable in downstream modules | Incompatible | Incompatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the enum type changes from unavailable to available in downstream modules | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the enum type's availability in downstream modules remains unchanged | Compatible | Compatible |

##### Modifying the Modifier Part

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Access modifier changed from public to non-public | Incompatible | Incompatible |
| Access modifier changed from non-public to public | Compatible | Compatible |
| Access modifier changed among the other three non-public cases (protected/internal/private) | Compatible | Compatible |

##### Modifying Generic Parameters and Generic Parameter Constraints

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding a generic parameter | Incompatible | Incompatible |
| Deleting a generic parameter | Incompatible | Incompatible |
| Modifying the name of a generic parameter | Compatible | Compatible |
| Modifying the order of generic parameters | Incompatible | Incompatible |
| Adding a constraint upper bound to a generic parameter | Incompatible | Incompatible |
| Deleting a certain upper bound constraint of a generic parameter | Compatible | Incompatible |
| Modifying a certain upper bound constraint A of a generic parameter to B | When B is a parent type of A, compatible; otherwise, incompatible | Incompatible |
| Modifying the order among generic parameter constraints or among constraint upper bounds | Compatible | Compatible |

##### Modifying Implemented Interfaces

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding an implemented interface | Compatible | Compatible |
| Deleting an implemented interface | Incompatible | Incompatible |
| Modifying an implemented interface from A to B | Incompatible | Incompatible |
| Modifying the order of implemented interfaces | Compatible | Compatible |

##### Modifying Static Member Functions

**Adding and Deleting Static Member Functions**

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding a static member function | Compatible | Adding a static member function with overriding semantics (with the redef modifier or with the redef modifier omitted), incompatible; otherwise, compatible |
| Deleting a static member function with the public modifier | Incompatible | Incompatible |
| Deleting a static member function without the public modifier | Compatible | Compatible |

**Modifying the Definition of a Static Member Function**

For an enum type visible outside the module, we call its static member function **visible outside the module** when it satisfies any of the following conditions:

1. The static member function has the public modifier.
2. The static member function is depended upon by other module-externally-visible declarations, and these declarations are marked with const or @Frozen.

Modifications to any static member function not visible outside the module satisfy API + ABI compatibility; below we only discuss the module-externally-visible scenario.

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding, deleting, or modifying annotations | Compatible | Compatible |
| Adding the @Deprecated compile marker | Based on the @Deprecated parameter conditions, when the static member function is unavailable in downstream modules, incompatible; otherwise, compatible | Based on the @Deprecated parameter conditions, when the static member function is unavailable in downstream modules, incompatible; otherwise, compatible |
| Deleting the @Deprecated compile marker | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the static member function changes from available to unavailable in downstream modules | Incompatible | Incompatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the static member function changes from unavailable to available in downstream modules | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the static member function's availability in downstream modules remains unchanged | Compatible | Compatible |
| Adding the @Frozen compile marker | Compatible | Compatible |
| Deleting the @Frozen compile marker | Compatible | Incompatible |
| Access modifier changed from public to non-public | Incompatible | Incompatible |
| Access modifier changed from non-public to public | Compatible | Compatible |
| Access modifier changed among the other three non-public cases (protected/internal/private) | Compatible | Compatible |
| Adding the unsafe modifier | Incompatible | Incompatible |
| Deleting the unsafe modifier | Compatible | Compatible |
| Adding the const modifier | Compatible | Compatible |
| Deleting the const modifier | Incompatible | Incompatible |
| Deleting the static modifier | Incompatible | Incompatible |
| Modifying the function name | Incompatible | Incompatible |
| Adding a generic parameter | Incompatible | Incompatible |
| Deleting a generic parameter | Incompatible | Incompatible |
| Modifying the name of a generic parameter | Compatible | Compatible |
| Modifying the order of generic parameters | Incompatible | Incompatible |
| Adding a constraint upper bound to a generic parameter | Incompatible | Incompatible |
| Deleting a certain upper bound constraint of a generic parameter | Compatible | Incompatible |
| Modifying a certain upper bound constraint A of a generic parameter to B | When B is a parent type of A, compatible; otherwise, incompatible | Incompatible |
| Modifying the order among generic parameter constraints or among constraint upper bounds | Compatible | Compatible |
| Adding a function parameter | Incompatible | Incompatible |
| Deleting a function parameter | Incompatible | Incompatible |
| Modifying the name of a non-named parameter | Compatible | Compatible |
| Modifying the name of a named parameter | Incompatible | Incompatible |
| Changing a non-named parameter to a named parameter | Incompatible | Incompatible |
| Changing a named parameter to a non-named parameter | Incompatible | Incompatible |
| Modifying the order of non-named parameters | Incompatible | Incompatible |
| Modifying the order of named parameters | Compatible | Incompatible |
| Modifying the type of a parameter from A to B | When B is a parent type of A, compatible; otherwise, incompatible | Incompatible |
| Adding a default value to a parameter | Compatible | Compatible |
| Deleting an existing default value of a parameter | Incompatible | Incompatible |
| Modifying the default value of a parameter of a const function or a function marked with @Frozen | Compatible | Incompatible |
| Modifying the default value of a parameter of a non-const function not marked with @Frozen | Compatible | Compatible |
| Modifying the return type from A to B | When B is a subtype of A, compatible; otherwise, incompatible | When B is a subtype of A, and both A and B are class or interface types, compatible; otherwise, incompatible |
| Modifying the function body of a const function or a function marked with @Frozen | Compatible | Incompatible |
| Modifying the function body of a non-const function not marked with @Frozen | Compatible | Compatible |

**Modifying the Order of Static Member Functions**

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Modifying the order of static member functions | Compatible | Compatible |

##### Modifying Static Properties

**Adding and Deleting Static Properties**

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding a static property | Compatible | Adding a static property with overriding semantics (with the redef modifier or with the redef modifier omitted), incompatible; otherwise, compatible |
| Deleting a public static property | Incompatible | Incompatible |
| Deleting a non-public static property | Compatible | Compatible |

**Modifying the Definition of a Static Property**

For an enum type visible outside the module, we call its static property **visible outside the module** when it satisfies any of the following conditions:

1. The static property has the public modifier.
2. The static property is depended upon by other module-externally-visible declarations, and these declarations are marked with const or @Frozen.

Modifications to any static property not visible outside the module satisfy API + ABI compatibility; below we only discuss the module-externally-visible scenario.

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding, deleting, or modifying annotations | Compatible | Compatible |
| Adding the @Deprecated compile marker | Based on the @Deprecated parameter conditions, when the static property is unavailable in downstream modules, incompatible; otherwise, compatible | Based on the @Deprecated parameter conditions, when the static property is unavailable in downstream modules, incompatible; otherwise, compatible |
| Deleting the @Deprecated compile marker | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the static property changes from available to unavailable in downstream modules | Incompatible | Incompatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the static property changes from unavailable to available in downstream modules | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the static property's availability in downstream modules remains unchanged | Compatible | Compatible |
| Adding the @Frozen compile marker | Compatible | Compatible |
| Deleting the @Frozen compile marker | Compatible | Incompatible |
| Access modifier changed from public to non-public | Incompatible | Incompatible |
| Access modifier changed from non-public to public | Compatible | Compatible |
| Access modifier changed among the other three non-public cases (protected/internal/private) | Compatible | Compatible |
| Deleting the static modifier | Incompatible | Incompatible |
| Modifying the property name | Incompatible | Incompatible |
| Modifying the property type | Incompatible | Incompatible |
| Modifying the getter function of a property not marked with @Frozen | Compatible | Compatible |
| Modifying the getter function of a property marked with @Frozen | Compatible | Incompatible |

**Modifying the Order of Static Properties**

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Modifying the order of static properties | Compatible | Compatible |

##### Modifying Instance Member Functions

**Adding and Deleting Instance Member Functions**

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding an instance member function | Compatible | Adding an instance member function with overriding semantics (with the override modifier or with the override modifier omitted), incompatible; otherwise, compatible |
| Deleting a public instance member function | Incompatible | Incompatible |
| Deleting a non-public instance member function | Compatible | Compatible |

**Modifying the Definition of an Instance Member Function**

For an enum type visible outside the module, we call its instance member function **visible outside the module** when it satisfies any of the following conditions:

1. The instance member function has the public modifier.
2. The instance member function is depended upon by other module-externally-visible declarations, and these declarations are marked with const or @Frozen.

Modifications to any instance member function not visible outside the module satisfy API + ABI compatibility; below we only discuss the module-externally-visible scenario.

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding, deleting, or modifying annotations | Compatible | Compatible |
| Adding the @Deprecated compile marker | Based on the @Deprecated parameter conditions, when the instance member function is unavailable in downstream modules, incompatible; otherwise, compatible | Based on the @Deprecated parameter conditions, when the instance member function is unavailable in downstream modules, incompatible; otherwise, compatible |
| Deleting the @Deprecated compile marker | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the instance member function changes from available to unavailable in downstream modules | Incompatible | Incompatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the instance member function changes from unavailable to available in downstream modules | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the instance member function's availability in downstream modules remains unchanged | Compatible | Compatible |
| Adding the @Frozen compile marker | Compatible | Compatible |
| Deleting the @Frozen compile marker | Compatible | Incompatible |
| Access modifier changed from public to non-public | Incompatible | Incompatible |
| Access modifier changed from non-public to public | Compatible | Compatible |
| Access modifier changed among the other three non-public cases (protected/internal/private) | Compatible | Compatible |
| Adding the static modifier | Incompatible | Incompatible |
| Adding the unsafe modifier | Incompatible | Incompatible |
| Deleting the unsafe modifier | Compatible | Compatible |
| Adding the const modifier | Compatible | Compatible |
| Deleting the const modifier | Incompatible | Incompatible |
| Modifying the function name | Incompatible | Incompatible |
| Adding a generic parameter | Incompatible | Incompatible |
| Deleting a generic parameter | Incompatible | Incompatible |
| Modifying the name of a generic parameter | Compatible | Compatible |
| Modifying the order of generic parameters | Incompatible | Incompatible |
| Adding a constraint upper bound to a generic parameter | Incompatible | Incompatible |
| Deleting a certain upper bound constraint of a generic parameter | Compatible | Incompatible |
| Modifying a certain upper bound constraint A of a generic parameter to B | When B is a parent type of A, compatible; otherwise, incompatible | Incompatible |
| Modifying the order among generic parameter constraints or among constraint upper bounds | Compatible | Compatible |
| Adding a function parameter | Incompatible | Incompatible |
| Deleting a function parameter | Incompatible | Incompatible |
| Modifying the name of a non-named parameter | Compatible | Compatible |
| Modifying the name of a named parameter | Incompatible | Incompatible |
| Changing a non-named parameter to a named parameter | Incompatible | Incompatible |
| Changing a named parameter to a non-named parameter | Incompatible | Incompatible |
| Modifying the order of non-named parameters | Incompatible | Incompatible |
| Modifying the order of named parameters | Compatible | Incompatible |
| Modifying the type of a parameter from A to B | When B is a parent type of A, compatible; otherwise, incompatible | Incompatible |
| Adding a default value to a parameter | Compatible | Compatible |
| Deleting an existing default value of a parameter | Incompatible | Incompatible |
| Modifying the default value of a parameter of a const function or a function marked with @Frozen | Compatible | Incompatible |
| Modifying the default value of a parameter of a non-const function not marked with @Frozen | Compatible | Compatible |
| Modifying the return type from A to B | When B is a subtype of A, compatible; otherwise, incompatible | When B is a subtype of A, and both A and B are class or interface types, compatible; otherwise, incompatible |
| Modifying the function body of a const function or a function marked with @Frozen | Compatible | Incompatible |
| Modifying the function body of a non-const function not marked with @Frozen | Compatible | Compatible |

**Modifying the Order of Instance Member Functions**

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Modifying the order of instance member functions | Compatible | Compatible |

##### Modifying Instance Properties

**Adding and Deleting Instance Properties**

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding an instance property | Compatible | Adding an instance property with overriding semantics (with the override modifier or with the override modifier omitted), incompatible; otherwise, compatible |
| Deleting a public instance property | Incompatible | Incompatible |
| Deleting a non-public instance property | Compatible | Compatible |

**Modifying the Definition of an Instance Property**

For an enum type visible outside the module, we call its instance property **visible outside the module** when it satisfies any of the following conditions:

1. The instance property has the public modifier.
2. The instance property is depended upon by other module-externally-visible declarations, and these declarations are marked with const or @Frozen.

Modifications to any instance property not visible outside the module satisfy API + ABI compatibility; below we only discuss the module-externally-visible scenario.

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding, deleting, or modifying annotations | Compatible | Compatible |
| Adding the @Deprecated compile marker | Based on the @Deprecated parameter conditions, when the instance property is unavailable in downstream modules, incompatible; otherwise, compatible | Based on the @Deprecated parameter conditions, when the instance property is unavailable in downstream modules, incompatible; otherwise, compatible |
| Deleting the @Deprecated compile marker | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the instance property changes from available to unavailable in downstream modules | Incompatible | Incompatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the instance property changes from unavailable to available in downstream modules | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the instance property's availability in downstream modules remains unchanged | Compatible | Compatible |
| Adding the @Frozen compile marker | Compatible | Compatible |
| Deleting the @Frozen compile marker | Compatible | Incompatible |
| Access modifier changed from public to non-public | Incompatible | Incompatible |
| Access modifier changed from non-public to public | Compatible | Compatible |
| Access modifier changed among the other three non-public cases (protected/internal/private) | Compatible | Compatible |
| Adding the static modifier | Incompatible | Incompatible |
| Modifying the property name | Incompatible | Incompatible |
| Modifying the property type | Incompatible | Incompatible |
| Modifying the getter function of a property not marked with @Frozen | Compatible | Compatible |
| Modifying the getter function of a property marked with @Frozen | Compatible | Incompatible |

**Modifying the Order of Instance Properties**

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Modifying the order of instance properties | Compatible | Compatible |

##### Modifying Constructors

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding a constructor to an exhaustive enum type | Incompatible | Incompatible |
| Adding a constructor to a non-exhaustive enum type | Compatible | If the added constructor is not at the end of all constructors except the ... constructor, incompatible; if all original constructors have no parameters and the added constructor contains parameters, incompatible; otherwise, compatible |
| Deleting a constructor | Incompatible | Incompatible |
| Modifying the definition of a constructor | Incompatible | Incompatible |
| Modifying the order of constructors | Compatible | Incompatible |

#### Modifying the Definition Location of an enum Type

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Modifying the file where an enum type is defined (note: including renaming the file and moving to another file) | Compatible | If the enum type has the private modifier, incompatible; otherwise, compatible |
| Modifying the position of an enum type definition within the file | Compatible | Compatible |

## Compatibility Rules for class Types

### Addition and Deletion Scenarios

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding a class type | Compatible | Compatible |
| Deleting a class type without the public modifier or whose package does not have the public modifier | Compatible | Compatible |
| Deleting a class type with the public modifier and whose package has the public modifier | Incompatible | Incompatible |

### Modification Scenarios

We call a class type **visible outside the module** when it satisfies any of the following conditions:

1. The class type has the public modifier and the package where it resides has the public modifier.
2. The class type is inherited by other module-externally-visible class types.

Modifications to any class type not visible outside the module satisfy API + ABI compatibility; below we only discuss the module-externally-visible scenario.

#### Modifying the Definition Content of a class Type

##### When the class Type is Visible Outside the Module

###### Modifying Annotations

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding, deleting, or modifying annotations | Compatible | Compatible |

###### Modifying the Compile Marker Part

| Change Behavior | API Compatibility | ABI Compatibility | Note |
|-|-|-|-|
| Adding the @Deprecated compile marker | Based on the @Deprecated parameter conditions, when the class type is unavailable in downstream modules, incompatible; otherwise, compatible | Based on the @Deprecated parameter conditions, when the class type is unavailable in downstream modules, incompatible; otherwise, compatible | NA |
| Deleting the @Deprecated compile marker | Compatible | Compatible | NA |
| Modifying the @Deprecated compile marker parameter conditions, such that the class type changes from available to unavailable in downstream modules | Incompatible | Incompatible | NA |
| Modifying the @Deprecated compile marker parameter conditions, such that the class type changes from unavailable to available in downstream modules | Compatible | Compatible | NA |
| Modifying the @Deprecated compile marker parameter conditions, such that the class type's availability in downstream modules remains unchanged | Compatible | Compatible | NA |
| Adding or deleting the @Java compile marker | Incompatible | Incompatible | The @Java compile marker affects the class's mangle name and underlying implementation |

###### Modifying the Modifier Part

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Access modifier changed from public to non-public | Incompatible | Incompatible |
| Access modifier changed from non-public to public | Compatible | Compatible |
| Access modifier changed among the other three non-public cases (protected/internal/private) | Compatible | Compatible |
| Adding the open modifier | Compatible | Compatible |
| Deleting the open modifier | Incompatible | Incompatible |
| Adding the sealed modifier | Incompatible | Incompatible |
| Deleting the sealed modifier | If the class has the public modifier, compatible; otherwise, incompatible | If the class has the public modifier, compatible; otherwise, incompatible |
| Adding the abstract modifier | Incompatible | Incompatible |
| Deleting the abstract modifier | Incompatible | Incompatible |

###### Modifying Generic Parameters and Generic Parameter Constraints

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding a generic parameter | Incompatible | Incompatible |
| Deleting a generic parameter | Incompatible | Incompatible |
| Modifying the name of a generic parameter | Compatible | Compatible |
| Modifying the order of generic parameters | Incompatible | Incompatible |
| Adding a constraint upper bound to a generic parameter | Incompatible | Incompatible |
| Deleting a certain constraint upper bound of a generic parameter | Compatible | Incompatible |
| Modifying a certain constraint upper bound A of a generic parameter to B | When B is a parent type of A, compatible; otherwise, incompatible | Incompatible |
| Modifying the order among generic parameter constraints or among constraint upper bounds | Compatible | Compatible |

###### Modifying Implemented Interfaces

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding an implemented interface | If it is an abstract class and there are interfaces without implementations among the added interface and all its parent interfaces, incompatible; otherwise, compatible | Compatible |
| Deleting an implemented interface | Incompatible | Incompatible |
| Modifying an implemented interface from A to B | Incompatible | Incompatible |
| Modifying the order of implemented interfaces | Compatible | Compatible |

###### Modifying the Parent Class

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding a parent class | Compatible | Incompatible |
| Deleting a parent class | Incompatible | Incompatible |
| Modifying the parent class from A to B | Incompatible | Incompatible |

###### Modifying Static Member Variables

**Adding and Deleting Static Member Variables**

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding a static member variable | Compatible | Compatible |
| Deleting a static member variable with the public/protected modifier | Incompatible | Incompatible |
| Deleting a static member variable without the public/protected modifier | Compatible | Compatible |

**Modifying the Definition of a Static Member Variable**

For a class type visible outside the module, we call its static member variable **visible outside the module** when it satisfies any of the following conditions:

1. The static member variable has the public/protected modifier.
2. The static member variable is depended upon by other module-externally-visible declarations, and these declarations are marked with const or @Frozen.

Modifications to any static member variable not visible outside the module satisfy API + ABI compatibility; below we only discuss the module-externally-visible scenario.

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding, deleting, or modifying annotations | Compatible | Compatible |
| Adding the @Deprecated compile marker | Based on the @Deprecated parameter conditions, when the static member variable is unavailable in downstream modules, incompatible; otherwise, compatible | Based on the @Deprecated parameter conditions, when the static member variable is unavailable in downstream modules, incompatible; otherwise, compatible |
| Deleting the @Deprecated compile marker | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the static member variable changes from available to unavailable in downstream modules | Incompatible | Incompatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the static member variable changes from unavailable to available in downstream modules | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the static member variable's availability in downstream modules remains unchanged | Compatible | Compatible |
| Access modifier changed from public/protected to internal/private | Incompatible | Incompatible |
| Access modifier changed from internal/private to public/protected | Compatible | Compatible |
| Access modifier changed from public to protected | Incompatible | Incompatible |
| Access modifier changed from protected to public | Compatible | Compatible |
| Access modifier changed from internal to private | Compatible | Compatible |
| Access modifier changed from private to internal | Compatible | Compatible |
| Changing the let modifier to var modifier | Compatible | Compatible |
| Changing the var modifier to let modifier | Incompatible | Incompatible |
| Changing the let modifier to const modifier | Compatible | Incompatible |
| Changing the var modifier to const modifier | Incompatible | Incompatible |
| Changing the const modifier to let modifier | Incompatible | Incompatible |
| Changing the const modifier to var modifier | Incompatible | Incompatible |
| Deleting the static modifier | Incompatible | Incompatible |
| Modifying the variable name | Incompatible | Incompatible |
| Modifying the variable type from A to B | When B is a subtype of A, compatible; otherwise, incompatible | When B is a subtype of A, and both A and B are class or interface types, compatible; otherwise, incompatible |
| Modifying the initial value of a non-const variable | Compatible | Compatible |
| Modifying the initial value of a const variable | Compatible | Incompatible |

**Modifying the Order of Static Member Variables**

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Modifying the order of static member variables | Compatible | Compatible |

###### Modifying Instance Member Variables

**Adding and Deleting Instance Member Variables**

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding an instance member variable | Compatible | If the class type has no open modifier and the added instance member variable is at the end of all instance member variables, compatible; otherwise, incompatible |
| Deleting an instance member variable with the public or protected modifier | Incompatible | Incompatible |
| Deleting an instance member variable without the public or protected modifier | Compatible | Incompatible |

**Modifying the Definition of a public/protected Instance Member Variable**

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding, deleting, or modifying annotations | Compatible | Compatible |
| Adding the @Deprecated compile marker | Based on the @Deprecated parameter conditions, when the instance member variable is unavailable in downstream modules, incompatible; otherwise, compatible | Based on the @Deprecated parameter conditions, when the instance member variable is unavailable in downstream modules, incompatible; otherwise, compatible |
| Deleting the @Deprecated compile marker | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the instance member variable changes from available to unavailable in downstream modules | Incompatible | Incompatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the instance member variable changes from unavailable to available in downstream modules | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the instance member variable's availability in downstream modules remains unchanged | Compatible | Compatible |
| Access modifier changed from public/protected to private/internal | Incompatible | Incompatible |
| Changing the let modifier to var modifier | Compatible | Compatible |
| Changing the var modifier to let modifier | Incompatible | Incompatible |
| Modifying the variable name | Incompatible | Incompatible |
| Modifying the instance member variable type from A to B | Incompatible | Incompatible |
| Modifying the variable initial value | Compatible | If the class type contains an init function annotated with const or @Frozen, incompatible; otherwise, compatible |

**Modifying the Definition of a private/internal Instance Member Variable**

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding, deleting, or modifying annotations | Compatible | Compatible |
| Adding the @Deprecated compile marker | Compatible | Compatible |
| Deleting the @Deprecated compile marker | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions | Compatible | Compatible |
| Access modifier changed from private/internal to public/protected | Compatible | Compatible |
| Access modifier changed between the two cases of private and internal | Compatible | Compatible |
| Changing the let modifier to var modifier | Compatible | Compatible |
| Changing the var modifier to let modifier | Compatible | Compatible |
| Modifying the variable name | Compatible | Compatible |
| Modifying the instance member variable type from A to B | Compatible | Incompatible |
| Modifying the variable initial value | Compatible | If the class type contains an init function annotated with const or @Frozen, incompatible; otherwise, compatible |

**Modifying the Order of Instance Member Variables**

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Modifying the order of instance member variables | Compatible | Incompatible |

###### Modifying Static Member Functions

**Adding and Deleting Static Member Functions**

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding a static member function | Compatible | Adding a static member function with overriding semantics (with the redef modifier or with the redef modifier omitted), incompatible; otherwise, compatible |
| Deleting a static member function with the public/protected modifier | Incompatible | Incompatible |
| Deleting a static member function without the public/protected modifier | Compatible | Compatible |

**Modifying the Definition of a Static Member Function**

For a class type visible outside the module, we call its static member function **visible outside the module** when it satisfies any of the following conditions:

1. The static member function has the public/protected modifier.
2. The static member function is depended upon by other module-externally-visible declarations, and these declarations are marked with const or @Frozen.

Modifications to any static member function not visible outside the module satisfy API + ABI compatibility; below we only discuss the module-externally-visible scenario.

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding, deleting, or modifying annotations | Compatible | Compatible |
| Adding the @Deprecated compile marker | Based on the @Deprecated parameter conditions, when the static member function is unavailable in downstream modules, incompatible; otherwise, compatible | Based on the @Deprecated parameter conditions, when the static member function is unavailable in downstream modules, incompatible; otherwise, compatible |
| Deleting the @Deprecated compile marker | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the static member function changes from available to unavailable in downstream modules | Incompatible | Incompatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the static member function changes from unavailable to available in downstream modules | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the static member function's availability in downstream modules remains unchanged | Compatible | Compatible |
| Adding the @Frozen compile marker | Compatible | Compatible |
| Deleting the @Frozen compile marker | Compatible | Incompatible |
| Access modifier changed from public/protected to internal/private | Incompatible | Incompatible |
| Access modifier changed from internal/private to public/protected | Compatible | Compatible |
| Access modifier changed from public to protected | Incompatible | Incompatible |
| Access modifier changed from protected to public | Compatible | Compatible |
| Access modifier changed from internal to private | Compatible | Compatible |
| Access modifier changed from private to internal | Compatible | Compatible |
| Adding the unsafe modifier | Incompatible | Incompatible |
| Deleting the unsafe modifier | Compatible | Compatible |
| Adding the const modifier | Compatible | Compatible |
| Deleting the const modifier | Incompatible | Incompatible |
| Deleting the static modifier | Incompatible | Incompatible |
| Modifying the function name | Incompatible | Incompatible |
| Adding a generic parameter | Incompatible | Incompatible |
| Deleting a generic parameter | Incompatible | Incompatible |
| Modifying the name of a generic parameter | Compatible | Compatible |
| Modifying the order of generic parameters | Incompatible | Incompatible |
| Adding a constraint upper bound to a generic parameter | Incompatible | Incompatible |
| Deleting a certain upper bound constraint of a generic parameter | Compatible | Incompatible |
| Modifying a certain upper bound constraint A of a generic parameter to B | When B is a parent type of A, compatible; otherwise, incompatible | Incompatible |
| Modifying the order among generic parameter constraints or among constraint upper bounds | Compatible | Compatible |
| Adding a function parameter | Incompatible | Incompatible |
| Deleting a function parameter | Incompatible | Incompatible |
| Modifying the name of a non-named parameter | Compatible | Compatible |
| Modifying the name of a named parameter | Incompatible | Incompatible |
| Changing a non-named parameter to a named parameter | Incompatible | Incompatible |
| Changing a named parameter to a non-named parameter | Incompatible | Incompatible |
| Modifying the order of non-named parameters | Incompatible | Incompatible |
| Modifying the order of named parameters | Compatible | Incompatible |
| Modifying the type of a parameter from A to B | When B is a parent type of A, compatible; otherwise, incompatible | Incompatible |
| Adding a default value to a parameter | Compatible | Compatible |
| Deleting an existing default value of a parameter | Incompatible | Incompatible |
| Modifying the default value of a parameter of a const function or a function marked with @Frozen | Compatible | Incompatible |
| Modifying the default value of a parameter of a non-const function not marked with @Frozen | Compatible | Compatible |
| Modifying the return type from A to B | When B is a subtype of A, compatible; otherwise, incompatible | When B is a subtype of A, and both A and B are class or interface types, compatible; otherwise, incompatible |
| Modifying the function body of a const function or a function marked with @Frozen | Compatible | Incompatible |
| Modifying the function body of a non-const function not marked with @Frozen | Compatible | Compatible |

**Modifying the Order of Static Member Functions**

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Modifying the order of static member functions | Compatible | Compatible |

###### Modifying Static Properties

**Adding and Deleting Static Properties**

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding a static property | Compatible | Adding a static property with overriding semantics (with the redef modifier or with the redef modifier omitted), incompatible; otherwise, compatible |
| Deleting a public/protected static property | Incompatible | Incompatible |
| Deleting a non-public/protected static property | Compatible | Compatible |

**Modifying the Definition of a Static Property**

For a class type visible outside the module, we call its static property **visible outside the module** when it satisfies any of the following conditions:

1. The static property has the public/protected modifier.
2. The static property is depended upon by other module-externally-visible declarations, and these declarations are marked with const or @Frozen.

Modifications to any static property not visible outside the module satisfy API + ABI compatibility; below we only discuss the module-externally-visible scenario.

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding, deleting, or modifying annotations | Compatible | Compatible |
| Adding the @Deprecated compile marker | Based on the @Deprecated parameter conditions, when the static property is unavailable in downstream modules, incompatible; otherwise, compatible | Based on the @Deprecated parameter conditions, when the static property is unavailable in downstream modules, incompatible; otherwise, compatible |
| Deleting the @Deprecated compile marker | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the static property changes from available to unavailable in downstream modules | Incompatible | Incompatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the static property changes from unavailable to available in downstream modules | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the static property's availability in downstream modules remains unchanged | Compatible | Compatible |
| Adding the @Frozen compile marker | Compatible | Compatible |
| Deleting the @Frozen compile marker | Compatible | Incompatible |
| Access modifier changed from public/protected to internal/private | Incompatible | Incompatible |
| Access modifier changed from internal/private to public/protected | Compatible | Compatible |
| Access modifier changed from public to protected | Incompatible | Incompatible |
| Access modifier changed from protected to public | Compatible | Compatible |
| Access modifier changed from internal to private | Compatible | Compatible |
| Access modifier changed from private to internal | Compatible | Compatible |
| Deleting the static modifier | Incompatible | Incompatible |
| Adding the mut modifier and the corresponding setter function | Compatible | Compatible |
| Deleting the mut modifier and the corresponding setter function | Incompatible | Incompatible |
| Modifying the property name | Incompatible | Incompatible |
| Modifying the property type | Incompatible | Incompatible |
| Modifying the getter/setter functions of a property not marked with @Frozen | Compatible | Compatible |
| Modifying the getter/setter functions of a property marked with @Frozen | Compatible | Incompatible |

**Modifying the Order of Static Properties**

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Modifying the order of static properties | Compatible | Compatible |

###### Modifying Instance Member Functions

**Adding and Deleting Instance Member Functions**

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding an instance member function without open semantics | Compatible | Adding an instance member function with overriding semantics (with the override modifier or with the override modifier omitted), incompatible; otherwise, compatible |
| Adding an instance member function with open semantics | If the added instance member function has a default implementation, compatible; otherwise, incompatible | If the added instance member function is at the end of all instance member functions and instance properties with open semantics, and has no overriding semantics (no override modifier and not omitting the override modifier), compatible; otherwise, incompatible |
| Deleting a public/protected instance member function | Incompatible | Incompatible |
| Deleting a non-public/protected instance member function | Compatible | Compatible |

**Modifying the Definition of an Instance Member Function**

For the scenario of modifying the definition of an instance member function, we also need to further divide into two cases based on whether the instance member function is visible outside the module. For a class type visible outside the module, we call its instance member function visible outside the module when it satisfies any of the following conditions:

1. The instance member function has the public/protected modifier.
2. The instance member function is depended upon by other module-externally-visible declarations, and these declarations are marked with const or @Frozen.

Modifications to any instance member function not visible outside the module satisfy API + ABI compatibility; below we only discuss the module-externally-visible scenario.

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding, deleting, or modifying annotations | Compatible | Compatible |
| Adding the @Deprecated compile marker | Based on the @Deprecated parameter conditions, when the instance member function is unavailable in downstream modules, incompatible; otherwise, compatible | Based on the @Deprecated parameter conditions, when the instance member function is unavailable in downstream modules, incompatible; otherwise, compatible |
| Deleting the @Deprecated compile marker | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the instance member function changes from available to unavailable in downstream modules | Incompatible | Incompatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the instance member function changes from unavailable to available in downstream modules | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the instance member function's availability in downstream modules remains unchanged | Compatible | Compatible |
| Adding the @Frozen compile marker | Compatible | Compatible |
| Deleting the @Frozen compile marker | Compatible | Incompatible |
| Access modifier changed from public/protected to internal/private | Incompatible | Incompatible |
| Access modifier changed from internal/private to public/protected | Compatible | Compatible |
| Access modifier changed from public to protected | Incompatible | Incompatible |
| Access modifier changed from protected to public | Compatible | Compatible |
| Access modifier changed from internal to private | Compatible | Compatible |
| Access modifier changed from private to internal | Compatible | Compatible |
| Adding the static modifier | Incompatible | Incompatible |
| Adding the unsafe modifier | Incompatible | Incompatible |
| Deleting the unsafe modifier | Compatible | Compatible |
| Adding the const modifier | Compatible | Incompatible |
| Deleting the const modifier | Incompatible | Incompatible |
| Adding the open modifier | Compatible | Incompatible |
| Deleting the open modifier | Incompatible | Incompatible |
| Modifying the function name | Incompatible | Incompatible |
| Adding a generic parameter | Incompatible | Incompatible |
| Deleting a generic parameter | Incompatible | Incompatible |
| Modifying the name of a generic parameter | Compatible | Compatible |
| Modifying the order of generic parameters | Incompatible | Incompatible |
| Adding a constraint upper bound to a generic parameter | Incompatible | Incompatible |
| Deleting a certain upper bound constraint of a generic parameter | Compatible | Incompatible |
| Modifying a certain upper bound constraint A of a generic parameter to B | When B is a parent type of A, compatible; otherwise, incompatible | Incompatible |
| Modifying the order among generic parameter constraints or among constraint upper bounds | Compatible | Compatible |
| Adding a function parameter | Incompatible | Incompatible |
| Deleting a function parameter | Incompatible | Incompatible |
| Modifying the name of a non-named parameter | Compatible | Compatible |
| Modifying the name of a named parameter | Incompatible | Incompatible |
| Changing a non-named parameter to a named parameter | Incompatible | Incompatible |
| Changing a named parameter to a non-named parameter | Incompatible | Incompatible |
| Modifying the order of non-named parameters | Incompatible | Incompatible |
| Modifying the order of named parameters | Compatible | Incompatible |
| Modifying the type of a parameter from A to B | When the instance member function has no open semantics, and B is a parent type of A, compatible; otherwise, incompatible | Incompatible |
| Adding a default value to a parameter | Compatible | Compatible |
| Deleting an existing default value of a parameter | Incompatible | Incompatible |
| Modifying the default value of a parameter of a const function or a function marked with @Frozen | Compatible | Incompatible |
| Modifying the default value of a parameter of a non-const function not marked with @Frozen | Compatible | Compatible |
| Modifying the return type from A to B | When the instance member function has no open semantics, and B is a subtype of A, compatible; otherwise, incompatible | When the instance member function has no open semantics, and B is a subtype of A, and both A and B are class or interface types, compatible; otherwise, incompatible |
| Modifying the function body of a const function or a function marked with @Frozen | Compatible | Incompatible |
| Modifying the function body of a non-const function not marked with @Frozen | Compatible | Compatible |

**Modifying the Order of Instance Member Functions**

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Modifying the order of instance member functions | Compatible | When the modification does not cause a change in the order among instance member functions with open semantics, compatible; otherwise, incompatible |

###### Modifying Instance Properties

**Adding and Deleting Instance Properties**

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding an instance property without open semantics | Compatible | Adding an instance property with overriding semantics (with the override modifier or with the override modifier omitted), incompatible; otherwise, compatible |
| Adding an instance property with open semantics | If the added instance property has a default implementation, compatible; otherwise, incompatible | If the added instance property is at the end of all instance member functions and instance properties with open semantics, and has no overriding semantics (no override modifier and not omitting the override modifier), compatible; otherwise, incompatible |
| Deleting a public/protected instance property | Incompatible | Incompatible |
| Deleting a non-public/protected instance property | Compatible | Compatible |

**Modifying the Definition of an Instance Property**

For the scenario of modifying the definition of an instance property, we also need to further divide into two cases based on whether the instance property is visible outside the module. For a class type visible outside the module, we call its instance property visible outside the module when it satisfies any of the following conditions:

1. The instance property has the public/protected modifier.
2. The instance property is depended upon by other module-externally-visible declarations, and these declarations are marked with const or @Frozen.

Modifications to any instance property not visible outside the module satisfy API + ABI compatibility; below we only discuss the module-externally-visible scenario.

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding, deleting, or modifying annotations | Compatible | Compatible |
| Adding the @Deprecated compile marker | Based on the @Deprecated parameter conditions, when the instance property is unavailable in downstream modules, incompatible; otherwise, compatible | Based on the @Deprecated parameter conditions, when the instance property is unavailable in downstream modules, incompatible; otherwise, compatible |
| Deleting the @Deprecated compile marker | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the instance property changes from available to unavailable in downstream modules | Incompatible | Incompatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the instance property changes from unavailable to available in downstream modules | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the instance property's availability in downstream modules remains unchanged | Compatible | Compatible |
| Adding the @Frozen compile marker | Compatible | Compatible |
| Deleting the @Frozen compile marker | Compatible | Incompatible |
| Access modifier changed from public/protected to internal/private | Incompatible | Incompatible |
| Access modifier changed from internal/private to public/protected | Compatible | Compatible |
| Access modifier changed from public to protected | Incompatible | Incompatible |
| Access modifier changed from protected to public | Compatible | Compatible |
| Access modifier changed from internal to private | Compatible | Compatible |
| Access modifier changed from private to internal | Compatible | Compatible |
| Adding the static modifier | Incompatible | Incompatible |
| Adding the mut modifier and the corresponding setter function | Compatible | If the instance property has no open semantics, compatible; if the instance property has open semantics and is at the end of all instance member functions and instance properties with open semantics, compatible; otherwise, incompatible |
| Deleting the mut modifier and the corresponding setter function | Incompatible | Incompatible |
| Modifying the property name | Incompatible | Incompatible |
| Modifying the property type | Incompatible | Incompatible |
| Modifying the getter/setter functions of a property not marked with @Frozen | Compatible | Compatible |
| Modifying the getter/setter functions of a property marked with @Frozen | Compatible | Incompatible |

**Modifying the Order of Instance Properties**

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Modifying the order of instance properties | Compatible | When the modification does not cause a change in the order among instance properties with open semantics, compatible; otherwise, incompatible |

##### Modifying init Constructors

###### Adding and Deleting init Constructors

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding an init constructor | Compatible | Compatible |
| Deleting an init constructor without the public/protected modifier | Compatible | Compatible |
| Deleting an init constructor with the public/protected modifier | Incompatible | Incompatible |

###### Modifying the Definition of an init Constructor

For a class type visible outside the module, we call its constructor **visible outside the module** when it satisfies any of the following conditions:

1. The init constructor has the public/protected modifier.
2. The init constructor is depended upon by other module-externally-visible declarations, and these declarations are marked with const or @Frozen.

Modifications to any init constructor not visible outside the module satisfy API + ABI compatibility; below we only discuss the module-externally-visible scenario.

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding, deleting, or modifying annotations | Compatible | Compatible |
| Adding the @Deprecated compile marker | Based on the @Deprecated parameter conditions, when the init constructor is unavailable in downstream modules, incompatible; otherwise, compatible | Based on the @Deprecated parameter conditions, when the init constructor is unavailable in downstream modules, incompatible; otherwise, compatible |
| Deleting the @Deprecated compile marker | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the init constructor changes from available to unavailable in downstream modules | Incompatible | Incompatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the init constructor changes from unavailable to available in downstream modules | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the init constructor's availability in downstream modules remains unchanged | Compatible | Compatible |
| Adding the @Frozen compile marker | Compatible | Compatible |
| Deleting the @Frozen compile marker | Compatible | Incompatible |
| Access modifier changed from public/protected to internal/private | Incompatible | Incompatible |
| Access modifier changed from internal/private to public/protected | Compatible | Compatible |
| Access modifier changed from public to protected | Incompatible | Incompatible |
| Access modifier changed from protected to public | Compatible | Compatible |
| Access modifier changed from internal to private | Compatible | Compatible |
| Access modifier changed from private to internal | Compatible | Compatible |
| Adding the const modifier | Compatible | Compatible |
| Deleting the const modifier | Incompatible | Incompatible |
| Adding a function parameter | If the added function parameter is at the end of all function parameters and has a default value, compatible; otherwise, incompatible | Incompatible |
| Deleting a function parameter | Incompatible | Incompatible |
| Modifying the name of a non-named parameter | Compatible | Compatible |
| Modifying the name of a named parameter | Incompatible | Incompatible |
| Changing a non-named parameter to a named parameter | Incompatible | Incompatible |
| Changing a named parameter to a non-named parameter | Incompatible | Incompatible |
| Modifying the order of non-named parameters | Incompatible | Incompatible |
| Modifying the order of named parameters | Compatible | Incompatible |
| Modifying the type of a parameter from A to B | When B is a parent type of A, compatible; otherwise, incompatible | Incompatible |
| Adding a default value to a parameter | Compatible | Compatible |
| Deleting an existing default value of a parameter | Incompatible | Incompatible |
| Modifying the default value of a parameter of a const function or a function marked with @Frozen | Compatible | Incompatible |
| Modifying the default value of a parameter of a non-const function not marked with @Frozen | Compatible | Compatible |
| Modifying the function body of a const function or a function marked with @Frozen | Compatible | Incompatible |
| Modifying the function body of a non-const function not marked with @Frozen | Compatible | Compatible |

###### Modifying the Order of init Constructors

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Modifying the order of init constructors | Compatible | Compatible |

##### Modifying Primary Constructors

###### Adding and Deleting Primary Constructors

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding a primary constructor | Compatible | When the primary constructor has member variable parameters, incompatible; otherwise, compatible |
| Deleting a constructor without the public/protected modifier | When the primary constructor has member variable parameters modified with public/protected/internal, incompatible; otherwise, compatible | When the primary constructor has member variable parameters, incompatible; otherwise, compatible |
| Deleting a constructor with the public/protected modifier | Incompatible | Incompatible |

###### Modifying the Definition of a Primary Constructor

For a class type visible outside the module, we call its primary constructor **visible outside the module** when it satisfies any of the following conditions:

1. The primary constructor has the public/protected modifier.
2. The primary constructor is depended upon by other module-externally-visible declarations, and these declarations are marked with const or @Frozen.

**Modifying the Definition of a Module-Externally-Visible Primary Constructor**

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding, deleting, or modifying annotations | Compatible | Compatible |
| Adding the @Deprecated compile marker | Based on the @Deprecated parameter conditions, when the primary constructor is unavailable in downstream modules, incompatible; otherwise, compatible | Based on the @Deprecated parameter conditions, when the primary constructor is unavailable in downstream modules, incompatible; otherwise, compatible |
| Deleting the @Deprecated compile marker | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the primary constructor changes from available to unavailable in downstream modules | Incompatible | Incompatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the primary constructor changes from unavailable to available in downstream modules | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the primary constructor's availability in downstream modules remains unchanged | Compatible | Compatible |
| Adding the @Frozen compile marker | Compatible | Compatible |
| Deleting the @Frozen compile marker | Compatible | Incompatible |
| Access modifier changed from public/protected to internal/private | Incompatible | Incompatible |
| Access modifier changed from internal/private to public/protected | Compatible | Compatible |
| Access modifier changed from public to protected | Incompatible | Incompatible |
| Access modifier changed from protected to public | Compatible | Compatible |
| Access modifier changed from internal to private | Compatible | Compatible |
| Access modifier changed from private to internal | Compatible | Compatible |
| Adding the const modifier | Compatible | Compatible |
| Deleting the const modifier | Incompatible | Incompatible |
| Adding an ordinary function parameter | If the added ordinary function parameter is at the end of all function parameters and has a default value, compatible; otherwise, incompatible | Incompatible |
| Deleting an ordinary function parameter | Incompatible | Incompatible |
| Modifying the name of a non-named ordinary function parameter | Compatible | Compatible |
| Modifying the name of a named ordinary function parameter | Incompatible | Incompatible |
| Changing a non-named ordinary function parameter to a named ordinary function parameter | Incompatible | Incompatible |
| Changing a named ordinary function parameter to a non-named ordinary function parameter | Incompatible | Incompatible |
| Modifying the order of non-named ordinary function parameters | Incompatible | Incompatible |
| Modifying the order of named ordinary function parameters | Compatible | Incompatible |
| Modifying the type of an ordinary function parameter from A to B | When B is a parent type of A, compatible; otherwise, incompatible | Incompatible |
| Adding a default value to an ordinary function parameter | Compatible | Compatible |
| Deleting an existing default value of an ordinary function parameter | Incompatible | Incompatible |
| Modifying the default value of an ordinary function parameter of a const function or a function marked with @Frozen | Compatible | Incompatible |
| Modifying the default value of an ordinary function parameter of a non-const function not marked with @Frozen | Compatible | Compatible |
| Adding a member variable parameter | If the added member variable parameter is at the end of all function parameters and has a default value, compatible; otherwise, incompatible | Incompatible |
| Deleting a member variable parameter | Incompatible | Incompatible |
| Modifying the name of a non-named member variable parameter | If the non-named member variable parameter has the public/protected modifier, incompatible; otherwise, compatible | If the non-named member variable parameter has the public/protected modifier, incompatible; otherwise, compatible |
| Modifying the name of a named member variable parameter | Incompatible | Incompatible |
| Changing a non-named member variable parameter to a named member variable parameter | Incompatible | Incompatible |
| Changing a named member variable parameter to a non-named member variable parameter | Incompatible | Incompatible |
| Modifying the order of non-named member variable parameters | Incompatible | Incompatible |
| Modifying the order of named member variable parameters | Compatible | Incompatible |
| Modifying the type of a member variable parameter from A to B | Incompatible | Incompatible |
| Adding a default value to a member variable parameter | Compatible | If the class type contains an init function annotated with const or @Frozen, incompatible; otherwise, compatible |
| Deleting an existing default value of a member variable parameter | Incompatible | Incompatible |
| Modifying the default value of a member variable parameter of a const function or a function marked with @Frozen | Compatible | Incompatible |
| Modifying the default value of a member variable parameter of a non-const function not marked with @Frozen | Compatible | Compatible |
| Modifying the function body of a const function or a function marked with @Frozen | Compatible | Incompatible |
| Modifying the function body of a non-const function not marked with @Frozen | Compatible | Compatible |

##### Modifying static init Functions

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding a static init function | Compatible | Compatible |
| Deleting a static init function | Compatible | Compatible |
| Modifying the definition of a static init function | Compatible | Compatible |

#### Modifying the Definition Location of a class Type

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Modifying the file where a class type is defined (note: including renaming the file and moving to another file) | Compatible | If the class type has the private modifier, incompatible; otherwise, compatible |
| Modifying the position of a class type definition within the file | Compatible | Compatible |

## Compatibility Rules for interface Types

### Addition and Deletion Scenarios

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding an interface type | Compatible | Compatible |
| Deleting an interface type without the public modifier or whose package does not have the public modifier | Compatible | Compatible |
| Deleting an interface type with the public modifier and whose package has the public modifier | Incompatible | Incompatible |

### Modification Scenarios

We call an interface type **visible outside the module** when it satisfies any of the following conditions:

1. The interface type has the public modifier and the package where it resides has the public modifier.
2. The interface type is inherited by other module-externally-visible class/interface types.

Modifications to any interface type not visible outside the module satisfy API + ABI compatibility; below we only discuss the module-externally-visible scenario.

#### Modifying the Definition Content of an interface Type

##### Modifying Annotations

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding, deleting, or modifying annotations | Compatible | Compatible |

##### Modifying the Compile Marker Part

| Change Behavior | API Compatibility | ABI Compatibility | Note |
|-|-|-|-|
| Adding the @Deprecated compile marker | Based on the @Deprecated parameter conditions, when the interface type is unavailable in downstream modules, incompatible; otherwise, compatible | Based on the @Deprecated parameter conditions, when the interface type is unavailable in downstream modules, incompatible; otherwise, compatible | NA |
| Deleting the @Deprecated compile marker | Compatible | Compatible | NA |
| Modifying the @Deprecated compile marker parameter conditions, such that the interface type changes from available to unavailable in downstream modules | Incompatible | Incompatible | NA |
| Modifying the @Deprecated compile marker parameter conditions, such that the interface type changes from unavailable to available in downstream modules | Compatible | Compatible | NA |
| Modifying the @Deprecated compile marker parameter conditions, such that the interface type's availability in downstream modules remains unchanged | Compatible | Compatible | NA |
| Adding or deleting the @Java compile marker | Incompatible | Incompatible | The @Java compile marker affects the interface's mangle name and underlying implementation |

##### Modifying the Modifier Part

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Access modifier changed from public to non-public | Incompatible | Incompatible |
| Access modifier changed from non-public to public | Compatible | Compatible |
| Access modifier changed among the other three non-public cases (protected/internal/private) | Compatible | Compatible |
| Adding the sealed modifier | Incompatible | Incompatible |
| Deleting the sealed modifier | If the interface has the public modifier, compatible; otherwise, incompatible | If the interface has the public modifier, compatible; otherwise, incompatible |

##### Modifying Generic Parameters and Generic Parameter Constraints

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding a generic parameter | Incompatible | Incompatible |
| Deleting a generic parameter | Incompatible | Incompatible |
| Modifying the name of a generic parameter | Compatible | Compatible |
| Modifying the order of generic parameters | Incompatible | Incompatible |
| Adding a constraint upper bound to a generic parameter | Incompatible | Incompatible |
| Deleting a certain upper bound constraint of a generic parameter | Compatible | Incompatible |
| Modifying a certain upper bound constraint A of a generic parameter to B | When B is a parent type of A, compatible; otherwise, incompatible | Incompatible |
| Modifying the order among generic parameter constraints or among constraint upper bounds | Compatible | Compatible |

##### Modifying Inherited Interfaces

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding an inherited interface | If there are interfaces without implementations among the added interface and all its parent interfaces, incompatible; otherwise, compatible | Compatible |
| Deleting an inherited interface | Incompatible | Incompatible |
| Modifying an inherited interface from A to B | Incompatible | Incompatible |
| Modifying the order of inherited interfaces | Compatible | Compatible |

##### Modifying Member Functions

###### Adding and Deleting Member Functions

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding a member function | If the added member function has a default implementation, compatible; otherwise, incompatible | If the added member function is at the end of all member functions or properties, and has no overriding semantics (no redef/override modifier and not omitting the redef/override modifier), compatible; otherwise, incompatible |
| Deleting a member function | Incompatible | Incompatible |

###### Modifying the Definition of a Member Function

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding, deleting, or modifying annotations | Compatible | Compatible |
| Adding the @Deprecated compile marker | Based on the @Deprecated parameter conditions, when the instance member function is unavailable in downstream modules, incompatible; otherwise, compatible | Based on the @Deprecated parameter conditions, when the instance member function is unavailable in downstream modules, incompatible; otherwise, compatible |
| Deleting the @Deprecated compile marker | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the instance member function changes from available to unavailable in downstream modules | Incompatible | Incompatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the instance member function changes from unavailable to available in downstream modules | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the instance member function's availability in downstream modules remains unchanged | Compatible | Compatible |
| Adding the @Frozen compile marker | Compatible | Compatible |
| Deleting the @Frozen compile marker | Compatible | Incompatible |
| Adding the static modifier | Incompatible | Incompatible |
| Deleting the static modifier | Incompatible | Incompatible |
| Adding the unsafe modifier | Incompatible | Incompatible |
| Deleting the unsafe modifier | Compatible | Compatible |
| Adding the const modifier | Compatible | Incompatible |
| Deleting the const modifier | Incompatible | Incompatible |
| Adding the mut modifier | Incompatible | Incompatible |
| Deleting the mut modifier | Incompatible | Incompatible |
| Modifying the function name | Incompatible | Incompatible |
| Adding a generic parameter | Incompatible | Incompatible |
| Deleting a generic parameter | Incompatible | Incompatible |
| Modifying the name of a generic parameter | Compatible | Compatible |
| Modifying the order of generic parameters | Incompatible | Incompatible |
| Adding a constraint upper bound to a generic parameter | Incompatible | Incompatible |
| Deleting a certain upper bound constraint of a generic parameter | Compatible | Incompatible |
| Modifying a certain upper bound constraint A of a generic parameter to B | When B is a parent type of A, compatible; otherwise, incompatible | Incompatible |
| Modifying the order among generic parameter constraints or among constraint upper bounds | Compatible | Compatible |
| Adding a function parameter | Incompatible | Incompatible |
| Deleting a function parameter | Incompatible | Incompatible |
| Modifying the name of a non-named parameter | Compatible | Compatible |
| Modifying the name of a named parameter | Incompatible | Incompatible |
| Changing a non-named parameter to a named parameter | Incompatible | Incompatible |
| Changing a named parameter to a non-named parameter | Incompatible | Incompatible |
| Modifying the order of non-named parameters | Incompatible | Incompatible |
| Modifying the order of named parameters | Compatible | Incompatible |
| Modifying the type of a parameter from A to B | Incompatible | Incompatible |
| Modifying the return type from A to B | Incompatible | Incompatible |
| Modifying the function body of a const function or a function marked with @Frozen | Compatible | Incompatible |
| Modifying the function body of a non-const function not marked with @Frozen | Compatible | Compatible |

###### Modifying the Order of Member Functions

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Modifying the order of member functions | Compatible | Incompatible |

##### Modifying Properties

###### Adding and Deleting Properties

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding a member property | If the added member property has a default implementation, compatible; otherwise, incompatible | If the added member property is at the end of all member functions or properties, and has no overriding semantics (no redef/override modifier and not omitting the redef/override modifier), compatible; otherwise, incompatible |
| Deleting a member property | Incompatible | Incompatible |

###### Modifying the Definition of a Property

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding, deleting, or modifying annotations | Compatible | Compatible |
| Adding the @Deprecated compile marker | Based on the @Deprecated parameter conditions, when the instance member property is unavailable in downstream modules, incompatible; otherwise, compatible | Based on the @Deprecated parameter conditions, when the instance member property is unavailable in downstream modules, incompatible; otherwise, compatible |
| Deleting the @Deprecated compile marker | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the instance member property changes from available to unavailable in downstream modules | Incompatible | Incompatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the instance member property changes from unavailable to available in downstream modules | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the instance member property's availability in downstream modules remains unchanged | Compatible | Compatible |
| Adding the @Frozen compile marker | Compatible | Compatible |
| Deleting the @Frozen compile marker | Compatible | Incompatible |
| Adding the static modifier | Incompatible | Incompatible |
| Deleting the static modifier | Incompatible | Incompatible |
| Adding the mut modifier and the corresponding setter function | Incompatible | Incompatible |
| Deleting the mut modifier and the corresponding setter function | Incompatible | Incompatible |
| Modifying the property name | Incompatible | Incompatible |
| Modifying the property type | Incompatible | Incompatible |
| Modifying the getter/setter functions of a property not marked with @Frozen | Compatible | Compatible |
| Modifying the getter/setter functions of a property marked with @Frozen | Compatible | Incompatible |

###### Modifying the Order of Properties

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Modifying the order of properties | Compatible | Incompatible |

#### Modifying the Definition Location of an interface Type

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Modifying the file where an interface type is defined (note: including renaming the file and moving to another file) | Compatible | If the interface type has the private modifier, incompatible; otherwise, compatible |
| Modifying the position of an interface type definition within the file | Compatible | Compatible |

## Compatibility Rules for extend Types

### Addition and Deletion Scenarios

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding an extend | Compatible | Compatible |
| Deleting a non-exported extend | Compatible | Compatible |
| Deleting an exported extend | Incompatible | Incompatible |

### Modification Scenarios

Modifications to any non-exported extend satisfy API + ABI compatibility; below we only discuss the exported extend scenario.

#### Modifying the Definition Content of an extend

##### Modifying Generic Parameters and Generic Parameter Constraints

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding a generic parameter | Incompatible | Incompatible |
| Deleting a generic parameter | Incompatible | Incompatible |
| Modifying the name of a generic parameter | Compatible | Compatible |
| Modifying the order of generic parameters | Incompatible | Incompatible |
| Adding a constraint upper bound to a generic parameter | Incompatible | Incompatible |
| Deleting a certain upper bound constraint of a generic parameter | Compatible | Incompatible |
| Modifying a certain upper bound constraint A of a generic parameter to B | When B is a parent type of A, compatible; otherwise, incompatible | Incompatible |
| Modifying the order among generic parameter constraints or among constraint upper bounds | Compatible | Compatible |

##### Modifying Implemented Interfaces

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding an implemented interface | Compatible | Compatible |
| Deleting an implemented interface | Incompatible | Incompatible |
| Modifying an implemented interface from A to B | Incompatible | Incompatible |
| Modifying the order of implemented interfaces | Compatible | Compatible |

##### Modifying Static Member Functions

###### Adding and Deleting Static Member Functions

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding a static member function | Compatible | Adding a re-implemented static member function, incompatible; otherwise, compatible |
| Deleting a static member function with the public modifier | Incompatible | Incompatible |
| Deleting a static member function with the protected modifier | If the extended type is a class type, incompatible; if the extended type is another non-class type, compatible | If the extended type is a class type, incompatible; if the extended type is another non-class type, compatible |
| Deleting a static member function without the public/protected modifier | Compatible | Compatible |

###### Modifying the Definition of a Static Member Function

For an exported extend, we call its static member function **visible outside the module** when it satisfies any of the following conditions:

1. The extended type is a class type, and the static member function has the public/protected modifier
2. The extended type is a non-class type, and the static member function has the public modifier
3. The static member function is depended upon by other module-externally-visible declarations, and these declarations are marked with const or @Frozen.

Modifications to any static member function not visible outside the module satisfy API + ABI compatibility; below we only discuss the module-externally-visible scenario.

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding, deleting, or modifying annotations | Compatible | Compatible |
| Adding the @Deprecated compile marker | Based on the @Deprecated parameter conditions, when the static member function is unavailable in downstream modules, incompatible; otherwise, compatible | Based on the @Deprecated parameter conditions, when the static member function is unavailable in downstream modules, incompatible; otherwise, compatible |
| Deleting the @Deprecated compile marker | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the static member function changes from available to unavailable in downstream modules | Incompatible | Incompatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the static member function changes from unavailable to available in downstream modules | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the static member function's availability in downstream modules remains unchanged | Compatible | Compatible |
| Adding the @Frozen compile marker | Compatible | Compatible |
| Deleting the @Frozen compile marker | Compatible | Incompatible |
| Access modifier changed from public to protected | If the extended type is a class type, incompatible; if the extended type is another non-class type, compatible | If the extended type is a class type, incompatible; if the extended type is another non-class type, compatible |
| Access modifier changed from public to internal/private | Incompatible | Incompatible |
| Access modifier changed from protected to public | Compatible | Compatible |
| Access modifier changed from protected to internal/private |  If the extended type is a class type, incompatible; if the extended type is another non-class type, compatible | If the extended type is a class type, incompatible; if the extended type is another non-class type, compatible |
| Access modifier changed from internal/private to public/protected | Compatible | Compatible |
| Access modifier changed from internal to private | Compatible | Compatible |
| Access modifier changed from private to internal | Compatible | Compatible |
| Adding the unsafe modifier | Incompatible | Incompatible |
| Deleting the unsafe modifier | Compatible | Compatible |
| Adding the const modifier | Compatible | Compatible |
| Deleting the const modifier | Incompatible | Incompatible |
| Deleting the static modifier | Incompatible | Incompatible |
| Modifying the function name | Incompatible | Incompatible |
| Adding a generic parameter | Incompatible | Incompatible |
| Deleting a generic parameter | Incompatible | Incompatible |
| Modifying the name of a generic parameter | Compatible | Compatible |
| Modifying the order of generic parameters | Incompatible | Incompatible |
| Adding a constraint upper bound to a generic parameter | Incompatible | Incompatible |
| Deleting a certain upper bound constraint of a generic parameter | Compatible | Incompatible |
| Modifying a certain upper bound constraint A of a generic parameter to B | When B is a parent type of A, compatible; otherwise, incompatible | Incompatible |
| Modifying the order among generic parameter constraints or among constraint upper bounds | Compatible | Compatible |
| Adding a function parameter | Incompatible | Incompatible |
| Deleting a function parameter | Incompatible | Incompatible |
| Modifying the name of a non-named parameter | Compatible | Compatible |
| Modifying the name of a named parameter | Incompatible | Incompatible |
| Changing a non-named parameter to a named parameter | Incompatible | Incompatible |
| Changing a named parameter to a non-named parameter | Incompatible | Incompatible |
| Modifying the order of non-named parameters | Incompatible | Incompatible |
| Modifying the order of named parameters | Compatible | Incompatible |
| Modifying the type of a parameter from A to B | When B is a parent type of A, compatible; otherwise, incompatible | Incompatible |
| Adding a default value to a parameter | Compatible | Compatible |
| Deleting an existing default value of a parameter | Incompatible | Incompatible |
| Modifying the default value of a parameter of a const function or a function marked with @Frozen | Compatible | Incompatible |
| Modifying the default value of a parameter of a non-const function not marked with @Frozen | Compatible | Compatible |
| Modifying the return type from A to B | When B is a subtype of A, compatible; otherwise, incompatible | When B is a subtype of A, and both A and B are class or interface types, compatible; otherwise, incompatible |
| Modifying the function body of a const function or a function marked with @Frozen | Compatible | Incompatible |
| Modifying the function body of a non-const function not marked with @Frozen | Compatible | Compatible |

###### Modifying the Order of Static Member Functions

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Modifying the order of static member functions | Compatible | Compatible |

##### Modifying Static Properties

###### Adding and Deleting Static Properties

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding a static property | Compatible | Adding a re-implemented static property, incompatible; otherwise, compatible |
| Deleting a static property with the public modifier | Incompatible | Incompatible |
| Deleting a static property with the protected modifier | If the extended type is a class type, incompatible; if the extended type is another non-class type, compatible | If the extended type is a class type, incompatible; if the extended type is another non-class type, compatible |
| Deleting a static property without the public/protected modifier | Compatible | Compatible |

###### Modifying the Definition of a Static Property

For an exported extend, we call its static property **visible outside the module** when it satisfies any of the following conditions:

1. The extended type is a class type, and the static property has the public/protected modifier
2. The extended type is a non-class type, and the static property has the public modifier
3. The static property is depended upon by other module-externally-visible declarations, and these declarations are marked with const or @Frozen.

Modifications to any static property not visible outside the module satisfy API + ABI compatibility; below we only discuss the module-externally-visible scenario.

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding, deleting, or modifying annotations | Compatible | Compatible |
| Adding the @Deprecated compile marker | Based on the @Deprecated parameter conditions, when the static property is unavailable in downstream modules, incompatible; otherwise, compatible | Based on the @Deprecated parameter conditions, when the static property is unavailable in downstream modules, incompatible; otherwise, compatible |
| Deleting the @Deprecated compile marker | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the static property changes from available to unavailable in downstream modules | Incompatible | Incompatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the static property changes from unavailable to available in downstream modules | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the static property's availability in downstream modules remains unchanged | Compatible | Compatible |
| Adding the @Frozen compile marker | Compatible | Compatible |
| Deleting the @Frozen compile marker | Compatible | Incompatible |
| Access modifier changed from public to protected | If the extended type is a class type, incompatible; if the extended type is another non-class type, compatible | If the extended type is a class type, incompatible; if the extended type is another non-class type, compatible |
| Access modifier changed from public to internal/private | Incompatible | Incompatible |
| Access modifier changed from protected to public | Compatible | Compatible |
| Access modifier changed from protected to internal/private |  If the extended type is a class type, incompatible; if the extended type is another non-class type, compatible | If the extended type is a class type, incompatible; if the extended type is another non-class type, compatible |
| Access modifier changed from internal/private to public/protected | Compatible | Compatible |
| Access modifier changed from internal to private | Compatible | Compatible |
| Access modifier changed from private to internal | Compatible | Compatible |
| Deleting the static modifier | Incompatible | Incompatible |
| Adding the mut modifier and the corresponding setter function | Compatible | Compatible |
| Deleting the mut modifier and the corresponding setter function | Incompatible | Incompatible |
| Modifying the property name | Incompatible | Incompatible |
| Modifying the property type | Incompatible | Incompatible |
| Modifying the getter/setter functions of a property not marked with @Frozen | Compatible | Compatible |
| Modifying the getter/setter functions of a property marked with @Frozen | Compatible | Incompatible |

###### Modifying the Order of Static Properties

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Modifying the order of static properties | Compatible | Compatible |

##### Modifying Instance Member Functions

###### When the Extended Type is a non-class Type

**Adding and Deleting Instance Member Functions**

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding an instance member function | Compatible | Adding a re-implemented instance member function, incompatible; otherwise, compatible |
| Deleting a public instance member function | Incompatible | Incompatible |
| Deleting a non-public instance member function | Compatible | Compatible |

**Modifying the Definition of an Instance Member Function**

For an exported extend, we call its instance member function **visible outside the module** when it satisfies any of the following conditions:

1. The instance member function has the public modifier.
2. The instance member function is depended upon by other module-externally-visible declarations, and these declarations are marked with const or @Frozen.

Modifications to any instance member function not visible outside the module satisfy API + ABI compatibility; below we only discuss the module-externally-visible scenario.

| Change Behavior | API Compatibility | ABI Compatibility | Note |
|-|-|-|-|
| Adding, deleting, or modifying annotations | Compatible | Compatible | NA |
| Adding the @Deprecated compile marker | Based on the @Deprecated parameter conditions, when the instance member function is unavailable in downstream modules, incompatible; otherwise, compatible | Based on the @Deprecated parameter conditions, when the instance member function is unavailable in downstream modules, incompatible; otherwise, compatible | NA |
| Deleting the @Deprecated compile marker | Compatible | Compatible | NA |
| Modifying the @Deprecated compile marker parameter conditions, such that the instance member function changes from available to unavailable in downstream modules | Incompatible | Incompatible | NA |
| Modifying the @Deprecated compile marker parameter conditions, such that the instance member function changes from unavailable to available in downstream modules | Compatible | Compatible | NA |
| Modifying the @Deprecated compile marker parameter conditions, such that the instance member function's availability in downstream modules remains unchanged | Compatible | Compatible | NA |
| Adding the @Frozen compile marker | Compatible | Compatible | NA |
| Deleting the @Frozen compile marker | Compatible | Incompatible | NA |
| Access modifier changed from public to non-public | Incompatible | Incompatible | NA |
| Access modifier changed from non-public to public | Compatible | Compatible | NA |
| Access modifier changed among the other three non-public cases (protected/internal/private) | Compatible | Compatible | NA |
| Adding the static modifier | Incompatible | Incompatible | NA |
| Adding the unsafe modifier | Incompatible | Incompatible | NA |
| Deleting the unsafe modifier | Compatible | Compatible | NA |
| Adding the const modifier | Compatible | Compatible | NA |
| Deleting the const modifier | Incompatible | Incompatible | NA |
| Adding the mut modifier | Incompatible | Incompatible | NA |
| Deleting the mut modifier | Compatible | Incompatible | NA |
| Modifying the function name | Incompatible | Incompatible | NA |
| Adding a generic parameter | Incompatible | Incompatible | NA |
| Deleting a generic parameter | Incompatible | Incompatible | NA |
| Modifying the name of a generic parameter | Compatible | Compatible | NA |
| Modifying the order of generic parameters | Incompatible | Incompatible | NA |
| Adding a constraint upper bound to a generic parameter | Incompatible | Incompatible | NA |
| Deleting a certain upper bound constraint of a generic parameter | Compatible | Incompatible | NA |
| Modifying a certain upper bound constraint A of a generic parameter to B | When B is a parent type of A, compatible; otherwise, incompatible | Incompatible | NA |
| Modifying the order among generic parameter constraints or among constraint upper bounds | Compatible | Compatible | NA |
| Adding a function parameter | Incompatible | Incompatible | Even if the added function parameter is at the end and has a default value, it is still API incompatible, because Cangjie supports functions as first-class citizens, and the function type changes |
| Deleting a function parameter | Incompatible | Incompatible | NA |
| Modifying the name of a non-named parameter | Compatible | Compatible | NA |
| Modifying the name of a named parameter | Incompatible | Incompatible | NA |
| Changing a non-named parameter to a named parameter | Incompatible | Incompatible | The call site of a named parameter needs to use the parameter name, so it will cause a compilation error |
| Changing a named parameter to a non-named parameter | Incompatible | Incompatible | The call site of a named parameter needs to use the parameter name, so it will cause a compilation error |
| Modifying the order of non-named parameters | Incompatible | Incompatible | NA |
| Modifying the order of named parameters | Compatible | Incompatible | NA |
| Modifying the type of a parameter from A to B | When B is a parent type of A, compatible; otherwise, incompatible | Incompatible | NA |
| Adding a default value to a parameter | Compatible | Compatible | NA |
| Deleting an existing default value of a parameter | Incompatible | Incompatible | NA |
| Modifying the default value of a parameter of a const function or a function marked with @Frozen | Compatible | Incompatible | NA |
| Modifying the default value of a parameter of a non-const function not marked with @Frozen | Compatible | Compatible | NA |
| Modifying the return type from A to B | When B is a subtype of A, compatible; otherwise, incompatible | When B is a subtype of A, and both A and B are class or interface types, compatible; otherwise, incompatible | NA |
| Modifying the function body of a const function or a function marked with @Frozen | Compatible | Incompatible | NA |
| Modifying the function body of a non-const function not marked with @Frozen | Compatible | Compatible | NA |

**Modifying the Order of Instance Member Functions**

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Modifying the order of instance member functions | Compatible | Compatible |

###### When the Extended Type is a class Type

**Adding and Deleting Instance Member Functions**

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding an instance member function | Compatible | Adding a re-implemented instance member function, incompatible; otherwise, compatible |
| Deleting a public/protected instance member function | Incompatible | Incompatible |
| Deleting a non-public/protected instance member function | Compatible | Compatible |

**Modifying the Definition of an Instance Member Function**

For an exported extend, we call its instance member function visible outside the module when it satisfies any of the following conditions:

1. The instance member function has the public/protected modifier.
2. The instance member function is depended upon by other module-externally-visible declarations, and these declarations are marked with const or @Frozen.

Modifications to any instance member function not visible outside the module satisfy API + ABI compatibility; below we only discuss the module-externally-visible scenario.

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding, deleting, or modifying annotations | Compatible | Compatible |
| Adding the @Deprecated compile marker | Based on the @Deprecated parameter conditions, when the instance member function is unavailable in downstream modules, incompatible; otherwise, compatible | Based on the @Deprecated parameter conditions, when the instance member function is unavailable in downstream modules, incompatible; otherwise, compatible |
| Deleting the @Deprecated compile marker | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the instance member function changes from available to unavailable in downstream modules | Incompatible | Incompatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the instance member function changes from unavailable to available in downstream modules | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the instance member function's availability in downstream modules remains unchanged | Compatible | Compatible |
| Adding the @Frozen compile marker | Compatible | Compatible |
| Deleting the @Frozen compile marker | Compatible | Incompatible |
| Access modifier changed from public/protected to internal/private | Incompatible | Incompatible |
| Access modifier changed from internal/private to public/protected | Compatible | Compatible |
| Access modifier changed from public to protected | Incompatible | Incompatible |
| Access modifier changed from protected to public | Compatible | Compatible |
| Access modifier changed from internal to private | Compatible | Compatible |
| Access modifier changed from private to internal | Compatible | Compatible |
| Adding the static modifier | Incompatible | Incompatible |
| Adding the unsafe modifier | Incompatible | Incompatible |
| Deleting the unsafe modifier | Compatible | Compatible |
| Adding the const modifier | Compatible | Incompatible |
| Deleting the const modifier | Incompatible | Incompatible |
| Modifying the function name | Incompatible | Incompatible |
| Adding a generic parameter | Incompatible | Incompatible |
| Deleting a generic parameter | Incompatible | Incompatible |
| Modifying the name of a generic parameter | Compatible | Compatible |
| Modifying the order of generic parameters | Incompatible | Incompatible |
| Adding a constraint upper bound to a generic parameter | Incompatible | Incompatible |
| Deleting a certain upper bound constraint of a generic parameter | Compatible | Incompatible |
| Modifying a certain upper bound constraint A of a generic parameter to B | When B is a parent type of A, compatible; otherwise, incompatible | Incompatible |
| Modifying the order among generic parameter constraints or among constraint upper bounds | Compatible | Compatible |
| Adding a function parameter | Incompatible | Incompatible |
| Deleting a function parameter | Incompatible | Incompatible |
| Modifying the name of a non-named parameter | Compatible | Compatible |
| Modifying the name of a named parameter | Incompatible | Incompatible |
| Changing a non-named parameter to a named parameter | Incompatible | Incompatible |
| Changing a named parameter to a non-named parameter | Incompatible | Incompatible |
| Modifying the order of non-named parameters | Incompatible | Incompatible |
| Modifying the order of named parameters | Compatible | Incompatible |
| Modifying the type of a parameter from A to B | When B is a parent type of A, compatible; otherwise, incompatible | Incompatible |
| Adding a default value to a parameter | Compatible | Compatible |
| Deleting an existing default value of a parameter | Incompatible | Incompatible |
| Modifying the default value of a parameter of a const function or a function marked with @Frozen | Compatible | Incompatible |
| Modifying the default value of a parameter of a non-const function not marked with @Frozen | Compatible | Compatible |
| Modifying the return type from A to B | When B is a subtype of A, compatible; otherwise, incompatible | When B is a subtype of A, and both A and B are class or interface types, compatible; otherwise, incompatible |
| Modifying the function body of a const function or a function marked with @Frozen | Compatible | Incompatible |
| Modifying the function body of a non-const function not marked with @Frozen | Compatible | Compatible |

**Modifying the Order of Instance Member Functions**

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Modifying the order of instance member functions | Compatible | Compatible |

##### Modifying Instance Properties

###### When the Extended Type is a non-class Type

**Adding and Deleting Instance Properties**

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding an instance property | Compatible | Adding a re-implemented instance property, incompatible; otherwise, compatible |
| Deleting a public instance property | Incompatible | Incompatible |
| Deleting a non-public instance property | Compatible | Compatible |

**Modifying the Definition of an Instance Property**

For an exported extend, we call its instance property **visible outside the module** when it satisfies any of the following conditions:

1. The instance property has the public modifier.
2. The instance property is depended upon by other module-externally-visible declarations, and these declarations are marked with const or @Frozen.

Modifications to any instance property not visible outside the module satisfy API + ABI compatibility; below we only discuss the module-externally-visible scenario.

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding, deleting, or modifying annotations | Compatible | Compatible |
| Adding the @Deprecated compile marker | Based on the @Deprecated parameter conditions, when the instance property is unavailable in downstream modules, incompatible; otherwise, compatible | Based on the @Deprecated parameter conditions, when the instance property is unavailable in downstream modules, incompatible; otherwise, compatible |
| Deleting the @Deprecated compile marker | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the instance property changes from available to unavailable in downstream modules | Incompatible | Incompatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the instance property changes from unavailable to available in downstream modules | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the instance property's availability in downstream modules remains unchanged | Compatible | Compatible |
| Adding the @Frozen compile marker | Compatible | Compatible |
| Deleting the @Frozen compile marker | Compatible | Incompatible |
| Access modifier changed from public to non-public | Incompatible | Incompatible |
| Access modifier changed from non-public to public | Compatible | Compatible |
| Access modifier changed among the other three non-public cases (protected/internal/private) | Compatible | Compatible |
| Adding the static modifier | Incompatible | Incompatible |
| Adding the mut modifier and the corresponding setter function | Incompatible | Incompatible |
| Deleting the mut modifier and the corresponding setter function | Incompatible | Incompatible |
| Modifying the property name | Incompatible | Incompatible |
| Modifying the property type | Incompatible | Incompatible |
| Modifying the getter/setter functions of a property not marked with @Frozen | Compatible | Compatible |
| Modifying the getter/setter functions of a property marked with @Frozen | Compatible | Incompatible |

**Modifying the Order of Instance Properties**

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Modifying the order of instance properties | Compatible | Compatible |

###### When the Extended Type is a class Type

**Adding and Deleting Instance Properties**

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding an instance property | Compatible | Adding a re-implemented instance property, incompatible; otherwise, compatible |
| Deleting a public/protected instance property | Incompatible | Incompatible |
| Deleting a non-public/protected instance property | Compatible | Compatible |

**Modifying the Definition of an Instance Property**

For an exported extend, we call its instance property visible outside the module when it satisfies any of the following conditions:

1. The instance property has the public/protected modifier.
2. The instance property is depended upon by other module-externally-visible declarations, and these declarations are marked with const or @Frozen.

Modifications to any instance property not visible outside the module satisfy API + ABI compatibility; below we only discuss the module-externally-visible scenario.

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding, deleting, or modifying annotations | Compatible | Compatible |
| Adding the @Deprecated compile marker | Based on the @Deprecated parameter conditions, when the instance property is unavailable in downstream modules, incompatible; otherwise, compatible | Based on the @Deprecated parameter conditions, when the instance property is unavailable in downstream modules, incompatible; otherwise, compatible |
| Deleting the @Deprecated compile marker | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the instance property changes from available to unavailable in downstream modules | Incompatible | Incompatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the instance property changes from unavailable to available in downstream modules | Compatible | Compatible |
| Modifying the @Deprecated compile marker parameter conditions, such that the instance property's availability in downstream modules remains unchanged | Compatible | Compatible |
| Adding the @Frozen compile marker | Compatible | Compatible |
| Deleting the @Frozen compile marker | Compatible | Incompatible |
| Access modifier changed from public/protected to internal/private | Incompatible | Incompatible |
| Access modifier changed from internal/private to public/protected | Compatible | Compatible |
| Access modifier changed from public to protected | Incompatible | Incompatible |
| Access modifier changed from protected to public | Compatible | Compatible |
| Access modifier changed from internal to private | Compatible | Compatible |
| Access modifier changed from private to internal | Compatible | Compatible |
| Adding the static modifier | Incompatible | Incompatible |
| Adding the mut modifier and the corresponding setter function | Compatible | Compatible |
| Deleting the mut modifier and the corresponding setter function | Incompatible | Incompatible |
| Modifying the property name | Incompatible | Incompatible |
| Modifying the property type | Incompatible | Incompatible |
| Modifying the getter/setter functions of a property not marked with @Frozen | Compatible | Compatible |
| Modifying the getter/setter functions of a property marked with @Frozen | Compatible | Incompatible |

**Modifying the Order of Instance Properties**

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Modifying the order of instance properties | Compatible | Compatible |

#### Modifying the Definition Location of an extend

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Modifying the file where an extend is defined (note: including renaming the file and moving to another file) | Compatible | Compatible |
| Modifying the position of an extend definition within the file | Compatible | Compatible |

## Compatibility Rules for import Statement Changes

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding an import statement | Compatible | Incompatible |
| Deleting an import statement | When the deleted import statement is a non-public import, compatible; otherwise, incompatible | Incompatible |
| Modifying the module or package name imported by the import statement | When the modified import statement is a non-public import, compatible; otherwise, incompatible | Incompatible |
| Modifying the declaration content imported by the import statement | When the modified import statement is a non-public import, compatible; otherwise, incompatible | When the modified import statement is a non-public import, compatible; otherwise, incompatible |

## Compatibility Rules for typeAlias Statement Changes

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding a typeAlias statement | Compatible | Compatible |
| Deleting a typeAlias statement with the public modifier | Incompatible | Incompatible |
| Deleting a typeAlias statement without the public modifier | Compatible | Compatible |
| Modifying a typeAlias statement with the public modifier | Incompatible | Incompatible |
| Modifying a typeAlias statement without the public modifier | Compatible | Compatible |

## Compatibility Rules for package Declaration Statement Changes

| Change Behavior | API Compatibility | ABI Compatibility |
|-|-|-|
| Adding, deleting, and modifying package declaration statements | Incompatible | Incompatible |

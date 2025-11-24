# Rules for Converting C Language to Cangjie Glue Code

HLE automatically generates glue code from C to Cangjie, supporting the translation of functions, structures, enums, and global variables. Type support includes: basic types, structure types, pointers, arrays, and strings.

### Basic Types

The tool supports the following basic types:

| C Type             | Cangjie Type     |
| -------------------- | ------------------ |
| void               | unit             |
| NULL               | CPointer         |
| bool               | Bool             |
| char               | UInt8            |
| signed char        | Int8             |
| unsigned char      | UInt8            |
| short              | Int64            |
| int                | Int32            |
| unsigned int       | UInt32           |
| long               | Int64            |
| unsigned long      | UInt64           |
| long long          | Int64            |
| unsigned long long | UInt64           |
| float              | Float32          |
| double             | Float64          |
| int arr[10]        | Varry            |

### Complex Types

The tool supports complex types including: struct types, pointer types, enum types, strings, and arrays.

#### Struct Types

.h declaration file:

```c
struct Point {
    struct {
        int x;
        int y;
    };
    int z;
};

struct person {
    int age;
};

typedef struct {
    long long x;
    long long y;
    long long z;
} Point3D;
```

The corresponding generated glue code is as follows:

```cangjie
@C
public struct _cjbind_ty_1 {
    public let x: Int32
    public let y: Int32

    public init(x: Int32, y: Int32) {
        this.x = x
        this.y = y
    }
}

@C
public struct Point {
    // todo 实现有问题
    public let __cjbind_anon_1: _cjbind_ty_1
    public let z: Int32

    public init(__cjbind_anon_1: _cjbind_ty_1, __cjbind_anon_2: _cjbind_ty_1, z: Int32, __cjbind_anon_3: _cjbind_ty_1) {
        this.__cjbind_anon_1 = __cjbind_anon_1
        this.z = z
    }
}

@C
public struct person {
    public let age: Int32

    public init(age: Int32) {
        this.age = age
    }
}

@C
public struct Point3D {
    public let x: Int64
    public let y: Int64
    public let z: Int64

    public init(x: Int64, y: Int64, z: Int64) {
        this.x = x
        this.y = y
        this.z = z
    }
}
```

#### Pointer Types

.h declaration file:

```c
void* testPointer(int a);
```

The generated glue code is as follows:

```cangjie
foreign func testPointer(a: Int32): CPointer
```

#### Function Types

.h declaration file:

```c
void test(int a);
```

The corresponding generated glue code is as follows:

```cangjie
foreign func test(a: Int32): Unit
```


#### Enumeration Types

.h declaration file:

```c
enum Color {
RED,
GREEN,
BLUE = 5,
YELLOW
};
```

The generated glue code is as follows:

```cangjie
public const Color_RED: Color = 0
public const Color_GREEN: Color = 1
public const Color_BLUE: Color = 5
public const Color_YELLOW: Color = 6

public type Color = UInt32
```

#### String

.h declaration file:

```c
void test(char* a);
```

The corresponding generated glue code is as follows:

```cangjie
foreign func test(a: CString): Unit
```

#### Array Type

.h declaration file:

```c
void test(int arr[3]);
```

The corresponding generated glue code is as follows:

```cangjie
foreign func test(arr: VArray<Int32, $3>): Unit
```

### Unsupported Specifications

Unsupported specifications include: bit fields, unions, macros, opaque types, flexible arrays, extended types

#### Bit Fields

.h declaration file:

```c
struct X {
    unsigned int isPowerOn : 1;
    unsigned int hasError : 1;
    unsigned int mode : 2;
    unsigned int reserved : 4;
};
```

The generated corresponding Cangjie code is as follows, and the user needs to manually correct it:

```cangjie
@C
public struct X {
    let _cjbind_opaque_blob: UInt32

    public init() {
        this._cjbind_opaque_blob = unsafe { zeroValue<UInt32>() }
    }
}
```

#### Union

.h declaration file:

```c
union X {
    int a;
    void* ptr;
};
```

The generated corresponding Cangjie code is as follows, and the user needs to manually correct it:

```cangjie
@C
public struct X {
    let _cjbind_opaque_blob: UInt64

    public init() {
        this._cjbind_opaque_blob = unsafe { zeroValue<UInt64>() }
    }
}
```

#### Macros

Currently, Cangjie does not have a suitable expression parsing library, so it cannot directly compute the value of macros. When encountering a macro, the current implementation will skip the entire #define.

#### Opaque Types

.h declaration file:

```c
typedef struct OpaqueType OpaqueType;

OpaqueType* create_opaque(int initial_value);
void set_value(OpaqueType* obj, int value);
int get_value(OpaqueType* obj);
void destroy_opaque(OpaqueType* obj);
```

The generated corresponding Cangjie code is as follows, and the user needs to manually correct it:

```cangjie
@C
public struct OpaqueType {
    init() {
        throw Exception("type should not be inited")
    }
}

foreign func create_opaque(initial_value: Int32): CPointer<OpaqueType>

foreign func set_value(obj: CPointer<OpaqueType>, value: Int32): Unit

foreign func get_value(obj: CPointer<OpaqueType>): Int32

foreign func destroy_opaque(obj: CPointer<OpaqueType>): Unit
```

#### Flexible Arrays

.h declaration file:

```c
typedef struct {
    int length;
    char data[];
} FlexibleString;
```

The generated corresponding Cangjie code is as follows, and the user needs to manually correct it:

```cangjie
@C
public struct OpaqueType {
    init() {
        throw Exception("type should not be inited")
    }
}

foreign func create_opaque(initial_value: Int32): CPointer<OpaqueType>

foreign func set_value(obj: CPointer<OpaqueType>, value: Int32): Unit

foreign func get_value(obj: CPointer<OpaqueType>): Int32

foreign func destroy_opaque(obj: CPointer<OpaqueType>): Unit
```

#### Extension Types

.h declaration file:

```c
#include <complex.h> 
#include <stdatomic.h>

float _Complex c_float; 
double _Complex c_double;
long double _Complex c_ld; 

long double pi_high = 3.14159265358979323846264338327950288L;
long double Planck_constant = 6.62607015e-34L; 

_Atomic(int) counter = 0;
```

The generated corresponding Cangjie code is as follows, and the user needs to manually correct it:

```cangjie
public const pi_high: Float64 = 3.141592653589793
public const Planck_constant: Float64 = 6.62607015e-34
public const counter: Int32 = 0i64
```

> Explanation:
>
> 1. Memory Alignment: Cangjie does not provide syntax for alignment control, so the HLE tool uses the default C alignment. If the C code uses `#pragma pack` or `__attribute__((packed))` to control alignment, the generated binding code cannot be guaranteed to be correct.
> 2. Calling Conventions: The Cangjie documentation does not clearly describe calling conventions. In fact, the default calling convention is used. Currently, the HLE tool will try to infer the calling convention based on the function signatures in the C code, but correctness cannot be guaranteed.
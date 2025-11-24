# C语言转换到仓颉胶水代码的规则

HLE自动生成c到仓颉的胶水代码，支持函数、结构体、枚举和全局变量的翻译，类型支持：基础类型、结构体类型、指针、数组和字符串。 

### 基础类型

工具支持以下几种基础类型：

| C Type             | cangjie Type     |
| -------------------- | ------------------ |
| void               | unit             |
| NULL               | CPointer   |
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
| int arr[10]        | Varry |

### 复杂类型

工具支持的复杂类型有：struct类型、指针类型、枚举类型、字符串、数组。

#### struct类型

.h声明文件：

```
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

对应生成的胶水代码如下：

```
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

#### 指针类型

.h声明文件：

```
void* testPointer(int a);
```

对应生成的胶水代码如下：

```
foreign func testPointer(a: Int32): CPointer
```

#### 函数类型

.h声明文件：

```
void test(int a);
```

对应生成的胶水代码如下：

```
foreign func test(a: Int32): Unit
```


#### 枚举类型

.h声明文件：

```
enum Color {
RED,
GREEN,
BLUE = 5,
YELLOW
};
```

对应生成的胶水代码如下：

```
public const Color_RED: Color = 0
public const Color_GREEN: Color = 1
public const Color_BLUE: Color = 5
public const Color_YELLOW: Color = 6

public type Color = UInt32
```

#### 字符串

.h声明文件：

```
void test(char* a);
```

对应生成的胶水代码如下：

```
foreign func test(a: CString): Unit
```

#### 数组类型

.h声明文件：

```
void test(int arr[3]);
```

对应生成的胶水代码如下：

```
foreign func test(arr: VArray<Int32, $3>): Unit
```

### 不支持的规格

不支持的规格有：位域、联合体、宏、不透明类型、柔性数组、扩展类型

#### 位域

.h声明文件：

```
struct X {
    unsigned int isPowerOn : 1;
    unsigned int hasError : 1;
    unsigned int mode : 2;
    unsigned int reserved : 4;
};
```

#### 联合体

.h声明文件：

```
union X {
    int a;
    void* ptr;
};
```

#### 宏

仓颉目前没有合适的表达式解析库，因此无法直接计算宏的值。在遇到宏时，当前会跳过整个 #define。

#### 不透明类型

.h声明文件：

```
typedef struct OpaqueType OpaqueType;

OpaqueType* create_opaque(int initial_value);
void set_value(OpaqueType* obj, int value);
int get_value(OpaqueType* obj);
void destroy_opaque(OpaqueType* obj);
```

#### 柔性数组

.h声明文件：

```
typedef struct {
    int length;
    char data[];
} FlexibleString;
```

#### 扩展类型

.h声明文件：

```
#include <complex.h> // C 标准复数头文件
#include <stdatomic.h>

float _Complex c_float; // float 复数类型
double _Complex c_double; // double 复数类型
long double _Complex c_ld; // long double 复数类型

long double pi_high = 3.14159265358979323846264338327950288L; // 后缀 L 表示 long double 字面量
long double Planck_constant = 6.62607015e-34L; // 普朗克常数（高精度需求）

// 用 _Atomic 关键字声明原子类型（int 的原子封装）
_Atomic(int) counter = 0;
```

> 说明：
> 
> 1. 内存对齐：仓颉没有提供对齐控制的语法，因此 HLE工具使用 C 默认的对齐方式。如果 C 代码中使用了 `#pragma pack` 或者 `__attribute__((packed))` 等方式来控制对齐，生成的绑定代码不保证正确性。
> 2. 调用约定：仓颉文档中对于调用约定描述不清晰，其实是采用了默认的调用约定，目前HLE工具会尝试根据C代码中的函数签名推断调用约定，但是不保证正确性。> 说明：
> 
> 1. 内存对齐：仓颉没有提供对齐控制的语法，因此 HLE工具使用 C 默认的对齐方式。如果 C 代码中使用了 `#pragma pack` 或者 `__attribute__((packed))` 等方式来控制对齐，生成的绑定代码不保证正确性。
> 2. 调用约定：仓颉文档中对于调用约定描述不清晰，其实是采用了默认的调用约定，目前HLE工具会尝试根据C代码中的函数签名推断调用约定，但是不保证正确性。
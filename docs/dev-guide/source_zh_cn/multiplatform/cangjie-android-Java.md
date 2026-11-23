# 仓颉-Java 互操作

> **注意**：
>
> Java 互操作特性为实验特性，尚在持续完善中。

仓颉跨平台方案支持开发者将仓颉语言接入 Android 应用开发，无论是项目中待实现的新逻辑，还是已存在的存量逻辑，都可通过仓颉语言完成开发与适配。

镜像类型是仓颉跨平台实现跨语言、跨运行时互操作的核心机制。它允许一门语言中定义的类型向另一门语言暴露接口，进而实现该类型在不同语言环境中的直接使用。

在仓颉侧，镜像类型允许仓颉 `class` 在遵循仓颉语法和语义的前提下，继承 Java `class`、实现 Java `interface`。而这个仓颉 `class` 通过镜像类型反向暴露给 Java 侧后，如同本身就是 Java 实现。而在 Java 侧，镜像类型同样能够使得仓颉类型以 Java 类型表示出来。总体来说，仓颉跨平台让仓颉和 Java 在 Android 应用工程中做到尽可能无缝衔接，同时也意味着，开发者可以在仓颉代码中，通过跨语言互操作调用 Android 操作系统提供的 API。

## 互操作实现思路与底层机制

仓颉和 Java 虽然都是支持继承和多态的面向对象语言，但其各自的语义、底层实现的对象模型和执行模型等却存在显著差异，因此，试图在 Java 代码中直接使用仓颉语言，或反之在仓颉代码中直接使用 Java，均无法实现。

两种语言均各自拥有各自独立的托管运行时，自动内存管理、线程模型、异常处理等底层特性各不相同。让两个复杂编程语言的运行时通过相互感知来实现互操作，无疑会让整个应用的复杂度剧增。

因此，仓颉跨平台对于仓颉与 Java 的互操作的实现思路是分别站在仓颉和 Java 侧，均将另一方视作低级语言。具体来说，仓颉与 Java 通过 Java 本地接口（ JNI ）实现互操作。JNI 可让 Java 调用 C/C++ 开发的本地接口，功能强大，但作为底层 API，手动编写绑定层费时费力，CJMP 提供相应工具链，能够有效降低使用复杂度。

## Android 版本支持

仓颉 Android SDK 支持 Android 6（API 23）及更高系统。若代码使用 Java 8+ 特性或接口，项目构建脚本设置的最低 Android API 版本需兼容对应能力，详情见下文：

Android 工具链通过一种称为脱糖（desugaring）的字节码转换技术，在旧版 Android 上支持新的 Java 语言特性和 API。D8 编译器在将 Java 类文件转换为可在 Android 设备上运行的 DEX 代码时会执行这些转换。

然而，这些较新的特性和 API 如果经过了脱糖处理，则无法在运行时供仓颉代码使用。

需注意，接口中的 `static` 方法和 `default` 方法在 Android 6（API 23）上不受支持。因此，如果在主 Gradle 构建脚本中将 `minSdk` 设置为 `23`，则仓颉代码中不得使用此类方法。

同样地，除非应用仅面向那些支持新 Java 平台 API（无需脱糖）的 Android 版本，否则这些 API 也无法使用。如果仓颉代码中使用了脱糖后的 API，在任何 Android 版本上均将导致运行时错误。

例如，在 Android 6 到 Android 8.1（API 级别 23-27）上，仓颉 Android SDK 不支持 Java 8+ 平台 API，如 Stream API 和 `java.util.Optional`。如果仓颉代码使用了这些 API，则必须在 Gradle 构建脚本中将 `minSdk` 设置为 `28` 或更高版本。

有关各 Java 8+ 特性与 API 在目标 Android 版本上的可用性，请参阅 Android 开发者文档。

## 核心概念

### 镜像类型

可通过以下方式理解镜像类型：仓颉和 Java 两种语言之间进行互操作，若一种语言 A 的源码中定义有镜像类型 `T'`，则意味着在另一种语言 B 的源码中实际存在由 B 语言定义的类型 `T`。于是，在语言 A 的源码中就可以通过直接使用镜像类型 `T'` 来实现间接使用类型 `T`，最终实现语言 A 仿佛直接使用语言 B 的类型的效果。该操作存在特定限制，将在下文中详细说明。

诸如布尔类型和数值类型等两种语言之间本质上等价的类型本身就是相互的镜像类型，例如， Java 视角下，其 `int` 类型就是仓颉 `Int32` 类型在 Java 侧的镜像类型；反过来，仓颉视角下，其 `Int32` 类型就是 Java `int` 类型在仓颉侧的镜像类型。不过，对于部分无法建立对应关系的数值类型来说，这种镜像关系并不存在，例如仓颉的 `Float16` 在 Java 侧就没有任何类型能够与之对应，故在 Java 视角下就不存在一种镜像类型来匹配仓颉的 `Float16` 类型，也可以理解为，仓颉的 `Float16` 类型无法被镜像为任何 Java 基本类型。

对于 `class`、`struct`、`interface` 和 `enum` 等用户自定义类型，语言 A 中的类型 `T` 在另一门语言 B 中的镜像类型 `T'`，是在语言 B 中所能找到的最接近的等价类型。举例来说，仓颉的 `struct` 或元组类型在 Java 中所能找到的最佳等价类型是 Java 的 `final class` 类型。

若要在语言 B 中通过镜像类型使用语言 A 定义的类型，该镜像类型仅会暴露语言 A 的类型中“语言 B 理论上可访问和调用”的成员与构造函数。举例来说：若某个仓颉成员函数的返回类型为 `Float16`，由于 `Float16` 无法镜像为 Java 类型，该仓颉成员函数也无法生成对应的镜像，导致 Java 侧无法通过镜像类型调用此函数，这类场景需根据实际情况采用特定方法解决。

正常情况下，无论是仓颉类型的镜像类型还是 Java 类型的镜像类型，以及镜像类型本身依赖的其他类型的镜像类型，都能够以某种方式自动生成获得。 CJMP 提供了独立的工具—— Java 镜像生成器，来实现为 Java 类型自动生成镜像类型；为仓颉类型生成镜像类型也同样是自动完成的，加上特定编译选项的 cjc 编译过程会将仓颉类型的镜像类型定义作为副产品生成，具体步骤将在本文档中详细解释。

**将 Java 类型镜像为仓颉类型：**

 cjc 在编译过程中会将所有仓颉源码中用到的 Java 镜像类型替换为相应的胶水代码，这意味着，真正对编译结果起作用的核心信息只有两点：一是被使用的 Java 镜像类型的名称，二是该镜像类型中各可用成员的名称及其类型。因此在编写仓颉代码时，Java 镜像类型定义中只需要包含各个可用成员的声明就够了，换句话说，Java 镜像类型中并不需要保留构造函数体、成员函数体和成员属性体，成员变量也不需要初始化器。另一方面，Java 类型中定义的 `private` 与包内私有的成员对仓颉侧来说不可见，因此这类成员同样不会出现在 Java 镜像类型定义中。

需注意，上述 Java 镜像类型定义的写法是不符合仓颉语法/语义规格的，故 Java 镜像类型定义必须带有 `@JavaMirror` 注解，该注解用于在编译期协助 cjc 区分正常的仓颉类型定义与 Java 镜像类型定义，从而对后者进行特殊处理。

示例如下，假设存在如下的 Java  `class`：

```java
public class Node {
    public static final int A = 0xDeadBeef;
    private int _id;
    public Node(int id) { _id = id; }
    public int id() { return _id; }
}
```

其对应的 Java 镜像类型定义可能如下：

<!-- compile -->
```cangjie
@JavaMirror
public open class Node {
    public static let A: Int32
    public init(id: Int32)
    public open func id(): Int32
}
```

互操作库中预置了几个基础的 Java 类型的镜像类型，即 `java.lang.Object`、`java.lang.String` 和 Java 数组类型，详情请参见 [互操作库预置 API 参考](#互操作库预置-api-参考)。

### 互操作类

互操作类本质上是一个仓颉 `class`，从一个或多个镜像类型派生而来，这种仓颉 `class` 能够同时被仓颉和 Java 侧使用，这是因为其所有构造函数和非继承而来的 `public` 成员函数，都会通过共同的 Java 包装类（由 cjc 编译时自动生成），对 Java 代码暴露。这个 Java 包装类本身可能会定义若干辅助方法，但 Java 侧仅能调用由仓颉侧对外暴露的方法、以及该类继承所得的方法；仓颉侧代码的调用权限规则与之相同。

接下来将举例说明，当使用 cjc 编译以下互操作类时：

<!-- compile -->
```cangjie
@JavaImpl
public class BooleanNode <: Node {
    private let flag: Bool
    public init(id: Int32, flag: Bool) {
        super(id)
        this.flag = flag
    }
    public func isFlagged(): Bool {
        flag
    }
}
```

 cjc 将同时生成一份 Java 源码，其内容类似于以下代码块：

```java
public class BooleanNode extends Node {
    /* 胶水代码 */
    public BooleanNode(int id, boolean flag) {
        /* 胶水代码，构造一个 Java 的 BooleanNode 包装类实例，
         * 并将其与仓颉侧的 BooleanNode 实例关联起来
         */
    }
    public boolean isFlagged() {
        /* 胶水代码，调用该 Java 的 BooleanNode 包装类实例所关联的
         * 仓颉 BooleanNode 实例的'isFlagged'实例成员函数，并返回调用结果
         */
    }
    /* 其他胶水代码 */
}
```

### 外部类型

镜像类型和互操作类均有别于语言本身的用户自定义类型，为简洁起见，本文档将它们统称为外部类型。

### Java 兼容类型

以下仓颉类型均为 Java 兼容类型：

* 所有拥有等价的 Java 基本类型的仓颉值类型，例如 `Int16` 拥有等价的 Java 基本类型 `short`，故 `Int16` 为 Java 兼容类型；而 `UInt8` 无等价的 Java 基本类型，故 `UInt8` 不是 Java 兼容类型
* 所有外部类型
* `Option<T>` 类型，且其中类型变元 `T` 为外部类型

互操作库中预置的特殊泛型镜像类型 `JArray<T>` 对应 Java 的数组类型，其类型变元 `T` 必须是 Java 兼容类型。

需注意，外部类型定义中可见性为 `public` 成员函数的形参类型和返回类型必须是 Java 兼容类型，否则将导致 cjc 编译报错；可见性为 `public` 的构造函数同理。

<!-- compile -->
```cangjie
@JavaImpl
class WeightedNode <: Node {
    public let weight: Float64      // 该实例成员变量不会暴露至 Java 侧
    public init(weight: Float64) {
        this.weight = weight
    }
}
```

<!-- compile -->
```cangjie
@JavaImpl
class ColoredNode <: Node {
    private let _color: Int32
    public prop color: Int32 {     // 该实例成员属性将以方法 'int getColor()' 的形式暴露给 Java 侧
        get() { _color }
    }
    public init(color: Int32) {
        _color = color
    }
}
```

## 在仓颉侧使用 Java

通过以下步骤来实现 Android 应用中 Java 与仓颉的互操作：

1. 基于 Java 类和方法，设计互操作胶水层 API，由开发者完成互操作胶水层的设计（以 Java 伪代码形式呈现）。

2. 根据上一步设计的胶水层，借助 Java 镜像生成器，为所有现存相关的 Java 类和接口生成仓颉侧可用的 @JavaMirror 类型定义，即将 .class 文件转换为 .cj 镜像类型定义文件。

3. 使用仓颉语言编写实现互操作层，在仓颉代码中按需使用 @JavaMirror 镜像类型，例如创建镜像类型的实例、调用其成员函数等。即开发者依据互操作胶水层设计和 .cj 镜像类型定义，完成 .cj 互操作层实现代码的编写。

4. 将 @JavaMirror 镜像类型定义和第 3 步中仓颉实现的互操作层代码一并使用 cjc 编译器进行编译，编译产物包括：

    * 包含互操作层逻辑的动态库（.so 文件）。
    * 若干 Java 侧可用的镜像类型定义源文件（.java 文件）。

    即：.cj 源文件（镜像类型定义 + 互操作层实现）经 cjc 编译后，生成 .so 动态库和 .java 胶水层镜像类型定义。

5. 将以下中间产物添加进 Android Studio 工程：

    * 第 4 步中由 cjc 编译产生的若干 .java 源文件，其中包含后续 Java 侧可能用到的互操作胶水层代码。
    * 第 4 步中由 cjc 编译得到的 .so 动态库文件，其中包含了由仓颉实现的胶水层逻辑。
    * 仓颉 SDK 中所有必要的运行时库，包括 .so 和 .jar 等。

以下将通过一个端到端的例子来详细说明上述流程。

### 第一步：设计互操作胶水层

在这一步，开发者需要从 Java 源码的视角，设计一个或多个互操作类。互操作类由仓颉编写实现，但最终会由 cjc 编译生成镜像类型以便 Java 侧使用，因此 Java 侧无需关心互操作类的具体实现，而只需要关心 Java 侧需要哪些功能。因此，对每个互操作类，开发者只需要考虑以下要点：

* 互操作类应该放在哪个 Java 包中？
* 互操作类是默认继承 `java.lang.Object`，还是需要继承其他 Java 类？
* 互操作类是否需要实现任何 Java 接口？
* 互操作类中需要拥有哪些 `public` 构造方法/成员方法？开发者目前只需要知道它们的功能以确定其函数签名，真正的实现将在后续步骤中用仓颉语言完成。

更多约束查看：[互操作类的特性与限制](#互操作类的特性与限制)。

**例子：**

假设，开发者希望在 Java 侧通过调用一个静态方法来将控制流从 Java 侧切换到仓颉侧，这个静态方法的名称为 `m`，接收 3 个形参，形参类型分别为 `com.example.a.A`、`java.lang.String` 和 `int`，并返回类型为 `com.example.b.B` 的值。开发者还希望这个静态方法属于一个叫做 `Interop` 的类，且该类位于名为 `cjworld` 的 Java 包中，不继承任何 Java 类，也就是说，默认继承 `java.lang.Object`，也不实现任何接口。

根据上述描述，步骤四中 cjc 将为互操作类生成的镜像类型定义大致如下：

```java
// Java 包名为 `cjworld`
package cjworld;

// 为定义静态方法 `m`，需要依赖以下两个其他包中定义的类型
import com.example.a.A;
import com.example.b.B;

// 互操作类在 Java 侧的镜像类型定义
public class Interop {
    /* 胶水代码 */
    public static B m(A a, String s, int i) {
        /* 调用仓颉侧静态成员函数`m`实现逻辑的胶水代码 */
    }
    /* 其他胶水代码 */
}
```

### 第二步：生成镜像类型声明

现在，切换到仓颉侧源码的视角，开发者需要获得在仓颉侧编写互操作类所依赖的所有 Java 类型的镜像类型，根据上一步可知具体依赖哪些 Java 类型：互操作类的父类型、形参类型、返回类型，甚至可能还有这些类型本身所依赖的类型。

> **注意**：
>
> CJMP 互操作库中预置了 `java.lang.Object`、`java.lang.String` 和泛型 Java 数组类型的镜像类型，而 Java 基本数据类型也无需镜像，在仓颉侧使用对应的仓颉基本数据类型即可。如果开发者的互操作类中并没有用到除了前述这几种类型外的其他 Java 类型，可直接跳过步骤二。

以下说明了如何使用 [Java 镜像生成器](#java-镜像生成器参考) 来为依赖的 Java 类型生成镜像类型：

```bash
java -Dpackage.mode=true -Dpackage.name=package-name \
    -jar ${CANGJIE_HOME}/tools/bin/java-mirror-gen.jar \
    --boot-class-path path-to-android-jar \
    --class-path full-application-classpath \
    -d output-directory \
    names-of-mirrored-types
```

或：

```bash
java -Dpackage.mode=true -Dpackage.name=package-name -Djar.mode=true \
    -jar ${CANGJIE_HOME}/tools/bin/java-mirror-gen.jar \
    --boot-class-path path-to-android-jar \
    --class-path full-application-classpath \
    -d output-directory \
    jar-file
```

**例子:**

延续之前的例子，开发者注意到将定义的互操作类 `cjworld.Interop` 依赖如下类型：

* 父类型 `java.lang.Object`
* 静态成员函数 `m` 的形参类型 `com.example.a.A`、`java.lang.String` 和 `int`
* 静态成员函数 `m` 的返回类型 `com.example.b.B`

对于上述类型，开发者不需要为 Java 基本数据类型 `int` 生成镜像类型，而 `java.lang.Object` 和 `java.lang.String` 这两个 Java 类型的镜像类型则在 CJMP 互操作库中预置了。所以开发者只需要为 `com.example.a.A` 和 `com.example.b.B` 这两个 Java 类型生成镜像类型即可。假设我们希望将生成的镜像类型放在名为 `javaworld` 的仓颉包中，以下是一条 Java 镜像生成器的命令行调用：

```bash
java -Dpackage.mode=true -Dpackage.name=javaworld \
    -jar ${CANGJIE_HOME}/tools/bin/java-mirror-gen.jar \
    --boot-class-path ${ANDROID_SDK}/platforms/android-35/android.jar \
    --class-path ${ANDROID_SDK}/platforms/android-35/android.jar:./App.jar \
    -d ./src/cj \
    com.example.a.A com.example.b.B
```

### 第三步：实现互操作类

现在开始用仓颉语言为第一步中描述的 Java 类框架编写实现逻辑，请参考以下要点：

1. 互操作类所在的包名和类名与步骤一中的设计保持一致（ cjc 编译互操作类自动生成的 Java 封装类的包名和类名与互操作类的包名和类名是完全一样的）。
2. 导包 `java.lang.*`。
3. 导入第二步通过 Java 镜像生成器产出、实现互操作类所需的镜像类型，暂不引入其余依赖类型。
4. 为互操作类加上注解 `@JavaImpl`。
5. 互操作类继承某 Java 类的镜像类型。标注了 `@JavaImpl` 的互操作类默认继承预置在互操作库中的 `java.lang.Object` 的镜像类型。
6. 仓颉代码中，任何需要使用 `java.lang.Object`、`java.lang.String` 和 Java 数组的地方，分别使用 `JObject`、`JString` 和 `JArray<T>` 来实现相应功能逻辑。

 Java 类型到仓颉类型的映射关系（`T'` 为对应的仓颉值类型，或相应的镜像类型）：

 Java 类型 (`T`)  | 仓颉类型 (`T'`)
---------------- | -----------------------------
`boolean`        | `Bool`
`byte`           | `Int8`
`short`          | `Int16`
`char`           | `UInt16`
`int`            | `Int32`
`long`           | `Int64`
`float`          | `Float32`
`double`         | `Float64`
`Object`         | `JObject` 或 `?JObject` \*
`String`         | `JString`、`?JString`、`String` 或 `?String` \*
`class C`        | `C'` 或 `?C'` \*
`interface I`    | `I'` 或 `?I'` \*
`T[]`            | `JArray<T'>` 或 `?JArray<T'>` \*

\* 对于可能接收或持有 `null` 值的镜像类型和互操作类的形参类型、返回类型和局部变量类型，请使用 `Option<T'>`，而不是 `T'`。

Java 侧返回类型为 `void` 的方法，在仓颉侧的对应函数返回类型是 `Unit`。

另请参见 [互操作类的特性与限制](#互操作类的特性与限制)。

延续之前的例子，实现的互操作类类似如下：

<!-- compile -->
```cangjie
package cjworld

import java.lang.*
import javaworld.*

@JavaImpl
public class Interop {
    public static func m(a: ?A, s: ?JString, i: Int32): ?B {
        /* 此处可以实现各种逻辑 */
        B() // 假设 com.example.b.B 拥有一个 public 无参构造方法
    }
}
```

在上述例子中，如果静态成员函数 `m` 在设计上不可能返回 `null` 到 Java 侧，那么完全可以将 `m` 的返回类型改为 `B`，互操作依然可以正常工作。

### 第四步：编译互操作类

使用以下命令进行对互操作类实现进行编译:

```bash
cjc --output-type=dylib \
    --target=aarch64-linux-android31 \
    -p source-directory \
    -ljava.lang -ljava.internal \
    --output-javagen-dir=java-output-directory \
    --sysroot=${ANDROID_NDK_HOME}/toolchains/llvm/prebuilt/linux-x86_64/sysroot \
    -B ${ANDROID_NDK_HOME}/toolchains/llvm/prebuilt/linux-x86_64/bin
```

在上述命令中：

`source-directory` 是保存互操作类和镜像类型声明源文件所在的目录的路径。

`java-output-directory` 是预期存放编译过程中自动生成的 Java 源文件的目录的路径。

上述 cjc 编译命令的直接编译产物是包含了互操作类定义的 `.so` 仓颉动态库文件和若干保存 Java 包装类的 Java 源文件。

例如：

```bash
cjc --output-type=dylib \
    --target=aarch64-linux-android31 \
    -p src/cjworld \
    -ljava.lang -ljava.internal \
    --output-javagen-dir=src/java
    --sysroot=${ANDROID_NDK_HOME}/toolchains/llvm/prebuilt/linux-x86_64/sysroot \
    -B ${ANDROID_NDK_HOME}/toolchains/llvm/prebuilt/linux-x86_64/bin
```

编译将生成两个文件：`libcjworld.so` 和 `src/java/cjworld/Interop.java`，后者包含了 Java 侧可以使用的互操作类的镜像类型。

### 第五步：集成编译产物至安卓工程

1. 将以下文件添加至 Android Studio 工程：
    * 第四步中由 cjc 生成的所有 Java 源文件，添加至 `src/main` 目录下，根据其实际包名，创建必要的目录结构，将源文件放至对应目录位置。
    * 将第四步中由 cjc 编译得到的 .so 文件放入 `src/main/jniLibs/arm64-v8a` 目录下，若该目录不存在，手动创建即可。
    * 将 `$CANGJIE_HOME/runtime/lib/linux_android_aarch64_cjnative` 目录下的所有 `.so` 文件复制进 `src/main/jniLibs/arm64-v8a` 目录下。
    * 将安卓 NDK 中的 `libc++_shared.so` 文件复制进 `src/main/jniLibs/arm64-v8a` 目录下。该文件位于安卓 NDK 根目录下的 `toolchains/llvm/prebuilt/<host>/sysroot/usr/lib/aarch64-linux-android` 目录下，其中 `<host>` 是开发者构建安卓工程所在的平台的 `${os}-${arch}` 组合。例如，如果是在 x64 架构 Linux 上构建安卓工程，则 `<host>` 为 `linux-x86_64`。
    * 将 `$CANGJIE_HOME/lib/library-loader.jar` 作为安卓工程的 JAR 包依赖。

2. 重要提示：必须强制使用传统规范，将所有 `APK` 中的 `.so` 文件进行压缩，否则应用运行时，将在尝试加载仓颉库的时候发生崩溃。请找到安卓工程中的 Gradle 构建脚本（一般名为 `build.gradle.kts`），在其中找到 `android {}` 配置块，检查配置块中是否已经存在以下配置信息。如果没有，请将以下配置信息插入其中：

   ```java
   // ...
   android {
       // ...
       packaging {
           jniLibs {
               useLegacyPackaging = true
           }
       }
   }
   // ...
   ```

3. 请重新构建安卓工程，确保截至目前安卓工程能够成功构建，不存在任何问题。构建成功后，也可以尝试推送安装应用检查是否存在安装问题。

4. 现在开发者就可以在 Java 源码中编写原先预想的调用互操作类的代码逻辑了。编写完成后，再次重新构建安卓工程。

延续之前的例子，在 Java 侧，现在开发者就可以编写逻辑调用 `Interop.m` 方法了，调用这个方法就会使得程序控制交给仓颉侧的互操作类的实现逻辑：

```java
// ...
B b = Interop.m(new A(), "Test", 0);
// ...
```

### 仓颉侧调用 Java

现在开发者已经设计了胶水层，实现并构建了互操作类，将各个必要的产物集成进了安卓工程，接下来可以继续往仓颉侧的互操作类中加入更多的代码逻辑。类型映射关系与 [在仓颉侧使用 Java](#在仓颉侧使用-java) 完全相同。

仓颉类型 (`T'`)           |  Java 类型 (`T`)
----------------------------- | ---------------
`Bool`                        | `boolean`
`Int8`                        | `byte`
`Int16`                       | `short`
`UInt16`                      | `char`
`Int32`                       | `int`
`Int64`                       | `long`
`Float32`                     | `float`
`Float64`                     | `double`
`JObject` 或 `?JObject`       | `Object`
`JString` 或 `?JString`       | `String`
`T'` 或 `?T'`                 | `T` \*
`JArray<T'>` 或 `?JArray<T'>` | `T[]` †

\* `T'` 必须要么是互操作类，要么是 Java 类型 `T` 的镜像类型。如果 `T'` 是互操作类，`T` 则是 Java 侧的一个包装类，且该包装类是由 cjc 在编译互操作类 `T'` 时自动生成的。

† `T'` 必须要么是互操作类，要么是镜像类型，要么是上表中列举的值类型（例如 `Int32`）。

**使用限制**：形参为可变参数（Java 类型 `T...`）的方法，在镜像中对应一个类型为 `?JArray<T'>` 的普通形参。仓颉代码调用此类方法或构造函数前，必须显式创建并初始化 `JArray<T'>` 实例以传入实参。

正常构建安卓工程之后，按以下步骤实现仓颉侧对 Java 侧暴露的接口的调用：

### 步骤一：为仓颉侧生成 Java 类型的镜像类型声明

如果开发者在互操作类的内部实现中，只会调用互操作类中的成员函数，那么由于这些成员函数的函数签名中的所有 Java 类型均已生成镜像类型，理论上可以直接跳过这步，无需使用 Java 镜像生成器生成更多的镜像类型。

使用 Java 镜像生成器为 Java 类型生成镜像类型的命令行：

```bash
java -Dpackage.mode=true -Dpackage.name=package-name \
    -jar ${CANGJIE_HOME}/tools/bin/java-mirror-gen.jar \
    --boot-class-path path-to-android-jar \
    --class-path full-application-classpath \
    -d output-directory \
    names-of-mirrored-types
```

或：

```bash
java -Dpackage.mode=true -Dpackage.name=package-name -Djar.mode=true \
    -jar ${CANGJIE_HOME}/tools/bin/java-mirror-gen.jar \
    --boot-class-path path-to-android-jar \
    --class-path full-application-classpath \
    -d output-directory \
    jar-file
```

在上述命令中：

* `package-name` 指定了为 Java 类型生成的镜像类型希望的包名。之所以镜像类型的包名不一定能与原 Java 类型的包名保持一致，与 [循环导入依赖](#处理循环导入依赖) 有关。

* `path-to-android-jar` 为你所使用的 Android SDK 中 `android.jar` 文件的路径，例如 `${ANDROID_SDK}/platforms/android-35/android.jar`。

* `full-application-classpath` 指定了本次镜像生成所采用的类路径，包括安卓 SDK 的 `android.jar`，和安卓项目构建得到的 `App.jar` 等，类路径之间由冒号分隔。

* `output directory` 指定了生成的包含镜像类型的仓颉源文件希望放置在哪个目录下，例如 `./src/cj`。

* `names-of-mirrored-types` 是一到多个 Java 引用类型的完全限定名，之间以空格分隔。这些类型是在互操作类设计过程中开发者所识别出来的除了 `java.lang.Object`、`java.lang.String` 和 Java 数组类型外的其他 Java 引用类型，`java-mirror-gen` 将为这些类型生成镜像。

* `jar-file` 是单个 `jar` 文件的路径，这个 `jar` 中的所有 `.class` 文件中的 `public` 的 `class` 和 `interface` 均会生成镜像，且这些类型所依赖的类型（在 `<full-application-classpath>` 的类路径下找到）也会被生成镜像。

延续之前的例子，假设开发者希望在仓颉侧的 `Interop.m` 静态成员函数中，调用 Java 侧定义的 `com.example.c.C` 的签名为 `String g(A a, int i)` 静态方法，其定义如下：

```java
package com.example.c;

import com.example.a.A;

public class C {
    public static String g(A a, int i) {
        /* Some  Java  code returning a string */
    }
}
```

由于 `com.example.c.C` 是新引入的互操作类中用到的类型，尚不存在其镜像类型供互操作类使用，故需要重新执行 Java 镜像生成器命令。这次额外新增一个入参 `com.example.c.C`，其他则保持不变：

```bash
java -Dpackage.mode=true -Dpackage.name=javaworld \
    -jar ${CANGJIE_HOME}/tools/bin/java-mirror-gen.jar \
    --boot-class-path /home/user/Android/Sdk/platforms/android-35/android.jar \
    --class-path /home/user/Android/Sdk/platforms/android-35/android.jar:App.jar \
    -d ./src/cj \
    com.example.a.A com.example.b.B com.example.c.C
```

这条命令所生成的所有镜像类型声明文件，是在之前的基础上，新增一个 `src/javaworld/src/C.cj`，并且如果 `com.example.c.C` 类型本身依赖其他需要生成镜像类型的 Java 类型，且这些类型尚未被镜像，那么也会同时生成这些类型的镜像类型声明文件。

新生成的文件 `src/cj/javaworld/src/C.cj` 的内容如下：

<!-- compile -->
```cangjie
package javaworld

import java.lang.*

@JavaMirror["com.example.c.C"]
public class C {
    public static func g(a: ?A, i: Int32): ?JString
}
```

### 步骤二：导入镜像类型并实现互操作类的逻辑

确保用于实现互操作类的所有镜像类型均已生成，并导入它们，接着就可以把它们完全当成仓颉类型来使用，实现互操作类中构造函数和成员函数的逻辑了。

延续之前的例子，这时开发者就可以在 `cjworld.Interop.m` 函数体中使用 `javaworld.C` 了：

<!-- compile -->
```cangjie
package cjworld

import java.lang.*
import javaworld.A
import javaworld.B
// 新增导入
import javaworld.C

@JavaImpl
public class Interop {
    public static func m(a: ?A, s: ?JString, i: Int32): ?B {
        let s1: JString = match (a) {
            case Some(aa) => C.g(aa, i) ?? JString("")
            case None => JString("")
        }
        B(s1)  // 假设B存在一个签名为 B(String) 的构造方法
    }
}
```

### 步骤三：重编仓颉部分的源码

使用 cjc 重新编译更新后的仓颉互操作层代码，详情请参见 [编译互操作类](#第四步编译互操作类)。

### 步骤四：更新并重新构建安卓工程

只要开发者确定在前几步中没有改变互操作类所暴露的 `public` 接口的签名，那么理论上只需要在重编仓颉实现源码后更新安卓工程中的 `.so` 文件。只要互操作类所暴露的 `public` 接口签名保持不变， `cjc` 所生成的 Java 胶水层源码内容理论上是完全一致的。

将步骤三中新生成或更新了的 `.so` 文件和 `.java` 文件（如有必要）更新到安卓工程的对应位置，然后重新构建安卓工程，详情请参见 [集成编译产物至安卓工程](#第五步集成编译产物至安卓工程)。

### 互操作类的特性与限制

* 互操作类必须是 `@JavaMirror class` 的直接子类。互操作类当不显式指定继承哪个父类时，将默认继承互操作库中的 [`java.lang.JObject`](#javalangjobject)，而非 `std.core.Object`。

* 互操作类可能实现一到若干个 `@JavaMirror interface`，但禁止实现任何普通仓颉 `interface`。反过来，普通仓颉类型禁止实现或继承 `@JavaMirror interface`。

* 互操作类禁止声明为 `open` 或 `abstract`，且禁止 `extend`，否则均将导致编译报错。

* 互操作类中允许定义实例成员变量，且变量类型可以是任何仓颉类型，这是因为互操作类中的实例成员变量一定不会暴露至 Java 侧。互操作类中允许重写其父类中的成员函数。

* 互操作类的构造函数体中可以通过 `super()` 调用父类的构造函数，其对调用实例成员函数的先后顺序的规格限制，与普通仓颉构造函数的是完全一致的。另外，构造函数体中同样也需要为所有互操作类新定义的实例成员变量进行初始化，否则将导致编译报错。

* 在互操作类的实例成员函数中，可通过 `super.` 调用父类的实例成员函数，即使该函数已在当前互操作类中被重写。

* 互操作类中可见性为 `public` 的构造函数和成员函数的函数签名中所用到的类型，只允许是：(a) 镜像类型或互操作类；(b) 100% 对应于 Java 基本数据类型的仓颉基本数据类型；(c) 仓颉 `String` 类型。该限制对于成员属性同样存在。

安卓 / JVM 平台特有约束：

* 所有镜像类型和互操作类型所对应的 Java 类型，都必须由同一个类加载器所加载。

* 与 Java 及其他 JVM 语言不同，仓颉禁止包之间存在循环导入依赖关系。该限制给镜像生成的流程带来了挑战，详情请参见 [处理循环导入依赖](#处理循环导入依赖) 章节。

#### 自动字符串转换

Java 的 `java.lang.String` 与仓颉的 `std.core.String` 在二进制层面不兼容，因此需要在两种字符串表示之间进行拷贝式转换，方能使一方语言的代码处理来自另一方的字符串数据。内置的镜像类型 [`java.lang.JString`](#javalangjstring) 提供专用构造函数与成员函数：构造函数接收仓颉 `String`，基于其 UTF-8 字符数据生成等效 UTF-16 格式 Java 字符串；成员函数 `JString.toString()` 则实现反向转换，输出仓颉 `String`。

```java
public class J {
    public static String s2s(String s) { /* ... */ }
}
```

<!-- compile -->
```cangjie
@JavaMirror
public class J {
    public static func s2s(s: ?JString): ?JString
}

let s: String = J.s2s(JString("Cangjie string")).getOrThrow().toString()
```

仓颉与 Java 互操作中，若在每次跨越边界时一律执行全量转换，在特定场景下会引入不必要的 CPU 与内存开销——典型场景如：仓颉代码仅将某个 Java 字符串透传回 Java，并不实际访问其内容。此时该字符串只是"路过"仓颉侧，转换完全属于浪费，在移动端等资源敏感环境中尤其应当避免。

因此，互操作类作者可以针对需要在 Java 侧暴露为 `String` 类型的特定构造函数形参、成员函数形参、返回类型及成员属性，选择性地启用自动转换：将这些实体在仓颉侧声明为 `String` 类型（而非 `JString`）即可。唯一限制是，此类成员函数不得重写其镜像 Java 父类中的方法（见下文说明）。生成的 Java 包装类在相应位置使用 `String` 类型，桥接代码在 Java 调用这些构造函数和方法时自动完成两种字符串之间的转换。

若需要支持接收或返回 `null`，应使用 `?String` 而非 `String`。详情请参见 [Java `null` 值处理](#java-null-值处理)。

`String` 与 `JString` 是互不相关的类型，不存在子类型关系。因此，互操作类中重写镜像 Java 父类方法的成员函数，其形参与返回类型必须与父类方法一致（通常为 `?JString`），`override` 修饰符才能正确生效：

<!-- compile -->
```cangjie
@JavaMirror
public class J {
    public func f(s: ?JString): Unit
}

@JavaImpl
public class CJ <: J {
    override public func f(s: ?String): Unit {}   // 错误
    public func f(s: ?String): Unit {}            // 重载，而非重写
    override public func f(s: ?JString): Unit {}  // 正确
}
```

虽然也可以在镜像类型定义中手动将 `JString` 替换为 `String`，但不建议这样做：由于二者无子类型关系，若未在所有子类型中完全一致地替换，可能引发难以诊断的重写/重载歧义。

[`java.lang.JArray<T>`](#javalangjarrayt) 的类型变元不支持 `String`，因此 Java 字符串数组应映射为 `JArray<JString>`，更常见的是 `?JArray<?JString>`。

## 由 Java 到仓颉的映射关系

当前版本的 Java 镜像生成器遵循以下所描述的 Java 到仓颉的类型映射规格。

> **说明：**
>
> Java 镜像生成器的直接输入是 Java 的 `.class` 文件而不是 `.java` 源文件，因此任何 `javac` 没有从 Java 源代码传播到类文件的信息，Java 镜像生成器都无法感知。正是由于这个原因，部分映射规则受到影响，其中最主要的是对 [Java 泛型](#java-泛型) 和方法形参名称的处理。

### Java 名称

 Java 类型、字段及方法的原名称会被尽可能地保留，但如果原名称由于下述的任何原因无法保留，原名称将通过 `@JavaMirror` 注解传播到仓颉侧，供 cjc 还原出原 Java 名称：

* 与仓颉关键字冲突的 Java 标识符，如 `func`、`main`、`Int32` 等，将会由反引号 ` `` ` 包裹以作为仓颉标识符，例如：

    ```java
    public static final long Int32 = 0xffff_ffff;
    ```

    <!-- compile -->
    ```cangjie
    public static let `Int32`: Int64
    ```

* Java 标识符中可能包含仓颉标识符所禁止的字符，最典型的就是 `$` 符号，其一般被用作嵌套 Java 类型在 `.class` 文件中二进制形式的类型名。这类字符将被替换为下划线 `_`，例如：

    ```java
    public class Outer {
        public class Inner {}
        public Inner getInner() { return new Inner(); }
    }
    ```

    <!-- compile -->
    ```cangjie
    @JavaMirror["Outer"]
    public open class Outer {
        public init()

        public open func getInner(): ?Outer_Inner
    }

    @JavaMirror["Outer$Inner"]
    public open class Outer_Inner {
        public init(p0: ?Outer)
    }
    ```

* Java 用户自定义类型中字段、成员类型和方法允许拥有相同的标识符。同一类型中的实例方法和静态方法如果方法签名不同，也是允许使用相同的标识符作为方法名的。而在仓颉中，除重载函数外，禁止成员之间拥有相同名称。仓颉没有成员类型的概念，Java 的成员类型将被映射为仓颉的顶层类型，因此不可能存在相同名称带来的冲突。

    因此，为了符合仓颉的规则，如果存在上述的命名冲突， Java 镜像生成器将为实例成员变量的名称末端追加 `_${type-name}`，为静态成员函数的名称末端追加 `Static`。Java 侧的原名称依然将通过 `@ForeignName` 注解得以留存，例如：

    ```java
    public class Node {
        public int id;
        public Node(int id) { this.id = id; }
        public static int id(long x) { return (int)x; }
        public static int id(short x) { return x; }
        public int id() { return id; }
        public void id(int newId) { this.id = newId; }
    }
    ```

    将被镜像为：

    <!-- compile -->
    ```cangjie
    public open class Node {
        @ForeignName["id"]
        public var id_Node: Int32

        public init(arg0: Int32)

        @ForeignName["id"]
        public static func idStatic(arg0: Int64): Int32

        @ForeignName["id"]
        public static func idStatic(arg0: Int16): Int32

        public open func id(): Int32

        public open func id(arg0: Int32): Unit
    }
    ```

* Java 包名无法保留，这是因为 Java 支持包间循环依赖，且大量的包存在循环依赖的用法，但仓颉则是禁止包间循环依赖的，如果保留 Java 包名将难以避免镜像得到的仓颉包间存在循环依赖从而导致仓颉侧编译失败。详情请参见 [处理循环导入依赖](#处理循环导入依赖) 章节。

### Java 基本类型

 Java 基本类型将被镜像为对应的仓颉值类型：

 Java 类型        | 仓颉类型
---------------- | ------------
`boolean`        | `Bool`
`byte`           | `Int8`
`short`          | `Int16`
`char`           | `UInt16`
`int`            | `Int32`
`long`           | `Int64`
`float`          | `Float32`
`double`         | `Float64`

### Java `class` 与 `interface` 类型

Java `class` 和 `interface` 类型定义将分别被镜像为仓颉 `class` 和 `interface` 类型定义，得到的类型定义将拥有 `@JavaMirror` 注解。`@JavaMirror` 注解的有且仅有一个字符串实参的值是被镜像的 Java 类型的完全限定名。如果 Java 类型的简单名称中不包含仓颉标识符所禁止的字符，那么镜像得到的仓颉类型的名称将保持与 Java 类型简单名称一致；否则，镜像得到的仓颉类型名称将按特殊规则生成，例如 Java 类型的简单名称中包含 `$`，或是一个嵌套类型（嵌套类型经`javac`编译得到的类型简单名称由其所在类型的简单名称和该类型的简单名称通过`$` 拼接而成），这些 `$` 将被自动替换为下划线 `_`。

被镜像的字段类型和方法的形参类型和返回类型 `T`，如果是 `class` 或 `interface` 类型，会自动装包为 `Option<T'>` 类型，其中 `T'` 是 `T` 的镜像类型。详情请参见 [null 值处理](#java-null-值处理) 章节。

被 `@JavaMirror` 注解的类型定义与正常的仓颉类型定义存在若干差异：

* `@JavaMirror class` 的继承层次结构的根类不是 `std.core.Object`，而是一个内置镜像类型 [`java.lang.JObject`](#javalangjobject)，而 `java.lang.JObject` 直接继承自 `std.core.Object`。

* Java 的 `java.lang.String` 在仓颉侧的镜像是一个内置镜像类型 [`java.lang.JString`](#javalangjstring)。

* 镜像得到的类型定义中仅保留符号和类型信息，变量初始化器、函数体、属性体等均不会在 `@JavaMirror` 类型定义中体现。

示例如下，假设存在以下 Java 类定义：

```java
public class Node {
    public static final int A = 0xDeadBeef;
    private int id;
    public Node(int id) { this.id = id; }
    public int id() { return id; }
}
```

其镜像得到的 `@JavaMirror` 类可能如下：

<!-- compile -->
```cangjie
@JavaMirror["Node"]
public open class Node {
    public static let A: Int32
    public init(id: Int32)
    public func id(): Int32
}
```

* 访问修饰符为 `public` 的 Java 类和接口会被镜像，其他则不会被镜像。

* 非 `final` 的 Java 类被镜像得到的仓颉类将拥有 `open` 修饰符。

* Java 的 `sealed`、`non-sealed` 以及遗留的 `strictfp` 修饰符均将被忽略。

* 访问修饰符为默认或 `private` 的构造方法、实例/静态字段、实例/静态方法不会被镜像。

* 静态初始化块和实例初始化块均不会被镜像。

* 如果 Java 类型的成员名称与镜像得到的仓颉类型的成员名称不同（原因请参考 [Java 名称](#java-名称) 小节），那么 Java 类型的成员名称信息将通过 `@ForeignName` 注解传递到仓颉侧，例如：

```java
CurrencyAmount priceInUS$Per(WeightUnit wu) { /* ... */ }
```

<!-- compile -->
```cangjie
@ForeignName["priceInUS$Per"]
public open priceInUS_Per(arg0: WeightUnit): CurrencyAmount
```

> **注意**：
>
> Java 和仓颉的访问修饰符 `protected` 的含义是不同的。
>
> 在 Java 中，`protected` 成员的可见范围是**所在包**内，以及所在类的子类。
>
> 而在仓颉中，`protected` 成员的可见范围是**所在模块**内，以及所在类的子类。
>
> 不过，通常情况下这个差异不会导致问题。

**字段**将被镜像为成员变量，变量类型为字段类型相应的镜像类型；变量名称与字段名称保持一致（一般情况下如此，特殊情况请参见 [Java 名称](#java-名称) 小节）；实例字段将被镜像为实例成员变量，静态字段将被镜像为静态成员变量；访问修饰符 `public`、`protected` 将直接保留；非访问修饰符 `transient`、`volatile` 将被忽略；`final` 字段将被镜像为 `let` 成员变量，非 `final` 字段将被镜像为 `var` 成员变量；字段初始化器将被忽略。

**方法**将被镜像为成员函数，其函数名与方法名保持一致（一般情况下如此，特殊情况请参见 [Java 名称](#java-名称) 小节）；其形参类型和返回类型为相应的镜像类型；返回类型为 `void` 的方法将被镜像为返回类型为 `Unit` 的成员函数；实例方法将被镜像为实例成员函数，静态方法将被镜像为静态成员函数；访问修饰符 `public`、`protected` 将直接保留；`default` 修饰符将被转换为标记于仓颉成员函数上的 `@JavaHasDefault` 注解；非访问修饰符 `native`、`synchronized` 及遗留的 `strictfp` 将被忽略；非 `final` 方法将被镜像为 `open` 成员函数。

**构造方法**将被镜像为构造函数，其形参类型均被替换为相应镜像类型；由于未定义构造方法而被隐式声明的默认构造方法也会被镜像；访问修饰符 `public`、`protected` 将直接保留。

> **注意**：
>
> 1. `@JavaMirror` 类中禁止包含主构造函数。
> 2. `@JavaMirror` 类中如果没有任何显式定义的构造函数，并不会像普通仓颉类那样存在隐式定义的构造函数，于是该类并不能通过调用构造函数来实例化。对于自动生成的 `@JavaMirror` 类，出现这种情况一般意味着被镜像的 Java 类中仅声明有访问范围为默认或 `private` 的构造方法，而这样做一般是有意阻止下游用户直接通过调用构造方法来实例化该类。
> 3. Java 镜像生成器的输入是 `.class` 文件，而方法/构造方法的形参名一般并不会保存在 `.class` 文件中，这种情况下，Java 镜像生成器会为生成的镜像自动合成形参名，诸如 `arg0`、`arg1`。`javac` 的编译选项 `-parameters` 可以使形参名得以在 `.class` 文件中留存，但只对 `class` 类型有效，`interface` 类型则依旧无法保留。调试信息生成相关选项 `-g`/`-g:vars` 与之同理。

成员类型将被镜像为顶层类型定义，因为仓颉并不支持嵌套类型定义；镜像类型的名称是成员类型的二进制名称，也就是该成员类型的直接所在类型的二进制名称，加上 `$`分隔符，再加上该成员类型自己的简单名称，依此类推，且由于仓颉标识符不支持`$`，所有 `$` 均被替换为下划线 `_`（可参考 [Java 名称](#java-名称) 小节）；访问修饰符 `public`、`protected` 将直接保留；非访问修饰符 `static` 将被忽略；镜像类型的构造函数将新增一个额外的形参，该形参用于传入该成员类型直接所在类型的实例（在 Java 中，该形参是被隐式声明且被隐式传入的）。

```java
public class Outer {
    public class Inner {}
    public static class Static {}
    public Inner getInner() { return new Inner(); }
    public Static getStatic() { return new Static(); }
}
```

<!-- compile -->
```cangjie
@JavaMirror["Outer"]
public open class Outer {
    public init()

    public open func getInner(): ?Outer_Inner
    public open func getStatic(): ?Outer_Static
}

@JavaMirror["Outer$Inner"]       // Original binary name is retained
public open class Outer_Inner {  // '$' is replaced with '_'
    public init(p0: ?Outer)      // Extra parameter for enclosing instance
}

@JavaMirror["Outer$Static"]       // Original binary name is retained
public open class Outer_Static {  // '$' is replaced with '_'
    public init()
}
```

所有镜像得到的成员函数和构造函数均无函数体，代码外观上与正常仓颉的抽象成员函数相似。于是存在以下约束条件：

* 抽象方法的 `abstract` 修饰符将被保留，否则单从仓颉侧无法区分原 Java 方法是否是抽象的，例如：

    ```java
    public abstract class A {
        public void c() {}
        public abstract void a();
    }
    ```

    <!-- compile -->
    ```cangjie
    @JavaMirror["A"]
    public abstract class A {
        public init()

        public open func c(): Unit

        public open abstract func a(): Unit
    }
    ```

* 默认接口方法所镜像得到的成员函数将带有 `@JavaHasDefault` 注解，否则单从仓颉侧无法区分原 Java 接口方法是否拥有默认实现，例如：

    ```java
    public interface I {
        default void c() {}
        void a();
    }
    ```

    <!-- compile -->
    ```cangjie
    @JavaMirror["I"]
    public interface I {
        @JavaHasDefault
        func c(): Unit
    
        func a(): Unit
    }
    ```

> **注意**：
>
> 不支持可变参数，对于声明了可变参数的方法和构造方法，其参数列表中的 `...` 部分将被忽略。

#### `@JavaMirror`类型的继承层次结构

`@JavaMirror class` 和 `@JavaMirror interface` 自成一套继承层次结构，也就是说：

* `@JavaMirror class` 的继承层次结构的根类并不是 `std.core.Object`，而是一个内置 `@JavaMirror class`，即 [`java.lang.JObject`](#javalangjobject)。

* `@JavaMirror interface` 可以继承其他 `@JavaMirror interface`，该继承关系反映的是原 Java 侧接口之间的继承关系。`@JavaMirror interface` 禁止继承除 `std.core.Any` 以外的普通仓颉 `interface`（所有 `@JavaMirror interface` 均隐式继承 `std.core.Any`），普通仓颉 `interface` 也禁止继承 `@JavaMirror interface`。

* `@JavaMirror class` 可以继承其他 `@JavaMirror class`，该继承关系反映的是原 Java 侧 `class` 之间的继承关系。`@JavaMirror class` 禁止继承普通仓颉 `class`，普通仓颉 `class` 也禁止继承 `@JavaMirror class`。

* `@JavaMirror class` 可以实现 `@JavaMirror interface`，该实现关系反映的是原 Java 侧 `class/interface` 之间的实现关系。`@JavaMirror class` 禁止实现除 `std.core.Any` 以外的普通仓颉 `interface`（所有 `@JavaMirror class` 均隐式实现 `std.core.Any`），普通仓颉 `class` 也禁止实现 `@JavaMirror interface`。

* 镜像类不得使用 `extend` 扩展，任何类型亦不得以镜像接口进行接口扩展。

* 在 Java 中，所有接口都是 `java.lang.Object` 类的子类型。这是因为 Java 中只有类才能实现接口。仓颉则不同：所有接口都是内置 `Any` 接口的子类型，而 `Any` 并非 `std.core.Object` 的子类型。因此，镜像接口不是 [`java.lang.JObject`](#javalangjobject) 的子类型，以下 Java 方法 `test()` 无法在仓颉中改写：

  ```java
  public interface I {}

  public class C {
      public static void accept(Object o) {}
      static void test(I i) { accept(i); }     // 在 Java 中可行
  }
  ```

#### Java 泛型

Java 泛型在经 `javac` 编译得到 `.class` 的过程中将被擦除，故由 Java 镜像生成器自动生成的 `@JavaMirror` 类型总是非泛型的，且所有原泛型参数均被替换为其最左边界类型的相应镜像类型。

禁止手写泛型 `@JavaMirror` 类型定义。

内置类型 [`JArray<T>`](#javalangjarrayt)（详情请参见 [Java 数组类型](#java-数组类型) 小节）除外，虽然是 `@JavaMirror` 类型，但支持泛型。

`@JavaMirror` 类型禁止作为普通仓颉泛型类型的类型实参，其中唯一的例外是仓颉 [`std.core.Option<T>` 类型](#java-null-值处理)。

### Java 数组类型

[`JArray<T>`](#javalangjarrayt)是一个特殊的内置类型，其为 Java 数组类型的镜像类型。

元素类型为 `T` 的 Java 数组（即类型为 `T[]`）被镜像为（其中 `T'` 为 `T` 的镜像类型）：

* 若 `T` 为基本数据类型，则镜像为 `?JArray<T'>`

* 若 `T` 为引用类型，则镜像为 `?JArray<?T'>`

其中 `T'` 是 `T` 的镜像类型。

如需了解进行 `Option<T>` 封装的原因，请参见 [null 值处理](#java-null-值处理) 章节。

> **注意**：
>
> Java 数组是协变的，而仓颉泛型是不变的，这个规格对于 `JArray<T>` 类型同样成立。

### Java 枚举类

 Java 枚举类 `E` 将被镜像为 `@JavaMirror` 类 `E'`，该类直接继承 `java.lang.Enum` 类的镜像。`E'` 既不可能是 `open` 也不可能是 `sealed`，故无法被继承。

`@JavaMirror` 类 `E'` 中包含有：

* Java 枚举常量的镜像，形式为可见性为 `public` 的 `let` 静态成员变量，变量类型为 `E'`。

* Java 枚举类型中隐式定义的若干方法的镜像，即 `public static E'[] values()` 和 `public static E' valueOf(String name)`。

* Java 枚举类型中所有访问范围为 `public`/`protected` 的字段和方法，其镜像规格与 Java 类的镜像规格完全一致。

`@JavaMirror` 类 `E'` 中无任何显式定义的构造函数，且 `@JavaMirror` 类本身也不会隐式定义默认构造函数，从而杜绝了通过调用构造函数实例化 `E'` 的可能性。

### Java 记录类

Java 记录类（`record`）本质上是语法糖，镜像规则与等价的普通类相同（参见 [Java `class` 与 `interface` 类型](#java-class-与-interface-类型)）。

记录类定义在语义上等价于一个满足以下条件的普通 Java 类：

* 为 `final` 类，且非 `abstract`
* 直接继承 `java.lang.Record`
* 包含一个或多个组件字段：即 `private` 实例字段，各配有对应的访问器方法
* 可包含 `static` 字段，但除组件字段外不得有其他实例字段
* 包含与组件字段一一对应的标准（canonical）构造函数，也可包含其他构造函数
* 重写 `java.lang.Object` 的 `equals()`、`hashCode()` 和 `toString()`，不重写其他 `Object` 方法

> **注意**：
>
> 内置镜像 [`java.lang.JObject`](#javalangjobject) 中，`hashCode()` 和 `toString()` 已分别重命名为 `hashCode32()` 和 `toJString()`，记录类镜像中亦遵循此约定。

**示例：**

```java
public record Node (int value, Node next) {}
```

<!-- compile -->
```cangjie
@JavaMirror["Node"]
public class Node <: Record {
    public init(value: Int32, next: ?Node)

    public func toJString(): JString
    public func hashCode32(): Int32
    public func equals(o: ?JObject): Bool

    public func value(): Int32
    public func next(): ?Node
}
```

### Java `null` 值处理

仓颉没有空引用的概念，因此对 Java `null` 类型没有直接对应物。如果 Java 引用类型直接镜像为对应的镜像类型，那么任一从 Java 方法返回至仓颉的 `null` 值都将导致 `NoneValueException`。若仓颉代码访问装包为镜像类型的、实际含 `null` 的字段，亦会如此。反之，若 Java 调用[互操作类](#互操作类)方法并传入 `null` 形参，将抛出 Java `NullPointerException`；仓颉侧亦无法向设计为接受 `null` 的 Java 方法或构造函数传递 `null`。

因此，如果 Java 侧的字段类型、数组元素类型、方法形参类型或返回类型等，是引用类型 `R`，该实体的镜像所声明的类型将是 `Option<R'>`，其中的 `R'` 是 `R` 的镜像类型。在仓颉侧，`None` 代表的是 `null` 值，而 `Some(r)` 代表非 `null` 的引用值，其中 `r` 是类型为 `R'` 的值。为了实现上述规格，假设存在镜像类型或互操作类 `T`， cjc 会将 `Option<T>` 识别为 Java 兼容类型，并据此对 `T` 值进行装包/拆包操作。

示例如下：

```java
public interface I {
    Object f(Object[] xs);
}
```

形参 `xs` 本身可能为 `null`，数组 `xs` 的每个元素亦可能为 `null`，方法 `f` 的返回值同样可能为 `null`。因此，对该接口进行镜像时最稳妥的方式如下：

<!-- compile -->
```cangjie
@JavaMirror
public interface I {
    func f(xs: ?JArray<?JObject>): ?JObject
}
```

另一示例如下：

```java
interface Concatenator {
    String concat(String[] ss);
}
```

对于以上的 Java `interface`，形参 `ss` 本身可能为 `null`，`ss` 作为数组，其中每个元素都有可能为 `null`，`concat` 方法的返回值也同样可能为 `null`。因此，对于该 `interface` 来说最保险的镜像的方式如下：

<!-- compile -->
```cangjie
@JavaMirror
interface Concatenator {
    func concat(ss: ?JArray<?JString>): ?JString
}
```

`Option<T>` 装包同样完全适用于[互操作类](#互操作类)，因为此类类型的值可能传入/传出 Java 侧。此外，建议对互操作类暴露给 Java 的构造函数与成员函数的形参及返回类型均使用 `Option<T>` 装包。

同理，当开发者在互操作类中定义具有外部类型 `T` 的局部变量时，也应该使用 `Option<T>` 而不是直接 `T`，除非开发者能百分之百确定该局部变量不会被赋 `null` 值。

<!-- compile -->
```cangjie
// 假设M是 Java 镜像类型
let m: M = M() // 如果M()能够成功返回，开发者能够保证一定返回 M 实例而不是空引用
```

`Option<T>` 封装保证了即便 Java 侧往仓颉侧传入空引用也不会导致程序崩溃。然而这种封装也带来了代价：引入了额外的性能与内存开销，同时导致 [型变丢失](#型变丢失)，并使 [可空类型的类型测试与转换](#可空类型的类型测试与转换) 更为繁琐。

#### 型变丢失

为 Java 镜像类型和互操作类进行 [`Option<T>`装包](#java-null-值处理) 带来了一个显著的限制：像这样装包的类型在所有其他方面均完全遵循仓颉语义规则。具体而言，根据仓颉语义规则，`Option<T>` 对其类型变元 `T` 是不变的，换句话说，对于两个类型 `U` 和 `T`，除非 `U` 和 `T` 是相同的类型，否则即便 `U` 是 `T` 的子类型，`Option<U>` 也与 `Option<T>` 不存在任何子类型关系。这意味着，对于镜像类型中存在重写关系的方法，如果这两个方法的返回类型存在协变的关系，这个协变的关系无法在仓颉侧保留下来，子类中的重写方法的返回类型的镜像必须改为父类中方法的返回类型的镜像。

示例如下，在以下代码片段中，`class Foo` 是 `class Bar` 的父类：

```java
public class Foo {}

public class Bar extends Foo {}
```

`interface C` 中的 `get` 方法的返回类型是 `Foo`：

```java
public interface C {
    public Foo get();
}
```

`interface D` 作为 `interface C` 的子类型，可以通过重写 `get` 方法，来让方法的返回类型更加精确，从 `Foo` 改为 `Bar`：

```java
public interface D extends C {
    @Override
    public Bar get();
}
```

假设不进行 `Option<T>` 的装包，上述 Java 类型定义将被镜像为以下仓颉类型定义：

<!-- compile -->
```cangjie
@JavaMirror
public open class Foo {}

@JavaMirror
public open class Bar <: Foo {}

@JavaMirror
public interface C {
    public open func get(): Foo
}

@JavaMirror
public interface D <: C {
    public override open func get(): Bar // 此处存在返回类型协变
}
```

但正如前文所述，若不进行 `Option<T>` 装包，而调用 `get` 实际返回 `null` 值，则将导致抛出异常。

如果进行 `Option<T>` 装包，就可以解决 `null` 的问题，不过所有重写的成员函数的返回类型就不得不降级为原始的（定义在父类型中的）成员函数的返回类型：

<!-- compile -->
```cangjie
@JavaMirror
public open class Foo {}

@JavaMirror
public open class Bar <: Foo {}

@JavaMirror
public open interface C {
    public open func get(): ?Foo
}

@JavaMirror
public open interface D <: C {
    // public open func get(): ?Bar    // 错误，Option<T> 对于 T 不协变，?Bar 不是 ?Foo 的子类型
    public open func get(): ?Foo       // 正确，但返回类型降级了
}
```

 Java 的可空性注解，如 `@Nullable`、`@NotNull` 等，可以部分缓解上述问题，但当前版本尚不支持此处理。

#### 可空类型的类型测试与转换

**注意**：经 `Option<T>` 装包的可空外部类型值，在进行类型测试与转换之前，必须先做空值检测并拆包，原因如下：

1. 仓颉泛型对其类型变元是不变的，因此 `e is Option<T>` 仅当 `e` 的类型恰好为 `Option<T>` 时才为 `true`，而非某个 `Option<U>`（其中 `U <: T`）。同理，无法直接对 `Option<T>` 装包的值进行上转型或下转型。

   <!-- compile -->
   ```cangjie
   open class Foo {}
   class Bar <: Foo {}
       // ...
       let bar: Bar = Bar()
       let foo: Foo = bar                         // 正确
       let bar2: Bar = (foo as Bar).getOrThrow()  // 正确


       let maybeBar: ?Bar = Some(Bar())
       let maybeFoo: ?Foo = maybeBar                                // 错误：类型不匹配
       let maybeBar2: ?Bar = (maybeFoo as Option<Bar>).getOrThrow() // 抛出 NoneValueException
   ```

   可分别使用 `Option` 的成员函数 `.map()` 和 `.flatMap()` 编写上转型与下转型的简写：

   <!-- compile -->
   ```cangjie
   // 上转型（由子类型转为父类型）：
   let maybeFoo: ?Foo = maybeBar.map { bar => bar }
   
   // 下转型（由父类型转为子类型）：
   let maybeBar2 = maybeFoo.flatMap { foo => foo as Bar }
   ```

2. 在 Java 中，对任意引用类型 `T`，`null instanceof T` 均为 `false`；而在仓颉中，若 `e` 的值为 `Option<T>.None`，则 `e is Option<T>` 为 `true`。

   可在仓颉中按如下方式复现 Java `instanceof` 的语义：

   ```java
   // Java：
   void f(o: Object) {
       if (o instanceof T) { ... }
   }
   ```

   <!-- compile -->
   ```cangjie
   // 仓颉：
   func f(o: ?JObject): Unit {
       if (let Some(t) <- o && t is T) { ... }
   }
   ```

### 外部类型的转换与类型测试

仓颉运算符 `is` 和 `as`，以及 `match` 表达式中的类型模式 `v: T`，均支持所有[外部类型](#外部类型)（`if-let` 与 `while-let` 中的类型模式支持尚不完善）。不过，这些运算的语义与 Java 的 `instanceof` 及强制类型转换一致，一般情况下实际类型检测与转换在 JVM 中完成。

如 [Java `null` 值处理](#java-null-值处理) 所述，经 `Option<T>` 装包的镜像类型与互操作类值，在类型测试前必须先做空值检测并拆包：

1. `v is ?T` 仅当 `v` 的类型恰好为 `Option<T>` 时为 `true`，而非 `Option<U>`（`U <: T`）。
2. 在 Java 中 `null instanceof T` 为 `false`；在仓颉中，若 `v` 等于 `Option<T>.None`，则 `v is ?T` 为 `true`。

复现 Java `instanceof` 语义的方式如下：

```java
// Java：
void f(o: Object) {
    if (o instanceof T) { ... }
}
```

<!-- compile -->
```cangjie
// 仓颉：
func f(x: ?JObject): Unit {
    if (let Some(t) <- x && t is T) { ... }
}
```

### 处理循环导入依赖

 Java 源码中普遍存在包间的循环导入，例如，Java 最基础的类，`java.lang` 包中的 `String` 类依赖：

* `java.io` 包中的 `Serializable` 接口
* `java.nio.charset` 包中的 `Charset` 类
* `java.util` 包中的 `Locale` 类

而上述的所有类型均无一例外依赖 `java.lang` 包中的 `Object` 类，从而构成循环导入依赖。

之所以 Java 允许循环导入依赖，是因为对于每个类型，都会被编译为单独的 `.class` 文件。而对于仓颉来说，假设一个仓颉包 `a` 导入另一个包 `b`，则包 `b` 必须先于包 `a` 编译完毕，然后才能正常编译包 `a`。因此，同属于一个仓颉包的所有源文件必须在同一次 cjc 编译中被编译得到一个单独的二进制文件。仓颉的最小编译单元是一个包，而不是像 Java 那样的单个源文件，故在仓颉源码中，包间循环导入依赖是禁止的。

#### 单包模式

仓颉语言不支持 Java 那样的循环导入，因此镜像生成器无法将原始 Java 包名直接用作生成的仓颉包名。为避免仓颉侧出现循环导入依赖，镜像生成器必须将所有生成的镜像类型统一放在同一个仓颉包中，即使这些类型在 Java 侧来源于多个不同的包。该仓颉包名由用户通过系统属性 `package.name` 指定，默认为 `UNNAMED`。

举例来说，假设开发者运行镜像生成器，指定以下系统属性：

`-Dpackage.name=java.world`

镜像生成器将把生成的仓颉类型放在 `java.world` 包中，而原 Java 包名则通过 `@JavaMirror` 注解的参数得以传递至仓颉侧：

<!-- compile -->
```cangjie
package java.world

import java.lang.*

@JavaMirror["java.lang.Cloneable"]
public interface Cloneable {
}
```

#### Java 镜像类型名称冲突

不同 Java 包中可能定义有完全限定名不同，但拥有相同简单名称的类型。例如，JDK 的 `javax.management` 包中存在 `Attribute` 类，`javax.naming.directory` 包中存在 `Attribute` 接口。显然，如果它们生成的镜像类型的简单名称不改名，同为 `Attribute`，那么这两个类型的镜像类型无法同时存在在一个仓颉包中。因此，镜像生成器在 [单包模式](#单包模式) 下，将自动检测命名冲突，对所有存在冲突的类型名称进行修饰。具体而言，存在冲突的镜像类型的名称会采用原 Java 类型的完全限定名，其中的点 `.` 均替换为下划线 `_`。

<!-- compile -->
```cangjie
// src/java/world/src/javax_management_Attribute.cj
package java.world

import java.lang.*

@JavaMirror["javax.management.Attribute"]
public open class javax_management_Attribute <: Serializable {
 // ...
}
```

<!-- compile -->
```cangjie
// src/java/world/src/javax_naming_directory_Attribute.cj
package java.world

import java.lang.*

@JavaMirror["javax.naming.directory.Attribute"]
public interface javax_naming_directory_Attribute <: Cloneable & Serializable {
 // ...
}
```

#### 增量镜像

将所有 Java 镜像类型放入单一仓颉包（如 `java.world`）不仅会引发 [Java 镜像类型名称冲突](#java-镜像类型名称冲突)，还会增加编译开销，使用上也不便。将声明拆分到多个包中因此十分可取。并非所有 Java 包都处于同一导入依赖环中，故至少可以进行一定程度的拆分。

在任何 Android 应用中，以下三类包之间通常不存在循环导入依赖：

* 应用自身的包
* Android API 包
* JDK API 包

因此，即便在最坏情况下，也可将镜像类型分别归入三个仓颉包，可分别命名为 `app`、`android` 和 `java`。

此外，自 Java 9 起 JDK API 已模块化，Java 模块之间不允许循环导入依赖。镜像时完全可以复现 JDK 模块结构，例如将 `java.base` 模块导出包（`java.lang` 及其子包、`java.io`、`java.math` 等）的镜像放入仓颉包 `java.base`。

第三方库通常也不应与应用组件及 API 形成循环导入。若应用中包含需在仓颉代码中使用的此类库，可将每个库分别镜像到独立的仓颉包。

增量镜像流程支持上述拆分。对每个目标仓颉包各运行一次镜像生成器（通过 `package.name` 系统属性指定包名，用法同 [单包模式](#单包模式)），每次额外指定以下两个列表：

1. 要镜像到该仓颉包的 Java 包完整列表（这些 Java 包之间可有循环依赖，也可导入此前已镜像的包/类型）。
2. 所有此前已镜像 Java 类型的完全限定名映射累积列表（初次为空）。

生成器为列表 1 中各包的 `public` 类与接口及其尚未在列表 2 中的依赖生成镜像，并将新建立的由 Java 到仓颉的名称映射追加到列表 2。

##### 增量镜像命令行语法

增量镜像目前仅支持[单 JAR 包模式](#java-镜像生成器命令行语法)：命令行须设置 `jar.mode=true`，并传入 jar 文件路径（而非类型的完全限定名）：

```shell
java -Djar.mode=true \
    [system-properties] \
    -jar ${CANGJIE_HOME}/tools/bin/java-mirror-gen.jar \
    [options] \
    jar-file
```

`system-properties` 必须包含：

* `-Dpackage.mode=true`

* `-Dpackage.name=target-package-name`

  `target-package-name` 为目标仓颉包名，本次生成的**所有**镜像类型均放入该包。

  > **注意**：
  >
  > 增量模式下，每次运行镜像生成器都必须指定一个**全新的、此前未使用过的**目标仓颉包名。若首次以 `-Dpackage.name=my.java.libs` 镜像一批类型，再次运行时不更改 `package.name`，将完全破坏镜像结果的一致性。
  > 从某种意义上说，目标仓颉包是镜像的最小单元，正如仓颉源码包是 cjc 编译的最小单元：同一包中的源文件不能分开编译。

* `-Djar.mode.packages=pathname`

  `pathname` 为纯文本文件路径，每行一个 Java 完全限定包名，可选后缀 `.*`：

  ```text
  com.example.model
  com.example.ui.*
  ```

  > **注意**：
  >
  > 通配符 `.*` 表示匹配该包**及其所有子包**，语义与 Java/仓颉 import 语句中的通配符不同。

  生成器为所列包中所有非匿名、非 `private` 类型及其依赖（映射文件中已有的除外）生成镜像，均放入 `package.name` 指定的仓颉包。此选项仅用于单 JAR 包模式。

* `-Dimports.config=import-mappings-file`

  `import-mappings-file` 为**导入映射文件**路径，累积记录已生成镜像类型的由 Java 到仓颉的完全限定名映射。启动时读取；其中已有映射的类型不再重复镜像。成功完成后，将本次新映射追加并写回（默认为 `./imports_config.txt`）。

  > **注意**：
  >
  > 若 `import-mappings-file` 设为 `./imports_config.txt`，该文件会被覆盖。调试时如需保留中间映射，请在每次增量运行后备份。

#### 使用增量镜像

[增量镜像](#增量镜像) 可将应用中需在仓颉侧使用的任意组件拆分到不同仓颉包，前提是该组件与 Java 代码其余部分之间无循环导入依赖。每个需在仓颉中使用的第三方 Java 库的 public 类型，均可镜像到独立命名的仓颉包。

先将应用包划分为彼此无循环导入依赖的子集，再按增量流程操作。做法如下：

1. 选取不依赖其他子集的任一子集，一次性镜像到合适命名的包。
2. 镜像仅依赖已镜像子集的子集。
3. 重复上一步，直至所有需在仓颉中使用的 Java 类型均已镜像，每次使用新的目标仓颉包名。

对于 JDK API 的模块化镜像，可查阅 JDK 模块列表及其依赖图，收集各模块导出包列表，按增量镜像命令行模板逐模块运行生成器。`java.base` 模块的许多实现依赖内部 JDK 包，但镜像类型仅为声明、不暴露私有成员，生成器不会处理那些内部包中的类文件。

### 闭包深度限制

除 [处理循环导入依赖](#处理循环导入依赖) 所述的包间循环依赖外，标准 Java API 及众多流行库还具有依赖闭包庞大的特点。例如，若调用镜像生成器为空枚举类型生成镜像：

```java
public enum E {}
```

则生成器还会连带生成 `java.lang.Enum` 及其在标准 Java 库中的全部依赖，合计约 300 个镜像类型：

```text
AbstractInterruptibleChannel.cj
AbstractStringBuilder.cj
AccessControlContext.cj
AccessMode.cj
AccessibleObject.cj
...
ZoneOffsetTransitionRule.cj
ZoneRules.cj
ZonedDateTime.cj
```

cjc 会为所有这些镜像类型生成胶水代码，而实际程序大多用不到——在 Android 等移动平台上尤其不可接受。

要求开发者精确列出需镜像的类型与成员并不可行。不过，在计算依赖闭包时**限制搜索深度**即可显著减少生成量（往往一个数量级以上）。例如，将上述空枚举 `E` 的镜像深度限制为 2 时，除 `E` 自身外仅再生成六个镜像类型：枚举基类 `java.lang.Enum`、其实现的三个接口，以及两个方法的返回类型：

```text
Class.cj
Comparable.cj
Constable.cj
E.cj
Enum.cj
Optional.cj
Serializable.cj
```

深度限制规则如下：

* 镜像**类型**集合始终包含：
    - 映射为仓颉值类型的 Java 基本类型；
    - 命令行显式指定的 [类与接口](#java-class-与-interface-类型)；
    - 分别预镜像为 `JObject` 和 `JString` 的 `java.lang.Object` 与 `java.lang.String`（见[互操作库预置 API 参考](#互操作库预置-api-参考)）；
    - 元素类型属于上述类型的 [数组类型](#java-数组类型)，镜像为 `JArray<T>`。

* 对集合中类型的**成员**按以下规则过滤：
    - 继承成员不作为“自有”成员镜像，即便其父类型不在集合中；
    - 字段类型不在集合中的字段省略；
    - 签名中含不在集合中的类型的方法省略；
    - 成员类型与顶层类型同等对待：除非命令行显式指定，否则仅在有限深度闭包计算中被纳入集合时才镜像。

* [命令行](#java-镜像生成器命令行语法) 系统属性 `-Dgen.closure.depth` 设置显式指定类/接口类型的深度上限。详见 [系统属性](#系统属性)。

* 某类型 `T` 的深度上限为 **0** 时，**不**将 `T` 所依赖的任何类型加入集合，**包括其超类型**。此时仅镜像 `T` 自身在 `T` 中声明的非 `private` 成员（签名中的类型须已在集合中，如 `JString` 会自动纳入）。

  **示例：**

  ```java
  // A.java
  public class A {
      public void f(String s) {}
  }

  // B.java
  public class B extends A {
      public void g(String s) {}
  }
  ```

  若指定镜像生成器以深度 0 镜像类 `B`，则仅生成 `B` 及其方法 `g(String)` 的镜像（因 `JString` 会自动纳入镜像类型集合）。同时，`B` 从 `A` 继承的方法 `f(String)` **不可**用：

  <!-- compile -->
  ```cangjie
  // B.cj
  // ...
  @JavaMirror["B"]
  public open class B {
      public init()

      public open func g(s: ?JString): Unit
  }
  ```

* 深度上限为**正整数** `N` 时，除上述外还将以深度 `N-1` 纳入：`T` 的全部（递归）超类型；`T` 自身声明的非 `private` 字段类型；`T` 的非 `private` 构造函数的形参类型；`T` 自身声明的非 `private` 方法的形参与返回类型（**不**扫描继承方法）。若某类型已在集合中但深度更低，则提升至 `N-1`。对新纳入类型递归重复此过程。

  **示例：**

  ```java
  // A.java
  public class A {
      public void f(C c) {}
  }

  // B.java
  public class B extends A {
      public void g(D d) { }
  }

  // C.java
  public class C { }

  // D.java
  public class D extends C {}
  ```

  若指定镜像生成器以深度 1 镜像类 `B`，则生成 `B`、其超类 `A`，以及 `B.g(D)` 唯一形参的类型 `D` 的镜像。但 `A` 与 `D` 的镜像深度为 0，故类 `C` 及方法 `A.f(C)` **不会**被镜像：

  <!-- compile -->
  ```cangjie
  // A.cj
  // ...
  @JavaMirror["A"]
  public open class A {
      public init()
  }

  // B.cj
  // ...
  @JavaMirror["B"]
  public open class B <: A {
      public init()

      public open func g(d: ?D): Unit
  }

  // D.cj
  // ...
  @JavaMirror["D"]
  public open class D {
      public init()
  }
  ```

  最终将深度上限设为 **2** 时，镜像生成器输出中还将包含类 `C` 及方法 `A.f(C)` 的镜像。

## Java 镜像生成器参考

 Java 镜像生成器依赖 JDK17，在使用前请确保开发者本地已安装 JDK17 并配置好相应的 `PATH` 环境变量。

开发者需要知道本地所有需要为之生成镜像的 jar 文件的路径和 `.class` 文件所在目录的路径，这包括安卓标准库的 jar，以及安卓应用运行时的类路径等。

### Java 镜像生成器命令行语法

共有两种使用 Java 镜像生成器的方式：

* 默认模式：

```bash
java [system-properties]                               \
    -jar ${CANGJIE_HOME}/tools/bin/java-mirror-gen.jar \
    [options] type-names
```

这一种方式用于为一到若干个 Java 类或接口，及其所依赖的所有其他 Java 类型生成镜像。

* 单 JAR 包模式：

```bash
java -Djar.mode=true [system-properties]               \
    -jar ${CANGJIE_HOME}/tools/bin/java-mirror-gen.jar \
    [options] jar-file
```

这一种方式则用于为指定的 jar 文件 `jar-file` 中包含的所有 `.class` 文件中的所有类型，以及其所依赖的所有其他 Java 类型生成镜像。注意：后者这些依赖的 Java 类型所在的 `.class` 文件可能并不在指定的 Jar 包中，而存在于其他类路径上（如果确实存在）。

在上述命令中：

* `options` 是 Java 镜像生成器的若干 [命令行选项](#java-镜像生成器命令行参数)。

* `type-names` 是需要为之生成镜像的 Java 类和接口的完全限定名。

* `jar-file` 是单个 jar 文件的路径。

### Java 镜像生成器命令行参数

* `--boot-class-path` `pathname`   (必选)

    `pathname` 必须是用于构建安卓应用 Java 部分的安卓 SDK 中 `android.jar` 文件的路径。该选项必须指定，且必须指向一个存在的文件。

* `-d ``directory`

    `directory` 指定了一个目录路径，镜像生成器将把生成的镜像仓颉源文件放置在该目录中，放置的目录结构将遵循 CJMP 相关的要求。如果该选项未被指定，默认为当前目录。

* `-cp ``path`, `--class-path` `path`    (必选)

    `path` 是一系列的目录路径、jar 文件路径或 zip 文件路径，不同路径之间使用冒号 `:`（非 Windows）或分号 `;`（Windows）分隔。镜像生成器在为指定的类型 `type-names` 及其依赖类型生成镜像时，将会在这些路径下尝试搜索这些类型。`path` 必须以 `android.jar` 文件的路径开头。

* `-h`, `-?`, `--help`

    指定该选项， Java 镜像生成器将打印帮助信息，简要解释各命令行选项的用法然后终止。

### 系统属性

* `-Dpackage.mode=true`   (必选)

    在当前版本中，必须指定。

* `-Dpackage.name=some_name`   (必选)

    `some_name` 是仓颉包名，镜像生成器将把所有本次生成的镜像仓颉类型置于该包中。详情请参见 [单包模式](#单包模式)。

* `-Dgen.closure.depth=number`

    `number` 为非负十进制整数值，限制在确定需镜像的类型及其成员时，依赖图扫描的深度。详情请参见 [闭包深度限制](#闭包深度限制)。

* `-Djar.mode=true`

    启用单 JAR 包模式（参见上文 [命令行语法](#java-镜像生成器命令行语法)）。

* `-Djar.mode.packages=pathname`

    `pathname` 为包含 Java 包名列表的纯文本文件路径。仅镜像所列包及其依赖中的类型。须配合单 JAR 包模式使用。详情请参见 [增量镜像命令行语法](#增量镜像命令行语法)。

* `-Dimports.config=pathname`

    `pathname` 为导入映射文件路径，累积历次镜像生成的类型映射。须配合单 JAR 包模式及 `-Djar.mode.packages` 使用。详情请参见 [增量镜像命令行语法](#增量镜像命令行语法)。

### Java 镜像生成器使用示例

```shell
java -Dpackage.mode=true -Dpackage.name=com.example \
     -jar ${CANGJIE_HOME}/tools/bin/java-mirror-gen.jar \
    --boot-class-path ${ANDROID_SDK}/platforms/android-35/android.jar \
    --class-path ${ANDROID_SDK}/platforms/android-35/android.jar:App.jar \
    -d ./mirrors \
    com.example.subpkg1.A com.example.subpkg2.B
```

```shell
java -Djar.mode=true -Dpackage.mode=true -Dpackage.name=com.example \
     -jar ${CANGJIE_HOME}/tools/bin/java-mirror-gen.jar \
    --boot-class-path ${ANDROID_SDK}/platforms/android-35/android.jar \
    --class-path ${ANDROID_SDK}/platforms/android-35/android.jar:./Lib.jar \
    -d ./mirrors \
    App.jar
```

## 互操作库预置 API 参考

 CJMP 所提供的 Java 互操作库 `java.lang` 中预置了 `java.lang.Object` 和 `java.lang.String` 这两个基础 Java 类的镜像类型，以及一个对标 Java 数组的泛型镜像类型。

由于 `Object`、`String` 和 `Array` 在仓颉 `std.core` 包中均有同名的类型，为了避免 [Java 镜像类型名称冲突](#java-镜像类型名称冲突)，这几个 Java 类型的镜像类型分别改名为 `JObject`、`JString` 和 `JArray<T>`。

为了提升互操作使用体验，这几个镜像类型中特别重命名了部分成员函数，也新增了若干成员函数，详情请参见下文阐述。

### `java.lang.JObject`

`java.lang.JObject` 是整个 Java 镜像类和互操作类继承层次结构的根类，其本身是 `java.lang.Object` 的 Java 镜像类，不过删除了部分不支持的成员函数，重命名或新增了部分成员函数，以更好地与仓颉标准库保持协调。

> **注意**：
>
> 被删除的成员函数是 `clone()`、`finalize()` 和 `getClass()`。由于 `JObject` 是所有镜像类的根类，这些被删除的成员函数在所有其他镜像类中自然也不可用。

<!-- compile -->
```cangjie
package java.lang

@JavaMirror["java.lang.Object"]
open class JObject {
    public open func equals(obj: ?JObject): Bool

    public func hashCode(): Int64
    @ForeignName["hashCode"]
    public open func hashCode32(): Int32

    public func toString(): String
    @ForeignName["toString"]
    public open func toJString(): JString

    public func wait(timeoutMillis: Int64): Unit
    public func wait(timeoutMillis: Int64, nanos: Int32): Unit
    public func wait(): Unit
    public func notifyAll(): Unit
    public func notify(): Unit
}
```

`equals` 以及所有 `wait/notify` 相关实例成员函数都是对应 Java 实例方法的镜像。

 Java 的 `java.lang.Object` 的 `hashCode` 方法的返回类型是 `int`，对应仓颉 `Int32`，而仓颉标准库中的 `hashCode` 成员函数的返回类型则是 `Int64`。因此，`java.lang.JObject` 内置了两个不同的 `hashCode` 成员函数来解决这个差异：

<!-- compile -->
```cangjie
public func hashCode(): Int64
```

该实例成员函数将调用 Java 实例方法 `hashCode`，并将 Java 侧的 32 位的 `int` 返回值强制类型转换为 `Int64`，从而更符合仓颉开发者的习惯预期。而另一个实例成员函数：

<!-- compile -->
```cangjie
@ForeignName["hashCode"]
public open func hashCode32(): Int32
```

则是原本的 Java 方法 `hashCode` 的镜像，为避免名称冲突而重命名为 `hashCode32`。

<!-- compile -->
```cangjie
public func toString(): String
```

该实例成员函数将调用原本的 Java 的 `toString` 实例方法，并将返回值转换为仓颉 `String`。Java 侧 `toString` 方法的调用极低概率会返回 `null`，但为了方便使用，并没有为此进行 `Option<T>` 装包，而是在返回 `null` 时，仓颉侧的 `toString` 实例成员函数将抛出异常。

> **注意**：
>
> 该成员函数的返回类型是仓颉 `String` 类型，这明显违反了规格，因为规格要求镜像类的所有 `public` 成员函数的形参类型和返回类型必须是 Java 兼容类型。之所以可行是因为 cjc 针对该成员函数有专门的支持。

<!-- compile -->
```cangjie
@ForeignName["toString"]
public open func toJString(): JString
```

该实例成员函数是 Java 原方法 `toString` 的镜像类型，为避免名称冲突而改名为 `toJString`。

同上述原因， Java 的 `toString` 方法极低概率返回 `null`，故 `toJString` 函数返回类型设计为 `JString` 而不是 `?JString`。

### `java.lang.JString`

<!-- compile -->
```cangjie
package java.lang

@JavaMirror["java.lang.String"]
open class JString {
    // ...
    public init(cjString: String)
    // ...
}
```

<!-- compile -->
```cangjie
public init(cjString: String)
```

将仓颉 `String` 实例转换为 `JString`。

> **注意**：
>
> 该构造函数的形参类型是仓颉 `String` 类型，这明显违反了规格，因为规格要求镜像类的所有 `public` 构造函数的形参类型必须是 Java 兼容类型。之所以可行是因为 cjc 针对 `JString` 有专门的支持。

`JString` 从 [`JObject`](#javalangjobject) 继承得到以下成员函数：`equals`、`hashCode`、`hashCode32`、`toString`、`toJString`、`wait/notify` 等。

### `java.lang.JArray<T>`

`JArray<T>` 是互操作库中内置的特殊泛型镜像类型，用作所有 Java 数组类型的镜像类型，类型变元 `T` 必须是能够映射至 Java 基本数据类型的值类型（例如 `Int32` 和 `Bool` 等）、镜像类型或互操作类。

当前版本存在以下使用限制：

* 不支持数组元素类型为可空引用的类型。换句话说，假设 `T` 是镜像类型或互操作类，则 `JArray<T>` 类型是支持的，而对元素类型进行了 `Option<T>` 装包的 `JArray<?T>` 类型则是不支持的。

* 不支持变量和形参的类型为可空引用的 `JArray<T>` 类型。也就是说，即便 `JArray<T>` 是支持的，`?JArray<T>` 却是不支持的。

`JArray<T>` 提供的 API 相比仓颉标准库类型 `Array<T>` 较为受限，除 `JArray<T>` 构造函数外，仅提供了用于获取数组长度的 `length` 实例成员属性、数组元素访问的操作符重载函数，以及由 `java.lang.JObject` 继承而来的若干成员函数。

<!-- compile -->
```cangjie
public init(length: Int32)
```

实例化一个长度为 `length` 的 Java 数组。

<!-- compile -->
```cangjie
public prop length: Int32
```

获取 Java 数组的元素个数。

<!-- compile -->
```cangjie
public operator func [](index: Int32): T
public operator func [](index: Int32, value!: T): Unit
```

数组元素访问 `[]` 操作符重载函数。

`JArray<T>` 从 [`JObject`](#javalangjobject) 继承得到以下成员函数：`equals`、`hashCode`、`hashCode32`、`toString`、`toJString`、`wait/notify` 等。

## 运行时行为

### Java 类加载器

所有镜像类型与互操作类对应的 Java 类型，必须由**同一个**类加载器加载。

### 初始化

当控制流首次进入仓颉代码时，所有全局及 `static` 仓颉变量初始化完毕，所有仓颉类型的静态初始化器完成调用。这发生在第一个互操作类包装类初始化时。

Java 类与接口在首次使用时初始化，触发条件包括：实例化、调用 `static` 方法、对 `static` 字段赋值、读取非常量 `static` 字段、子类初始化、实现类的初始化（仅当接口声明了 `default` 方法时），或某些反射调用。

> **注意**：
>
> 上述仓颉初始化代码**不得**以任何方式使用镜像类型或互操作类，否则将导致死锁或崩溃。

### 终结器

#### Java 终结器

互操作类不得实现 `java.lang.Object` 的 `finalize()` 方法。内置镜像 [`java.lang.JObject`](#javalangjobject) 甚至未声明该方法。在互操作类中定义 `finalize()` 将与互操作支撑代码产生名称冲突并导致编译错误。

若在互操作类中尝试定义 Java 终结器：

<!-- compile -->
```cangjie
@JavaImpl
public class C {
    // ...
    public func finalize(): Unit {
        // ...
    }
}
```

将因互操作支撑代码自身使用 Java 终结器而产生名称冲突，导致编译错误。

#### 仓颉终结器

镜像类声明中不得包含仓颉终结器（`~init()`）。[互操作类](#互操作类)是否可包含终结器**尚待确定**。

### 异常

Java 与仓颉均支持异常。在 CJMP 双向互操作场景中，调用链可能在两种语言的方法/函数间多次传递，栈上 Java 与仓颉帧可能交错；异常抛出时，栈展开过程可能跨越语言边界。

若从仓颉代码调用 Java 构造方法或方法时因未捕获的 Java 异常而异常完成，将立即抛出具有以下特征的仓颉异常：

* 若 Java 异常为 `Error`（`java.lang.Error` 或其子类），仓颉异常亦为 `Error`；否则为 `Exception`。
* `message` 属性的值**包含** Java 异常 `getMessage()` 返回的字符串。
* `getStackTrace()` 返回的数组元素**尚待确定**。

若该仓颉异常在抵达 Java 帧之前未被捕获，则重新抛出原始 Java 异常，过程重复。（若线程由仓颉 `spawn` 创建，栈上可能无 Java 帧，此时行为**尚待确定**。）

若异常在仓颉代码内部抛出（非上述跨语言调用所致），且栈展开抵达 Java 帧，则抛出具有以下特征的 Java 异常（`java.lang.Exception`）：

* `getMessage()` 返回的字符串包含仓颉异常的 `message` 属性值。
* `getStackTrace()` 返回的数组元素**尚待确定**。

该 Java 异常除消息外不保留原仓颉异常信息，此后按普通 Java 异常处理（参见上文）。

无法在仓颉代码中 `throw` Java 异常，亦无法在 Java 代码中 `throw` 仓颉异常。

### 内存管理

Java 与仓颉对象分别驻留于各自堆中，由各自运行时管理。互操作库与桥接代码确保：只要另一语言的可访问变量或数据结构中仍持有引用，一侧堆中的对象就不会被垃圾回收。

> **注意**：
>
> 跨语言引用一致性机制有两项重大限制，使用互操作功能时须始终牢记：
>
> 1. Android VM 对同时存在于 VM **外部**（尤其是仓颉变量与数据结构中）的 Java 堆对象引用数设有硬性上限。应尽量避免在仓颉全局变量及长生命周期数据结构中保存此类引用（**包括**镜像类型与互操作类的实例）。这些引用在仓颉侧不再需要后的释放时机完全取决于仓颉垃圾回收器；即便从开发者角度看引用已短暂失效，也未必立即释放。在循环中遍历大型 Java 数组或集合可能在仓颉 GC 运行前创建大量此类引用，使应用触及 Android VM 上限并异常终止。
>
> 2. Java 与仓颉垃圾回收器各自仅在其托管环境内运行，跨语言循环引用可能导致内存泄漏。应避免创建此类引用，或在对象从 Java 与仓颉代码两侧均不可达之前主动打破循环。

### 线程

由仓颉 `spawn` 创建的线程在首次跨越语言边界时（实例化镜像/互操作类、调用未在仓颉中（重）定义的 `static` 方法等）自动以守护线程身份附着到 JVM。

当前版本中，在**非** Java VM 创建的线程中进行互操作，须用 `exclusiveScope<T>` 包裹相关代码。例如以下代码：

<!-- compile -->
```cangjie
let x = SomeJavaClass()
return x.foo().bar() + 1  // 设 bar() 返回 Int32
```

须改写为：

<!-- compile -->
```cangjie
return exclusiveScope<Int32> {
    let x = SomeJavaClass()
    return x.foo().bar() + 1  // 设 bar() 返回 Int32
}
```

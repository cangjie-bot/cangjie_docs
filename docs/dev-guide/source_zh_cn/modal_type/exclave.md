# exclave 表达式

> **说明：**
>
> local 模式是实验特性。本节示例都按开启 local 模式编写，编译时需要加上 `--experimental --enable-local-mode` 编译选项，参见[开启 local 模式](./modal_overview.md#开启-local-模式)。

## 为什么需要 exclave

每个函数都有自己的 local region，函数内创建的 local 对象在函数返回时随之释放。这带来两个问题：

- 函数无法把内部创建的 local 对象交给调用者，因为该对象在函数返回时就会被释放；
- 函数无法让调用者传入的 `@ local!` 参数引用函数内部创建的 local 对象，因为参数对象属于调用者的 local region，比函数内部的对象活得更久。

下例中，`normalize` 想把调用者传入的数组中的第一个路径替换为一个新建的字符串，编译器会报错：

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func normalize(paths: Array<String> @ local!) {
    let fixed: String @ local! = "/index.html"
    paths[0] = fixed // Error, fixed belongs to this function's region
}
```

`fixed` 属于 `normalize` 的 local region，而 `paths` 属于调用者的 local region；不能让调用者的数组引用在 `normalize` 返回时就被释放的字符串。

exclave 表达式用于解决这类问题。

## exclave 的含义

exclave 表达式的形式为 `exclave { 块 }`，执行过程如下：

1. 当前函数的 local region 先行结束，其中的 local 对象被释放；
2. 接着执行块中的代码，块中新创建的 local 对象改为分配在调用者的 local region 中；
3. 块执行完毕后，像 `return` 一样把块的值返回给调用者。

exclave 执行完就会离开函数，因此 exclave 表达式本身的类型是 `Nothing`，块中最后一个表达式的类型需要符合函数的返回类型。

在 exclave 中，`@ local!` 参数与块内新建的 local 对象位于同一个 local region（即调用者的 local region），因此可以互相赋值，块内新建的 local 对象也可以作为返回值返回。

### 修改 @ local! 参数

把上面的 `normalize` 改为在 exclave 中创建并赋值：

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

执行结果为：

```text
/index.html
```

### exclave 修饰符

用 `exclave` 修饰函数或构造函数，相当于将其整个函数体包在一个 `exclave { ... }` 块中。块中可以包含多条语句。

例如，`exclave func f() { ... }` 可以展开为 `func f() { exclave { ... } }`；`exclave init` 的展开方式相同。下面使用修饰符写法：

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

执行结果为：

```text
/new
```

`this @ local!` 和 `this @ local?` 的构造函数必须写成 exclave，参见[构造 local 实例](./local_mode_rules.md#构造-local-实例)。

## 返回 local 对象

在 exclave 中构造的 local 对象可以返回给调用者，调用者拿到的对象位于调用者自己的 local region 中，调用者返回时才释放：

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

执行结果为：

```text
285
```

不使用 exclave 时，函数内创建的 local 对象不能返回：

<!-- compile.error -->
<!-- cfg="--experimental --enable-local-mode" -->

```cangjie
func makeSquares(n: Int64): Array<Int64> @ local! {
    let squares = Array<Int64>(n, {i => i * i}) @ local!
    squares // Error, a @ local! value created in this function cannot be returned
}
```

如果需要把传入的 `@ local!` 参数放进新的数组或元组后返回，也应在 exclave 中构造。下面将字符串参数放入新数组；数组及其元素都位于调用者的 local region：

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

执行结果为：

```text
hello
hello
```

## 使用条件

exclave 只能用于以下函数（包括 lambda 和构造函数）：

- 有非 `Copyable` 类型的 `@ local!` 参数（包括 `this @ local!`）；
- 返回类型为非 `Copyable` 类型的 `@ local!` 或 `@ local?`；
- `this @ local!` 或 `this @ local?` 的构造函数。

其他函数中使用 exclave 会报错。`abstract` 函数没有函数体，也不能用 `exclave` 修饰：

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

## 使用限制

exclave 有以下限制：

- 同一个函数中的 exclave 不能嵌套：外层 exclave 已经结束了函数的 local region，内层无法再结束一次。
- exclave 块中不能访问在块外声明的、非 `Copyable` 类型的 `@ local!` 或 `@ local?` 局部变量，因为当前函数的 local region 在进入 exclave 时已经结束，这些变量引用的对象可能已被释放。`Copyable` 类型的变量不受此限制。
- 不能在 spawn 块、`main` 函数、终结器和静态初始化器中直接使用 exclave；也不能在 try 或 catch 块中用 exclave 结束包含该块的函数。

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

下面的函数都有非 `Copyable` 类型的 `@ local!` 参数，满足 exclave 的使用条件，但仍不能在 try 块或 catch 块中使用 exclave：

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

spawn 块、`main`、终结器和静态初始化器中不能直接使用 exclave：

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

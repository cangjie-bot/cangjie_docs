# CJO 产物说明

本章介绍仓颉编程语言编译过程中生成的 CJO（Cangjie Object）产物的作用和相关信息。

## 什么是 CJO 产物

CJO（Cangjie Object）是仓颉编译器生成的二进制格式的AST（抽象语法树）文件，主要承载类似C语言头文件的作用。CJO 文件包含了包的接口信息、类型定义、函数声明等元数据，用于在编译时进行语义检查和类型检查。除了接口元数据外，CJO 文件在某些特殊场景下也会包含部分实现内容，如内联函数、部分表达式等需要在编译时展开的代码片段。

## CJO 文件的作用

### 1. 接口描述

CJO 文件记录了包的公共接口信息，包括类型定义、函数签名、常量声明等，类似于C语言中的头文件作用。

### 2. 编译时依赖解析

在编译时，编译器通过 CJO 文件获取被导入包的接口信息，进行语义检查和类型检查，而无需重新解析源代码。

### 3. 模块化支持

CJO 文件实现了模块间的解耦，允许包的接口和实现分离，支持独立编译和分发。

### 4. 跨包依赖管理

通过 CJO 文件，编译器可以有效管理包之间的依赖关系，确保类型安全和接口一致性。

## CJO 文件的生成

### 编译包时自动生成

使用 `cjc` 编译包时会自动生成对应的 CJO 文件：

```shell
$ cjc -p mypackage --output-type=staticlib
```

编译器会生成 `mypackage.cjo` 文件和相应的库文件。

### 指定输出目录

可以使用 [`--output-dir`](compile_options.md#--output-dir-value-frontend) 选项指定 CJO 文件的输出目录：

```shell
$ cjc -p mypackage --output-type=staticlib --output-dir ./build
```

### 宏包的 CJO 文件

对于[宏包](../Macro/defining_and_importing_macro_package.md)，编译时会生成带有宏属性的 CJO 文件：

```shell
$ cjc --compile-macro macro_define.cj --output-dir ./target
```

## CJO 文件的使用

### 与 --import-path 配合使用

CJO 文件主要通过 [`--import-path`](compile_options.md#--import-path-value-frontend) 选项来指定搜索路径。假设有以下目录结构：

```text
.
├── libs
|   └── myModule
|       ├── log.cjo
|       └── libmyModule.a
└── main.cj
```

可以通过以下命令使用 CJO 文件：

```shell
$ cjc main.cj --import-path ./libs libmyModule.a
```

编译器会使用 `./libs/myModule/log.cjo` 文件来对 `main.cj` 文件进行语义检查与编译。

### 在 CJPM 中的使用

在 `cjpm.toml` 配置文件中，可以通过 `bin-dependencies` 配置 CJO 文件依赖：

```toml
[target.x86_64-unknown-linux-gnu.bin-dependencies]
path-option = ["./test/pro0", "./test/pro1"]

[target.x86_64-unknown-linux-gnu.bin-dependencies.package-option]
"pro0.xoo" = "./test/pro0/pro0.xoo.cjo"
"pro0.yoo" = "./test/pro0/pro0.yoo.cjo"
```

### 依赖扫描

可以使用 [`--scan-dependency`](compile_options.md#--scan-dependency-frontend) 选项扫描 CJO 文件的依赖关系：

```shell
$ cjc --scan-dependency mypackage.cjo
```

## CJO 文件的特点

### 1. 二进制 AST 格式

CJO 文件采用二进制格式存储抽象语法树信息，相比文本格式具有更高的解析效率。

### 2. 接口信息完整性

CJO 文件包含完整的包接口信息，包括类型定义、函数签名、可见性等元数据。

### 3. 编译器版本兼容性

自LTS版本开始，仓颉编译器承诺向后兼容，后续版本的编译器能够正确处理之前版本生成的 CJO 文件。在LTS版本之前，不同版本编译器生成的 CJO 文件可能存在兼容性问题。

### 4. 宏信息支持

对于宏包，CJO 文件会携带宏属性信息，支持宏的编译时展开。

## 使用建议

### 1. 使用 CANGJIE_PATH 环境变量

可以通过设置 [`CANGJIE_PATH`](compile_options.md#--import-path-value-frontend) 环境变量指定 CJO 文件的搜索路径，[`--import-path`](compile_options.md#--import-path-value-frontend) 选项具有更高优先级。

### 2. 二进制库分发

将 CJO 文件与对应的库文件（.a 或 .so）一起分发，提供给其他模块作为二进制依赖。

### 3. 版本兼容性管理

自LTS版本开始，仓颉编译器提供向后兼容保障，新版本编译器能够正确处理之前版本生成的 CJO 文件。在LTS版本之前，建议在升级编译器后重新生成 CJO 文件。

### 4. 路径管理

使用 [`--trimpath`](compile_options.md#--trimpath-value-frontend) 选项可以移除 CJO 文件中的路径前缀信息，便于跨环境使用。

## 常见问题

### Q: CJO 文件与源文件不同步怎么办？

A: 删除过期的 CJO 文件，重新编译对应的包即可。

### Q: 如何查看 CJO 文件的依赖信息？

A: 可以使用 [`cjc --scan-dependency`](compile_options.md#--scan-dependency-frontend) `mypackage.cjo` 命令查看依赖关系。

### Q: 为什么找不到 CJO 文件？

A: 检查 [`--import-path`](compile_options.md#--import-path-value-frontend) 设置是否正确，确保 CJO 文件在指定的搜索路径中。

### Q: CJO 文件可以跨平台使用吗？

A: CJO 文件主要包含接口信息，通常可以跨平台使用，但建议在目标平台重新生成。

## 相关选项

以下编译选项与 CJO 文件生成和使用相关：

- [`--import-path <value>`](compile_options.md#--import-path-value-frontend)：指定导入模块的 AST 文件搜索路径
- [`--output-dir <value>`](compile_options.md#--output-dir-value-frontend)：控制 CJO 文件的保存目录
- [`-p, --package`](compile_options.md#--package--p-frontend)：编译包时自动生成对应的 CJO 文件
- [`--scan-dependency`](compile_options.md#--scan-dependency-frontend)：扫描 CJO 文件的依赖关系
- `--compile-macro`：编译宏包生成带有宏属性的 CJO 文件
- [`--trimpath <value>`](compile_options.md#--trimpath-value-frontend)：移除 CJO 文件中的路径前缀信息

## 注意事项

1. **搜索路径**：确保通过 [`--import-path`](compile_options.md#--import-path-value-frontend) 或 [`CANGJIE_PATH`](compile_options.md#--import-path-value-frontend) 正确设置 CJO 文件的搜索路径。

2. **文件同步**：CJO 文件必须与对应的库文件保持同步，版本不匹配可能导致编译错误。

3. **版本兼容性**：自LTS版本开始，仓颉编译器承诺向后兼容，新版本能够正确处理之前版本生成的 CJO 文件。在LTS之前的版本中，升级编译器后建议重新生成 CJO 文件。

4. **宏包处理**：宏包的 CJO 文件需要与对应的动态库文件配合使用，路径设置要保持一致。

5. **依赖解析**：编译器通过 CJO 文件进行语义检查，确保导入的包接口信息完整准确。
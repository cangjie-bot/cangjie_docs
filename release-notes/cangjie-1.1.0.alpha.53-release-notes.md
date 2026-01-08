# Cangjie 1.1.0.alpha.53 Release Notes

## 版本介绍

该版本版本号 Cangjie 1.1.0.alpha.53, 构建时间 xxx，构建 id xxx.

本版新增 xx 等特性，删除 xx 等特性，变更特性和修复问题做一个总结，描述需要开发者关注的重大特性和问题修复。

* 本版本新增 xxx，xxx 等特性。

* 删除 xxx，xxx 等特性。

* xxx 等特性发生了变更。

* 修复了 xxx 等若干 bug。

详见后续章节介绍。

## 编译器

### 新增特性

* 互操作支持 Java 使用 Cangjie 泛型类型，详见[仓颉-Java 互操作](../docs/dev-guide/source_zh_cn/multiplatform/cangjie-android-Java.md)。
* 互操作支持 Java 使用 Cangjie Lambda表达式类型，详见[仓颉-Java 互操作](../docs/dev-guide/source_zh_cn/multiplatform/cangjie-android-Java.md)。
* 互操作支持 Java 使用 Cangjie 元组类型，详见[仓颉-Java 互操作](../docs/dev-guide/source_zh_cn/multiplatform/cangjie-android-Java.md)。
* 互操作支持 ObjC 使用 Cangjie Open class 类型，详见[仓颉-ObjC 互操作](../docs/dev-guide/source_zh_cn/multiplatform/cangjie-ios-objc.md)。
* 互操作支持 ObjC 使用 Cangjie 泛型类型，详见[仓颉-ObjC 互操作](../docs/dev-guide/source_zh_cn/multiplatform/cangjie-ios-objc.md)。
* 支持 iOS x86_64 模拟器交叉编译能力，详见[`cjc` 编译选项](../docs/dev-guide/source_zh_cn/Appendix/compile_options.md) `--target` 编译选项的说明。
* platform 声明函数返回类型支持为 common 的子类型，详见[跨平台](docs/dev-guide/source_zh_cn/multiplatform/common_platform.md)。
* common/platform 有初始值的变量支持不显式指定类型，详见[跨平台](docs/dev-guide/source_zh_cn/multiplatform/common_platform.md)。
* 当 common class/struct/enum 或 common extend 的所有成员函数都包含实现时，支持省略 platform 声明，详见[跨平台](docs/dev-guide/source_zh_cn/multiplatform/common_platform.md)。
* 支持 common abstract class 匹配 platform sealed abstract class，详见[跨平台](docs/dev-guide/source_zh_cn/multiplatform/common_platform.md)。

### 变更特性

* common/platform 声明新增了注解一致性检查，详见[跨平台](docs/dev-guide/source_zh_cn/multiplatform/common_platform.md)。
* common/platform 不支持修饰主构造函数，详见[跨平台](docs/dev-guide/source_zh_cn/multiplatform/common_platform.md)。
* abstract platform class 不支持添加额外的抽象成员，详见[跨平台](docs/dev-guide/source_zh_cn/multiplatform/common_platform.md)。
* common sealed class 的所有直接子类型必须在同一个 source set，详见[跨平台](docs/dev-guide/source_zh_cn/multiplatform/common_platform.md)。
* 函数默认参数可在 common 或 platform 声明任一侧定义，详见[跨平台](docs/dev-guide/source_zh_cn/multiplatform/common_platform.md)。

### 修复问题

* 修复[【缺陷】运行时崩溃Thread "main" catched unhandled SIGSEGV (Segmentation fault) from runtime frame.](https://gitcode.com/Cangjie/UsersForum/issues/2815) 问题。

## 运行时

### 新增特性

【[issue262](https://gitcode.com/Cangjie/cangjie_runtime/issues/275)】仓颉运行时支持 ios-x86_64 模拟器构建以及运行

### 修复问题

【[issue275](https://gitcode.com/Cangjie/cangjie_runtime/issues/275)】优化`ohos cjthread`日志
【[issue288](https://gitcode.com/Cangjie/cangjie_runtime/issues/288)】修复使用“`--static`”编译选项时出现的“`__gnu_cxx::__verbose_terminate_handler()` 的多重定义”问题
【[issue289](https://gitcode.com/Cangjie/cangjie_runtime/issues/289)】修复 `windows` 静态链接运行报错的问题
【[issue293](https://gitcode.com/Cangjie/cangjie_runtime/issues/293)】修复上游包新增满足兼容性的更新时，部分程序会运行错误的问题
【[issue320](https://gitcode.com/Cangjie/cangjie_runtime/issues/320)】修复 `mac` 平台用户注册的信号处理函数中抛出未捕获异常，程序仍然因为该信号结束的问题
【[issue331](https://gitcode.com/Cangjie/cangjie_runtime/issues/331)】修复计算 `target` 所属 `ExtensionDef` 错误
【[issue332](https://gitcode.com/Cangjie/cangjie_runtime/issues/332)】修复 `lds` 脚本中合并 `text section` 问题，可能会导致 `section` 过多，链接失败问题

## 标准库

### 新增特性

    注意该节按需提供，若没有，则删除

### 变更特性

    注意该节按需提供，若没有，则删除

|变更前|变更后|适配举例|
|---|---|--|
|   |   |   |

### 修复问题

    注意该节按需提供，若没有，则删除 

## 工具链

### IDE插件

#### 新增特性

    注意该节按需提供，若没有，则删除

#### 变更特性

    注意该节按需提供，若没有，则删除

|变更前|变更后|适配举例|
|---|---|--|
|   |   |   |

#### 修复问题

    注意该节按需提供，若没有，则删除 

### cjpm

#### 新增特性

    注意该节按需提供，若没有，则删除

#### 变更特性

    注意该节按需提供，若没有，则删除

|变更前|变更后|适配举例|
|---|---|--|
|   |   |   |

#### 修复问题

    注意该节按需提供，若没有，则删除 

### cjdb

#### 新增特性

    注意该节按需提供，若没有，则删除

#### 变更特性

    注意该节按需提供，若没有，则删除

|变更前|变更后|适配举例|
|---|---|--|
|   |   |   |

#### 修复问题

    注意该节按需提供，若没有，则删除 

### cjfmt

#### 新增特性

    注意该节按需提供，若没有，则删除

#### 变更特性

    注意该节按需提供，若没有，则删除

|变更前|变更后|适配举例|
|---|---|--|
|   |   |   |

#### 修复问题

    注意该节按需提供，若没有，则删除 

### cjlint

#### 新增特性

    注意该节按需提供，若没有，则删除

#### 变更特性

    注意该节按需提供，若没有，则删除

|变更前|变更后|适配举例|
|---|---|--|
|   |   |   |

#### 修复问题

    注意该节按需提供，若没有，则删除 

### cjcov

#### 新增特性

    注意该节按需提供，若没有，则删除

#### 变更特性

    注意该节按需提供，若没有，则删除

|变更前|变更后|适配举例|
|---|---|--|
|   |   |   |

#### 修复问题

    注意该节按需提供，若没有，则删除 

### cjprof

#### 新增特性

    注意该节按需提供，若没有，则删除

#### 变更特性

    注意该节按需提供，若没有，则删除

|变更前|变更后|适配举例|
|---|---|--|
|   |   |   |

#### 修复问题

    注意该节按需提供，若没有，则删除 

### cjtrace-recover

#### 新增特性

    注意该节按需提供，若没有，则删除

#### 变更特性

    注意该节按需提供，若没有，则删除

|变更前|变更后|适配举例|
|---|---|--|
|   |   |   |

#### 修复问题

    注意该节按需提供，若没有，则删除 

### HLE

#### 新增特性

    注意该节按需提供，若没有，则删除

#### 变更特性

    注意该节按需提供，若没有，则删除

|变更前|变更后|适配举例|
|---|---|--|
|   |   |   |

#### 修复问题

    注意该节按需提供，若没有，则删除 

## 遗留问题

|issue|问题描述|影响及规避方案|
|---|---|--|
|   |   |   |

## 文档变更说明

    填写《仓颉开发指南》，《仓颉标准库API》，《仓颉工具使用指南》中重大的章节类资料变更
# Cangjie 1.1.5 Release Notes

## 版本介绍

`Cangjie 1.1.5` 为 `STS 1.1` 第三个发布版本，同时也是 `STS 1.1` 最后一个维护版本。

**相比 1.1.3 版本**，本版本新增编译器 `Mock` 特性与 `cjprof` 性能分析工具，标准库新增中心仓支持并增强 `TLS` 安全性（存在1处行为变更，见”变更说明“），同时修复了编译器、运行时、标准库、工具链的若干问题；各模块修复问题详见本文档各章节介绍。

## 变更说明

| 场景描述 | 变更前| 变更后 | 适配方案 |
| --- | --- | --- | --- |
| `stdx.net.http` 客户端发起 TLS 请求时，<br>服务器证书与请求主机名不匹配 | 不按请求主机名校验 TLS 服务器证书 | 按请求主机名校验服务器证书；<br>未显式指定 `serverName` 时从请求 URL 的 host 回退取值 | 证书与主机名不匹配的服务器，<br>需显式指定匹配的 `serverName` 或更换服务器证书 |

## 编译器

### 新增特性

- [`2636ad3f`](https://gitcode.com/Cangjie/cangjie_compiler/commit/2636ad3f) Mock 支持 foreign 函数的 inout 入参 mocking。
- [`f7c2d875`](https://gitcode.com/Cangjie/cangjie_compiler/commit/f7c2d875) + [`d8eb83e0`](https://gitcode.com/Cangjie/cangjie_compiler/commit/d8eb83e0) + [`30b61c19`](https://gitcode.com/Cangjie/cangjie_compiler/commit/30b61c19) Mock 实现重构：移至 sema 之后的 desugar 阶段，并清理 clang 告警。

### 修复问题

**代码生成与优化**

- [`a39ebfa7`](https://gitcode.com/Cangjie/cangjie_compiler/commit/a39ebfa7) 修复运算符翻译错误。
- [`e0f7f879`](https://gitcode.com/Cangjie/cangjie_compiler/commit/e0f7f879) 修复去虚化（devirt）后首个参数类型错误。
- [`1dabbdcc`](https://gitcode.com/Cangjie/cangjie_compiler/commit/1dabbdcc) 修复去虚化中无法使用 `Any&` 的问题。
- [`1e83f2eb`](https://gitcode.com/Cangjie/cangjie_compiler/commit/1e83f2eb) 修复链式访问嵌套泛型 `struct` 字段时字段偏移错误。
- [`55272e46`](https://gitcode.com/Cangjie/cangjie_compiler/commit/55272e46) 修复去虚化 int box 问题。
- [`8e00feab`](https://gitcode.com/Cangjie/cangjie_compiler/commit/8e00feab) 修复内联优化后去虚化阶段源 class 类型计算错误。
- [`f60c5544`](https://gitcode.com/Cangjie/cangjie_compiler/commit/f60c5544) `GenerateOptionLikeNonRef` 改用 `tuple.GetOperand(1)->GetType()` 获取关联类型。
- [`d4f6ebc9`](https://gitcode.com/Cangjie/cangjie_compiler/commit/d4f6ebc9) 为 mut wrapper 保留泛型信息。
- [`2f63b6e4`](https://gitcode.com/Cangjie/cangjie_compiler/commit/2f63b6e4) 移除 Android 的 global-struct tag。
- [`330a0e45`](https://gitcode.com/Cangjie/cangjie_compiler/commit/330a0e45) 修复 LTO 直接链接依赖 `.bc` 时宏 dylib 名错误。
- [`0c6319d2`](https://gitcode.com/Cangjie/cangjie_compiler/commit/0c6319d2) 回退 `dyld_fallback_library_path` 与空 `LD_LIBRARY_PATH` 相关修改。

**语义分析与前端**

- [`44208f88`](https://gitcode.com/Cangjie/cangjie_compiler/commit/44208f88) VArray 类型不匹配错误改经 sourceExpr 上报，定位更准确。
- [`f12fc3ce`](https://gitcode.com/Cangjie/cangjie_compiler/commit/f12fc3ce) 避免运行时 envsetup 中出现空 `LD_LIBRARY_PATH` 条目。
- [`c3cb838a`](https://gitcode.com/Cangjie/cangjie_compiler/commit/c3cb838a) 修复哈希常量命名冲突问题。
- [`40462f9f`](https://gitcode.com/Cangjie/cangjie_compiler/commit/40462f9f) 保留嵌套注解宏的 vtable 候选。
- [`7b950135`](https://gitcode.com/Cangjie/cangjie_compiler/commit/7b950135) + [`fbc22898`](https://gitcode.com/Cangjie/cangjie_compiler/commit/fbc22898) 重载决议排除 broken 声明，并澄清 broken 函数查找行为。
- [`14f01003`](https://gitcode.com/Cangjie/cangjie_compiler/commit/14f01003) 避免 const 函数中 pattern 变量触发 ICE。
- [`dff39652`](https://gitcode.com/Cangjie/cangjie_compiler/commit/dff39652) 拒绝 synchronized 块中的声明。
- [`688091ce`](https://gitcode.com/Cangjie/cangjie_compiler/commit/688091ce) 为字符串插值脱糖节点补充源码位置；修复 
- [`b893a0c2`](https://gitcode.com/Cangjie/cangjie_compiler/commit/b893a0c2) debug 构建下 SignalTest 回调未定义引用。
- [`6d469070`](https://gitcode.com/Cangjie/cangjie_compiler/commit/6d469070) intrinsic 函数作非一等公民使用时补充诊断信息。
- [`f8d9d717`](https://gitcode.com/Cangjie/cangjie_compiler/commit/f8d9d717) 修复 `IsSubtype` 循环替换缺陷。
- [`6a65b4e5`](https://gitcode.com/Cangjie/cangjie_compiler/commit/6a65b4e5) stdlib `.cjo` 未找到时报 `package_search_error`。

**CFunc 与互操作**

- [`b93b75da`](https://gitcode.com/Cangjie/cangjie_compiler/commit/b93b75da) 修复多版本 super type 时 CFunc 引用调用问题。
- [`8aa4c84d`](https://gitcode.com/Cangjie/cangjie_compiler/commit/8aa4c84d) CFunc 静态调用补充前缀 ref。
- [`3b050f59`](https://gitcode.com/Cangjie/cangjie_compiler/commit/3b050f59) 修复 cfunc 中虚方法调用问题。

**编译器后端（llvm-project）**

- [`d22cd1f78dd7`](https://gitcode.com/Cangjie/llvm-project/commit/d22cd1f78dd7) 识别 tbd v4 文件中的 arm64e.x1 目标。
- [`077e7c77a63c`](https://gitcode.com/Cangjie/llvm-project/commit/077e7c77a63c) CJDevirtualOpt 中折叠 `is_subtype`。
- [`49b4e81d0bff`](https://gitcode.com/Cangjie/llvm-project/commit/49b4e81d0bff) 避免生成非法的 CJ TBAA 元数据。

## 运行时

### 修复问题

- [`1b974e85`](https://gitcode.com/Cangjie/cangjie_runtime/commit/1b974e85) 修复 mutator 锁超时问题。
- [`dc0c8b5e`](https://gitcode.com/Cangjie/cangjie_runtime/commit/dc0c8b5e) Windows 下改用文档化 API 枚举运行时模块。
- [`cc39b83c`](https://gitcode.com/Cangjie/cangjie_runtime/commit/cc39b83c) reflection API 读取对象引用时补充 GC barrier。
- [`6b0bec80`](https://gitcode.com/Cangjie/cangjie_runtime/commit/6b0bec80) 修复 Windows 静态链接崩溃问题。
- [`d59fcf35`](https://gitcode.com/Cangjie/cangjie_runtime/commit/d59fcf35) 修复 LOS(Large Object Space) 导致的内存碎片问题。
- [`1f4ab31c`](https://gitcode.com/Cangjie/cangjie_runtime/commit/1f4ab31c) 修复 cjthread 抢占导致 GC 数据同步所需 safepoint 错误通过的问题。
- [`1acfbe2a`](https://gitcode.com/Cangjie/cangjie_runtime/commit/1acfbe2a) 修复 foreign thread 关闭 socket 失败问题。
- [`83f9c42e`](https://gitcode.com/Cangjie/cangjie_runtime/commit/83f9c42e) 修复 Android stack tagged pointer 问题。
- [`cd2a9893`](https://gitcode.com/Cangjie/cangjie_runtime/commit/cd2a9893) 修复文档与源码不一致问题。

## 标准库

### 新增特性

- [`798fdd4`](https://gitcode.com/Cangjie/cangjie_stdx/commit/798fdd4) 支持中心仓。

### 修复问题

- [`5142428`](https://gitcode.com/Cangjie/cangjie_stdx/commit/5142428) http 客户端按请求主机名校验 TLS 服务器证书（行为变更见"变更说明"）。
- [`186655f`](https://gitcode.com/Cangjie/cangjie_stdx/commit/186655f) 修复 http client 请求与超时的竞争问题。
- [`bbbd5e3`](https://gitcode.com/Cangjie/cangjie_stdx/commit/bbbd5e3) 防止 http1.1 client 中 `HttpBufferedBody` 并发访问。
- [`9ffb511`](https://gitcode.com/Cangjie/cangjie_stdx/commit/9ffb511) JSON 流读取增加 buffer 边界检查。
- [`979c385`](https://gitcode.com/Cangjie/cangjie_stdx/commit/979c385) 异常消息保持与原始信息一致。
- [`8ec85ad`](https://gitcode.com/Cangjie/cangjie_stdx/commit/8ec85ad) `stdx.net.tls` 恢复 OpenSSL fallback 探测。
- [`2d34a18`](https://gitcode.com/Cangjie/cangjie_stdx/commit/2d34a18) `-Wframe-larger-than` 告警阈值从 100000 提升至 110000。
- [`97fdd1e`](https://gitcode.com/Cangjie/cangjie_stdx/commit/97fdd1e) 文档代码块格式化。

## 工具链

### cjpm

#### 修复问题

- [`2fddac21`](https://gitcode.com/Cangjie/cangjie_tools/commit/2fddac21) 修复 cjpm 动态库 LTO 问题。
- [`877cb64c`](https://gitcode.com/Cangjie/cangjie_tools/commit/877cb64c) + [`d3c33beb`](https://gitcode.com/Cangjie/cangjie_tools/commit/d3c33beb) 修复 cjpm 交叉编译链接问题及 compile-option 问题。

### cjprof

#### 新增特性

- [`f523833a`](https://gitcode.com/Cangjie/cangjie_tools/commit/f523833a) 新增 cjprof 性能分析工具支持。

#### 修复问题

- [`fd6784a2`](https://gitcode.com/Cangjie/cangjie_tools/commit/fd6784a2) 修复 cjprof 文件异常处理。
- [`a721410b`](https://gitcode.com/Cangjie/cangjie_tools/commit/a721410b) 生成 `*.o` 文件时移除链接命令。

### cjcov

#### 修复问题

- [`777b5ddc`](https://gitcode.com/Cangjie/cangjie_compiler/commit/777b5ddc) + [`0f48514a`](https://gitcode.com/Cangjie/cangjie_compiler/commit/0f48514a) 移除 chir 中的 update pos；修复 cjcov 冗余行。

## 文档变更说明

### 开发指南与语言文档

- [`b6c5dc4`](https://gitcode.com/Cangjie/cangjie_docs/commit/b6c5dc4) 澄清泛型约束示例。
- [`b2e0929`](https://gitcode.com/Cangjie/cangjie_docs/commit/b2e0929) 为类型模式补充子类型转换说明。
- [`6588c21`](https://gitcode.com/Cangjie/cangjie_docs/commit/6588c21) 修复 HLE 类型错误。
- [`9ced22d`](https://gitcode.com/Cangjie/cangjie_docs/commit/9ced22d) 修复拼写错误。
- [`924600d`](https://gitcode.com/Cangjie/cangjie_docs/commit/924600d) 修复文档缺陷。
- [`ce522cb`](https://gitcode.com/Cangjie/cangjie_docs/commit/ce522cb) 文档标准化。
- [`01bf085`](https://gitcode.com/Cangjie/cangjie_stdx/commit/01bf085) 补充 `serverName` 从请求 URL host 回退取值的文档说明。

### 工具与构建文档

- [`63bf3c77`](https://gitcode.com/Cangjie/cangjie_compiler/commit/63bf3c77) [issue#519](https://gitcode.com/Cangjie/cangjie_compiler/issues/519) Android 构建文档中 macOS NDK 路径改为 Linux 路径。
- [`a6b8e7a4`](https://gitcode.com/Cangjie/cangjie_compiler/commit/a6b8e7a4) [issue#76](https://gitcode.com/Cangjie/cangjie_compiler/issues/76) 修正 `--product` 选项默认值的文档描述。
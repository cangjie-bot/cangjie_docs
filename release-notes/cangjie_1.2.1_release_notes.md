# Cangjie 1.2.1 Release Notes

## 版本介绍

本版本为 `STS` 版本，版本号 `1.2.1`。

相比 `1.2.0` 版本，本版本无新增特性，主要修复编译器、运行时、标准库与工具链的若干问题：
 * 编译器修复畸形类头、`foreign block` 宏展开、非法泛型约束、接口默认方法抽象重声明等一批语义问题，以及嵌套泛型 `struct` 字段偏移等代码生成问题。
 * 运行时修复 `schdfd` 初始化竞态、`descriptor lookup` UAF、`mutator` 锁超时、`cjthread` 抢占 `safepoint` 等并发与 `GC` 问题，优化 `LOS` 碎片。
 * 标准库增强 `TLS` 安全性（按请求主机名校验服务器证书，存在 1 处行为变更），修复 `HTTP` 请求与超时竞争问题。
 * 工具链修复`macOS` 语言服务器启动失败等问题。

详见后续各部分介绍。

---
## 编译器

### 修复问题

* 修复畸形类头之后类体丢失的问题，支持错误恢复后继续解析类体。
* 修复宏展开期间 `foreign block` 上下文丢失、以赋值方式复制 `foreign block` 修饰符的问题。
* 修复重复性检查中未跳过非法泛型约束名的问题。
* 修复接口默认方法被继承时未检测抽象重声明的问题。
* 修复 `tuple-destructuring` `lambda` 别名缓存未清理导致的问题。
* `VArray` 类型不匹配问题通过  `sourceExpr` 上报，并将括号位置复制到从 `VArray` 调用脱糖的 `ArrayExpr`。
* 为字符串插值脱糖节点补充源码位置。
* 修复 `LookUpImpl::SwapResultsWithInstTy` 中 `erase` 越界问题。
* 修复 `override` 检查错误。
* 修复链式访问嵌套泛型 `struct` 字段时字段偏移错误。
* 修复 `arm32` `cangjie epilogue` 隐式返回问题。
* 识别 `tbd v4` 文件中的 `arm64e.x1` 目标。

---

## 运行时

### 修复问题

* 修复 `schdfd slots` 初始化竞态问题。
* 修复 `cjthread` 抢占导致 `GC` 数据同步所需 `safepoint` 错误通过的问题。
* 修复 `GC` 潜在问题。
* 修复 `iOS` `schmon allocator` 问题。
* 修复 `timer heap` 调整释放大小问题。
* 修复 `descriptor lookup` UAF 问题。
* 反射 `API` 读取对象引用时补充 `GC` 屏障。
* 修复 `mutator` 锁超时问题。
* 修复 `Windows` 静态链接崩溃问题；`Windows` 下改用文档化 `API` 枚举运行时模块。
* `LOS` 碎片优化。
* 空 `treemap` 的 `forward`/`backward` 返回空迭代器。
---

## 标准库

### 变更特性

* `TLS` 安全性增强：`http` 客户端按请求主机名校验 `TLS` 服务器证书。

  | 变更前 | 变更后 | 适配举例 |
  |---|---|---|
  | 发起 `TLS` 请求时，不按请求主机名校验服务器证书 | 按请求主机名校验服务器证书；未显式指定 `serverName` 时，从请求 `URL` 的 `host` 回退取值 | 对证书与主机名不匹配的服务器，显式指定匹配的 `serverName`，或更换服务器证书 |

### 修复问题

* 修复 `HTTP` 请求与超时的竞争问题。
* 修正 `sort` 的 `run stack` 容量估算，恢复 `timsort run-stack` 不变式。

---

## 工具链

### IDE 插件

#### 修复问题

* 修复语言服务器在 `macOS` 上启动失败问题。

---

## 文档变更说明

* 修复 `hashset` 声明示例类型不匹配问题。
* 补充 `serverName` 从请求 `URL` `host` 回退取值的说明。
* 修正已知示例预期输出不匹配问题。

# Cangjie 1.0.6 Release Notes

## 版本介绍

该版本版本号 `1.0.6`（定版于 2026-05-13），本文档汇总 `release/1.0` 分支 26/02/11 ~ 26/11/07 期间的变更（含定版后的维护提交）。本版本**相对于 1.0.5 版本**无新增语言特性，以问题修复与稳定性加固为主：

- 编译器修复 `foreign block` 宏展开、畸形类头错误恢复、管道操作符类型推断 `ICE` 等一批语义问题，并加固去虚化、内联、`LTO` 等代码生成路径。
- 运行时修复 `schdfd slots` 初始化竞态、`foreign` 线程 `socket` 关闭、`dirHandle` 泄漏等并发与资源问题。
- 标准库修复 `HTTP`/`JSON` 健壮性问题（并发访问、超时竞争、缓冲边界检查）。
- 工具链修复 `cjpm` 依赖路径、`cjlint` VAR01 告警、语言服务器 `macOS` 环境适配等问题。

该版本包含使用仓颉开发应用的必备能力，包括编译器、运行时、标准库和工具链，欢迎各位开发者使用。如有任何问题，欢迎在仓颉社区[提交issue](https://gitcode.com/org/Cangjie/issues)。

## 编译器

### 修复问题

**foreign block 与宏展开**

【[`3a91a401`](https://gitcode.com/Cangjie/cangjie_compiler/commit/3a91a401) + [`f5b96d2e`](https://gitcode.com/Cangjie/cangjie_compiler/commit/f5b96d2e)】修复 `foreign block` 解析问题，保留宏展开中解析的 `foreign` 声明上的 `MACRO_EXPANDED_NODE` 标记。

【[`32383f0c`](https://gitcode.com/Cangjie/cangjie_compiler/commit/32383f0c) + [`8e74ff66`](https://gitcode.com/Cangjie/cangjie_compiler/commit/8e74ff66)】修复宏展开期间 `foreign block` 上下文丢失、以赋值方式复制 `foreign block` 修饰符的问题。

【[`25a4cccb`](https://gitcode.com/Cangjie/cangjie_compiler/commit/25a4cccb) + [`e29e3ba9`](https://gitcode.com/Cangjie/cangjie_compiler/commit/e29e3ba9)】宏展开测试适配 `MacroTest` 环境，随后回退新增单测、仅保留缺陷修复。

**错误恢复与诊断**

【[`26f4bc46`](https://gitcode.com/Cangjie/cangjie_compiler/commit/26f4bc46)】畸形类头之后支持恢复类体，改进错误恢复。

【[`44a9a18a`](https://gitcode.com/Cangjie/cangjie_compiler/commit/44a9a18a)】重复性检查中跳过非法泛型约束名。

【[`cc2d9e39`](https://gitcode.com/Cangjie/cangjie_compiler/commit/cc2d9e39)】修复管道操作符结合 `lambda` 调用重载函数时的类型推断 `ICE`。

【[`11db99a6`](https://gitcode.com/Cangjie/cangjie_compiler/commit/11db99a6)】避免对脱糖后的 `operator '()'` 调用产生 `ICE`。

【[`3f2a7d05`](https://gitcode.com/Cangjie/cangjie_compiler/commit/3f2a7d05)】`VArray` 类型不匹配错误改经 `sourceExpr` 上报，定位更准确。

【[`9dc6a3a6`](https://gitcode.com/Cangjie/cangjie_compiler/commit/9dc6a3a6)】修复 `LookUpImpl::SwapResultsWithInstTy` 中的 `erase` 越界。

【[`429aec16`](https://gitcode.com/Cangjie/cangjie_compiler/commit/429aec16)】为字符串插值脱糖节点补充源码位置。

【[`6be17b50`](https://gitcode.com/Cangjie/cangjie_compiler/commit/6be17b50)】修复 `LSP` 场景宏内枚举构造子问题。

【[`e1fa8664`](https://gitcode.com/Cangjie/cangjie_compiler/commit/e1fa8664)】修复在声明前的 `extend` 中给只读属性赋值导致的编译器崩溃。

【[`0dce68cd`](https://gitcode.com/Cangjie/cangjie_compiler/commit/0dce68cd)】宏调试文件替换后刷新 `import` 声明映射。

**类型系统与继承**

【[`03b918e8`](https://gitcode.com/Cangjie/cangjie_compiler/commit/03b918e8)】修复 `IsSubtype` 的循环替换缺陷。

【[`c3425ed2`](https://gitcode.com/Cangjie/cangjie_compiler/commit/c3425ed2)】修复 `override` 检查错误。

【[`88df9ab7`](https://gitcode.com/Cangjie/cangjie_compiler/commit/88df9ab7)】为 `extends` 增加额外约束。

【[`8f897e66`](https://gitcode.com/Cangjie/cangjie_compiler/commit/8f897e66)】将间接 `extend` 注册到其被扩展类型。

【[`67917661`](https://gitcode.com/Cangjie/cangjie_compiler/commit/67917661)】修复父类继承泛型接口问题。

【[`d1e58225`](https://gitcode.com/Cangjie/cangjie_compiler/commit/d1e58225)】`GenerateOptionLikeNonRef` 中改用 `tuple.GetOperand(1)->GetType()` 获取关联类型。

【[`06387022`](https://gitcode.com/Cangjie/cangjie_compiler/commit/06387022)】修复从 `extend` 声明收集类型时的缺陷。

**代码生成与优化**

【[`94be3a81`](https://gitcode.com/Cangjie/cangjie_compiler/commit/94be3a81)】去虚化（devirtualization）时选取最优方法。

【[`160ec4fc`](https://gitcode.com/Cangjie/cangjie_compiler/commit/160ec4fc) + [`21fb9bac`](https://gitcode.com/Cangjie/cangjie_compiler/commit/21fb9bac)】内联后正确计算源类类型并转换为正确类型。

【[`56125ddb`](https://gitcode.com/Cangjie/cangjie_compiler/commit/56125ddb)】优先翻译实例化的局部函数，再翻译其泛型版本。

【[`498764e4`](https://gitcode.com/Cangjie/cangjie_compiler/commit/498764e4)】为 `struct` 包装 `mut` 函数。

【[`59854124`](https://gitcode.com/Cangjie/cangjie_compiler/commit/59854124)】为携带 `VFE` 元数据的函数添加 `NoMerge` 属性（与 `llvm` 侧 `MergeFunctions` 修复配套）。

【[`40bbaf5b`](https://gitcode.com/Cangjie/cangjie_compiler/commit/40bbaf5b)】哈希常量命名防冲突。

【[`4d84b1fd`](https://gitcode.com/Cangjie/cangjie_compiler/commit/4d84b1fd) + [`d59dc774`](https://gitcode.com/Cangjie/cangjie_compiler/commit/d59dc774)】修复 `gvinit` 中的注解访问；消除冗余加载。

【[`a924b1a6`](https://gitcode.com/Cangjie/cangjie_compiler/commit/a924b1a6)】修复 `LTO` 直接链接依赖 `.bc` 时宏 `dylib` 名错误。

【[`f1299586`](https://gitcode.com/Cangjie/cangjie_compiler/commit/f1299586) + [`2fbde31c`](https://gitcode.com/Cangjie/cangjie_compiler/commit/2fbde31c)】禁止空包名。

【[`f24ee4ef`](https://gitcode.com/Cangjie/cangjie_compiler/commit/f24ee4ef)】修复 `option static` 与 `dy-std` 问题。

**Driver 与平台适配**

【[`2701c7a3`](https://gitcode.com/Cangjie/cangjie_compiler/commit/2701c7a3)】`Driver` `posix_spawn` 失败时补充错误信息。

【[`47d36b3a`](https://gitcode.com/Cangjie/cangjie_compiler/commit/47d36b3a)】条件编译 `OS` 支持列表中移除 `HarmonyOS`。

【[`b0781773`](https://gitcode.com/Cangjie/cangjie_compiler/commit/b0781773)】`macOS` 平台动态库搜索路径由 `DYLD_LIBRARY_PATH` 改为 `DYLD_FALLBACK_LIBRARY_PATH`。

**Mock 与测试支持**

【[`ef5e047f`](https://gitcode.com/Cangjie/cangjie_compiler/commit/ef5e047f)】修复 `mock` 泛型参数非法属性。

【[`51c7d24f`](https://gitcode.com/Cangjie/cangjie_compiler/commit/51c7d24f)】同步 `mock` 多项修复。

【[`29d1213a`](https://gitcode.com/Cangjie/cangjie_compiler/commit/29d1213a)】`HandleDeclsToExportForTest` 遍历 `AST` 前先收集局部声明。

**编译器后端（llvm-project）**

【[`df4cd7f96eb9`](https://gitcode.com/Cangjie/llvm-project/commit/df4cd7f96eb9)】修复 `macOS` 动态加载问题。

【[`fae6a83a8713`](https://gitcode.com/Cangjie/llvm-project/commit/fae6a83a8713)】避免生成非法的 `CJ TBAA` 元数据。

【[`4efbff3d9a5e`](https://gitcode.com/Cangjie/llvm-project/commit/4efbff3d9a5e)】`MergeFunctions` 不合并带 `NoMerge` 属性的函数。

【[`c38aa32aa582`](https://gitcode.com/Cangjie/llvm-project/commit/c38aa32aa582)】`CJDevirtualOpt` 中折叠 `is_subtype`。

【[`a4c1f7b41161`](https://gitcode.com/Cangjie/llvm-project/commit/a4c1f7b41161)】识别 `tbd v4` 文件中的 `arm64e.x1` 目标。

【[`f102bb175f7d`](https://gitcode.com/Cangjie/llvm-project/commit/f102bb175f7d)】`PromotePass` 跳过 `optnone` 函数。

【[`9ea483bf4603`](https://gitcode.com/Cangjie/llvm-project/commit/9ea483bf4603)】消除 `PEA`（部分逃逸分析）定义链循环。

## 运行时

### 修复问题

【[`626ef24c`](https://gitcode.com/Cangjie/cangjie_runtime/commit/626ef24c)】解决 `schdfd slots` 初始化竞态。

【[`27969f56`](https://gitcode.com/Cangjie/cangjie_runtime/commit/27969f56)】修复 `thread local cleaner` 不生效的问题。

【[`807587ff`](https://gitcode.com/Cangjie/cangjie_runtime/commit/807587ff)】修复 `foreign` 线程关闭 `socket` 的问题。

【[`5be1bd6a`](https://gitcode.com/Cangjie/cangjie_runtime/commit/5be1bd6a)】修复 `f()` 抛异常时 `dirHandle` 泄漏。

【[`215cc9b7`](https://gitcode.com/Cangjie/cangjie_runtime/commit/215cc9b7)】修正 `IPv4` 组播（multicast）判断。

【[`8165eeab`](https://gitcode.com/Cangjie/cangjie_runtime/commit/8165eeab)】`std` 构建中排除 `unwind c` 与 `clang_rt` 库。

【[`4585d0cf`](https://gitcode.com/Cangjie/cangjie_runtime/commit/4585d0cf)】修复使用 `libast` 的静态链接错误。

【[`e709d173`](https://gitcode.com/Cangjie/cangjie_runtime/commit/e709d173)】`Windows` 下改用文档化 `API` 枚举运行时模块。

【[`db9a9ec5`](https://gitcode.com/Cangjie/cangjie_runtime/commit/db9a9ec5)】空 `treemap` 的 `forward`/`backward` 返回空迭代器。

【[`dfa359c1`](https://gitcode.com/Cangjie/cangjie_runtime/commit/dfa359c1)】修正 `sort` 的 `run stack` 容量估算，恢复 `timsort run-stack` 不变式。

【[`6ef699d7`](https://gitcode.com/Cangjie/cangjie_runtime/commit/6ef699d7) + [`7088ee22`](https://gitcode.com/Cangjie/cangjie_runtime/commit/7088ee22) + [`667ea6a6`](https://gitcode.com/Cangjie/cangjie_runtime/commit/667ea6a6)】修复文档-代码一致性问题与文档笔误。

## 标准库

### 修复问题

【[`2d88fb2`](https://gitcode.com/Cangjie/cangjie_stdx/commit/2d88fb2)】防止 `http1.1 client` 并发访问 `HttpBufferedBody`。

【[`1b23e1a`](https://gitcode.com/Cangjie/cangjie_stdx/commit/1b23e1a)】修复 `HTTP` 请求与超时的竞争问题。

【[`ec3accd`](https://gitcode.com/Cangjie/cangjie_stdx/commit/ec3accd)】`JSON` 流读取器增加缓冲区边界检查。

【[`b20e742`](https://gitcode.com/Cangjie/cangjie_stdx/commit/b20e742)】异常消息保持与原始信息一致。

## 工具链

### cjpm

#### 修复问题

【[`782d46fe`](https://gitcode.com/Cangjie/cangjie_tools/commit/782d46fe)】修复 `cjpm` 依赖路径问题。

### cjlint

#### 修复问题

【[`ef324de2`](https://gitcode.com/Cangjie/cangjie_tools/commit/ef324de2)】修复 `cjlint` `VAR01` 告警（新增数据流规则检查）。

### IDE 插件

#### 修复问题

【[`baca1474`](https://gitcode.com/Cangjie/cangjie_tools/commit/baca1474)】语言服务器 `macOS` 环境适配：`DYLD_LIBRARY_PATH` 改为 `DYLD_FALLBACK_LIBRARY_PATH`。

【[`4faccafd`](https://gitcode.com/Cangjie/cangjie_tools/commit/4faccafd)】类体错误恢复后更新 `document symbol` 范围。

### 其他

【[`228bb677`](https://gitcode.com/Cangjie/cangjie_tools/commit/228bb677) 等】加强单元测试与覆盖率基建：补充语言服务器单测、接入 `LCOV` 覆盖率、解决覆盖率 `CI` 中 `try-catch` 引起的编译错误、补充 `derive` 测试用例。

## 不兼容变更说明

- 条件编译 `OS` 支持列表中移除 `HarmonyOS`：依赖 `HarmonyOS` 条件编译分支的代码需要适配。
- `macOS` 平台动态库搜索路径由 `DYLD_LIBRARY_PATH` 改为 `DYLD_FALLBACK_LIBRARY_PATH`，依赖原搜索顺序的部署环境需验证动态库加载行为。

## 文档变更说明

【[`e35a1a9`](https://gitcode.com/Cangjie/cangjie_docs/commit/e35a1a9)】修正 `examples` 分支 `release1.0` 中不正确的目录结构描述。

【[`07e87ec`](https://gitcode.com/Cangjie/cangjie_docs/commit/07e87ec)】移除不再支持的 `mac-x64` 相关描述。

【[`3c756fb`](https://gitcode.com/Cangjie/cangjie_docs/commit/3c756fb)】修正 `cjcov` 文档。

【[`c3c635a`](https://gitcode.com/Cangjie/cangjie_docs/commit/c3c635a)】修正 `OpenHarmonyOS` 相关表述。

【[`f982abd`](https://gitcode.com/Cangjie/cangjie_docs/commit/f982abd)】统一文档格式。

【[`c2f8f1c6`](https://gitcode.com/Cangjie/cangjie_compiler/commit/c2f8f1c6)】修正 `--product` 选项默认值的文档描述。

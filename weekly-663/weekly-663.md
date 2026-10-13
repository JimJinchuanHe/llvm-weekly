# LLVM Weekly 第 663 期笔记

原文：[LLVM Weekly #663, September 14th 2026](https://llvmweekly.org/issue/663)

### News and articles

- Dominic Stöcker 总结了其 GSoC 项目：[为 Clang C++ API Notes 增加针对特定重载的注解能力](https://blog.llvm.org/posts/2026-08-21-gsoc-clang-api-notes-overloads/)。

API Notes 可以在不修改第三方或系统头文件的情况下，通过外部 YAML 文件给声明添加注解，常用于改善 C/C++ API 到 Swift 的映射。

以前 API Notes 主要按函数名匹配。对于 `setValue(int)` 和 `setValue(double)` 这样的重载函数，只写 `setValue` 会匹配整个重载集合，无法分别注解。

新方案增加 `Where` 选择器：

- `Where.Parameters`：根据显式参数列表选择具体重载，参数数量和顺序都有意义。
- `Where.Parameters: []`：明确匹配无参数重载；省略 `Where.Parameters` 则表示不限制参数。
- `Where.Object`：匹配成员函数隐式对象的 `const`、`volatile` 以及左值/右值引用限定符。
- 类型匹配会处理别名、无关空格、顶层 cv 限定符和 nullability 等差异。
- 旧的按名称匹配方式保持不变，因此已有 API Notes 文件不受影响。

该项目涉及 YAML 解析、二进制序列化、声明查找、Sema 集成、类型规范化和诊断等多个 Clang 模块。

### On the forums

- Renato Golin、Adam Siemieniuk 和 Tuomas Karna [分享了 MLIR Project Lighthouse 的进展](https://discourse.llvm.org/t/mlir-project-lighthouse-update/91790)。

Lighthouse 的目标不是再造一个完整编译器，而是在 MLIR 之上提供可复用的导入、执行、变换流水线和自动调优基础设施，让项目不必从头搭建编译器。

当前进展：

- KernelBench 前 200 个 kernel 中约 80% 已能在 CPU 上运行并验证结果。
- CPU 性能约为已有 MLIR 参考实现的 60%～80%，主要差距来自尚未识别部分高层特定模式。
- 已支持 Intel GPU 的 lowering 与执行，多数 GEMM、elementwise kernel 和部分 layer norm 可以运行。
- GEMM 自动调优通常可达到参考实现约 80%～100% 的性能。
- 已接入 AI-Bench 和 TPP-MLIR，验证这些基础设施能够被下游项目复用。

长期方向是把通用变换、模式和 matcher 上游到 MLIR；Lighthouse 只保留流水线组装、自动调优和少量胶水代码。

- [第 77 期 MLIR News](https://discourse.llvm.org/t/mlir-news-77th-edition-13th-september-2026/91815) 已发布。

- Andrew Haberlandt 提议[为 Sanitizer 引入稳定的报告格式](https://discourse.llvm.org/t/rfc-a-stable-report-format-for-sanitizers/91696)。

目前 ASan、TSan 等 Sanitizer 的报告主要是供人阅读的文本，由运行库中的一系列 `Printf` 生成，没有稳定格式。外部工具如果解析这些文本，会遇到两个问题：

1. 格式可能随错误类型和 LLVM 版本变化。
2. 不同 Sanitizer 的报告结构不同，换行、提示信息和调用栈位置也不统一。

RFC 建议增加一个**带版本号的二进制结构化格式**，至少记录：

- Sanitizer 名称和错误类型；
- 故障地址、分配地址和分配大小；
- 与错误有关的多组调用栈；
- 每组调用栈的类型和文字描述。

这样操作系统崩溃报告器、LLDB 等工具可以稳定消费 Sanitizer 信息，不必复制数百行文本解析或报告渲染逻辑。版本化设计也允许未来扩展字段，同时兼容旧系统。

- Andy Kaylor 正在推进 LLVM ABI library 对 AArch64 的支持，并给出了正在评审的 PR。

- [LLVM 23.1.1](https://discourse.llvm.org/t/llvm-23-1-1-released/91760) 已发布。

- Yaxun (Sam) Liu 提议增加 [`block.uniformity.profile` 和 `branch.uniformity.profile` 两种 LLVM IR 元数据](https://discourse.llvm.org/t/rfc-profile-based-block-and-branch-uniformity-metadata/91764)。

GPU 中一个 wave 的多个 lane 一起执行指令。如果不同 lane 选择不同分支，硬件通常需要分别执行两条路径，并在另一部分 lane 上屏蔽结果，这就是控制流发散。

普通 PGO 的分支权重只告诉优化器每条边被走了多少次，却无法说明多个 lane 是否一起选择该分支。例如 32 个 lane 中 31 个走 A、1 个走 B，统计上是 31:1 的强偏置，但每个 wave 实际仍要执行 A、B 两条路径。

新元数据分别表示：

- `block.uniformity.profile`：运行时该基本块通常在所有 lane 都活跃时执行。
- `branch.uniformity.profile`：活跃 lane 通常选择相同的后继分支。

这些信息只用于优化成本判断，不能作为程序永远 uniform 的正确性证明。RFC 给出的案例中，uniformity-aware CHR 不仅避免了错误优化造成的 24% 回退，还在一次 workload 上将中位 kernel 时间从 5.554ms 降至 2.068ms，同时显著减少寄存器 spill。

- Leonardo Román Carrillo 提议[给 LLVM IR 增加逐函数的 code model 属性](https://discourse.llvm.org/t/rfc-add-per-function-code-model-attribute/91786)。

当前 code model 主要是 module 级设置。在 LTO/ThinLTO 中，如果把用 `-mcmodel=small` 编译的函数导入 `large` code model 模块，该函数会丢失原来的设置，并按 `large` 生成代码。这可能破坏预编译代码或手写汇编对函数地址距离的假设。

RFC 建议允许函数定义和声明携带 `code_model "small"`、`code_model "large"` 等属性。ThinLTO 导入函数时即可保留其 code model，后端也能据此选择正确的指令序列和 section 布局。

- Wenlong Mu 希望[让 BOLT 能够优化已剥离符号的 ELF 二进制文件](https://discourse.llvm.org/t/rfc-bolt-enabling-post-link-optimization-for-stripped-elf-binaries/91765)。

BOLT 是链接后优化器，通常需要符号、重定位和函数边界等信息分析二进制。支持 stripped ELF 的意义是：即使发布二进制已经移除符号表，仍有机会利用 profile 做代码布局和性能优化。核心难点在于如何可靠恢复函数边界、控制流及重定位相关信息。

- Jacob Hodgson 提议[扩展 `llvm-profgen`，直接从只含采样指令地址的基础 perf 事件生成 LLVM sample profile](https://discourse.llvm.org/t/rfc-basic-sample-support-in-llvm-profgen/91782)。

目前这类 sampled-IP 数据通常依赖 LLVM 树外的 AutoFDO `create_llvm_prof` 工具处理；`llvm-profgen` 已支持 x86 LBR 分支栈和 Arm ETM trace，但不能直接处理普通 IP 采样。

计划增加 `--basic-events`（别名 `-ba`）模式：

- 输入可以是原始 `perf.data` 或 perfscript。
- 结合被测二进制和调试信息，将采样地址映射到函数及源码行。
- 支持多进程、地址重定位、fork 后映射继承等场景。
- 输出标准的 flat、line-number LLVM sample profile，用于内联、冷热分类和权重决策。
- 方案与具体架构无关，只要平台能提供 sampled IP 就能复用。

与 LBR 相比，基础事件没有分支边和调用目标等丰富信息，但采集成本和硬件依赖更低，也能把现有 AutoFDO 工作流收归 LLVM 上游工具链。

### LLVM commits

- X86 启用了 `preferZeroCompareBranch`。[59c5d5e](https://github.com/llvm/llvm-project/commit/59c5d5efb734)

该选项让后端在合适时优先生成与零比较后跳转的形式。零比较通常可以复用指令设置的标志位，或生成更适合目标架构的指令序列，从而减少额外比较指令。

- `LoopInterchange` 现在支持 partially-perfect loop nests。[821ff7c](https://github.com/llvm/llvm-project/commit/821ff7cd6803)

Loop Interchange 会交换循环嵌套次序，以改善内存访问局部性或向量化机会。过去通常要求循环嵌套近似“完美”，即外层循环体主要只包含下一层循环。

支持 partially-perfect nest 后，即使循环之间夹有部分语句，只要依赖关系允许，也可以尝试交换。这扩大了该优化能够处理的真实代码范围。

- LoongArch 增加内存屏障优化 Pass。[c9a1fc8](https://github.com/llvm/llvm-project/commit/c9a1fc8d1dee)

内存屏障用于约束内存操作顺序，但过多或重复的屏障会影响性能。该 Pass 会在不破坏内存模型的前提下消除冗余屏障或合并相关约束。

- `llvm.loop.align` 元数据获得正式文档。[cebfe6c](https://github.com/llvm/llvm-project/commit/cebfe6c25f45)

该元数据用于向后端传递循环对齐提示。合理对齐循环头可能减少取指跨界、改善指令缓存和前端吞吐，但也可能因 padding 增大代码体积。

- 引入 `thread-model` module flag。[e17b8cb](https://github.com/llvm/llvm-project/commit/e17b8cb5341d)

module flag 允许在 LLVM IR 模块中记录线程模型，让 LTO、链接以及后续 codegen 阶段能够保留并检查这一信息，而不是只依赖驱动层的命令行状态。

### Clang commits

- 实现 `-fkeep-inline-functions`。[28efe19](https://github.com/llvm/llvm-project/commit/28efe19f9256)

通常一个函数如果已被内联到所有调用者，编译器可能不再输出其独立函数体。开启该选项后，即使所有调用点都已内联，也会保留可独立调用的定义。

这对调试、动态符号查找、二进制插桩或依赖函数实体存在的工具有用，代价是增大目标文件和最终二进制体积。

- Clang 的 diagnostics reference 改为生成 Markdown。[9f52468](https://github.com/llvm/llvm-project/commit/9f5246879112)

这使诊断参考文档更容易在常见文档工具和代码托管平台中处理、审阅与复用，也降低了维护专用输出格式的成本。

- 补充 ClangIR 对 dynamic exception specification 的处理文档。[f471750e](https://github.com/llvm/llvm-project/commit/f471750e042c)

动态异常规范是早期 C++ 中类似 `throw(Type)` 的语法。虽然现代 C++ 已弃用或移除它，但 ClangIR 在兼容旧代码和正确 lowering 时仍需明确相关语义。

### Other project commits

- LLDB 新增 Fortran language plugin。[07f6736](https://github.com/llvm/llvm-project/commit/07f6736c8a7e)

language plugin 负责语言相关的名称、类型、表达式和运行时行为。独立的 Fortran 插件为后续完善 Fortran 断点、表达式求值、类型展示和 Flang 调试支持建立基础。

- BOLT 获得多项 RISC-V 修复。[c22d6a8](https://github.com/llvm/llvm-project/commit/c22d6a86d2e4)、[5503fa5](https://github.com/llvm/llvm-project/commit/5503fa587536)、[7caac9b](https://github.com/llvm/llvm-project/commit/7caac9be9a7f)、[6b72801](https://github.com/llvm/llvm-project/commit/6b728014512a)

这些修改继续完善 BOLT 对 RISC-V 二进制分析和重写的正确性，为该架构上的链接后布局优化铺路。

- LLVM libc 实现 `fmemopen`。[8ed9cade](https://github.com/llvm/llvm-project/commit/8ed9cadeb622)

`fmemopen` 可以把一段内存缓冲区包装为 C 标准 I/O 的 `FILE *`，随后使用 `fread`、`fwrite`、`fprintf` 等接口操作，而不需要创建真实文件。

- MLIR 新增 `tosa-narrow-f32-to-f16` Pass。[59f30df](https://github.com/llvm/llvm-project/commit/59f30df017e9)

该 Pass 将合适的 TOSA `f32` 运算窄化为 `f16`，用于降低模型的存储、带宽和计算成本。由于精度下降可能改变结果，它需要按照 Pass 的适用条件和目标硬件能力使用。

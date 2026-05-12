### On the forums

- Sameer Sahasrabuddhe shared a [proposal for a memory model incubated under AMDGPU target that is weaker than the LLVM memory model](https://discourse.llvm.org/t/rfc-a-vulkan-style-memory-model-for-amdgpu-and-beyond/90498), aiming for compatibility with the Vulkan memory model.

Vulkan memory model:
https://docs.vulkan.net.cn/spec/latest/appendices/memorymodel.html#memory-model

llvm内存模型：
GPU 编程的现实：
**LLVM原本的内存模型是“为CPU编写的通用规则”，而GPU硬件执行的是一套更激进、更灵活的“专用规则”（即Vulkan模型）。** 两者的“差距”源于设计目标的不同，而这次提案正是为了将这些“专用规则”正式纳入LLVM，弥补这个差距

- You might have seen GitHub recently share a preview of a stacked PR workflow. LLVM has [asked for access](https://discourse.llvm.org/t/native-stacked-prs-in-github/90608/6).

stacked PR workflow：
将一个大的功能改动有逻辑地拆解成一系列**小的、相互依赖的PR**，像“堆积木”一样层层叠加
**已经主动为LLVM项目提交了试用申请**，并正在等待GitHub批准

- LLVM 22.1.4 [was released](https://discourse.llvm.org/t/llvm-22-1-4-released/90622).
    
- Simon Tatham started an RFC discussing on [improving function size estimation for Arm](https://discourse.llvm.org/t/rfc-arm-fixing-function-size-estimation/90626).
function size estimation：**估算一个函数编译后机器码（二进制）大小的过程**

Background：
Arm v6-M架构仅支持16位的Thumb指令集
**编译的顺序问题**是：必须在编译早期决定是否保护LR，但决定是否需要`BL`指令的最终代码大小，却要等到编译晚期的`ARMConstantIslandsPass`才能确定。
当前方案及失效原因

LLVM当前的解决方案是**估算 (EstimateFunctionSizeInBytes)**，在早期猜测函数大小以决定是否提前保护LR[](https://discourse.llvm.org/t/rfc-arm-fixing-function-size-estimation/90626#p-362146-background-2)。但这个估算机制有两大致命缺陷：

**低估导致崩溃**：如果估算小于实际大小，本应需要但未保护LR的代码无法正确生成，会直接导致**编译器崩溃 (compiler crash)**。
    
**估算本身不准确**：估算函数很难准确预测后续优化步骤（如重复的常量池字面量，或为访问它们而拆分基本块引入的额外跳转）引入的代码膨胀。

- EuroLLVM roundtable notes were shared for [PAuthABI](https://discourse.llvm.org/t/eurollvm-2026-pauthabi-roundtable-meeting-notes/90640), [MLIR assembly dialects](https://discourse.llvm.org/t/assembly-dialects-roundtable/90647), and [LLVM_ENABLE_RUNTIMES](https://discourse.llvm.org/t/round-table-llvm-enable-runtimes/90654), [embedded toolchains](https://discourse.llvm.org/t/round-table-embedded-toolchains/90655).

| 议题                         | 核心讨论内容                                                                                                                                                                                                       |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **PAuthABI**               | 将指针认证从Linux拓展至裸机场景；为不同需求定义“配置文件”；将`pauthtest`改为更稳定的命名；在Rust中推进支持                                                                                                                                             |
| **MLIR Assembly Dialects** | 旨在厘清MLIR生态中汇编/指令集级别工作的现状；讨论不同项目中IR编码方式的权衡（如SSA链维护）                                                                                                                                                           |
| **LLVM_ENABLE_RUNTIMES**   | 改进运行时库构建系统，更好地支持multilib以应对不同架构变体；简化外部sysroot构建流程；重构`compiler-rt`以解决技术债务                                                                                                                                     |
| **Embedded Toolchains**    | 希望LLD（链接器）能生成机器可读的map文件（如JSON）[](https://discourse.llvm.org/t/round-table-embedded-toolchains/90655)；优化对嵌入式特定链接脚本的支持[](https://discourse.llvm.org/t/round-table-embedded-toolchains/90655)；建立嵌入式性能与代码大小的跟踪体系 |

`LLVM_ENABLE_RUNTIMES` 的新特性，实现只需 **一条命令**，就能自动为所有目标架构生成对应的运行时库
`LLVM_ENABLE_RUNTIMES` 是LLVM构建系统中用于构建**运行时库**（如`libc++`, `compiler-rt`等）的关键机制。


- Hassnaa Hamdi is checking for [interest in setting up a GitHub roadmap board for vectorisers work](https://discourse.llvm.org/t/rfc-public-roadmap-board-for-the-vectorizers/90632).

随着向量化器代码日益复杂，并行项目众多（如 VPlan 迁移、尾部折叠尾声支持等），**贡献者很难从全局视角看清自己的工作如何融入大图景，也难以预判可能影响自己工作的底层变更**

创建一个看板去跟踪各种pr。
这个原型打不开，如果可以看到对理解llvm中向量化的相关的代码是有帮助的。


- Felipe de Azevedo Piovezan proposed [adding a new packet type to the gdbremote protocol in LLDB for setting/removing multiple breakpoints](https://discourse.llvm.org/t/rfc-a-new-packet-to-set-remove-multiple-breakpoints/90623).

lldb相关：
在LLDB调试器的协议层面，设置大量断点曾是个耗时操作。Felipe的这份RFC提出了一个巧妙方案：将多个断点请求打包成一个网络包发送，将频繁的往返通信简化为“批发”处理，从而显著降低延迟.
耗时：从16s下降到4s。
它在数据库、消息队列等需批量操作的系统中也广泛存在。

    
- Pankaj Dwivedi suggests [adding a ValueDeletionListener class to LLVM IR](https://discourse.llvm.org/t/rfc-valuedeletionlistener-context-level-value-deletion-notifications/90624).

背景：悬空指针与静默错误
在LLVM中，部分分析Pass（如 **UniformityAnalysis**）会在内部使用`DenseSet`等数据结构**存储裸指针**（即 `Value *` 指针）来追踪指令[](https://discourse.llvm.org/t/rfc-valuedeletionlistener-context-level-value-deletion-notifications/90624)。但当某个指令被删除时，这些指针就会**变成悬空指针**。如果这块内存被新指令复用，分析Pass就会在无意中报告新指令的状态，从而导致**静默的编译错误**

原始：CallbackVH：正确但是开销大，每一个value都需要插入一次
新的：ValueDeletionListener。“为每个值注册句柄”转向“在上下文（Context）级别注册监听”

-**核心工作原理**：它颠覆了“为每个值注册句柄”的模式，而是让分析Pass直接**在 `LLVMContext` 上注册一个监听**。当上下文中任意`Value`被销毁（`~Value()`）时，所有已注册的监听都会被通知[](https://discourse.llvm.org/t/rfc-valuedeletionlistener-context-level-value-deletion-notifications/90624)。
    
 **内存管理**：此监听器类采用 **RAII（资源获取即初始化）** 机制，会在构造时自动向`LLVMContext`注册，在析构时自动移除，简化了生命周期管理[](https://discourse.llvm.org/t/rfc-valuedeletionlistener-context-level-value-deletion-notifications/90624)。
    
-**内部实现**：`LLVMContext` 内部使用 `SmallPtrSet` 存储所有监听器；`Value`的析构函数会遍历此集合并逐一通知

- Aryan Magoon is seeking feedback on [handling CFG transforms on branch-divergent/SPMD targets](https://discourse.llvm.org/t/rfc-target-provided-cfg-transform-hints-for-divergent-targets/90630).

许多LLVM的通用CFG优化（如SimplifyCFG、跳转线程等）会复制代码、合并分支，这些在CPU上是有效的优化。然而，对于SPMD目标，这些优化可能会产生严重副作用

- Oskar Wirga [shared a set of PRs adding arm64e support to LLD](https://discourse.llvm.org/t/macho-adding-arm64e-support-to-lld/90656) (Apple’s ABI variant of arm64 that implements Armv8.3 pointer authentication).

为LLD带来处理`arm64e`特有重定位、跳板和链式修复等能力，填补了LLD在Apple指针认证ABI上的空白

- Zequan Wu posted an RFC on [extending LLVM IR DILocation and the DWARF line table to track the source location history or merged instructions during optimisations](https://discourse.llvm.org/t/rfc-multi-sloc-dwarf-line-table-extension/90659).

-**LLVM IR DILocation**：是LLVM IR（中间表示）中的一个**元数据节点**，用于记录IR指令所对应的**源文件位置**，包括文件、行号、列号等信息。在编译器进行指令合并等优化时，会调用 `DILocation::getMergedLocation()` 函数来生成一个新的、合并后的 `DILocation`，但这个函数目前会**丢失原始的源位置信息**[](https://discourse.llvm.org/t/rfc-multi-sloc-dwarf-line-table-extension/90659)。
    
-**DWARF line table**：是 DWARF 调试信息格式中的核心部分，它维护着一张**机器指令地址与源代码行号之间的映射表**。这张表由一系列条目组成，每个条目标记一个地址范围的起点和该范围内的源位置。如果某个地址范围的起点和终点相同，就会形成一个“空地址范围

一种创新的调试信息扩展方案，旨在解决编译器优化过程中的一个固有问题：当优化Pass将来自不同源码位置的指令合并后，现有的调试信息机制会丢失部分源码位置历史，进而影响基于调试信息的优化决策（如AFDO，即自动反馈驱动优化）的效果

### LLVM commits

- The Mips delay slot filler was extended to fix an issue when there is a combination of branch and load delay slots on MIPS1. [458e9c4](https://github.com/llvm/llvm-project/commit/458e9c452c10).

MIPS1架构。不看

- GlobalISel FPInfo was implemented for AArch64, allowing ‘ambiguous’ FP types (fp types with the same width as others). [c95a333](https://github.com/llvm/llvm-project/commit/c95a333de710).

这个提交通过引入`FPInfo`和新的匹配表操作码，补全了GlobalISel框架在类型系统上的最后一块拼图，为AArch64后端实现了对多种扩展浮点类型的健壮支持。
核心目标是**为AArch64架构的GlobalISel指令选择框架，正式启用对多种浮点类型（尤其是bfloat16等扩展类型）的完整支持**，解决了此前因类型信息缺失而无法进行正确指令选择的难题

- Parsing the frame-pointer attribute is now done when creating a MachineFunction, resulting in a small but measurable speedup for codegen. [f2efeab](https://github.com/llvm/llvm-project/commit/f2efeabe314b).

`TargetOptions::DisableFramePointerElim()` 是一个**热路径函数**，在 AArch64 的 `-O0 -g` 编译模式下频繁出现在性能分析中（通过 `AArch64FrameLowering::hasFPImpl` 调用）
这个 commit 提升的是 **LLVM 自身作为编译器的编译速度**（即用 LLVM 编译其他代码的速度），不是被编译程序的运行性能。
关键点：这里改成缓存
![image.png](./figures/1.png 'image.png')


- The Os and Oz pipelines were removed in favour of using O2 with the optsize or minsize attributes instead. [86b9775](https://github.com/llvm/llvm-project/commit/86b9775612f8).

移除了Os和Oz
对于最终用户（用 [clang](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-browser/workbench/workbench.html) 编译），**没有任何影响**，因为 Clang 前端会自动处理转换。只有直接操作 `opt` 工具或编写 LLVM 内部代码的开发者才需要注意这个变化。
**彻底移除了 `Os` 和 `Oz` 作为独立的优化流水线级别**，强制要求改为 `O2 + optsize/minsize 函数属性`

Os和Oz的问题：

- Volatile loads are no longer `willreturn`. [d28eeaa](https://github.com/llvm/llvm-project/commit/d28eeaa99735).

Volatile loads含义：在 C/C++ 里，`volatile` 是关键字，告诉编译器：**"这个变量的值随时可能被外部改变，你不能对它做任何假设，必须每次老老实实地从内存读/写。"**
volatile 操作本身就是一种**副作用**（side effect）。
**普通 load**：编译器可以随意移动、消除、缓存——反正读出来的值一样。
**volatile load**：

- 不能被消除（必须真的执行这次读）
- 不能被合并（两次读不能变成一次）
- 不能随意换顺序
**这个 commit 的关键点**
之前 LLVM 认为 volatile load **一定会返回**（`willreturn = true`），所以可以把普通 load 提升到 volatile load 之前：
**一句话总结**：`volatile load` 就是"小心翼翼地从内存读一次，这次读本身可能会触发意外（trap），编译器不能对它做任何激进的优化假设。"修改后，LLVM 认为 volatile load **也可能不返回**（比如访问了坏地址、触发硬件 trap），所以：
-不敢把东西挪到它前面
-不能推断含 volatile load 的函数是 `willreturn` 的

- TargetInfo for ABI information was added and implemented for BPF. [b06f62f](https://github.com/llvm/llvm-project/commit/b06f62f7fc82),[ebbaa93](https://github.com/llvm/llvm-project/commit/ebbaa93e005e).

这次改动是把这部分逻辑**从 Clang 里搬出来**，放到 LLVM 通用层，让它：

- **不依赖** Clang，不依赖任何前端的 AST
- **任何前端**（Clang、Flang、rustc via LLVM 等）都可以调用
- 操作的是 `llvm::abi::Type` 而不是 `clang::QualType`

所以本质上是一个**编译器基础设施重构**，把原本只属于 Clang 的功能下沉到 LLVM 公共层

- The SLP vectorizer gained initial support for non-power-of-2 vectorisation. [cb9b66c](https://github.com/llvm/llvm-project/commit/cb9b66cb6107).

**之前**：SLP 向量化只能凑成 2/4/8/16 个操作一组，如果代码里天然是 3 个或 6 个操作，就无法向量化或得凑 padding。

**这个 commit**：SLP 树内部允许出现 3/5/6/7 等任意宽度的向量节点，更贴合实际代码的结构，特别对 RISC-V 和 AArch64 SVE 这类支持可变长度向量的架构收益更大。

现代 CPU 有 SIMD 指令
对于三个的，如果强行凑成 4 个（填 1 个 padding），会浪费一个 lane

### Clang commits

- The `[[clang:unsafe_buffer_usage]]` attribute is now supported in API notes.[849de61](https://github.com/llvm/llvm-project/commit/849de61619cc).

能够通过外部配置文件注入到第三方/系统库函数上，而不需要修改这些库的源代码
[clang:unsafe_buffer_usage] 安全选项相关的一个attribute。
ApiNote：
clang 编译流程：

1. 解析 #include "Foo.h"
        ↓
2. 同时查找 Foo.apinotes（按头文件路径或模块名）
        ↓
3. 解析 YAML，生成内部注解数据库（二进制缓存，存在 .pcm 或缓存目录）
        ↓
4. 做语义分析（Sema）时，对每个函数声明，
   查表 → 发现有 APINotes 注解 → 自动给 AST 节点附加 Attribute
        ↓
5. 后续的警告、代码生成等，跟直接写 [[attribute]] 完全一致


- `-fstrict-bool` was implemented, allowing control over whether Clang can assume bool values loaded from pattern can’t have a bit pattern other than 0 or 1. [5659f86](https://github.com/llvm/llvm-project/commit/5659f86af5ab).

截取，

- Vector OFP8 types were added to RISC-V vector intrinsics. [228fabd](https://github.com/llvm/llvm-project/commit/228fabd5be82).

这是为 RISC-V 向量扩展（RVV）新增两种 **8位浮点（FP8）向量类型** 的 Clang 前端支持

- clangd gained a persistent cache for a clangd built module file. [28d2537](https://github.com/llvm/llvm-project/commit/28d2537af2b6).

clangd：VS Code 写代码时自动弹出补全、跳转定义，背后就是 clangd 在工作
**语言服务器（Language Server）**，不做编译，专门服务 IDE/编辑器。

clangd 在处理 C++20 模块时需要构建 **BMI（Built Module Interface，即 `.pcm` 文件）**。问题是：

- 每次启动新的 clangd 进程（比如重新打开 IDE），或者关闭所有标签页再重新打开，都会**从头重新构建所有 `.pcm` 文件**
- 特别糟糕的情况：如果用户只打开了一个文件，clangd 需要在**单线程**里串行构建所有依赖的模块文件，非常慢

**解决方案：持久化磁盘缓存**


### Other project commits

- TypeSanitizer can now build for Hexagon. [87a9cba](https://github.com/llvm/llvm-project/commit/87a9cbaed1d6).
    
- SIMD compiler directives and an `INLINEALWAYS` compiler directive were added to Flang. [f5e80c9](https://github.com/llvm/llvm-project/commit/f5e80c985804), [839a22f](https://github.com/llvm/llvm-project/commit/839a22f449b3).
    
- LLVM libc test running was moved to lit. [fc9f14e](https://github.com/llvm/llvm-project/commit/fc9f14e42422).

测试调整，Ctest转lit
    
- `views::enumerate` and `ranges::stride_view` was implemented in libcxx. [2039a51](https://github.com/llvm/llvm-project/commit/2039a51881bb),[2f28e1d](https://github.com/llvm/llvm-project/commit/2f28e1db535b).
    
- Input file loaded was parallelised in LLD’s ELF linker. [83f8eee](https://github.com/llvm/llvm-project/commit/83f8eee57d5a).
    
- LLDB’s docs gained an index of previous dev meeting talks on debugging. [df6792b](https://github.com/llvm/llvm-project/commit/df6792b285e7).
    
- Documentation was added on kernel record replay for OpenMP. [254fcbe](https://github.com/llvm/llvm-project/commit/254fcbeface8).
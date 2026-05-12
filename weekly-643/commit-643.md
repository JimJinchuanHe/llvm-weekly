##### Pankaj Dwivedi suggests [adding a ValueDeletionListener class to LLVM IR](https://discourse.llvm.org/t/rfc-valuedeletionlistener-context-level-value-deletion-notifications/90624).

背景：悬空指针与静默错误
在LLVM中，部分分析Pass（如 **UniformityAnalysis**）会在内部使用`DenseSet`等数据结构**存储裸指针**（即 `Value *` 指针）来追踪指令[](https://discourse.llvm.org/t/rfc-valuedeletionlistener-context-level-value-deletion-notifications/90624)。但当某个指令被删除时，这些指针就会**变成悬空指针**。如果这块内存被新指令复用，分析Pass就会在无意中报告新指令的状态，从而导致**静默的编译错误**

原始：CallbackVH：正确但是开销大，每一个value都需要插入一次
新的：ValueDeletionListener。“为每个值注册句柄”转向“在上下文（Context）级别注册监听”

-**核心工作原理**：它颠覆了“为每个值注册句柄”的模式，而是让分析Pass直接**在 `LLVMContext` 上注册一个监听**。当上下文中任意`Value`被销毁（`~Value()`）时，所有已注册的监听都会被通知[](https://discourse.llvm.org/t/rfc-valuedeletionlistener-context-level-value-deletion-notifications/90624)。
    
 **内存管理**：此监听器类采用 **RAII（资源获取即初始化）** 机制，会在构造时自动向`LLVMContext`注册，在析构时自动移除，简化了生命周期管理[](https://discourse.llvm.org/t/rfc-valuedeletionlistener-context-level-value-deletion-notifications/90624)。
    
-**内部实现**：`LLVMContext` 内部使用 `SmallPtrSet` 存储所有监听器；`Value`的析构函数会遍历此集合并逐一通知


许多LLVM的通用CFG优化（如SimplifyCFG、跳转线程等）会复制代码、合并分支，这些在CPU上是有效的优化。然而，对于SPMD目标，这些优化可能会产生严重副作用
##### GlobalISel FPInfo was implemented for AArch64, allowing ‘ambiguous’ FP types (fp types with the same width as others). [c95a333](https://github.com/llvm/llvm-project/commit/c95a333de710)

LLVM 有两套指令选择框架：

| 框架                      | 特点               |
| ----------------------- | ---------------- |
| **SelectionDAG (SDAG)** | 老框架，按基本块处理       |
| **GlobalISel**          | 新框架，全函数范围处理，更易扩展 |
BiSheng用的是**GlobalISel**
理由：入口走到了IRTranslator Pass

**GlobalISel 的流水线：**
LLVM IR
   ↓  [IRTranslator]
Generic Machine IR (G_ADD, G_LOAD, G_STORE ...)
   ↓  [Legalizer]        ← 把不合法的类型/操作变合法
Generic Machine IR (合法化后)
   ↓  [RegBankSelect]    ← 分配寄存器组（整数/浮点）
   ↓  [InstructionSelect] ← 用 MatchTable 匹配，映射为真实机器指令
Target Machine IR (e.g. ADDXrr, LDRXui ...)

**LLT（Low-Level Type）** 是 GlobalISel 的类型系统：

- `LLT::scalar(32)` — 32 位标量（不区分 int/float）
- `LLT::integer(32)` — 明确是 32 位整数（此 commit 新增/推广）
- `LLT::fixed_vector(4, LLT::integer(32))` — `<4 x i32>`

这个 commit 是 AArch64 GlobalISel 支持浮点扩展类型的重要基础设施升级，核心思路是：**让类型系统能精确区分整数和浮点，同时让 MatchTable 在合适的地方忽略这种区别，以减少冗余模式**

GlobalISel 使用 `LLT`（Low-Level Type）表示寄存器类型。在此 commit 之前：

- `LLT::scalar(N)` 创建一个 N-bit 的标量类型，**不区分整数还是浮点**。
- 当新的浮点类型（如 `bf16`、`ppc128float`）被引入到 GlobalISel 中时，所有现有代码用 `scalar(N)` 构造的"中间整数"类型（如条件码 `i1`、偏移量 `i64` 等）无法在语义上与浮点标量区分开，导致 pattern matching 混乱。

**对新扩展类型（bf16等）：这是让它们能正确工作的基础设施**
 bfloat, ppc128float

float32  = 1 + 8 + 23 = 32 bit
bfloat16 = 1 + 8 + 7  = 16 bit  ← 把 float32 的尾数砍掉
float16  = 1 + 5 + 10 = 16 bit




##### The Os and Oz pipelines were removed in favour of using O2 with the optsize or minsize attributes instead. [86b9775](https://github.com/llvm/llvm-project/commit/86b9775612f8).

移除了Os和Oz
对于最终用户（用 [clang](vscode-file://vscode-app/Applications/Visual%20Studio%20Code.app/Contents/Resources/app/out/vs/code/electron-browser/workbench/workbench.html) 编译），**没有任何影响**，因为 Clang 前端会自动处理转换。只有直接操作 `opt` 工具或编写 LLVM 内部代码的开发者才需要注意这个变化。
**彻底移除了 `Os` 和 `Oz` 作为独立的优化流水线级别**，强制要求改为 `O2 + optsize/minsize 函数属性`

Os和Oz的问题：
; 普通函数，没有 size 属性
define i32 @foo() { ... }

; 标记了 optsize，告诉编译器"这个函数优先优化体积"
define i32 @bar() optsize { ... }

; 标记了 minsize，极度压缩体积
define i32 @baz() minsize { ... }

用 -Os pipeline 编译时：
  inline 阈值 = 50（pipeline 级别决定）

函数上有 optsize 属性时：
  inline 阈值 = 50（属性级别决定）



// Pass A 的写法（查 pipeline 级别）：
if (OptLevel == OptimizationLevel::Os)
threshold = 50;

// Pass B 的写法（查函数属性）：
if (F.hasOptSize())
threshold = 50;

正常的clang的流程是没有问题的，这个commit是为了解决opt的问题。
正常clang逻辑：
clang -Os foo.c
    ↓
Clang 前端生成 IR，给每个函数自动加上 optsize 属性
    ↓
进入优化 pipeline，pipeline 级别是 Os

如果用Os处理一个普通的ir（没有optsize 属性）会有问题：
Opt处理这样的ir时，不同的pass会有不同的逻辑。



这个 commit 的立场是：**第一条（跳过哪些 pass）也应该由函数属性来决定**，而不是存在一个独立的 "Os pipeline"。这样每个函数的优化策略完全由函数自身的属性决定，pipeline 只负责"跑多强的优化"（O1/O2/O3），"跑多小的代码"是函数自己的事。


##### `-fstrict-bool` was implemented, allowing control over whether Clang can assume bool values loaded from pattern can’t have a bit pattern other than 0 or 1. [5659f86](https://github.com/llvm/llvm-project/commit/5659f86af5ab).

`-fstrict-bool`：

有可能不是0或者1
https://discourse.llvm.org/t/defining-what-happens-when-a-bool-isn-t-0-or-1/86778


lowest bit set：
![image.png](./figures/1.png 'image.png')

两种判断
truncate：
i8 值：  0000 0010  （= 2，非法 bool）
截断到 i1，只保留 bit0：
          ↓ bit0
          0
结论：false  ← 错误！2 应该是 true

icmp：
i8 值：  0000 0010  （= 2，非法 bool）
icmp ne 0：问"这个值等于0吗？"
不等于0  → true  ← 正确！

range metadata
; 有便利贴（range metadata）：告诉优化器"这个值只能是 0 或 1"
%x = load i8, ptr %p, !range !{i8 0, i8 2} 优化器会**放心地用 truncate 截断到 i1，不会丢信息**

在某些内存数据读取中（网络，apple等情况），读取到不那么合法的数据，这个时候会出现报错。

-fstrict-bool # 相信 bool 一定是 0/1，用 truncate（默认，快）

-fno-strict-bool=nonzero # 不信任内存数据，用 icmp ne 0（安全，稍慢）

-fno-strict-bool=truncate # 折中：用 & 1 取最低位
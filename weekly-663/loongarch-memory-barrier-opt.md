# LoongArch 内存屏障优化 Pass 详解

## 1. Commit 概览

- **Commit**：[c9a1fc8d1dee — [LoongArch] Add memory barrier optimization pass](https://github.com/llvm/llvm-project/commit/c9a1fc8d1dee)
- **提交时间**：2026-09-11
- **作者**：hev（Loongson）
- **关联 PR**：[#218597](https://github.com/llvm/llvm-project/pull/218597)
- **影响范围**：LoongArch 后端、Machine-level code generation

这个 commit 新增了一个 LoongArch 机器指令级优化 Pass，用于：

1. 删除冗余的 `DBAR` 内存屏障；
2. 合并相邻或控制流上安全的内存屏障；
3. 将独立的 `DBAR` 与原子内存操作合并成带屏障语义的 AMO 指令；
4. 在保证内存模型语义不变的前提下，减少屏障指令和原子操作带来的开销。

Commit 共修改 7 个文件，新增约 559 行 Pass 实现和大量测试用例。它没有改变 LLVM IR 的内存模型，而是在原子操作已经降低到 LoongArch MachineInstr 之后，针对 LoongArch 指令语义进行目标相关优化。

---

## 2. 为什么需要优化 DBAR

### 2.1 内存屏障的作用

LoongArch 使用 `DBAR` 指令约束内存访问顺序。LLVM IR 中的 `fence acquire`、`fence release`、`fence seq_cst` 等操作，最终可能被降低为不同参数的 `dbar`：

```asm
dbar 20    # 示例：一种 acquire 语义
...
dbar 16    # 示例：另一种 release/acquire 组合语义
```

屏障是并发正确性的基础，但它通常不是免费的：

- 可能阻止处理器重排内存操作；
- 可能增加流水线停顿或执行延迟；
- 会增加代码中的指令数量；
- 与原子读改写操作同时出现时，可能重复表达相同的排序约束。

因此，后端需要判断：多个屏障是否可以由一个屏障覆盖，以及一个原子操作本身是否已经包含所需的屏障语义。

### 2.2 原子指令本身也可能包含屏障语义

LoongArch 的原子操作大致可以分为几类：

- `LL_W`、`LL_D`：load-linked；
- `SC_W`、`SC_D`、`SC_Q`：store-conditional；
- `AMSWAP`、`AMADD`、`AMAND`、`AMOR`、`AMXOR`、`AMMAX`、`AMMIN`、`AMCAS` 等 AMO；
- 对应的带 `_DB` 后缀版本，例如 `amswap_db.w`。

例如，原本可能生成：

```asm
dbar 18
amswap.w  $a1, $a0, $zero
```

如果前面的 `DBAR` 和原子操作组合起来的排序语义，正好可以由带屏障版本的原子指令表达，就可以转换为：

```asm
amswap_db.w  $a1, $a0, $zero
```

这样既保留了语义，又避免单独执行一个 `DBAR`。

---

## 3. Pass 的核心变换规则

新 Pass 在源文件头部明确列出了主要规则。设 `x`、`y` 是 DBAR hint：

```text
DBAR x + DBAR y -> DBAR (x & y)
DBAR x + AMO_DB -> AMO_DB
DBAR x + AMO    -> AMO_DB
DBAR x + LL     -> LL
AMO_DB + DBAR x -> AMO_DB
AMO    + DBAR x -> AMO_DB
SC     + DBAR x -> SC
```

### 3.1 两个 DBAR 合并

```asm
dbar x
...              # 中间不能有影响内存顺序的操作
dbar y
```

如果后一个屏障能够覆盖前一个屏障要求，或者两个屏障的约束可以合并，则保留一个屏障，并将 hint 更新为合并后的值：

```text
merged_hint = x & y
```

`BarrierHint::merge()` 直接对两个 hint 做按位与。这里不是普通的“取更大值”，而是利用 LoongArch DBAR hint 位的组合语义计算一个同时满足两个屏障要求的 hint。

### 3.2 DBAR 与 AMO 合并

```asm
dbar x
amswap.w ...
```

如果 AMO 前置或后置的屏障语义已经覆盖 `dbar x`，则删除独立的 `DBAR`；如果 AMO 还没有带 DBAR 语义，则将它替换为对应的 `_DB` 形式：

```asm
amswap_db.w ...
```

这类转换是本 commit 最直接的收益来源之一：将两条指令压缩成一条，同时避免重复的内存排序操作。

### 3.3 DBAR 与 LL/SC 合并

`LL` 和 `SC` 在 Pass 中被抽象成具有特殊前后屏障语义的操作：

- `LL_W`、`LL_D`：前置约束较弱，后置约束较强；
- `SC_W`、`SC_D`、`SC_Q`：前置约束较强，后置约束较弱。

因此，Pass 可以判断独立 DBAR 是否已经被 LL/SC 覆盖，而不是把所有原子操作都简单地当成普通内存访问。

---

## 4. 用 BarrierHint 抽象屏障语义

实现没有为每一种屏障组合写大量特判，而是把一条指令抽象为两个属性：

```text
Pre  = 指令之前需要满足的内存约束
Post = 指令之后能够提供的内存约束
```

`BarrierHint` 目前只接受低 5 位有效的 hint：

```cpp
static inline bool isValid(unsigned Hint) {
  return (Hint & ~0x1f) == 0;
}
```

它提供两个关键操作：

```cpp
subsumes(O)  -> 当前 hint 是否覆盖 O 的约束
merge(O)     -> 两个 hint 合并后的 hint
```

不同指令被映射为不同的 `Pre`/`Post`：

| 指令类型 | Pre | Post | 含义 |
|---|---:|---:|---|
| `DBAR x` | `x` | `x` | 前后都提供同一个屏障约束 |
| `LL` | `0b10000` | `0b11111` | 读取链接操作的前后语义不同 |
| `SC` | `0b11111` | `0b10000` | 条件存储操作的前后语义不同 |
| 普通 AMO | `0b10000` | `0b10000` | 非 `_DB` AMO 的基础语义 |
| `_DB` AMO | `0b10000` | `0b10000` | 可与 DBAR 合并的原子操作 |

这种表示方式把“某条指令能不能替代某个屏障”的判断，统一转化为 bitmask 覆盖关系。

---

## 5. 跨基本块优化为什么需要支配树

简单的同一基本块场景可以直接扫描两条屏障之间的指令。但真实代码中，两条屏障可能位于不同基本块，不能只看线性指令顺序。

例如：

```text
        A: dbar
       / \\
      B   C
       \\
        D: atomic
```

要删除 `A`，必须确认从 `A` 到 `D` 的所有相关路径都不会绕过屏障或包含不可忽略的内存操作。

### 5.1 MachineDominatorTree

如果基本块 `A` 支配基本块 `B`，表示所有从函数入口到 `B` 的路径都必须经过 `A`。这有助于判断后一个屏障是否处于前一个屏障的必经控制流上。

### 5.2 MachinePostDominatorTree

如果基本块 `B` 后支配基本块 `A`，表示所有从 `A` 到函数出口的路径都必须经过 `B`。这有助于判断前一个屏障是否可以被后一个屏障统一覆盖。

Pass 同时要求：

```cpp
AU.addRequired<MachineDominatorTreeWrapperPass>();
AU.addRequired<MachinePostDominatorTreeWrapperPass>();
```

并声明保留这两个分析结果，因为它只修改指令，不改变基本块控制流结构。

---

## 6. `resolveBarrierRedundancy()` 的判断流程

这个函数返回一个 bitmask，表示两条候选指令中哪些可以删除：

```text
CandidateA = 1 << 0
CandidateB = 1 << 1
```

### 6.1 同一基本块

对于同一基本块中的 `A` 和 `B`：

1. 从 `A` 后面开始扫描；
2. 到达 `B` 之前，所有指令必须满足 `isSafeToSkip()`；
3. 如果遇到可能读写内存、调用、返回或未知副作用的指令，则停止优化；
4. 如果中间只有安全指令，就认为 `A` 和 `B` 之间没有破坏屏障关系的操作。

### 6.2 不同基本块

跨基本块时，Pass 先判断两种关系：

```text
ADomB   = A 所在块是否支配 B 所在块
BPostDomA = B 所在块是否后支配 A 所在块
```

如果两种关系都不存在，则两条指令之间不存在足够明确的必经路径关系，Pass 不做优化。

之后还会分别检查：

- `A` 所在块从 `A` 到块尾的指令；
- `B` 所在块从块首到 `B` 的指令；
- 两个基本块之间的所有路径。

`checkAllPathSafe()` 使用 worklist 遍历 CFG。除了起点和终点之外，路径上的基本块不能包含无法安全跳过的指令。

默认情况下，命令行选项：

```text
--loongarch-require-no-path-bypass=true
```

要求不存在绕过任一屏障的路径。这是偏保守的默认策略，优先保证并发语义正确，而不是最大化删除数量。

---

## 7. 哪些指令可以被安全跳过

`isSafeToSkip()` 的判断很重要：它决定了两条屏障之间是否真的“没有发生可能影响内存顺序的事情”。

以下情况通常不安全：

- `mayLoadOrStore()` 为真；
- 指令是 call 或 return；
- 指令带有未建模副作用；
- 无法识别的 inline assembly。

以下情况可以继续扫描：

- 纯算术指令；
- 不访问内存的寄存器操作；
- 可证明没有相关副作用的机器指令；
- 被识别为 DBAR 的 inline asm。

对 inline asm，Pass 默认会解析形如下面的汇编：

```asm
 dbar 22
 dbar  $0
```

并且可以通过：

```text
--loongarch-disable-inline-asm-barrier-opt=true
```

完全关闭对 inline asm 屏障的优化，避免因为程序集文本无法完整解析而产生风险。

---

## 8. 具体替换逻辑

### 8.1 删除前一个屏障

当 `A` 是 DBAR，且 `B` 的 `Pre` 能覆盖 `A` 的 `Post` 时，可以删除 `A`：

```text
DBAR x + B -> B
```

如果 `B` 是普通 AMO，则先替换为对应的 `_DB` AMO，再删除 `A`。如果两个 DBAR 的 hint 需要合并，则更新 `B` 的 hint。

### 8.2 删除后一个屏障

当 `B` 是 DBAR，且 `A` 的 `Post` 已经覆盖 `B` 的要求时，可以删除 `B`：

```text
A + DBAR x -> A
```

如果 `A` 是普通 AMO，则替换成 `_DB` AMO；如果 `A` 是 `SC`，则保留 `SC`，因为 SC 的语义已经满足对应屏障要求。

### 8.3 可选地用 NOP 保持代码布局

默认情况下，被消除的 DBAR 会直接从 MachineBasicBlock 中删除。

通过：

```text
--loongarch-replace-eliminated-dbar-to-nop=true
```

可以将被删除的位置替换成 `andi $r0, $r0, 0` 形式的 NOP。这样做不再减少指令槽位，但可以保持代码布局或地址相对稳定，适合对代码地址、补丁位置或调试布局敏感的场景。

---

## 9. Pass 在流水线中的位置

该 Pass 被加入 LoongArch 的 `addPreEmitPass2()`：

```cpp
addPass(createLoongArchExpandAtomicPseudoPass());
if (TM->getOptLevel() != CodeGenOptLevel::None)
  addPass(createLoongArchMemoryBarrierOptPass());
```

因此它位于：

1. 原子伪指令展开之后；
2. 汇编输出之前；
3. 只在优化级别不是 `CodeGenOptLevel::None` 时执行。

这个位置很关键：

- 太早时，原子操作还可能是 LLVM 的伪指令，无法准确匹配最终 LoongArch 指令；
- 太晚时，指令已经输出，无法再进行 MachineInstr 级别替换；
- 放在原子伪指令展开之后，Pass 可以直接识别 `LL`、`SC`、AMO 和 `_DB` 指令。

同时，commit 还更新了 `opt-pipeline.ll`，确认优化流水线中出现：

```text
MachineDominator Tree Construction
MachinePostDominator Tree Construction
LoongArch Memory Barrier Optimisation pass
```

---

## 10. 测试覆盖了什么

新增和扩展的 `memory-barrier-opt.ll` 覆盖了多个场景：

### 基本消除

```llvm
fence acquire
fence acquire
```

输出只保留一个 `dbar`。

### 不同屏障组合

测试了 acquire、release、acq_rel、seq_cst 等组合，验证合并后不会丢失更强的约束。

### 与 AMO 合并

例如：

```llvm
fence release
atomicrmw xchg ptr %p, i32 0 monotonic
```

可以生成带 `_DB` 的 AMO，而不是单独的 `dbar` 加普通 AMO。

### 不应优化的情况

如果中间存在真实的 load/store、调用、返回或其他不可忽略副作用，Pass 应保留原始屏障。

### inline asm

测试覆盖了：

- `dbar 22`；
- 带空格的 `dbar`；
- 带注释的 `dbar`；
- `dbar $0`、`dbar $1` 这类带约束操作数的 inline asm；
- 禁用 inline asm 优化的选项。

### 配置选项

测试使用多个 FileCheck 前缀验证不同设置：

| 选项 | 作用 |
|---|---|
| `--loongarch-require-no-path-bypass=false` | 放宽路径绕过限制 |
| `--loongarch-merge-amo-with-dbar=false` | 禁止把 AMO 与 DBAR 合并 |
| `--loongarch-disable-inline-asm-barrier-opt=true` | 禁止优化 inline asm 屏障 |
| `--loongarch-replace-eliminated-dbar-to-nop=true` | 删除位置改用 NOP 保留布局 |

---

## 11. 效果分析

### 11.1 可以明确确认的效果

从测试输出可以直接确认：

- 连续或可安全合并的 `DBAR` 数量减少；
- `DBAR + AMO` 可以变为单条 `_DB` AMO；
- 生成的机器指令数量减少；
- 某些场景下减少独立屏障，有利于降低排序开销；
- 优化发生在后端，不要求上游 IR 增加新的通用内存模型规则。

以典型转换为例：

```asm
# 优化前
 dbar 18
 amswap.w $a0, $zero, $zero

# 优化后
 amswap_db.w $a0, $zero, $zero
```

指令从两条减少为一条，并且保留需要的原子排序语义。

### 11.2 Commit 没有给出的数据

该 commit 本身没有提供 SPEC、数据库或微基准的运行时间对比，也没有给出统一的：

- DBAR 删除数量；
- 二进制大小变化；
- 周期数变化；
- 吞吐量或延迟改善百分比。

因此不能把“指令数减少”直接写成某个确定的性能提升百分比。实际收益取决于：

- 程序中原子操作和 fence 的密度；
- 目标 LoongArch 微架构对 DBAR 的实现成本；
- 优化后的指令是否位于热点路径；
- 是否因为保守的 CFG 条件而无法合并。

更准确的结论是：**该 commit 提供了减少冗余内存排序指令的能力，并通过测试证明了变换规则和保守条件，但 commit 页面没有公布端到端性能数字。**

---

## 12. 设计思想总结

### 12.1 在目标后端做目标相关优化

LLVM IR 的 fence 语义必须保持通用和架构无关。LoongArch 的 DBAR hint、LL/SC 以及 AMO `_DB` 指令属于目标相关细节，因此在 MachineInstr 阶段处理更合适。

### 12.2 用语义覆盖替代简单模式匹配

Pass 不只是搜索相邻的两条 `dbar`，而是把每条候选指令转换为 `Pre`/`Post` 约束，再判断一条指令是否覆盖另一条指令的要求。这让同一套框架可以处理：

- DBAR 与 DBAR；
- DBAR 与 AMO；
- DBAR 与 LL/SC；
- 普通指令与 inline asm。

### 12.3 先保证正确性，再扩大优化范围

跨基本块时同时使用 dominator、post-dominator 和路径扫描；遇到无法建模的内存访问、调用或副作用就放弃优化。默认启用 `require-no-path-bypass`，说明该实现更重视并发语义的可靠性。

### 12.4 为工程场景保留开关

几个 hidden 选项反映了实现中的工程权衡：

- 是否允许跨路径更激进地优化；
- 是否把 AMO 与 DBAR 合并；
- 是否处理 inline asm；
- 是否用 NOP 保持布局。

这些开关既方便回归测试，也便于后续定位特定平台或代码模式的问题。

---

## 13. 一句话总结

这个 commit 为 LoongArch 增加了一个位于最终机器码生成阶段的内存屏障优化 Pass。它通过 `BarrierHint` 建模 DBAR、LL/SC 和 AMO 的前后内存语义，再结合 MachineDominatorTree、MachinePostDominatorTree 和路径安全检查，删除冗余 DBAR，并把独立屏障合并到原子操作中。它的直接收益是减少指令数量和不必要的排序操作；但 commit 本身没有公布端到端性能提升百分比，因此效果应表述为“降低潜在屏障开销”，而不是给出未经验证的固定加速数字。

## 参考资料

- [Commit c9a1fc8d1dee](https://github.com/llvm/llvm-project/commit/c9a1fc8d1dee)
- [LoongArchMemoryBarrierOpt.cpp](https://github.com/llvm/llvm-project/blob/c9a1fc8d1deee527a3ff43592ed4e0e5b81d7fca/llvm/lib/Target/LoongArch/LoongArchMemoryBarrierOpt.cpp)
- [memory-barrier-opt.ll 测试](https://github.com/llvm/llvm-project/blob/c9a1fc8d1deee527a3ff43592ed4e0e5b81d7fca/llvm/test/CodeGen/LoongArch/memory-barrier-opt.ll)
- [LoongArch 后端流水线修改](https://github.com/llvm/llvm-project/blob/c9a1fc8d1deee527a3ff43592ed4e0e5b81d7fca/llvm/lib/Target/LoongArch/LoongArchTargetMachine.cpp)

- Wenlong Mu 希望[让 BOLT 能够优化已剥离符号的 ELF 二进制文件](https://discourse.llvm.org/t/rfc-bolt-enabling-post-link-optimization-for-stripped-elf-binaries/91765)。

BOLT 通常依靠符号表和静态重定位信息来识别函数及引用关系，才能安全地移动函数、调整布局并更新引用。但生产环境中的二进制经常被 strip；用户也可能拿不到原始源码、构建环境或可重链接的文件。

现有`--allow-stripped` 虽然允许处理这类文件，但不能恢复代码移动涉及的所有内部引用，无法保证优化结果正确；而且只能尽量在原地址附近改写，限制了函数重排等优化。

新增显式启用的`--recover-relocations` 模式。处理 stripped 文件时，还需要指定`--allow-stripped` 。整体思路是：

1. 利用 ELF 的 FDE 信息发现函数；AArch64 上再用有限模式匹配补充部分缺少 FDE 的函数。
2. 从机器码和数据中推断函数引用、跳转表项及函数指针，重建 BOLT 所需的重定位关系。
3. 把恢复出的引用交给 BOLT 现有的重定位模式流水线，允许移动函数和跳转表，并同步更新引用。
安全性上，只有目标能被明确识别时才更新引用；无法确定的引用保留原地址，避免跳转到错误位置。可选的`--aggressive-relocation-recovery` 会扩大数据段扫描范围，以恢复更多函数指针，但也增加把普通整数误判为指针的可能。

实现还需要处理 AArch64 的 ADRP/ADD 地址计算、缺少 FDE 的`_start` 和 Cortex-A53 erratum 843419 veneer，以及 stripped 二进制中的跳转表。跳转表项只有在能解析到已识别函数内的有效指令时才会接受。


### 实现 `-fkeep-inline-functions`。[28efe19](https://github.com/llvm/llvm-project/commit/28efe19f9256)

通常一个函数如果已被内联到所有调用者，编译器可能不再输出其独立函数体。开启该选项后，即使所有调用点都已内联，也会保留可独立调用的定义。

这对调试、动态符号查找、二进制插桩或依赖函数实体存在的工具有用，代价是增大目标文件和最终二进制体积。

1、并不是“禁止内联”，而是：
 内联可以照常发生，但不要因为已经内联或没有调用者，就删除独立的函数定义。

性能分析和插桩
 Profiler、动态插桩、二进制重写工具可能需要真实的函数入口。

例如：

- `perf`
- BOLT
- 动态探针
- 函数级 tracing
- 覆盖率或审计工具






### BOLT 获得多项 RISC-V 修复。[c22d6a8](https://github.com/llvm/llvm-project/commit/c22d6a86d2e4)、[5503fa5](https://github.com/llvm/llvm-project/commit/5503fa587536)、[7caac9b](https://github.com/llvm/llvm-project/commit/7caac9be9a7f)、[6b72801](https://github.com/llvm/llvm-project/commit/6b728014512a)

这些修改继续完善 BOLT 对 RISC-V 二进制分析和重写的正确性，为该架构上的链接后布局优化铺路。


### MLIR 新增 `tosa-narrow-f32-to-f16` Pass。[59f30df](https://github.com/llvm/llvm-project/commit/59f30df017e9)

该 Pass 将合适的 TOSA `f32` 运算窄化为 `f16`，用于降低模型的存储、带宽和计算成本。由于精度下降可能改变结果，它需要按照 Pass 的适用条件和目标硬件能力使用。


### `LoopInterchange` 现在支持 partially-perfect loop nests。[821ff7c](https://github.com/llvm/llvm-project/commit/821ff7cd6803)

Loop Interchange 会交换循环嵌套次序，以改善内存访问局部性或向量化机会。过去通常要求循环嵌套近似“完美”，即外层循环体主要只包含下一层循环。

支持 partially-perfect nest 后，即使循环之间夹有部分语句，只要依赖关系允许，也可以尝试交换。这扩大了该优化能够处理的真实代码范围。


# Lab 3：Sv39 分页式虚拟内存系统

!!! danger "DDL"

    本实验文档仍在编辑中，尚未正式发布。

## 实验简介

在前两个实验中，我们的内核一直运行在**物理地址空间**中。本次实验的目标，是让内核正式进入“虚拟世界”——实现 RISC-V 的 **Sv39 分页式虚拟内存系统**。

在理论课第八、九章中，我们已经学习了虚拟内存的概念与设计思想。现在，让我们从“为什么”出发，重新理一理它的演进脉络：

- 在早期系统中，程序员需要手动管理物理内存，容易出错，也难以实现隔离和保护。
- 随着系统复杂度提升，内存管理经历了从**单一连续分配 → 静态分段（Base and Bound） → 动态分段（Segmentation）→ 分页（Paging）**的演化。每种方式都有优缺点：分段容易产生**外部碎片**，分页则可能带来**内部碎片**。

    <figure markdown="span">
        ![vm_history](lab3.assets/vm_history.webp)
        <figcaption>
        内存管理的发展历程<br>
        <small>图片来源：Linux Kernel Development (3rd)
        </small>
        </figcaption>
    </figure>

- 分页机制的关键思想是：**用“虚拟地址”统一抽象内存，让程序认为自己独占整个地址空间**。

实现虚拟内存需要软硬件协作，如下图所示：

<figure markdown="span">
    ![vm_system](lab3.assets/vm_system.webp)
    <figcaption>
    虚拟内存的软硬件协作<br>
    <small>图片来源：[Piyush Itankar on X: "Here is how MMU (memory management unit) works!"](https://x.com/_streetdogg/status/1870679201424261309)
    </small>
    </figcaption>
</figure>

- CPU 内部的 **MMU（Memory Management Unit）** 负责解析页表、执行地址转换和权限检查，这是由 QEMU 完成的。
- 我们实现的操作系统需要负责**构建与维护页表**、**处理缺页异常**。

本实验要实现 RISC-V 规范中的 **Sv39 多级页表机制**，你将完成以下关键步骤：

1. **理解并掌握页表结构**：学习 Sv39 的三级页表格式，弄清每级索引、PTE 各字段的意义。
2. **构建内核页表**：为内核镜像的各个段（`.text`、`.rodata`、`.data`、`.bss`）建立映射，并配置正确的访问权限；同时实现必要的直接映射区域（如内核物理映射）。
3. **切换至虚拟地址执行**：配置 `satp` 寄存器，执行 `sfence.vma`，完成从“物理视角”到“虚拟视角”的安全过渡。
4. **实现最小化页异常处理**：识别缺页或越权异常（`scause`），打印上下文并采取合理的错误处理策略。
   （本实验只涉及**内核态**，用户态与 ELF 装载将在 Lab4 实现。）

完成本实验后，你的操作系统将第一次具备**完整的地址空间抽象能力**。这意味着你的内核不再依赖物理世界的直接寻址，而开始以“虚拟地址”的视角运行。

## 实验要求

见 [首页#要求和评分标准](index.md#要求和评分标准)

## Part 0：环境配置

### 更新代码

## Part 1：构造 Sv39 页表

### RISC-V Sv39 规范阅读

本节要求同学们**完整阅读**特权级手册中的以下两节：

- [12.3. Sv32: Page-Based 32-bit Virtual-Memory Systems](https://zju-os.github.io/doc/spec/riscv-privileged.html#sv32)
- [12.4. Sv39: Page-Based 39-bit Virtual-Memory System](https://zju-os.github.io/doc/spec/riscv-privileged.html#sv39)

!!! warning "不可跳过规范阅读"

    Sv39 是分页机制的核心，**历年考试几乎必考**。不读标准，你会在位宽、权限位、翻译流程上出现理解偏差。请务必认真通读并结合图表理解。

导读：

- 阅读目标是掌握 RISC-V 的**虚拟地址结构、页表层级与翻译算法**。阅读 **Sv32 了解机制**，阅读 **Sv39 明确差异**。
- **Sv32** 是给 32 位系统设计的，它详细描述了“地址翻译的全过程”，是学习流程的**模板**。
- **Sv39** 面向 64 位系统，只在 Sv32 的基础上说明“有何不同”，例如：

    - 页表层级从 2 级变为 3 级；
    - 虚拟地址长度从 32 位变为 39 位；
    - 页表项（PTE）扩展为 8 字节；
    - 支持 2 MiB、1 GiB 的大页（megapage/gigapage）。

阅读过程中画图是一个很好的习惯。下面这些图来自 [RISC-V Sv32,Sv39 を理解する](https://vlsi.jp/UnderstandMMU.html)，有助于同学们理解 Sv32/Sv39 的各种页类型与地址转换流程：

<figure markdown="span">
    <div style="display:flex; gap:8px; align-items:flex-start;">
        <img src="../lab3.assets/Sv32_Translation.webp" alt="Sv32 地址转换流程" loading="lazy">
        <img src="../lab3.assets/Sv32_Megapage.webp" alt="Sv32 2MiB 大页示意" loading="lazy">
    </div>
    <figcaption>
    Sv32 地址转换与 2MiB 大页<br>
    <small>[RISC-V Sv32,Sv39 を理解する](https://vlsi.jp/UnderstandMMU.html)
    </small>
    </figcaption>
</figure>

<figure markdown="span">
    <div style="display:flex; gap:8px; align-items:flex-start;">
        <img src="../lab3.assets/Sv39_Translation.webp" alt="Sv39 地址转换流程" loading="lazy">
        <img src="../lab3.assets/Sv39_Megapage.webp" alt="Sv39 2MiB 大页示意" loading="lazy">
        <img src="../lab3.assets/Sv39_Gigapage.webp" alt="Sv39 1GiB 大页示意" loading="lazy">
    </div>
    <figcaption>
        Sv39 地址转换与 2MiB、1GiB 大页<br>
        <small>[RISC-V Sv32,Sv39 を理解する](https://vlsi.jp/UnderstandMMU.html)</small>
    </figcaption>
</figure>

??? note "要点：Sv39 分页虚拟内存系统"

    | 类别 | 关键要点 |
    | --- | --- |
    | **地址结构** | Sv39 虚拟地址 39 位：VPN[2:0] 各 9 位，页内偏移 12 位。<br>CPU 访问时检查符号扩展：bit 63–39 必须与 bit 38 相同。 |
    | **页表结构** | 三级页表，每层 512 项、每项 8 B。<br>页表页大小 4 KiB；根页物理基址在 `satp.PPN`。 |
    | **页表项（PTE）格式** | 低 10 位含 V/R/W/X/U/G/A/D；其余为 PPN。<br>R=0 且 W=1 非法；A/D 由硬件在访问时更新。 |
    | **树形结构** | 叶子表项：R/W/X 任一为 1。<br>中间表项：V=1 且 R/W/X=0。 |
    | **权限控制** | V：有效位。<br>R/W/X：读/写/执行许可。<br>U：用户可访问。<br>G：全局映射。<br>A/D：访问/修改标记。<br>非法组合会触发页错误。 |
    | **地址转换流程** | 自 `satp.PPN` 逐级索引：VPN[2] → VPN[1] → VPN[0]。<br>遇叶子即停止；无效/越权/未对齐触发页错误。 |
    | **关键寄存器** | `satp.MODE`=8：Sv39。<br>`satp.ASID`：区分地址空间。|
    | **关键指令** | `sfence.vma`：刷新 TLB，避免旧映射。 |
    | **访存行为控制** | `sstatus.SUM`：允许内核访问 U=1 页。<br>`sstatus.MXR`：允许从可执行页读。 |





### Task 1：恒等映射大页表

!!! tip "术语：映射"

    **映射（mapping）**：将虚拟地址空间的一段连续区域对应到物理地址空间的一段连续区域的过程。

    在 RISC-V 中，映射是通过页表实现的。页表将虚拟地址转换为物理地址，并提供权限控制。

    所以，当我们说“建立映射”时，实际上是指在页表中创建相应的页表项（PTE），以实现虚拟地址到物理地址的转换。

`setup_vm()` 用于构造内核启动初期的页表 `early_pg_dir`。这是一个仅有 Gigapage 的 Sv39 页表，其中有以下映射：

- 恒等映射：虚拟地址 `0x80000000` ~ `0x9fffffff` 映射到物理地址 `0x80000000` ~ `0x9fffffff`；
- 内核映射（下一个 Task 介绍）：虚拟地址 `0xffffffd600000000` ~ `0xffffffffffffff` 映射到物理地址 `0x00000000` ~ `0x1fffffff`。

本次 Task 先做恒等映射。这意味着开启虚拟地址后，虚拟地址 = 物理地址，你的内核应该和 Lab2 一样正常运行。后文再学习怎么切换到内核映射。

你的任务是：

- 在 `setup_vm()` 中，完成对 `early_pg_dir` 的初始化，建立上述恒等映射的大页表。
- 在内核启动代码（`head.S` 的 `_start()`）中，调用 `setup_vm()`，然后设置 `satp` 寄存器，切换到虚拟地址执行。

!!! success "完成条件"

    - 写入 `satp` 寄存器后没有异常，内核和 Lab2 一样正常运行
    - QEMU Monitor 能够正确 Walk Page Table：

        ```text
        (qemu) gva2gpa 0x80000000
        Guest virtual address: 0x80000000
        ```

    - 通过 Lab3 Task1 测试

### 内存屏障与 TLB 刷新

### PC 相对寻址

在《计算机组成》课上我们学习过 RISC-V 有几种寻址方式：

- 立即数寻址（Immediate Addressing）
- 寄存器寻址（Register Addressing）
- 直接寻址（Direct Addressing）
- 间接寻址（Indirect Addressing）
- 基址加偏移寻址（Base plus Offset Addressing）
- **PC 相对寻址（PC-Relative Addressing）**

### 切换到虚拟内存

## Part 2：内核内存布局

RISC-V 架构下的 Linux 内核的内存布局见 [Virtual Memory Layout on RISC-V Linux — The Linux Kernel documentation](https://docs.kernel.org/arch/riscv/vm-layout.html)。我们采取简化的模型。

- **内核与用户空间：**

    同学们已经了解到，Sv39 分页式虚拟内存系统中，64b 的虚拟地址实际有效的只有 39b，其余高位必须与 bit 38 保持符号扩展一致。这意味着整个 $2^{64}\text{B}$ 的地址空间被划分为两个对称的区域：

    - **用户空间**：`0x0000000000000000` ~ `0x0000003fffffffff`（低地址半区）
    - 无效地址：多达 16M TB。
    - **内核空间**：`0xffffffc000000000` ~ `0xffffffffffffffff`（高地址半区）

    在本实验中，我们只实现内核空间的映射，用户空间的映射将在 Lab4 实现。

- **内核空间：**

    Linux 的内核空间布局较为复杂，包含设备 I/O、内核堆栈等区域，我们全部弃用。本实验只实现直接映射，即将 Lab2 中介绍的从 `0x80000000` 开始的物理内存全部映射到 `0xffffffd600000000` 开始的虚拟地址空间。

### Task 2：内核映射多级页表

目标：按照链接脚本中各段的布局，给内核镜像提供精确的页表映射与权限控制。

- 结合 `arch/riscv/kernel/vmlinux.lds` 中的符号（如 `_stext/_etext/_srodata/_erodata/_sdata/_edata/_sbss/_ebss`），完成：
    - `.text`：X|R，禁止 W；
    - `.rodata`：R，禁止 W/X；
    - `.data` 与 `.bss`：R|W，禁止 X；
    - 内核栈与早期栈：R|W，禁止 X；
    - 必要的 MMIO 区域映射（若需要），设置非可执行且符合设备访问的属性。
- 清理早期的临时恒等映射（若在 Task 1 使用过），验证仍可稳定运行。

!!! example "动手做"

- 有意对 `.text` 做一次写，期望触发 store page fault；
- 有意从 `.data` 执行一条指令，期望触发 instruction page fault。

!!! success "完成条件"

    - 通过 Lab3 Task2 测试。

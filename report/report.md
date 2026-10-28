# 操作系统 Lab3：中断与异常处理

小组成员：2412932 李潇健、2410718 姚钧、2410764 朱彦泽  
实验日期：2026-10-25

## 一、实验目标与整体逻辑

Lab3 在 Lab2 的内存管理内核上增加 Trap 处理能力。实现顺序是先建立 Trap frame 的保存/恢复规则，再将汇编入口接到 C 函数，之后按 `scause` 分发中断和异常，最后恢复现场并以 `sret` 回到原执行流。时钟中断用于验证处理能够重复发生；断点和非法指令用于验证异常能被识别且返回位置能越过已处理的指令。

RISC-V 硬件在 Trap 发生时记录 `sepc`、`scause`、`sstatus` 等状态并跳至 `stvec`。本实验的 `__alltraps` 使用 `SAVE_ALL` 将通用寄存器及相关 CSR 组成 `struct trapframe`，通过 `a0` 把 frame 地址传入 `trap()`。C 层处理完成后，`RESTORE_ALL` 取回寄存器和 CSR，`sret` 根据特权级状态返回。

## 二、核心模块

### Trap 上下文入口和恢复

`SAVE_ALL` 预留 `36 * REGBYTES` 空间，前 32 个槽对应 x0-x31，后四个槽对应 `status`、`epc`、`badvaddr`、`cause`。`sp` 槽保存的是 Trap 前的栈指针，不能把它当作普通寄存器在恢复过程中提前覆盖当前 frame 基址。实现将 `sstatus` 和 `sepc` 写回 CSR，在恢复其他通用寄存器之后最后恢复 `sp`。frame 大小在 RV64 上为 288 字节，是 16 字节的倍数，满足 C ABI 栈对齐要求。

### 中断处理

`trap_dispatch()` 使用 `scause` 最高位区分中断与异常。`interrupt_handler()` 清除最高位后识别中断号。对 `IRQ_S_TIMER`，先调用 `clock_set_next_event()` 安排下一次事件，再递增 `ticks`；每 100 tick 输出一次统计信息，达到 10 次统计后调用 SBI `sbi_shutdown()`。Supervisor 软件和外部中断输出类型信息；其他中断输出 cause 并调用 `print_trapframe()`。

### 异常处理

`exception_handler()` 处理非法指令和断点异常。RISC-V 压缩指令的低两位不为 `11`，长度为 2 字节；低两位为 `11` 时本实验所涉及的基础指令长度为 4 字节。处理器报告的 `sepc` 指向触发异常的指令，因此 handler 根据指令首个 16 位半字推进 `tf->epc`，避免返回后重复执行。其他异常通过 `print_trapframe()` 输出诊断信息。

## 三、Spec 与提示词

### 练习一：上下文保存恢复和 Trap 入口

Spec 要求：不修改 `SAVE_ALL`、`struct trapframe` 和 `trap()` 签名；严格依据现有槽位恢复寄存器与 CSR；正确处理特殊的 `sp` 槽；将 frame 地址按 ABI 传给 C handler；handler 返回后恢复现场并 `sret`。

提示词要求逐项对照汇编宏与结构体字段，解释 `sp` 保存关系、CSR 恢复顺序、C 调用参数寄存器和栈对齐，再只实现标记的两个汇编部分。完整摘要见 [`prompt.md`](./prompt.md)。

### 练习二：中断与异常 handler

Spec 要求：timer 每次重新设置下次事件并累计 tick；每 `TICK_NUM` 次打印；累计打印到 `SHUTDOWN_AFTER` 后关机；识别软件、外部、未知中断；对非法指令和断点打印 EPC 并按编码长度推进；其他异常打印完整 frame。

提示词要求只使用框架已有宏、`clock_set_next_event()`、`ticks`、`sbi_shutdown()` 和 `print_trapframe()` 等接口，不改变签名或改动无关函数。指令长度按低两位判定，不能统一假定 `epc + 4`。

## 四、AI 代码生成中的问题与修复

复核评分脚本时发现，未知中断测试除要求 frame 诊断外，还精确匹配 `Unknown interrupt: cause 15`。实现因此同时打印 cause 和完整 frame。另一个边界是压缩指令可能只有 2 字节；若用未对齐的 32 位读取判断长度，可能引入额外访问问题，因此改为读取 `uint16_t` 后判断低两位。

## 五、原理知识对照

| 原理知识 | 本实验体现 | 含义、关系和边界 |
| --- | --- | --- |
| 中断与异常 | `scause` 分发 | 中断为异步事件，异常由当前指令同步触发；RISC-V 将二者统称 Trap。 |
| 上下文保护 | `trapframe`、`SAVE_ALL`、`RESTORE_ALL` | 保存被打断执行流的状态，使 handler 可运行后恢复；这里只保存单个内核执行流的整数寄存器，不包含调度和 FPU 状态。 |
| 中断向量 | `stvec` 指向 `__alltraps` | 所有 Trap 进入统一入口后软件分发；本实验没有按原因建立独立向量表。 |
| 时钟中断 | SBI timer 与 `ticks` | 周期事件提供未来调度的时钟基础；当前没有进程时间片或定时器队列。 |
| 中断屏蔽与临界区 | `intr_enable()`、`intr_disable()`、`sync.h` | 关闭本地中断可避免当前 hart 被打断；它不等同于多核锁，不能单独保护跨 hart 共享状态。 |

OS 原理中重要但本实验未实现的内容包括进程和线程调度、系统调用、用户态 Trap 返回、多核同步、设备中断确认与路由、嵌套中断优先级、缺页异常恢复、软硬件上下文完整切换及信号机制。

## 六、实现与验证边界

代码实现位于 `kern/trap/trapentry.S`、`kern/trap/trap.c` 和 `kern/trap/trap.h`；另外从 Lab2 成果合入了完整的 `kern/mm/pmm.c` 与 `kern/mm/best_fit_pmm.c`，补齐 Lab3 框架的前置内存管理缺口。WSL 课程环境中执行 `source ~/os-lab/activate.sh` 后，使用 GCC 10.2.0 完整构建成功。`make grade` 的 Trap/断点、非法指令/兜底、持续时钟三项分别通过 35/35、25/25、40/40，总分 100/100。`timeout 5s make qemu` 也确认了内核启动、Best-Fit 自检成功并连续产生 tick；timeout 结束常驻内核属于预期。汇编对框架已有的 `sbadaddr` 名称发出 deprecated alias 警告，但不影响构建和测试。

当前 `SHUTDOWN_AFTER` 取 10 次统计输出；如果课程讲义或教师要求了其他阈值，应以其为准并重跑测试。成员分工和真实个人贡献见根目录 [`任务分工.md`](../任务分工.md)，完整环境和命令记录见 [`实验记录.md`](../实验记录.md)。

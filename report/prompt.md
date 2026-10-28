# Lab3 实际提示词与实现规格

本文件记录本次实现所依据的提示词规格摘要，便于按实验要求说明 AI 辅助代码生成过程。它不是逐字聊天导出，也不代表每位组员分别使用了相同提示词。小组成员：2412932 李潇健、2410718 姚钧、2410764 朱彦泽；成员贡献请按实际情况核对。

## P1：Trap 上下文恢复与入口

请阅读 `kern/trap/trapentry.S` 中现有的 `SAVE_ALL`、`kern/trap/trap.h` 中的 `pushregs`/`trapframe`，以及 C 接口 `trap(struct trapframe *tf)`。只实现标记的 `RESTORE_ALL` 和 `__alltraps`。必须逐项遵循现有 36 槽布局；恢复 `sstatus` 与 `sepc`；恢复所有通用寄存器；`sp` 保存的是陷入前栈指针，不能在仍需 frame 基址时提前覆盖当前 `sp`；将 frame 地址按 RISC-V ABI 传给 `trap()`；C 返回后进入 `__trapret` 并由 `sret` 返回。不要修改 `SAVE_ALL`、结构体、函数签名或其他模块。检查栈对齐以及所有槽位是否一一对应。

## P2：中断与异常

请根据 `trap.c` 中的占位、`riscv.h` 的 `IRQ_S_TIMER`、`IRQ_S_SOFT`、`IRQ_S_EXT`、`CAUSE_ILLEGAL_INSTRUCTION`、`CAUSE_BREAKPOINT`，以及 `clock.h`、`sbi.h` 实现 `interrupt_handler()` 和 `exception_handler()`。`scause` 最高位表示中断，先清除该位再判断 cause。时钟每次都调用 `clock_set_next_event()`、递增 `ticks`；每 100 次输出一次统计信息，累计达到 10 次统计后调用 `sbi_shutdown()`。软件和外部中断输出类型提示；未知中断打印原因并调用 `print_trapframe()`。非法指令和断点输出 EPC 信息，按 RISC-V 指令低两位判断 2 字节或 4 字节长度并推进 `tf->epc`，避免重复执行；其他异常调用 `print_trapframe()`。不要更改接口和规定外的函数。

## 发现的测试约束

`tools/grade.sh` 要求未知 IRQ 15 输出 `Unknown interrupt: cause 15`，并要求 timer 至少持续触发到 200 ticks。报告应同时记录测试输出、失败定位和修复，不得将静态复核写成已通过运行测试。

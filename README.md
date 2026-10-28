# Lab3：中断与异常处理

本目录以 `kern/`、`libs/`、`tools/` 和 `Makefile` 构成的 Lab3 原始框架为基础，完成 Trap 上下文保存/恢复、异常入口和时钟/异常处理。`report/` 保存实验报告与提示词记录，根目录的 [`任务分工.md`](./任务分工.md) 和 [`实验记录.md`](./实验记录.md) 记录职责及验收过程。

本机 WSL 的课程环境位于 `~/os-lab`。在 WSL 终端运行：

```sh
source ~/os-lab/activate.sh
cd ~/os-lab/projects/lab3
make -j2
make grade
```

`make grade` 会在临时副本中注入 Trap 上下文、非法指令和定时器测试，不会把测试代码写入正式源码。当前环境验证结果见 [`实验记录.md`](./实验记录.md)。

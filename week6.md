
## 进展

实现了基础的信号处理，包括：

- 增加 `axhal` 功能，处理函数参数、返回值、返回地址
- 从内核空间回到用户空间时的信号处理
- x86_64 上的 fork 实现
- 4 组相关测试
- 系统调用：`rt_sigprocmask`、`rt_sigaction`、`rt_sigreturn`、`rt_sigpending`、`kill`

[starry-next#28](https://github.com/oscomp/starry-next/pull/28)、[arceos#23](https://github.com/oscomp/arceos/pull/23)

## 记录

调查之后意识到 futex 优先级应该在实现线程后，因为 futex 本质上是基于共享的内存做 wait queues，如果没有实现线程或者 shared memory 的话 futex 是无法测试的。

开始实现 signal。Linux 信号分为 32 个标准信号和实时信号，准备先实现标准信号部分。

参考资料：

- [man 7 signal](https://man7.org/linux/man-pages/man7/signal.7.html)
  - 这里特别是 Execution of signal handlers 部分
- [man 2 sigaction](https://man7.org/linux/man-pages/man2/sigaction.2.html)
- [信号终结进程的返回值](https://unix.stackexchange.com/a/99143)

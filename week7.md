## 进展

- 完善了信号处理的相关功能，包括 `rt_sigtimedwait`、`rt_sigwaitinfo`、`rt_sigsuspend`，这部分是基于 `WaitQueue` 的。目前信号部分组件的完成阻塞在和进程的整合上
- 组件模块化：指针相关独立为 [axptr](https://github.com/Starry-OS/axptr)、信号相关独立为 [axsignal](https://github.com/Starry-OS/axsignal)
- 尝试适配了 libctest，前几个测试可以通过但后面有测试会死循环导致 CI 无法完成（CI 没有配置超时），所以暂时没有分

下周计划尽快将信号与进程整合，并尝试修复目前 libctest 超时的问题。整合完成后希望可以开始着手 futex 的实现。

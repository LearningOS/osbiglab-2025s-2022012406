## 进展

- 完善信号处理、支持实时信号 & set_tid：[9441db6](https://github.com/Starry-OS/starry-next/commit/9441db696675011160106df033e5021e69208583)、[e428b0f](https://github.com/Starry-OS/starry-next/commit/e428b0fff6465bf8a09634e38fc9404ebeb01de8)、[245827b](https://github.com/Starry-OS/starry-next/commit/245827b4b97a2b17e2c4a06927213a8ccec009b4)、[40d6613](https://github.com/Starry-OS/starry-next/commit/40d6613823b542937a29a51977d629e20fc44e9b)、[509e7a0](https://github.com/Starry-OS/starry-next/commit/509e7a0c642998388530b1153978328f25c8b523)、[81f0100](https://github.com/Starry-OS/starry-next/commit/81f010093d35159d40b92cb58c0ed8a4504b498e)、[d692afc](https://github.com/Starry-OS/starry-next/commit/d692afce362d7ec0f34376a867dad9554e42c60b)
- 实现基础 futex 功能（`FUTEX_WAIT`、`FUTEX_WAKE`、`FUTEX_REQUEUE`、`FUTEX_CMP_REQUEUE`：[509e7a0](https://github.com/Starry-OS/starry-next/commit/509e7a0c642998388530b1153978328f25c8b523)
- 实现杂项 syscall（`lstat`、`rename`、`faccessat`）：[333e96e](https://github.com/Starry-OS/starry-next/commit/333e96e2d7bdafe3580fda03015b3e7cbc49d32c)、[9a2be47](https://github.com/Starry-OS/starry-next/commit/9a2be47e43813c691340db632189b344fc9eed2c)
- 重构文件系统 axfs-ng，支持多核，现在实现了 fatfs（整合了 google fuchsia 的一些 patches）

下周计划：重构合并信号功能到主线，完成 axfs-ng

## 进展

- 整合信号相关 PR 到主线：[arceos#34](https://github.com/oscomp/arceos/pull/34)、[starry-next#38](https://github.com/oscomp/starry-next/pull/38)
- 添加了 SIGHAND 相关 flag 处理、修复了一些信号处理的 bug
- 添加了 `rt_sigqueueinfo` syscall：[starry-next#38](https://github.com/oscomp/starry-next/pull/38)
- 完善 `axfs-ng` 的高层 API，添加了相关 tests，发布在 https://github.com/Starry-OS/axfs-ng
- 尝试添加 ext4 支持，使用的是 https://github.com/PKTH-Jx/another_ext4 ，但实现好之后发现有一些问题，看到 `lwext4` 的代码发现有一些不符合需求

下周计划：基于 `lwext4-sys` 实现 ext4 支持，添加 `axfs-ng` 的相关测试，尝试整合到 arceos

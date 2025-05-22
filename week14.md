## 进展

- 通过全部基础测例（glibc 测例部分有问题，暂时使用 musl）
- 文件系统部分：
  - 添加 `statfs` 获取文件系统整体信息
  - 支持设备文件
  - 添加 `/dev/zero`、`/dev/null`、`/dev/random`、`/dev/urandom`、`/dev/rtc0`
  - 支持创建文件时传入的用户组信息
  - 修复重命名覆盖行为
  - 修复 mountpoint 解析
  - 修复可能的内存泄漏
- 支持共享内存
  - `axmm` 中添加新的 `Backend::Shared`
- 添加 Loongarch RTC 支持（Loongson LX7A）
- 实现 Robust list 实现（futex 相关）
- 添加同时最多文件描述符数限制
- 支持 `select` 系统调用

## 下周计划

整理报告

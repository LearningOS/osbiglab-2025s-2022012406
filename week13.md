## 进展

- 合并 futex、shebang 到主线，并修复了死锁问题: [arceos#39](https://github.com/oscomp/arceos/pull/39), [arceos#40](https://github.com/oscomp/arceos/pull/40), [starry-next#45](https://github.com/oscomp/starry-next/pull/45), [starry-next#46](https://github.com/oscomp/starry-next/pull/46), [starry-next#48](https://github.com/oscomp/starry-next/pull/48), [starry-next#49](https://github.com/oscomp/starry-next/pull/49)
- 实现了 mount/umount，添加了测试。这部分工作量比较大，包括将原有 `DirEntry` 包装为 `Location` 并整合底层操作、DeviceID 分配、多重 Mount 处理等
- 添加 FatFS inode 分配
- syscall 实现：软链接、chmod、chown、pread、pwrite、preadv、pwritev、utimensat、ftruncate、fsync、fdatasync、mkdir、rmdir、faccess、getdents64 以及他们的各种变种
- 实现 `/proc` 和 `/tmp` 的部分功能，搭建自定义 VFS 的基本框架
- 为现有的 `*at` 系列 syscall 添加/完善了 `AT_SYMLINK_NOFOLLOW` 和 `AT_EMPTY_PATH` 的支持
- 调试与修复了一些重入锁
- 修复 IRQ 死锁问题，由 [arceos#42](https://github.com/oscomp/arceos/pull/42) 造成，情况告知了助教
- 编写了 `EvaluationScript` 的本地测试脚本

- iozone：通过上面的 syscall 实现和修复通过了 sanity check，后续部分需要 shmem 实现才能得分

下周计划：整合 shmem，修复一些 libctest 测例，尝试在 iozone 取得分数，尝试将 axfs-ng 初步合入主线

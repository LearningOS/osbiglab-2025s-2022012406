
## 进展

- 合并 futex、shebang 到主线
- 实现了 mount/umount，添加了测试。这部分工作量比较大，包括将原有 `DirEntry` 包装为 `Location` 并整合底层操作、DeviceID 分配等
- 添加 FatFS inode 分配
- 文件系统功能与 syscall 实现：软链接、tmpfs、chmod、chown、pread、pwrite、preadv、pwritev、utimensat、ftruncate、fsync、fdatasync、mkdir、rmdir、faccess、getdents64 以及他们的各种变种
- 修复一些重入锁
- 修复 IRQ 死锁问题

- iozone：通过上面的 syscall 实现和修复通过了 sanity check，后续部分需要 shmem 实现才能得分

下周计划：整合 shmem，修复一些 libctest 测例，尝试在 iozone 取得分数，尝试将 axfs-ng 初步合入主线

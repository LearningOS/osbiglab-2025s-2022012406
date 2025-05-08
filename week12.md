## 进展

- 基于 `lwext4` 重新实现了 `lwext4_rust`，使用 RAII 管理资源
  - 这部分由于 `lwext4` 原生只支持全局单个 mountpoint，若需要支持同时创建多个文件系统实例需要将大部分核心逻辑（包括读取/写入文件、重命名、删除等）重新实现，花费较多时间
  - 原 `lwext4_rust` 实现不兼容 std（会 segfault），无法进行测试，添加了 feature 使得其可以兼容 no_std 和 std，可以正常运行测试
- `axfs-ng` 拆分为 `axfs-ng-vfs`（独立托管）和 `axfs-ng`（集成到 arceos）
- 重构了 `axfs-ng-vfs` 的设计（`Node` -> `DirEntry`），贴近 Linux VFS 实现，同时拆分 trait 易于实现文件系统（`NodeOps`、`FileNodeOps`、`DirNodeOps`）
- 将 `axfs-ng` 集成到 arceos 中，替换了原有大部分 syscall，通过了基础测试

推迟通过测例的原因在于，目前有两条线，一条是之前从主线 branch 出来的实现了较多功能的支线（也是目前得分较高的 branch），一条是主线。但在给主线整合信号功能 PR 的过程中又修复了许多 bug 并加入了特性，因此支线的实现在某些程度上又落后于主线，但将支线的实现 merge 到主线又显得费力不讨好（因为目前 axfs 的重构是在主线的基础上开发的）。因此目前的思路是逐渐将支线的所有特性 merge 到主线，完成之后再在目前的主线上新开支线过测例。

下周计划：实现 mount/umount，集成 shebang、futex 等 feature 到主线，调研 `iozone`

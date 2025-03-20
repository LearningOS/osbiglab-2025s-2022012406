
## 进展

- 添加了 `mprotect` syscall ([starry-next#14](https://github.com/oscomp/starry-next/pull/12))
- 添加了 `AddrSpace::populate_area` 以补全内存中的延迟加载页，为 `mprotect` 的实现提供支持，同时优化了用户访存（`UserPtr::get`）的性能 ([arceos#14](https://github.com/oscomp/arceos/pull/14))
- 修复了 ArceOS 中 `TaskExt` 无法被正确回收，导致内存泄漏的问题 ([arceos#20](https://github.com/oscomp/arceos/pull/20))
- 修复了 `AddrSpace` 回收时损坏内核页表的问题 ([arceos#20](https://github.com/oscomp/arceos/pull/20), [starry-next#17](https://github.com/oscomp/starry-next/pull/17))

本周还尝试实现线程、进程组的支持，但发现其依赖的未实现功能过多（信号、futex），因此暂时搁置。

下周计划实现 futex。

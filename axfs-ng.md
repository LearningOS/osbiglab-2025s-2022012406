# `axfs-ng`

该 crate 是对原有 `axfs` 的完全重构，主要改进如下：

- 摈弃原有的基于路径的文件系统操作，改为基于 inode。这样能够更好地支持多种系统调用（`openat`、`fstatat` 等），且效率更高
- 支持原 `axfs` 不提供的文件系统操作，包括完整的 `stat`（包含 inode、所有者、权限等）、软/硬 链接、重命名等
- 支持并发访问文件系统
- 灵活支持挂载操作（`mount`），并支持自定义实现文件系统（如 devfs、procfs 等）

## 架构设计

类似于原 `axfs-vfs` 与 `axfs`，我们也分为 `axfs-ng-vfs` 和 `axfs-ng` 两个 crate。

`axfs-ng-vfs` 提供文件系统的抽象，和基础的操作接口。`axfs-ng` 则包含了包括路径解析、当前路径（thread-local）以及具体文件系统的实现（如 ext4、vfat 等）。

### 节点

文件系统中的每个文件或目录都被称为一个节点（Node），关键定义如下：

```rust
/// 通用节点接口
pub trait NodeOps<M>: Send + Sync {
    /// 获取 inode 编号
    fn inode(&self) -> u64;
    /// 获取节点元数据
    fn metadata(&self) -> VfsResult<Metadata>;
    /// 更新节点元数据
    fn update_metadata(&self, update: MetadataUpdate) -> VfsResult<()>;
    /// 获取节点的文件系统操作接口
    fn filesystem(&self) -> &dyn FilesystemOps<M>;
    /// 获取节点大小
    fn len(&self) -> VfsResult<u64>;
    /// 同步节点到存储介质
    fn sync(&self, data_only: bool) -> VfsResult<()>;
    /// 将节点转换为任意类型
    fn into_any(self: Arc<Self>) -> Arc<dyn core::any::Any + Send + Sync>;
}

/// 文件节点特化的操作接口
pub trait FileNodeOps<M>: NodeOps<M> {
    /// 带偏移读取
    fn read_at(&self, buf: &mut [u8], offset: u64) -> VfsResult<usize>;
    /// 带偏移写入
    fn write_at(&self, buf: &[u8], offset: u64) -> VfsResult<usize>;
    /// 追加写入，返回写入的字节数和新的文件大小
    fn append(&self, buf: &[u8]) -> VfsResult<(usize, u64)>;
    /// 设置文件大小
    fn set_len(&self, len: u64) -> VfsResult<()>;
    /// 设置文件软链接目标
    fn set_symlink(&self, target: &str) -> VfsResult<()>;
}

/// 目录节点特化的操作接口
pub trait DirNodeOps<M: RawMutex>: NodeOps<M> {
    /// 读取目录内容，返回读取的条目数
    fn read_dir(&self, offset: u64, sink: &mut dyn DirEntrySink) -> VfsResult<usize>;
    /// 按名字查找目录条目
    fn lookup(&self, name: &str) -> VfsResult<DirEntry<M>>;
    /// 创建子节点
    fn create(
        &self,
        name: &str,
        node_type: NodeType,
        permission: NodePermission,
    ) -> VfsResult<DirEntry<M>>;
    /// 链接一个现有节点到目录
    fn link(&self, name: &str, node: &DirEntry<M>) -> VfsResult<DirEntry<M>>;
    /// 删除目录条目
    fn unlink(&self, name: &str) -> VfsResult<()>;
    /// 重命名
    fn rename(&self, src_name: &str, dst_dir: &DirNode<M>, dst_name: &str) -> VfsResult<()>;
}
```

实现自定义的操作系统时，需要对节点实现 `NodeOps`，再根据需要实现 `FileNodeOps` 或 `DirNodeOps`。

`FileNodeOps` 和 `DirNodeOps` 并不直接对外暴露，他们各自被 `FileNode` 和 `DirNode` 包装，提供了更具体的操作接口。

### `DirEntry`

虽然有 `FileNode` 与 `DirNode` 的包装，但他们类型并不兼容。`DirEntry` 将他们二者聚合为统一类型，并向外暴露了接口。此外，`DirEntry` 还包含了节点的具体类型（如常规文件或软链接）以及节点的名字和父节点，从而支持了遍历文件树、获取绝对路径等功能。

可以通过 `DirEntry::new_file` 和 `DirEntry::new_dir` 将 `FileNode` 和 `DirNode` 转换为 `DirEntry`。

### `Location`

但在实际的文件系统操作中，由于挂载（mount）功能的存在，单个 `DirEntry` 并不能完全描述一个文件或目录的位置。一个系统中可能同时存在多个文件系统的多个节点，因此我们引入了 `Location` 类型来描述一个节点的完整位置。其定义如下：

```rust
pub struct Location<M> {
    mountpoint: Arc<Mountpoint<M>>,
    entry: DirEntry<M>,
}
```

在此基础上，`Location` 封装提供了多种操作接口。**在大多数情况下，你都应该只使用 `Location` 来操作文件**。如果例外，可以通过 `Location::entry` 获取 `DirEntry` 来进行更底层的操作。

---

### `FsContext`

**从这里开始的内容都是 `axfs-ng` 中独有的**

`FsContext` 是解析路径的上下文，其包含两个字段：根目录与当前目录；二者类型均为 `Location`。在实际的操作系统中，可以通过 `chroot` 来改变根目录、`chdir` 来改变当前目录。

`FsContext::resolve` 将人类可读的路径解析为 `Location`，并在此过程中处理了符号链接的解析。此外，`FsContext` 还暴露了类似 `std::fs` 的接口，如 `create_dir`、`read_dir`、`rename`、`link`、`symlink` 等。

### `File`

`File` 在 `Location` 的基础上，包装了文件当前偏移量和文件打开权限，从而提供了 UNIX 风格的文件操作接口，如 `read`、`write`、`seek` 等。

### `OpenOptions`

`OpenOptions` 与 `std::fs::OpenOptions` 基本相同，用于配置打开文件的选项。不同之处在于：

- `OpenOptions` 包含更多底层选项，如配置打开文件时使用的用户身份、权限等
- 由于 `OpenOptions` 也支持打开文件夹，因此 `OpenOptions::open` 返回一个 `OpenResult` 而非 `File`

`OpenResult` 定义如下：

```rust
pub enum OpenResult<M> {
    File(File<M>),
    Dir(Location<M>),
}
```

## 具体实现

### 路径操作

路径操作借鉴了 `std::path` 的设计，提供了 `Path` 和 `PathBuf` 两个类型。与 `std::path` 不同的是我们不需要考虑非 UTF-8 以及 Windows 盘符的问题，因此设计上更简单。

实现了一些统一方便的接口，如 `Path::components` 方法来获取路径的各个组件、`Path::join` 方法来拼接路径、`Path::file_name` 来获取路径的文件名等。避免了像原 `axfs` 那样需要在几乎每个文件系统操作中都手动解析路径。

### 目录项缓存

类似于 Linux 中的 dentry，我们实现了目录项的缓存机制。其目的一方面在于加快目录项的查找速度，另一方面是确保单个目录项在内存中只存在一份，从而避免多个重复目录项存在情况下可能的并发错误。

其具体实现在 `DirNode` 中。`DirNode` 里包含 `cache: Mutex<BTreeMap<String, DirEntry>>` 成员，用互斥锁保护了一个缓存的目录项表。`DirNode::lookup` 方法会先在缓存中查找目录项，如果未找到再调用 `DirNodeOps::lookup` 方法生成对应的目录项，并将其加入缓存。`DirNode::rename`、`DirNode::unlink` 等方法也会在调用底层 `DirNodeOps` 的基础上清除对应的目录项缓存，保证了缓存的一致性。

### 挂载点

UNIX 系统支持将文件系统挂载到另一文件系统的子目录下，当在访问该子目录时，实际上访问的是挂载的文件系统。我们在 `axfs-ng` 中也实现了类似的功能。具体是定义了 `Mountpoint` 类型包含了挂载点的相关信息（如文件系统实例、根目录等），并在 `DirEntry` 中记录了 `mountpoint: Mutex<Option<Arc<Mountpoint>>>` 字段来记录该 `DirEntry` 上是否挂载了文件系统。

在 `Location::lookup` 中，在 `DirEntry::lookup` 的基础上，还会额外检查查找结果的目录项是否是挂载点。如果是，则会返回该挂载点的根目录作为查找结果。我们还正确处理了同一目录被挂载多次的情况，实现的部分均符合 UNIX 规范。

### vfat（fat16、fat32）

与 `axfs` 一样，我们基于 `rust-fatfs` 实现了对 vfat 的支持。此外，由于原 `rust-fatfs` 缺少对应实现，我们需要手动添加如获取文件更改时间、获取文件夹相关元数据之类的接口。

由于 `rust-fatfs` 的设计与我们类似，将文件和文件夹区分开来，因此对应实现也是相当直接的。

`rust-fatfs` 的接口只考虑了单线程访问，例如 `fatfs::Dir` 包含了对 `fatfs::FileSystem` 的引用，因此较难扩展到多线程。为了确保并发安全性，我们在全局的 `fatfs::FileSystem` 上添加了互斥锁，并定义了 `FsRef<T>` 类型用于包装 `fatfs::Dir` 和 `fatfs::File`，要求外部提供 `fatfs::FileSystem` 的引用来获取 `T` 的引用，从而确保了多线程访问的安全性。

### ext4

我们基于 `lwext4` 实现对 ext4 的支持。但其原有的 Rust 封装，也是 `axfs` 使用的 `lwext4_rust` 继承了 `axfs` 的设计，将文件系统操作与路径解析混合，因此完全重写了 `lwext4_rust`，提供了基于 inode 的文件系统操作接口。

此外，由于原 `lwext4` 部分操作强依赖于全局变量（如 `read`、`write` 等），无法同时创建多个实例，因此也需要重写这些接口。这部分完全在 Rust 中完成，避免了改变 `lwext4` 暴露的接口。

### 同步操作解耦

文件系统的实现中不可避免会涉及到同步操作（这里只用到互斥锁 `Mutex`）。为了避免对 `axsync` 的强耦合，我们对关键 trait 均添加了 `M: lock_api::RawMutex` 泛型参数。`lock_api` 基于 `RawMutex` trait 提供了具体的 `Mutex` 类型，可以根据需要选择不同的锁实现，达成了同步操作的解耦。

## 使用举例

```rust
/// 创建文件系统
let disk = RamDisk::from(&std::fs::read("resources/ext4.img")?);
let fs = fs::ext4::Ext4Filesystem::<RawMutex>::new(disk)?;

/// 创建挂载点对象（根目录也算挂载点）
let mount = Mountpoint::new_root(&fs);

/// 根据 mountpoint 创建文件系统上下文
let cx: FsContext<RawMutex> = FsContext::new(mount.root_location());

/// 列举根目录内容
for entry in cx.read_dir("/")? {
    let entry = entry?;
    println!("- {:?} ({:?})", entry.name, entry.node_type);
}

/// 打开文件
let mut file = File::create(&cx, "test.txt")?;
file.write_all(b"Hello, world!")?;
drop(file);

/// 创建文件夹
let mode = NodePermission::from_bits(0o766).unwrap();
cx.create_dir("temp", mode)?;


/// 挂载子文件系统
let disk = RamDisk::from(&std::fs::read("resources/fat16.img")?);
let sub_fs = fs::fat::FatFilesystem::<RawMutex>::new(disk);
cx.resolve("temp")?.mount(&sub_fs);
```

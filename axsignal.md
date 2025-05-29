# axsignal

该 crate 提供了与信号机制相关的核心类型，并提供了一些通用的信号处理逻辑以便上层使用。其并不包含具体的信号相关 syscall 实现，因此与操作系统解耦，可以在不同的操作系统上使用。

## 架构设计

### `Signo`

`Signo` 是一个枚举类型，表示不同的信号。包含了 POSIX 标准的信号和实时信号，共 64 种，编号 `1-64`。

### `SignalInfo`

对应 POSIX 的 `siginfo_t`，表示信号的详细信息。包含了信号编号、来源类型以及其他额外信息。

### `SignalSet`

对应 POSIX 的 `sigset_t`，表示一个信号的集合。提供了添加、删除、检查信号等操作。

### `SignalDisposition`

表示用户自定义的对特定信号的处理方式。可以设置为忽略、捕获或使用默认处理。

### `SignalAction`

对应 POSIX 的 `sigaction`，表示对特定信号的处理动作。在 `SignalDisposition` 的基础上，还包含了信号处理时的标志位（例如使用何种函数签名、是否使用自定义栈等）以及信号处理时使用的信号 mask。

### `SignalFrame`

在信号处理过程中，内核会在栈上创建一个 `SignalFrame`，用于保存信号处理的上下文信息。它包含了信号信息、寄存器状态等信息，以便在信号处理完成后恢复执行状态。

### `ProcessSignalManager`

提供给上层使用的，在进程层面需要存储的信号相关状态以及处理逻辑。他包含查询是否有待处理信号、发送信号、等待任意信号到达等功能。

### `ThreadSignalManager`

提供给上层使用的，在线程层面需要存储的信号相关状态以及处理逻辑。例如屏蔽信号、设置信号栈这些功能都是每个线程独立的。提供了处理信号、等待信号等功能。

`ThreadSignalManager::check_signals` 会检查是否有待处理的信号，如果有则对应处理。然而，为了和操作系统解耦，并没有实现具体的信号处理逻辑，而是返回 `SignalOSAction` 类型，表示操作系统应该如何处理该信号。其定义如下：

```rust
pub enum SignalOSAction {
    /// 终结进程
    Terminate,
    /// 生成核心转储并终结进程
    CoreDump,
    /// 暂停进程
    Stop,
    /// 继续进程
    Continue,
    /// 用户自定义信号处理函数
    Handler,
}
```

## 具体实现

### 自定义信号处理函数

如果用户自定义了信号处理函数，则在处理信号时（`ThreadSignalManager::check_signals`）用户的上下文会被保存到栈上的 `SignalFrame` 中，并被跳转到用户自定义的信号处理函数。

仅仅将程序指针寄存器设置为处理函数地址是不够的，返回地址该如何设置是需要考量的一环。用户信号处理函数执行完返回时依旧在用户态，因此返回地址对应的函数需要重新陷入内核态才能由内核恢复原有的执行状态。这部分函数（返回地址所对应的函数）被称为 restorer。在 POSIX 中，用户在设置信号处理函数时可以选择指定自定义的 restorer。但如果用户不提供 restorer，我们该如何处理呢？

Linux 上使用了 vdso 机制来处理该问题，其本质是将内核的部分代码映射到用户空间，这样在用户空间就可以访问我们预定义的 restorer 函数了。在 starry-next 中，我们使用了类似的机制，将预定义的 restorer 嵌入到内核代码中，并在创建用户空间时将其映射到用户空间。

### 信号队列

线程和进程各自需要维护一个信号队列，用于存储待处理的信号。可以分别通过 `tkill`（或 `rt_sigqueueinfo`）和 `kill` 系统调用来向线程或进程发送信号。进程收到的信号会被进程中的随机线程处理。

为此，内部使用 `PendingSignals` 类型来表示信号队列。其对 POSIX 标准信号和实时信号进行了区分，分别使用 `Option<SignalInfo>` 和 `VecDeque<SignalInfo>` 来存储这两种信号；并使用 `SignalSet` 来记录当前待处理的信号集合，从而实现快速查询。

### 操作系统解耦

为了和操作系统解耦，将核心需要用到的两项功能：互斥锁和 `WaitQueue` 抽象为 trait，并添加到泛型参数中。

## 使用举例

下面是 starry-next 中 `rt_sigsuspend` 系统调用的实现。其作用是，阻塞当前进程，屏蔽指定的信号集，直到有信号集之外的信号到达。

```rust
pub fn sys_rt_sigsuspend(
    tf: &mut TrapFrame,
    set: UserConstPtr<SignalSet>,
    sigsetsize: usize,
) -> LinuxResult<isize> {
    // 检查系统调用版本是否符合预期
    check_sigset_size(sigsetsize)?;

    let curr = current();
    let thr_data = curr.task_ext().thread_data();
    let mut set = *set.get_as_ref()?;

    // 不允许屏蔽 SIGKILL 与 SIGSTOP
    set.remove(Signo::SIGKILL);
    set.remove(Signo::SIGSTOP);

    // 替换原有的线程信号屏蔽集
    let old_blocked = thr_data
        .signal
        .with_blocked_mut(|blocked| mem::replace(blocked, set));

    // 默认返回值为 -EINTR
    tf.set_retval(-LinuxError::EINTR.code() as usize);

    // 等待信号到达
    loop {
        // check_signals 包装了 ThreadSignalManager::check_signals，检查当前线程是否有待处理的信号
        if check_signals(tf, Some(old_blocked)) {
            break;
        }
        // 如果没有待处理的信号，则阻塞当前线程，等待任意信号到达
        curr.task_ext().process_data().signal.wait_signal();
    }

    Ok(0)
}
```

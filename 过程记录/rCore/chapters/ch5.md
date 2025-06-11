# ch5

## 编程实验

简单

### spawn 一个新子进程

虽然不要将 spawn 生硬实现为 fork + exec, 但实际 spawn 的代码也差不多就是照着已有的 fork, exec 代码复制粘贴拼接。

```rust
// os/src/syscall/process.rs

/// YOUR JOB: Implement spawn.
/// HINT: fork + exec =/= spawn
pub fn sys_spawn(_path: *const u8) -> isize {
    trace!("kernel:pid[{}] sys_spawn", current_task().unwrap().pid.0);
    let token = current_user_token();
    let path = translated_str(token, _path);
    if let Some(data) = get_app_data_by_name(path.as_str()) {
        let current_task = current_task().unwrap();
        let new_task = current_task.spawn(data);
        let new_pid = new_task.pid.0;
        // add new task to scheduler
        add_task(new_task);
        new_pid as isize
    } else {
        -1
    }
}
```

仍然注意 sys_spawn 形参 _path 是**虚拟地址**。其次 spawn 是从父进程创建一个子进程，并非对调用进程的覆盖，无论成功与否，父进程都会收到相应返回值。

spawn 的完整实现：

```rust
// os/src/task/task.rs

impl TaskControlBlock {
    // ...

    pub fn spawn(self: &Arc<Self>, elf_data: &[u8]) -> Arc<Self> {
        // ---- access parent PCB exclusively
        let mut parent_inner = self.inner_exclusive_access();
        // memory_set with elf program headers/trampoline/trap context/user stack
        let (memory_set, user_sp, entry_point) = MemorySet::from_elf(elf_data);
        let trap_cx_ppn = memory_set
            .translate(VirtAddr::from(TRAP_CONTEXT_BASE).into())
            .unwrap()
            .ppn();
        // alloc a pid and a kernel stack in kernel space
        let pid_handle = pid_alloc();
        let kernel_stack = kstack_alloc();
        let kernel_stack_top = kernel_stack.get_top();
        let task_control_block = Arc::new(TaskControlBlock {
            pid: pid_handle,
            kernel_stack,
            inner: unsafe {
                UPSafeCell::new(TaskControlBlockInner {
                    trap_cx_ppn,
                    base_size: user_sp,
                    task_cx: TaskContext::goto_trap_return(kernel_stack_top),
                    task_status: TaskStatus::Ready,
                    memory_set,
                    parent: Some(Arc::downgrade(self)),
                    children: Vec::new(),
                    exit_code: 0,
                    heap_bottom: parent_inner.heap_bottom,
                    program_brk: parent_inner.program_brk,
                    stride: 0,
                    priority: 16,
                })
            },
        });
        // initialize trap_cx
        let inner = task_control_block.inner_exclusive_access();
        let trap_cx = inner.get_trap_cx();
        *trap_cx = TrapContext::app_init_context(
            entry_point,
            user_sp,
            KERNEL_SPACE.exclusive_access().token(),
            kernel_stack_top,
            trap_handler as usize,
        );
        drop(inner);
        // add child
        parent_inner.children.push(task_control_block.clone());
        // return
        task_control_block
    }

    // ...
}
```

需要注意的是 spawn **不需要**像 fork 那样去复制父进程地址空间，因此 memory_set 应像 exec 中那样调用 MemorySet::from_elf 而不是像 fork 中那样调用 MemorySet::from_existed_user。

```rust
// os/src/task/task.rs

impl TaskControlBlock {
    // ...

    pub fn exec(&self, elf_data: &[u8]) {
        // ...

        // **** access current TCB exclusively
        let mut inner = self.inner_exclusive_access();
        // substitute memory_set
        inner.memory_set = memory_set;
        // update trap_cx ppn
        inner.trap_cx_ppn = trap_cx_ppn;
        // initialize base_size
        inner.base_size = user_sp;
        // initialize trap_cx
        let trap_cx = inner.get_trap_cx();
        *trap_cx = TrapContext::app_init_context(
            entry_point,
            user_sp,
            KERNEL_SPACE.exclusive_access().token(),
            self.kernel_stack.get_top(),
            trap_handler as usize,
        );
        // **** release inner automatically
    }

    // ...
}
```

以及 spawn 对新进程中 inner 相关字段的设置（包括 memory_set, trap_cx_ppn, base_size, trap_cx）要按照 exec 中的来而不是按照 fork 中的来。这几个字段的功能分别是：

- ``memory_set`` 表示应用地址空间
    - ``fork`` 应该**复制**一份父进程的地址空间过来
    - ``exec`` 根据新进程 elf 文件内容设置
    - ``spawn`` 同 ``exec``
- ``trap_cx_ppn`` 指出了应用地址空间中的 Trap 上下文被放在的物理页帧的物理页号
    - ``fork``, ``exec``, ``spawn`` 均使用 memory_set 获取虚拟地址 TRAP_CONTEXT_BASE 对应的 pte 并提取 ppn 
- ``base_size`` 含义是：应用数据仅有可能出现在应用地址空间低于 base_size 字节的区域中。借助它我们可以清楚的知道应用有多少数据驻留在内存中
    - ``fork`` 同父进程 base_size
    - ``exec`` 可以查看 os/src/mm/memory_set.rs 下的 MemorySet 中的 from_elf 理解 base_size 的含义
        - elf 文件中有多个 program header，地址自低向高，每个 program header 描述了一个段，对应一个 MapArea；这之后地址再往高有一个 guard 页隔着；再往高就是用户栈空间了，从 user_stack_bottom 开始到 user_stack_top 大小为 USER_STACK_SIZE。于是 base_size 即为这个栈的 user_stack_top，即该应用的所有数据仅有可能出现在应用地址空间低于 user_stack_top 字节的区域中。
    - ``spawn`` 同 ``exec``
- ``trap_cx`` 地址处保存任务上下文，用于任务切换。其实就是 trap_cx_ppn.get_mut()
    - ``fork`` 的 ``trap_cx`` 地址处的内容还是同父进程，内容是在复制 memory_set 时从父进程复制过来的
        - from_existed_user 会将父 memory_set 中 areas 向量中的所有 MapArea 复制到子 memory_set
        - 而在一个进程被创建时 trap context 页便会被加入到其 areas 向量中
        ```rust
        // os/src/mm/memory_set.rs

        impl MemorySet {
            // ...

            /// Include sections in elf and trampoline and TrapContext and user stack,
            /// also returns user_sp_base and entry point.
            pub fn from_elf(elf_data: &[u8]) -> (Self, usize, usize) {
                let mut memory_set = Self::new_bare();
                // map trampoline
                memory_set.map_trampoline();
                
                // ...
                
                // map TrapContext
                memory_set.push(
                    MapArea::new(
                        TRAP_CONTEXT_BASE.into(),
                        TRAMPOLINE.into(),
                        MapType::Framed,
                        MapPermission::R | MapPermission::W,
                    ),
                    None,
                );
                
                // ...
            }

            // ...
        }
        ```
        - 所以在复制父进程的 memory_set 时，父进程 trap context 的内容会一并被复制到子进程中去
    - ``exec`` 的 ``trap_cx`` 处的内容应该用新应用的初始 trap_cx，即 TrapContext::app_init_context 来覆盖
    - ``spawn`` 同 ``exec``
    - 注意：[rCore 实现](https://rcore-os.cn/rCore-Tutorial-Book-v3/chapter4/5kernel-app-spaces.html)将 trap context 放在**用户地址空间**的[次高 4KiB 处](https://rcore-os.cn/rCore-Tutorial-Book-v3/_images/app-as-full.png)（从其 TRAP_CONTEXT_BASE 的**虚拟**地址值也可看出），只是为了不让用户访问该页因此未设置该页的 U 位

### 进程优先级

根据实验指导书来就可以。我的做法是：

- 在 tcb inner 里增加两个属性 

```rust
// os/src/task/task.rs

pub struct TaskControlBlockInner {
    // ...

    pub stride: usize
    pub priority: isize
}
```

- 在 sys_set_priority 中设置 priority
- 在 scheduler 切换进程时，对切换前的进程计算 pass 值并更新其 stride

```rust
// os/src/task/mod.rs

/// Suspend the current 'Running' task and run the next task in task list.
pub fn suspend_current_and_run_next() {
    // ...

    // increase stride
    let pass = BIG_STRIDE / (task_inner.priority as usize);
    task_inner.stride += pass;

    // ...
}
```
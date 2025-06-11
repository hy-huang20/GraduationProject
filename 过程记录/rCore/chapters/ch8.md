# ch8

## 编程实验

难度有些大。主要是要理解在银行家算法中如何设置 available, allocation, 尤其是 need 的初值。银行家算法具体实现完全照着实验指导书做即可。

在 pcb inner 中增加 deadlock_detect_enabled 属性，在 sys_mutex_lock 和 sys_semaphore_down 中添加死锁检测的逻辑，以下以 mutex 为例：

```rust
// os/src/syscall/sync.rs

pub fn sys_mutex_lock(mutex_id: usize) -> isize {
    // ...
    if process_inner.deadlock_detect_enabled {
        // ...
        if mutex_deadlock_exist(tid, mutex_id) {
            return -0xdead;
        }
        // ...
    }
    // ...
}

pub fn sys_enable_deadlock_detect(_enabled: usize) -> isize {
    let process = current_process();
    let mut process_inner = process.inner_exclusive_access();
    match _enabled {
        1 => process_inner.deadlock_detect_enabled = true,
        0 => process_inner.deadlock_detect_enabled = false,
        _ => return -1, // 参数不合法
    };
    return 0;
}
```

available, allocation, need 的初值在 mutex_deadlock_exist 中计算。

为了对 mutex 和 semaphore 分别进行死锁检测，需要在这些结构中增加一些属性以方便实现：

```rust
// os/src/sync/mutex.rs

pub struct MutexBlockingInner {
    // ..
    /// 增加一个属性记录该锁目前正被哪个线程占有
    pub lock_acquired_task: Option<Arc<TaskControlBlock>>,
    // ..
}
```

```rust
// os/src/sync/semaphore.rs

pub struct SemaphoreInner {
    // ..
    /// 增加一个分配列表以记录当前信号量已被哪些线程占有，可重复
    pub alloc_queue: Vec<Arc<TaskControlBlock>>,
    // ..
}
```

available, allocation, need 初值按照如下的方式确定：

```rust
fn mutex_deadlock_exist(tid: usize, mid: usize) -> bool {
    let process = current_process();
    let process_inner = process.inner_exclusive_access();
    let num_threads = process_inner.tasks.len();
    let num_resources = process_inner.mutex_list.len();
    if num_threads == 0 || num_resources == 0 {
        return false;
    }
    let mut worker = vec![0u32; num_resources]; // worker = available
    let mut allocation = vec![vec![0u32; num_resources]; num_threads];
    let mut need = vec![vec![0u32; num_resources]; num_threads]; // need = max - allocation
    for mutex_id in 0..num_resources {
        if let Some(mutex_any) = &process_inner.mutex_list[mutex_id] {
            let mutex_any = mutex_any.clone();
            if let Some(mutex_blocking) = mutex_any.as_any().downcast_ref::<MutexBlocking>() {
                let inner = mutex_blocking.inner.exclusive_access();
                if inner.locked {
                    let thread_id = inner.lock_acquired_task.as_ref().clone().unwrap().inner_exclusive_access().res.as_ref().unwrap().tid;
                    allocation[thread_id][mutex_id] = 1;
                } else {
                    worker[mutex_id] = 1;
                } 
                for tcb in inner.wait_queue.iter() {
                    let thread_id = tcb.inner_exclusive_access().res.as_ref().unwrap().tid;
                    need[thread_id][mutex_id] += 1;
                }
            }
        }
    }
    need[tid][mid] += 1;
    return banker_algorithm(num_resources, num_threads, &mut worker, &allocation, &need);
}
```

need 计算初值时应考虑目前在所有锁上等待的所有线程，以及该次调用 sys_mutex_lock 对应的线程。
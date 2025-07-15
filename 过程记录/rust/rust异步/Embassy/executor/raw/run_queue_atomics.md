# 从 RunQueue 源码出发

## RunQueue

```rust
// src/raw/run_queue_atomics.rs

pub(crate) struct RunQueueItem {
    next: SyncUnsafeCell<Option<TaskRef>>,
}

pub(crate) struct RunQueue {
    head: AtomicPtr<TaskHeader>,
}
```

``RunQueue`` 是一个无锁（lock-free，依赖硬件提供的原子操作而非传统的锁来保证线程安全）的、用单链表实现的任务队列。``head`` 是链表的头节点。``TaskHeader`` 中包含了 ``run_queue_item`` 属性，是上面列出来的 ``RunQueueItem`` 类型，其中包含的 ``next`` 属性包裹着一个 ``TaskRef`` 类型：

```rust
// src/raw/mod.rs

pub(crate) struct TaskHeader {
    // ...
    
    pub(crate) run_queue_item: RunQueueItem,
    
    // ...
}

#[derive(Clone, Copy, PartialEq)]
pub struct TaskRef {
    ptr: NonNull<TaskHeader>,
}
```

## enqueue()

```rust
// src/raw/run_queue_atomics.rs

impl RunQueue {
    #[inline(always)]
    pub(crate) unsafe fn enqueue(&self, task: TaskRef, _: super::state::Token) -> bool {
        let mut was_empty = false;

        self.head // 闭包的 prev 参数就是 self.head
            .fetch_update( // fetch_update 是原子操作
                Ordering::SeqCst, Ordering::SeqCst, |prev| {
                // 检查原队列是否为空
                was_empty = prev.is_null();
                unsafe {
                    // safety: the pointer is either null or valid
                    let prev = // Option<TaskRef>
                        NonNull::new(prev) // NotNull<TaskHeader>
                        .map(|ptr| // NotNull<TaskHeader>
                            TaskRef::from_ptr(ptr.as_ptr()));
                    // 将新 task 的 next 指向原 self.head
                    // safety: there are no concurrent accesses to `next`
                    task
                        .header() // &'static TaskHeader
                        .run_queue_item // RunQueueItem
                        .next // SyncUnsafeCell<Option<TaskRef>>
                        .set(prev);
                }
                // 返回。更新 self.head 为新 task 的指针（存在隐式类型转换）
                Some(task.as_ptr() as *mut _)
            })
            .ok();

        was_empty
    }
}
```

所以 ``enqueue()`` 的时候其实是将新 task 插入到链表头部。

## dequeue_all()

```rust
// src/raw/run_queue_atomics.rs

impl RunQueue {
    pub(crate) fn dequeue_all(&self, on_task: impl Fn(TaskRef)) {
        // Atomically empty the queue.
        let ptr = // *mut TaskHeader 
            self.head.swap(ptr::null_mut(), Ordering::AcqRel);

        // safety: the pointer is either null or valid
        let mut next = // Option<TaskRef> 
            unsafe { NonNull::new(ptr).map(|ptr| TaskRef::from_ptr(ptr.as_ptr())) };

        // Iterate the linked list of tasks that were previously in the queue.
        while let Some(task: TaskRef) = next {
            // If the task re-enqueues itself, the `next` pointer will get overwritten.
            // Therefore, first read the next pointer, and only then process the task.
            // safety: there are no concurrent accesses to `next`
            next = unsafe { task.header().run_queue_item.next.get() };

            task.header().state.run_dequeue();
            on_task(task);
        }
    }
}
```

``dequeue_all()`` 首先使用 ``AtomicPtr::swap()``，通过将链表的头节点 ``self.head`` 置为空从而实现清空链表，同时该函数会返回 ``self.head`` 旧值并由 ``ptr`` 获得，用于后续操作旧链表。之后的 while 循环从首部开始遍历旧链表的每一个 ``task`` 节点（``TaskRef`` 类型）。

### task.header().state.run_dequeue()

```rust
// src/raw/state_atomics_arm.rs

#[repr(C, align(4))]
pub(crate) struct State {
    /// Task is spawned (has a future)
    spawned: AtomicBool,
    /// Task is in the executor run queue
    run_queued: AtomicBool,
    pad: AtomicBool,
    pad2: AtomicBool,
}

impl State {
    // ...

    /// Unmark the task as run-queued. Return whether the task is spawned.
    #[inline(always)]
    pub fn run_dequeue(&self) {
        compiler_fence(Ordering::Release);

        self.run_queued.store(false, Ordering::Relaxed);
    }
}
```

清除 run queue 中所有 task 后，对于这些已经不在 run queue 中的所有 task，其 ``State`` 的 ``run_queued`` 属性都需要设置为 false。

### on_task(task)

``on_task`` 是一个接收 ``TaskRef`` 参数且无返回值的闭包，作为一个回调函数，由调用方传入，用于处理从队列中取出的每个任务。

这里关于 ``on_task`` 的介绍比较简略，因为目前并不清楚谁是 ``dequeue_all()`` 的调用方，因此不知道调用方会传入什么样的闭包。所以由谁来调用 ``RunQueue`` 的这些方法呢？

### 关于 dequeue_all() 的调用方

其实是在 [src/raw/mod.rs](./mod.md#调用-runqueuedequeue_all) 中：

```rust
// src/raw/mod.rs

pub(crate) struct TaskHeader {
    pub(crate) state: State,
    pub(crate) run_queue_item: RunQueueItem,
    pub(crate) executor: AtomicPtr<SyncExecutor>,
    poll_fn: SyncUnsafeCell<Option<unsafe fn(TaskRef)>>,

    /// Integrated timer queue storage. This field should not be accessed outside of the timer queue.
    pub(crate) timer_queue_item: timer_queue::TimerQueueItem,
}

pub(crate) struct SyncExecutor {
    run_queue: RunQueue,
    pender: Pender,
}

impl SyncExecutor {
    // ...

    pub(crate) unsafe fn poll(&'static self) {
        self.run_queue.dequeue_all(|p: TaskRef| {
            let task = // &'static TaskHeader
                p.header();

            // Run the task
            task
                .poll_fn // SyncUnsafeCell<Option<unsafe fn(TaskRef)>>
                .get() // Option<unsafe fn(TaskRef)>
                .unwrap_unchecked() // unsafe fn(TaskRef)
                (p);
        });
    }
}
```

>#### Option::unwrap_unchecked()
>
>与 ``Option::unwrap()`` 相比，``Option::unwrap_unchecked()`` 是一个 ``unsafe`` 的函数。``unwrap()`` 会在**运行时**检查 ``Option`` 是否为 ``None``，如果是则会触发 ``panic``。而 ``unwrap_unchecked()`` 则完全不进行检查，因此如果运行时 ``Option`` 实际为 ``None`` 会触发 Undefined Behavior。但由于后者免去了运行时检查，因此与前者相比会带来性能上的提升。

``poll_fn`` 中存放了一个 ``unsafe fn(TaskRef)``。这是一个[函数指针](../../../../rust-closure.md#函数指针)类型，指向一个：

- 接受 ``TaskRef`` 作为参数的函数，并且无返回值
- 被标记为 ``unsafe``（调用时需要 ``unsafe`` 块）
    - 注意这里的 ``SyncExecutor::poll()`` 自身就是 ``unsafe`` 的，所以在其中就**可以**不使用 ``unsafe`` 块包裹了

综上可知，``on_task()`` 其实就是 src/raw/mod.rs 文件 ``SyncExecutor::poll()`` 中传给 ``RunQueue::dequeue_all()`` 的闭包。然后，这个闭包中调用了 ``poll_fn`` 中存放的函数指针。所以这里的 ``TaskHeader::poll_fn`` 是什么呢？下面进行介绍，如需跳过介绍请[点此继续](#继续)

#### 关于 TaskHeader::poll_fn 属性

在 src/raw/mod.rs 中搜索 ``poll_fn`` 字段:

```rust
// src/raw/mod.rs

pub(crate) struct TaskHeader {
    // ...

    poll_fn: SyncUnsafeCell<Option<unsafe fn(TaskRef)>>,

    // ...
}

#[repr(C)]
pub struct TaskStorage<F: Future + 'static> {
    raw: TaskHeader,
    future: UninitCell<F>, // Valid if STATE_SPAWNED
}

impl<F: Future + 'static> TaskStorage<F> {
    
    /// Create a new TaskStorage, in not-spawned state.
    pub const fn new() -> Self {
        Self {
            raw: TaskHeader {
                // ...

                // Note: this is lazily initialized so that a static `TaskStorage` will go in `.bss`
                poll_fn: SyncUnsafeCell::new(None),

                // ...
            },
            future: UninitCell::uninit(),
        }
    }

    // ...

    unsafe fn poll(p: TaskRef) {
        let this = // &TaskStorage<F>
            &*p.as_ptr().cast::<TaskStorage<F>>();

        let future = // Pin<&mut F> 
            Pin::new_unchecked(this.future.as_mut());
        let waker = // Waker
            waker::from_task(p);
        let mut cx = // Context<'_>
            Context::from_waker(&waker);
        match future.poll(&mut cx) {
            Poll::Ready(_) => {
                // As the future has finished and this function will not be called
                // again, we can safely drop the future here.
                this.future.drop_in_place();

                // We replace the poll_fn with a despawn function, so that the task is cleaned up
                // when the executor polls it next.
                this.raw.poll_fn.set(Some(poll_exited));

                // Make sure we despawn last, so that other threads can only spawn the task
                // after we're done with it.
                this.raw.state.despawn();
            }
            Poll::Pending => {}
        }

        // the compiler is emitting a virtual call for waker drop, but we know
        // it's a noop for our waker.
        mem::forget(waker);
    }

    // ...
}

impl<F: Future + 'static> AvailableTask<F> {
    fn initialize_impl<S>(self, future: impl FnOnce() -> F) -> SpawnToken<S> {
        unsafe {
            self.task.raw.poll_fn.set(Some(TaskStorage::<F>::poll));
            self.task.future.write_in_place(future);

            let task = TaskRef::new(self.task);

            SpawnToken::new(task)
        }
    }

    pub fn initialize(self, future: impl FnOnce() -> F) -> SpawnToken<F> {
        self.initialize_impl::<F>(future)
    }

    #[doc(hidden)]
    pub unsafe fn __initialize_async_fn<FutFn>(self, future: impl FnOnce() -> F) -> SpawnToken<FutFn> {
        self.initialize_impl::<FutFn>(future)
    }
}
```

>##### 关于 SyncUnsafeCell 和 UninitCell
>
>这两个 cell 类型均定义于文件 src/raw/utils.rs 中，关于 utils 的记录可参考[此处](./utils.md)

综上可知 ``poll_fn`` 内部可能的值有：``None``, ``Some(poll_exited)``, ``Some(TaskStorage::<F>::poll)``。

``TaskStorage::<F>::poll`` 的代码已经在上面展示过了。``poll_exited`` 不做任何事：

```rust
// src/raw/mod.rs

unsafe fn poll_exited(_p: TaskRef) {
    // Nothing to do, the task is already !SPAWNED and dequeued.
}
```

发现 ``poll_fn`` 只在 src/raw/mod.rs 中 ``SyncExecutor::poll()`` 中被调用，``poll_fn`` 只在 src/raw/mod.rs 中 ``AvailableTask::initialize_impl()`` 中被设置为实际有意义的 ``TaskStorage::<F>::poll()``。``AvailableTask::initialize()`` 也调用了 ``AvailableTask::initialize_impl()``。因此接下来查看 ``AvailableTask`` 外部的哪些函数调用了上述两个 initialize 方法：

```rust
// src/raw/mod.rs

impl<F: Future + 'static, const N: usize> TaskPool<F, N> {
    fn spawn_impl<T>(&'static self, future: impl FnOnce() -> F) -> SpawnToken<T> {
        match self.pool.iter().find_map(AvailableTask::claim) {
            Some(task) => task.initialize_impl::<T>(future),
            None => SpawnToken::new_failed(),
        }
    }
}

impl<F: Future + 'static> TaskStorage<F> {
    /// Try to spawn the task.
    ///
    /// The `future` closure constructs the future. It's only called if spawning is
    /// actually possible. It is a closure instead of a simple `future: F` param to ensure
    /// the future is constructed in-place, avoiding a temporary copy in the stack thanks to
    /// NRVO optimizations.
    ///
    /// This function will fail if the task is already spawned and has not finished running.
    /// In this case, the error is delayed: a "poisoned" SpawnToken is returned, which will
    /// cause [`Spawner::spawn()`](super::Spawner::spawn) to return the error.
    ///
    /// Once the task has finished running, you may spawn it again. It is allowed to spawn it
    /// on a different executor.
    pub fn spawn(&'static self, future: impl FnOnce() -> F) -> SpawnToken<impl Sized> {
        let task = AvailableTask::claim(self);
        match task {
            Some(task) => task.initialize(future),
            None => SpawnToken::new_failed(),
        }
    }
}
```

``self.pool.iter().find_map(AvailableTask::claim)`` 是将 ``AvailableTask::claim()`` 运用到 ``TaskPool`` 中 pool（类型 ``[TaskStorage::NEW; N]``）的每个 ``TaskStorage``，并返回第一个非空的结果。去 src/raw/mod.rs 中查看 ``AvailableTask::claim()`` 的源码，发现当且仅当 task 对应的 ``State`` 为 mod.rs 中提到的 Not spawned 状态时，``claim()`` 不会返回空。而只有 ``claim()`` 不返回空，在上面展示的 ``TaskPool::spawn_impl`` 和 ``TaskStorage::spawn`` 函数中才能各自通过 match 获取到 ``claim()`` 返回值并最终如前所述对 ``TaskHeader::poll_fn`` 进行设置。也就是说，**将 ``poll_fn`` 设置为 ``TaskStorage::<F>::poll`` 仅会发生在处于 Not spawned 状态的 task 上**（应该是发生在状态转换 1 中）！调用 ``poll_fn`` 即是调用 ``TaskStorage::<F>::poll``（在这个调用中，如果 ``Poll::Ready``，则将 ``poll_fn`` 设置为 ``poll_exited``）。

#### 继续

弄清楚 ``poll_fn`` 后，继续研究 ``SyncExecutor::poll`` 的调用方。

TODO
# embassy_time_queue_utils::Queue 源码

embassy_time_queue_utils 的项目地址位于[此处](https://github.com/embassy-rs/embassy/tree/b528ed06e3025e0803e8fd6dc53ac968df9f49bc/embassy-time-queue-utils)。

``src/`` 下有两种 queue：``queue_generic.rs``, ``queue_integrated.rs``。由于在 ``lib.rs`` 中 cfg 只 pub 了 ``queue_integrated`` mod，因此这里只研究后者的源码。

## queue_integrated

代码不很长，因此都贴在这里了：

```rust
// src/queue_integrated.rs

//! Timer queue operations.
use core::cell::Cell;
use core::cmp::min;
use core::task::Waker;

use embassy_executor::raw::TaskRef;

/// A timer queue, with items integrated into tasks.
pub struct Queue {
    head: Cell<Option<TaskRef>>,
}

impl Queue {
    /// Creates a new timer queue.
    pub const fn new() -> Self {
        Self { head: Cell::new(None) }
    }

    /// Schedules a task to run at a specific time.
    ///
    /// If this function returns `true`, the called should find the next expiration time and set
    /// a new alarm for that time.
    pub fn schedule_wake(&mut self, at: u64, waker: &Waker) -> bool {
        let task = embassy_executor::raw::task_from_waker(waker);
        let item = task.timer_queue_item();
        if item.next.get().is_none() {
            // If not in the queue, add it and update.
            let prev = self.head.replace(Some(task));
            item.next.set(if prev.is_none() {
                Some(unsafe { TaskRef::dangling() })
            } else {
                prev
            });
            item.expires_at.set(at);
            true
        } else if at <= item.expires_at.get() {
            // If expiration is sooner than previously set, update.
            item.expires_at.set(at);
            true
        } else {
            // Task does not need to be updated.
            false
        }
    }

    /// Dequeues expired timers and returns the next alarm time.
    ///
    /// The provided callback will be called for each expired task. Tasks that never expire
    /// will be removed, but the callback will not be called.
    pub fn next_expiration(&mut self, now: u64) -> u64 {
        let mut next_expiration = u64::MAX;

        self.retain(|p| {
            let item = p.timer_queue_item();
            let expires = item.expires_at.get();

            if expires <= now {
                // Timer expired, process task.
                embassy_executor::raw::wake_task(p);
                false
            } else {
                // Timer didn't yet expire, or never expires.
                next_expiration = min(next_expiration, expires);
                expires != u64::MAX
            }
        });

        next_expiration
    }

    fn retain(&self, mut f: impl FnMut(TaskRef) -> bool) {
        let mut prev = &self.head;
        while let Some(p) = prev.get() {
            if unsafe { p == TaskRef::dangling() } {
                // prev was the last item, stop
                break;
            }
            let item = p.timer_queue_item();
            if f(p) {
                // Skip to next
                prev = &item.next;
            } else {
                // Remove it
                prev.set(item.next.get());
                item.next.set(None);
            }
        }
    }
}
```

### embassy_executor

[参考](./executor/readme.md)

### schedule_wake()

embassy-executor 有一个 run queue 和一个 timer queue，前者管理**立即可运行**的任务，后者存放**在特定时间点需要被唤醒**的任务。在 timer queue 中被唤醒的任务会被移入 run queue 中。

TODO：弄清楚 timer queue 任务转入 run queue 的调用时序

这里讨论 timer queue。timer queue 和 run queue 一样也是一个单链表。timer queue 中每个节点的 next 指向链表中的下一个节点，最后一个节点的值为 ``Some(dangling_pointer)``：

```rust
// src/raw/timer_queue.rs

/// An item in the timer queue.
pub struct TimerQueueItem {
    /// The next item in the queue.
    ///
    /// If this field contains `Some`, the item is in the queue. The last item in the queue has a
    /// value of `Some(dangling_pointer)`
    pub next: Cell<Option<TaskRef>>,

    /// The time at which this item expires.
    pub expires_at: Cell<u64>,
}
```

总是只要是 timer queue 中有意义的节点，其 next 都不会为空。这也解释了为什么 ``schedule_wake()`` 使用 ``if item.next.get().is_none()`` 来判断节点是否在 timer queue 中。

``schedule_wake()`` 的功能如注释所述，设定一个 task 在指定的时间 ``at`` 运行。如果该 task 原本不在 timer queue 中，则将其插入 timer queue 链表首部并设置其将要运行的时间为 ``at``；如果该 task 原本在 timer queue 中：

- 新设定的运行时间 ``at`` 早于原先设定的时间
    - 则更新其将要运行的时间
- 新设定的运行时间 ``at`` 晚于原先设定的时间
    - 无需更新，无需做任何事

### next_expiration()

由于 ``next_expiraton()`` 中调用了 ``Queue`` 同类型中的私有方法 ``retain()``，因此这里先讨论 ``retain()`` 的功能。

``retain()`` 对 timer queue 中的每一个节点调用闭包，如果闭包返回 true 则继续下一个节点，否则将当前节点从链表中移除后继续下一个节点。

TODO：没太理解这里的 ``// Remove it`` 的逻辑，总感觉有点问题

所以 ``next_expiration()`` 的功能就是唤醒 timer queue 中所有已经过期的 task 并移除，对于其它所有未过期的节点，记录**最近**需要唤醒的时间并返回。 
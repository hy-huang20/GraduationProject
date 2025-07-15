# src/raw/mod.rs 源码

TODO: ai 生成，继续完善 

```mermaid
graph TD
    %% 核心类型
    A[Executor] -->|包含| B[SyncExecutor]
    B -->|包含| C[RunQueue]
    B -->|包含| D[Pender]
    C -->|管理| E[TaskHeader]
    
    %% 任务存储结构
    F[TaskStorage<F>] -->|包含| E
    F -->|包含| G[UninitCell<F>]
    
    %% 任务池
    H[TaskPool<F,N>] -->|包含| I["[TaskStorage<F>; N]"]
    
    %% 引用关系
    J[TaskRef] -->|指向| F
    K[AvailableTask<F>] -->|包装| F
    
    %% 执行流程
    D -->|触发| A
    E -->|通过| L[State]
    L -->|状态控制| C
    E -->|通过| M[poll_fn]
    M -->|执行| F
    
    %% 队列关系
    C -->|入队/出队| E
    N[Waker] -->|传入| P["task_from_waker()"]
    P --> |转换| J
    J -->|唤醒| O["wake_task()"]
    O -->|通知| A
```

参考 src/raw/mod.rs 中的注释：

```rust
/// Raw task header for use in task pointers.
///
/// A task can be in one of the following states:
///
/// - Not spawned: the task is ready to spawn.
/// - `SPAWNED`: the task is currently spawned and may be running.
/// - `RUN_ENQUEUED`: the task is enqueued to be polled. Note that the task may be `!SPAWNED`.
///    In this case, the `RUN_ENQUEUED` state will be cleared when the task is next polled, without
///    polling the task's future.
///
/// A task's complete life cycle is as follows:
///
/// ```text
/// ┌────────────┐   ┌────────────────────────┐
/// │Not spawned │◄─5┤Not spawned|Run enqueued│
/// │            ├6─►│                        │
/// └─────┬──────┘   └──────▲─────────────────┘
///       1                 │
///       │    ┌────────────┘
///       │    4
/// ┌─────▼────┴─────────┐
/// │Spawned|Run enqueued│
/// │                    │
/// └─────┬▲─────────────┘
///       2│
///       │3
/// ┌─────▼┴─────┐
/// │  Spawned   │
/// │            │
/// └────────────┘
/// ```
///
/// Transitions:
/// - 1: Task is spawned - `AvailableTask::claim -> Executor::spawn`
/// - 2: During poll - `RunQueue::dequeue_all -> State::run_dequeue`
/// - 3: Task wakes itself, waker wakes task, or task exits - `Waker::wake -> wake_task -> State::run_enqueue`
/// - 4: A run-queued task exits - `TaskStorage::poll -> Poll::Ready`
/// - 5: Task is dequeued. The task's future is not polled, because exiting the task replaces its `poll_fn`.
/// - 6: A task is waken when it is not spawned - `wake_task -> State::run_enqueue`
```

一个 task 的 ``State`` 属性存放在 ``TaskHeader`` 类型中：

```rust
// src/raw/mod.rs

pub(crate) struct TaskHeader {
    pub(crate) state: State,

    // ...
}
```

关于 A task's complete life cycle 下面那张状态转移图中的状态，可以对照 State 类型的属性：

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
```

|task state|State::spawned|State::run_queued|
|---|---|---|
|``Not spawned``|false|false|
|``Spawned\|Run enqueued``|true|true|
|``Spawned``|true|false|
|``Not spawned\|Run enqueued``|false|true|

以下按照注释中的划分来研究 mod.rs 源码。

## 1: Task is spawned

### AvailableTask::claim()

```rust
impl<F: Future + 'static> AvailableTask<F> {
    /// Try to claim a [`TaskStorage`].
    ///
    /// This function returns `None` if a task has already been spawned and has not finished running.
    pub fn claim(task: &'static TaskStorage<F>) -> Option<Self> {
        task.raw.state.spawn().then(|| Self { task })
    }
}
```

``task.raw.state.spawn()`` 的功能主要是将 ``task.raw.state`` 的 ``spawned`` ``run_queued`` 属性均设置为 true 以**完成状态转移**，并返回是否成功的 bool 值。而 then 如果前面返回的为 true 则执行闭包，即返回状态转移后的 task。

### Executor::spawn()

```mermaid
sequenceDiagram
    participant Executor
    participant SyncExecutor
    participant RunQueue
    participant Pender
    participant state_atomics_arm

    activate Executor
        Executor->>Executor: spawn()
        Executor->>SyncExecutor: SyncExecutor::spawn()
        activate SyncExecutor
            SyncExecutor->>state_atomics_arm: state::locked()
            activate state_atomics_arm
                note over state_atomics_arm: 创建一个 Token
                state_atomics_arm->>SyncExecutor: 将 Token 传给闭包并执行闭包
                activate SyncExecutor
                    activate SyncExecutor
                        SyncExecutor->>SyncExecutor: enqueue()
                        SyncExecutor->>RunQueue: RunQueue::enqueue()
                        activate RunQueue
                            note over RunQueue: 新 task 插入链表首部
                            RunQueue-->>SyncExecutor: 返回
                        deactivate RunQueue
                        SyncExecutor->>Pender: Pender::pend()
                        activate Pender
                            Pender->>Pender: extern "Rust" unsafe fn __pender()
                            Pender-->>SyncExecutor: 返回
                        deactivate Pender
                    deactivate SyncExecutor
                    SyncExecutor-->>state_atomics_arm: 闭包执行完成返回
                deactivate SyncExecutor
                state_atomics_arm-->>SyncExecutor: 返回闭包的返回值
            deactivate state_atomics_arm
            SyncExecutor-->>Executor: 返回
        deactivate SyncExecutor
    deactivate Executor
```

#### 关于 __pender()

```rust
extern "Rust" {
    fn __pender(context: *mut ());
}
```

TODO

#### 关于 Token

TODO

## 2: During poll

### RunQueue::dequeue_all() 和 State::run_dequeue()

详细文字记录可[参考](./run_queue_atomics.md)。下图左，从调用 RunQueue::dequeue_all() 开始：

```mermaid
sequenceDiagram
    participant RunQueue
    participant State
    participant SyncExecutor

    activate RunQueue
        note over RunQueue: 清空链表
        activate RunQueue
            note over RunQueue: 对于旧链表中的每个 task
            RunQueue->>RunQueue: while {...}
            RunQueue->>State: State::run_dequeue()
            activate State
                note over State: 设置 State::run_queued 为 false
                State-->>RunQueue: 返回
            deactivate State
            activate SyncExecutor
                SyncExecutor->>SyncExecutor: poll()
                RunQueue->>SyncExecutor: 执行 on_task 闭包
                activate SyncExecutor
                    SyncExecutor->>SyncExecutor: TaskHeader::poll_fn.get().unwrap_unchecked()()
                    SyncExecutor-->>RunQueue: 返回
                deactivate SyncExecutor
            deactivate SyncExecutor
        deactivate RunQueue
    deactivate RunQueue
```

#### 关于 poll_fn

目前暂时不知道 poll_fn 作为一个函数指针是干什么用的

TODO

## 3: Task wakes itself, waker wakes task, or task exits

```mermaid
sequenceDiagram
    participant waker
    participant mod
    participant State
    participant state_atomics_arm
    participant SyncExecutor

    activate waker
        waker->>waker: wake()
        waker->>mod: wake_task()
        activate mod
            mod->>State: State::run_enqueue()
            activate State
                note over State: 设置 State::run_queued 为 true
                State->>state_atomics_arm: locked()
                activate state_atomics_arm
                    note over state_atomics_arm: 创建一个 Token
                    state_atomics_arm-->>mod: 将 Token 传给闭包并执行闭包
                    activate mod
                        mod->>SyncExecutor: SyncExecutor::enqueue()
                        activate SyncExecutor
                            note over SyncExecutor: 这之后的逻辑和之前的时序图存在重复
                            SyncExecutor-->>mod: 返回
                        deactivate SyncExecutor
                        mod-->>state_atomics_arm: 闭包执行完成
                    deactivate mod
                    state_atomics_arm-->>State: 闭包执行完成
                deactivate state_atomics_arm
                State-->>mod: 返回
            deactivate State
            mod-->>waker: 返回
        deactivate mod
    deactivate waker
```

#### 关于 src/raw/waker.rs 中的 wake()

似乎在 src/raw/ 中没有看到这个函数被调用过...因此暂时没搞懂注释中提到的 "task wakes itself" 的原理

TODO

## 4: A run-queued task exits

```mermaid
sequenceDiagram
    participant TaskStorage
    participant UninitCell
    participant State

    activate TaskStorage
        TaskStorage->>TaskStorage: TaskStorage::poll()
        activate TaskStorage
            TaskStorage->>TaskStorage: Future::poll()
            TaskStorage->>UninitCell: UninitCell::drop_in_place()
            activate UninitCell
                note over UninitCell: 析构
                UninitCell-->>TaskStorage: 返回
            deactivate UninitCell
            note over TaskStorage: 将 poll_fn 设为 Some(poll_exited)
            TaskStorage->>State: State::despawn()
            activate State
                note over State: 将 State::spawned 设为 false
                State-->>TaskStorage: 返回
            deactivate State
        deactivate TaskStorage
    deactivate TaskStorage
```

## 5: Task is dequeued

The task's future is not polled, because exiting the task replaces its ``poll_fn``.

应该是调用了 ``State::run_dequeue()`` 完成状态的转换。而这个函数在 src/raw/ 中只在 ``RunQueue::dequeue_all()`` 中调用过，[同 2](#2-during-poll)。

## 6: A task is waken when it is not spawned

[同 3](#3-task-wakes-itself-waker-wakes-task-or-task-exits)。
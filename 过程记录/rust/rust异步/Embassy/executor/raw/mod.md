# src/raw/mod.rs 源码

## 图示

可能需要完善

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

## 函数调用栈

```mermaid
sequenceDiagram
    participant Main
    participant FuncA
    participant FuncB
    participant FuncC

    Main->>FuncA: 调用 FuncA
    activate FuncA  # 压栈（模拟）
    FuncA->>FuncB: 调用 FuncB
    activate FuncB
    FuncB->>FuncC: 调用 FuncC
    activate FuncC
    FuncC-->>FuncB: 返回
    deactivate FuncC # 弹栈（模拟）
    FuncB-->>FuncA: 返回
    deactivate FuncB
    FuncA-->>Main: 返回
    deactivate FuncA
```
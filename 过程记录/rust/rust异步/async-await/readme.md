# rust async/await feature

这里是对 Philipp Oppermann 的博客 Write an OS in Rust 中 [async-await 部分](https://os.phil-opp.com/async-await/)的学习记录，博客有繁体中文[翻译版本](https://os.phil-opp.com/zh-TW/async-await/)

## Multitasking 多任务

>在 ``Multitasking`` 一节所讲的 ``Preemptive Multitasking``（抢占式多任务）和 ``Cooperative Multitasking``（协作式多任务）概念在 [rCore 实验指导书](https://rcore-os.cn/rCore-Tutorial-Book-v3/chapter3/index.html)中已经讲过。

## Async/Await in Rust

``async/await`` 是**协作式多任务**的一种形式：

>协作式多任务通常用于语言级别，比如协程 ``coroutines`` 或 ``async/await`` 的形式。其思想是程序员或编译器在程序中插入 ``yield`` 操作，这样可以放弃对 CPU 的控制，让其它任务运行。

### Preemptive Multitasking

#### Saving State

>由于调用栈可能非常大，操作系统通常为每个任务设置一个单独的调用栈，而不是在每次任务切换时备份调用栈内容。这种带有其自己的栈的任务被称为``执行线程`` [thread of execution](https://en.wikipedia.org/wiki/Thread_(computing)) 或简称为``线程`` thread。通过为每个任务使用单独的栈，只需要在上下文切换时保存寄存器内容（包括程序计数器和栈指针）。

这里原文第一句话的意思是，操作系统也可以设置一个位置统一放置所有任务的调用栈，同一时间这个位置只会存在一个任务的调用栈。每当任务切换时，换出旧任务的调用栈，并换入新任务的调用栈。

### Cooperative Multitasking

#### Saving State

>既然任务自己定义了暂停点，它们不需要操作系统保存它们的状态。相反，它们可以在暂停自己之前保存它们需要的状态，这通常会带来更好的性能。例如，一个刚完成了复杂计算的任务可能只需要备份计算的最终结果，因为它不再需要中间结果。

>协作式多任务的语言级实现通常甚至能够在暂停之前备份调用栈的必要部分。例如，Rust 的 async/await 实现会在暂停之前备份所有仍然需要的本地变量到一个自动生成的结构体中（[见下文](#saving-state-2)）。通过在暂停之前备份调用栈的相关部分，所有任务都可以共享一个调用栈，这导致每个任务的内存消耗大大降低。这使得可以创建几乎任意数量的协作式任务而不会耗尽内存。

### Future

``Future`` trait 的声明：

```rust
pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context) -> Poll<Self::Output>;
}
```

上面的 ``type Output`` 是 rust 中的 [关联类型 Associated Type](../../rust-associated-type.md)。

上述 poll 方法返回一个 ``Poll`` 枚举，允许检查 ``Self::Output`` 是否已经可用：

```rust
pub enum Poll<T> {
    Ready(T),
    Pending,
}
```

当 ``Self::Output`` 已经可用时，它被包装在 ``Ready`` 中返回。否则返回 ``Pending`` 以告诉调用者该值尚不可用。

### Future 组合器

#### string_len 组合器

文中提到的 ``string_len`` 组合器：

```rust
struct StringLen<F> {
    inner_future: F,
}

impl<F> Future for StringLen<F> where F: Future<Output = String> {
    type Output = usize;

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<T> {
        match self.inner_future.poll(cx) {
            Poll::Ready(s) => Poll::Ready(s.len()),
            Poll::Pending => Poll::Pending,
        }
    }
}

fn string_len(string: impl Future<Output = String>)
    -> impl Future<Output = usize>
{
    StringLen {
        inner_future: string,
    }
}

// Usage
fn file_len() -> impl Future<Output = usize> {
    let file_content_future = async_read_file("foo.txt");
    string_len(file_content_future)
}
```

上述 ``string_len`` 组合器代码的作用是：

- 组合异步操作
    - 包装一个 ``Future<String>`` 并返回一个新的 ``Future<usize>``
    - 类似于 ``map`` 操作，但适用于异步上下文
        - 虽然返回的仍然是 ``Future``, 但 ``map`` 的闭包只能是同步的，``then`` 的闭包可以是异步的

        ```rust
        let mapped_future = some_async_fn().map(|x| { /* 闭包 */ });
        // mapped_future 仍然是 Future 类型
        // 闭包的执行发生在：
        // 1. 当 mapped_future 被 poll（如 mapped_future.await）
        // 2. 且 some_async_fn() 的 Future 返回 Poll::Ready 时
        // 上述两个条件一旦满足，闭包会立刻同步执行，不会被延迟执行
        // 而如果使用了 async 闭包的 then，则闭包操作也可能延迟执行
        ```

- 延迟计算
    - 它不会立即计算字符串长度，而是等待 ``inner_future`` 完成后再计算
- 符合 rust 的[零成本抽象](../../函数式编程与零成本抽象.md)
    - 在编写高抽象的代码时，不会引入额外运行时开销，也就是说和低抽象代码（如 ``for`` 循环，``if`` 语句，原始指针）一样高效

#### 缺点

```rust
fn example(min_len: usize) -> impl Future<Output = String> {
    async_read_file("foo.txt").then(move |content| {
        if content.len() < min_len {
            Either::Left(async_read_file("bar.txt").map(|s| content + &s))
        } else {
            Either::Right(future::ready(content))
        }
    })
}
```

- ``Either`` 包装器 [参考](./either-wrapper.md)
- ``move`` 关键字的[作用](../../rust-move.md)
- 上述代码中闭包参数 ``content`` 的类型已经是 ``String`` 而非 ``impl Future<Output = String>``
    - ``impl Future<Output = String>``
        - 含义是：实现了 ``Future`` trait 的某个具体类型，且该 
    - ``then`` 闭包参数是前一个 ``Future`` 的输出值（``Output`` 类型）
- ``content + &s`` 中 ``&`` 的[原因](../../rust-add-string-and-string-slice.md)

### The Async/Await Pattern

与直接使用 ``async`` 相对应的 ``Future`` 代码如下：

```rust
/*
async fn foo() -> u32 {
    0
}
*/
// async 对应形式
fn foo() -> impl Future<Output = u32> {
    future::ready(0)
}
```

也就是说 ``async fn`` 返回值的类型**总是**为 ``impl Future<Output = T>``，编译器会**自动包装**返回值为上述类型。而对于任何 ``impl Future<Output = T>`` 类型的值使用 ``.await``，得到的值类型也**总是**为 ``T``。

此外，在 rust 中，只要函数体内存在任何 ``async`` 函数调用（即使所有这些调用都加了 ``.await``），则该函数本身也必须声明为 ``async``。

#### Saving State



## Implementation 实现

TODO
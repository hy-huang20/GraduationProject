# rust async/await feature

这是对 Philipp Oppermann 的博客 Write an OS in Rust 中 [async-await 部分](https://os.phil-opp.com/async-await/)的学习，有繁体中文[翻译版本](https://os.phil-opp.com/zh-TW/async-await/)

>林晨学长当时参考的学习资料 [futures explained in 200 lines of rust](https://stevenbai.top/rust/futures_explained_in_200_lines_of_rust) 已经[被原作者停止公开](https://github.com/cfsamson/examples-futures/issues/2)。不过这里有一份学长当时的[笔记](https://github.com/BITcyman/Rust-os-learning/blob/main/rust-future/200lines.md)。

## Multitasking 多任务

在 Multitasking 一节所讲的 Preemptive Multitasking（抢占式多任务）和 Cooperative Multitasking（协作式多任务）相关概念 [rCore 实验指导书](https://rcore-os.cn/rCore-Tutorial-Book-v3/chapter3/index.html)中已经讲过。

## Async/Await in Rust

async/await 是一种``协作式多任务``的形式：

>协作式多任务通常用于语言级别，比如协程 coroutines 或 async/await 的形式。其思想是程序员或编译器在程序中插入 yield 操作，这样可以放弃对 CPU 的控制，让其它任务运行。

### Future

``Future`` trait 的声明：

```rust
pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context) -> Poll<Self::Output>;
}
```

上面的 type Output 是 rust 中的 [关联类型 Associated Type](../../rust-associated-type.md)。

上述 poll 方法返回一个 ``Poll`` 枚举，允许检查 Self::Output 是否已经可用：

```rust
pub enum Poll<T> {
    Ready(T),
    Pending,
}
```

当 Self::Output 已经可用时，它被包装在 Ready 中返回。否则返回 Pending 以告诉调用者该值尚不可用。

### Future 组合器

#### string_len 组合器

文中 ``string_len`` 组合器代码的作用是：

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
    - 在编写高抽象的代码时，不会引入额外运行时开销，也就是说和低抽象代码（如 for 循环，if语句，原始指针）一样高效

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

- Either 包装器 [参考](./either-wrapper.md)
- move 关键字的[作用](../../rust-move.md)
- 闭包中 ``content`` 的类型已经是 String 而非 impl Future<Output = String>
    - ``then`` 闭包参数是前一个 ``Future`` 的输出值（``Output`` 类型）

## Implementation 实现
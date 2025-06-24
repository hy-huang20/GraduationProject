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
- 符合 rust 的[零成本抽象](../../rust-zero-cost-abstractions.md)
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

- ``Either`` 包装器 [参考](./rust-either-wrapper.md)
- ``move`` 关键字的[作用](../../rust-move.md)
- 上述代码中闭包参数 ``content`` 的类型已经是 ``String`` 而非 ``impl Future<Output = String>``
    - ``impl Future<Output = String>``
        - 含义是：实现了 ``Future`` trait 的某个具体类型，且该 
    - ``then`` 闭包参数是前一个 ``Future`` 的输出值（``Output`` 类型）
- ``content + &s`` 中 ``&`` 的[原因](../../rust-deref.md)

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

rust 的 ``async/await`` 代码会被编译器编译成``状态机``代码。

>为了能够从上一个等待状态继续，状态机必须在内部保持当前的状态。此外，它必须保存所有它需要在下一个 ``poll`` 调用中继续执行的变量。这就是编译器真正发挥作用的地方：因为它知道哪些变量何时使用，它可以自动生成具有确切所需变量的结构体。

作为一个习惯了 cpp “要求什么得到什么” 的自由特性后的 cpp 开发者，初期接触时说实话不太习惯 rust 的这样的自动化魔法。~~初期感觉到自己在被 rust 编译器“管教”“不信任”~~

### Pinning

#### Self-Referential Structs 

``自引用结构体``：结构体的某个字段引用了同一结构体的另一个字段的数据

考虑文中的 ``WaitingOnWriteState`` 结构体。当移动结构体时，由于 array 的元素的类型实现了 ``Copy`` trait, 所以默认行为是进行拷贝，因此移动后的结构体的 array 的地址会发生改变，从而 element 失效。

再考虑一个文中没有的例子：

```rust
struct SelfRef {
    data: String,
    self_ref: *const String, // 指向自己的指针
}

let sr = SelfRef {
    data: "hello".to_string(),
    self_ref: std::ptr::null(),
};
sr.self_ref = &sr.data; // 建立自引用

// 移动会导致self_ref成为悬垂指针！
let moved_sr = sr; // 危险！
```

*可选：回忆一下[String 内存结构](../../rust-string-memory-structure.md)*

``self_ref`` 记录了 data *栈*上元数据的地址。而移动 ``sr`` 时，这一部分元数据会被逐字节拷贝，也就是说元数据会被复制到栈上的另外一块内存处，而 ``self_ref`` 作为一个``裸指针``（或原始指针，raw pointer）由于也实现了 ``Copy`` trait，因此也会被原样拷贝，因此移动后 ``self_ref`` 仍然记录的是旧的元数据的地址。

#### Possible Solutions

rust **禁止移动自引用结构体**，因为它的原则是提供``零成本抽象``，这意味着抽象不应该带来额外的运行时成本。

#### Heap Values

文中使用堆分配创建的自引用结构体：

```rust
fn main() {
    let mut heap_value = Box::new(SelfReferential {
        self_ptr: 0 as *const _,
    });
    let ptr = &*heap_value as *const SelfReferential;
    heap_value.self_ptr = ptr;
    println!("heap value at: {:p}", heap_value);
    println!("internal reference: {:p}", heap_value.self_ptr);
}

struct SelfReferential {
    self_ptr: *const Self,
}
```

- 注意：Rust 不用像 C++ 一样先声明后定义，因此可能会遇到 ``main`` 函数作为文件中第一个函数的情况
    - **无需前置声明**
    - **任意顺序**：函数之间调用不依赖定义顺序，编译器会全局分析代码
    - ``main`` **函数位置**：可以放在文件的任何位置（开头、中间或末尾）
- 使用解引用运算符访问 ``Box<T>``
    - ``Box<T>`` 实现了 [``Deref``](../../rust-deref.md) trait
    - 流程
        - 调用 ``.deref()`` 返回 ``T`` 的引用 ``&T``
        - 然后通过普通解引用运算符解引用 ``&T``，最终访问 ``T`` 的值
- 使用 ``&*`` 访问 ``Box<T>``（常见）
    - 如果只有解引用运算符 ``*``，则如果 ``T`` 没有实现 ``Copy`` trait，那么会发生所有权的转移，数据被移动，原数据失效
    - ``&*`` 不会消耗 ``Box<T>`` 的所有权，只是借用其内部数据
    - 其实，完全可以直接显式调用一次 ``.deref()``，这和 ``&*`` 是完全等效的

所以上述代码中，``ptr`` 指向的是位于堆上的 ``SelfReferential``，从而 ``heap_value.self_ptr`` 指向的是 ``ptr`` 指向的数据，从而实现了自引用结构体，这里的结构体 ``SelfReferential`` 中的指针 ``self_ptr`` 指向结构体 ``SelfReferential`` 自身。

但是，这里的 ``Box<T>`` 是在栈上的结构体，只是内部指针指向堆上数据，于是 ``heap_value`` 是栈上的那部分数据（类型为 ``Box<SelfReferential>``），其地址也应该是栈上那部分数据的地址才对啊？为什么直接输出 ``heap_value`` 会输出堆上数据的地址呢？这是因为 ``Box<T>`` 实现了 ``Pointer`` trait：

*[关于 Pointer trait](https://doc.rust-lang.org/std/fmt/trait.Pointer.html)*

```rust
pub trait Pointer {
    // Required method
    fn fmt(&self, f: &mut Formatter<'_>) -> Result<(), Error>;
}
```

``Pointer`` trait 定义了当使用 ``{:p}`` 格式化时如何输出 ``Box`` 的指针值。

*[关于 ``Box`` 实现 ``Pointer`` trait](https://doc.rust-lang.org/src/alloc/boxed.rs.html#1930)*

```rust
#[stable(feature = "rust1", since = "1.0.0")]
impl<T: ?Sized, A: Allocator> fmt::Pointer for Box<T, A> {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        // It's not possible to extract the inner Uniq directly from the Box,
        // instead we cast it to a *const which aliases the Unique
        let ptr: *const T = &**self;
        fmt::Pointer::fmt(&ptr, f)
    }
}
```

一般如果 ``self`` 是 ``Box<T>`` 时，可能会看到 ``&**self`` 这样的形式。

- 第一次解引用 ``*self``
    - ``Box<T>`` 实现了 ``Deref`` trait，所以 ``*self`` 相当于 ``*(self.deref())``
    - 这会得到 ``Box`` 内部拥有的 ``T`` 类型的值（所有权转移）
- 第二次解引用 ``**self``
    - 如果 ``T`` 本身也是可以解引用的类型（比如 ``T`` 是 ``String``，``Vec`` 等实现了 ``Deref`` 的类型）
    - 则会继续调用 ``T`` 的 ``deref()`` 方法
    - 对于像 ``i32`` 这样的简单类型，第二次解引用无实际效果
- 最后取引用 ``&**self``
    - 对最终解引用得到的值取引用

|原类型|第一次解引用|第二次解引用|最后取引用|
|---|---|---|---|
|``Box<String>``|``String``|``str``|``&str``|
|``Box<i32>``|``i32``|``i32``|``&i32``|

这样设计便可让 ``Box`` 作为智能指针，在这一点上表现得像普通指针。想想看，如果 ``heap_value`` 是个普通裸指针，并指向堆上的一块数据，那么直接输出 ``heap_value`` 的结果，是不是就是堆上数据的地址了？

#### ``Pin<Box<T>>`` and ``Unpin``

> ``Unpin`` trait 是一个 ``auto trait``，Rust 自动为所有类型实现了它，除了那些明确地选择了不实现的类型。通过使自引用结构体选择不实现 ``Unpin`` 的类型，对于它们来说，要从 ``Pin<Box<T>>`` 类型获得 ``&mut T`` 是没有（**安全的**）办法的。

那么如何选择不实现 ``Unpin`` trait 呢？只需要在结构体**定义**和**初始化**的时候在结构体内添上这么一句，便可使类型成为 ``!Unpin``：

```rust
_pin: PhantomPinned,
```

另外，注意上面“**安全的**”表述，也就是这种情况下仍然有办法获取到 ``&mut T``。比如可以在 ``unsafe`` 块中使用 [get_unchecked_mut](https://doc.rust-lang.org/nightly/core/pin/struct.Pin.html#method.get_unchecked_mut) 函数。但是使用这个函数的开发者需要保证：

- **不移动数据**：返回的 ``&mut T`` 不能用于移动 T（如文中提到的 ``mem::replace`` 或 ``mem::swap``）
- **不使 pin 失效**：不能通过这个引用做任何会导致 pin 保证被破坏的操作
- **生命周期有效**：返回的引用不能超过原始 ``Pin`` 的生命周期

#### Stack Pinning and ``Pin<&mut T>``

#### Pinning and Futures

### Executors and Wakers

#### Executors

>负责轮询所有 ``future`` 直到它们完成。

#### Wakers

由 ``Executor`` 创建的 ``Waker`` 类型被包装在 ``Context`` 类型中被传给每个 ``poll`` 方法，当 ``future`` 异步任务完成时，``Waker`` 会通知 ``Executor`` 调用该 ``future`` 的 ``poll``。

### Cooperative Multitasking?

不同于显式地使用 ``yield`` 操作，``futures`` 通过返回 ``Poll::Pending`` 来放弃 CPU 控制（或者在结束时返回 ``Poll::Ready``）

- 没有任何东西强制 ``futures`` 放弃 CPU 控制。如果它们想要，它们可以永远不从 ``poll`` 返回。例如，通过在循环中无休止地旋转
- 由于每个 ``future`` 都可以阻塞执行器中的其它 ``future`` 的执行，我们需要相信它们不是恶意的

## Implementation 实现


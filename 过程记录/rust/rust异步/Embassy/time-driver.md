# embassy_time 包提供的三种 time driver

embassy_time 项目地址位于[此处](https://github.com/embassy-rs/embassy/tree/b528ed06e3025e0803e8fd6dc53ac968df9f49bc/embassy-time)。

## 1. driver_mock

位于 ``src/driver_mock.rs``。

### 功能

模拟时间，用于测试，不依赖真实硬件或系统时间，而是通过 ``advance()`` 调用实现**手动**控制时间流逝，以便于编写确定性测试：

```rust
impl MockDriver {
    /// Advances the time by the specified [`Duration`].
    /// Calling any alarm callbacks that are due.
    pub fn advance(&self, duration: Duration) {
        critical_section::with(|cs| {
            let inner = &mut *self.0.borrow_ref_mut(cs);

            inner.now += duration;
            // wake expired tasks.
            inner.queue.next_expiration(inner.now.as_ticks());
        })
    }
}
```

>### critical_section::with
>
>critical section 意为“临界区”。这是在 Rust 嵌入式开发中（尤其是使用 ``cortex-m`` 等微控制器时）常见的**临界区保护**模式。``with()`` 会接收一个闭包，这个闭包中的代码可以**不被中断或抢占地**安全地执行。 
>
>[官方文档](https://docs.rs/critical-section/latest/critical_section/#usage-in-no-std-binaries)中记载了这么一个 example：
>
>```rust
>use core::cell::Cell;
>use critical_section::Mutex;
>
>static MY_VALUE: Mutex<Cell<u32>> = Mutex::new(Cell::new(0));
>
>critical_section::with(|cs| {
>    // This code runs within a critical section.
>
>    // `cs` is a token that you can use to "prove" that to some API,
>    // for example to a `Mutex`:
>    MY_VALUE.borrow(cs).set(42);
>});
>```

``advance()`` 修改的是 ``MockDriver`` 中的 ``InnerMockDriver::now`` 属性，这个属性就是**现在的时间**。

源文件注释中给出了一个使用 ``advance()`` 的 example：

```rust
fn has_a_second_passed(reference: Instant) -> bool {
    Instant::now().duration_since(reference) >= Duration::from_secs(1)
}

fn test_second_passed() {
    let driver = embassy_time::MockDriver::get();
    let reference = Instant::now();
    assert_eq!(false, has_a_second_passed(reference));
    driver.advance(Duration::from_secs(1));
    assert_eq!(true, has_a_second_passed(reference));
}
```

### schedule_wake

实现 ``Driver`` trait 便需要实现其 ``schedule_wake`` 方法：

```rust
// src/driver_mock.rs

use critical_section::Mutex as CsMutex;
use embassy_time_driver::Driver;
use embassy_time_queue_utils::Queue;

pub struct MockDriver(CsMutex<RefCell<InnerMockDriver>>);

struct InnerMockDriver {
    now: Instant,
    queue: Queue,
}

embassy_time_driver::time_driver_impl!(static DRIVER: MockDriver = MockDriver::new());

impl Driver for MockDriver {
    // ...

    fn schedule_wake(&self, at: u64, waker: &Waker) {
        critical_section::with(|cs| {
            let inner = &mut *self.0.borrow_ref_mut(cs);
            // enqueue it
            inner.queue.schedule_wake(at, waker);
            // wake it if it's in the past.
            inner.queue.next_expiration(inner.now.as_ticks());
        })
    }
}
```

这里使用了 ``embassy_time_driver`` 包中的 ``time_driver_impl!`` 宏，宏的具体实现以及作用可参考 [timer.md 中的记述](./timer.md)。

这里使用的 ``embassy_time_queue_utils::Queue`` 中的两个方法 ``schedule_wake()`` 和 ``next_expiration()`` 可参考[关于 Queue](./queue.md)。

所以，``MockDriver::schedule_wake()`` 其实就是将一个将在 ``at`` 时运行的 task 插入 timer queue 并唤醒所有过期的 task。

## 2. driver_std

TODO

## 3. driver_wasm

TODO
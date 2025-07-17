# embassy_time 包提供的三种 time driver

embassy_time 项目地址位于[此处](https://github.com/embassy-rs/embassy/tree/b528ed06e3025e0803e8fd6dc53ac968df9f49bc/embassy-time)

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

另外，driver_mock 使用的是 embassy-time package 自己实现的 ``Instant`` 和 ``Duration``（见 src/instant.rs）。

## 2. driver_std

### struct Signaler

``driver_std`` 中**基于 std** 实现了 ``Signaler`` 类型，用于实现线程之间的**等待-通知**机制。

```rust
// src/driver_std.rs

struct Signaler {
    mutex: Mutex<bool>,
    condvar: Condvar,
}

impl Signaler {
    const fn new() -> Self {
        Self {
            mutex: Mutex::new(false),
            condvar: Condvar::new(),
        }
    }

    fn wait_until(&self, until: StdInstant) {
        let mut signaled = self.mutex.lock().unwrap();
        while !*signaled {
            let now = StdInstant::now();

            if now >= until {
                break;
            }

            let dur = until - now;
            let (signaled2, timeout) = self.condvar.wait_timeout(signaled, dur).unwrap();
            signaled = signaled2;
            if timeout.timed_out() {
                break;
            }
        }
        *signaled = false;
    }

    fn signal(&self) {
        let mut signaled = self.mutex.lock().unwrap();
        *signaled = true;
        self.condvar.notify_one();
    }
}
```

``Signaler::wait_until()`` 中的局部变量 ``signaled`` 表示该等待线程是否已经收到 notify 通过。该函数可能因超时而返回，也可能因为被 notify 了而返回。该函数返回后，被阻塞的线程继续执行。

### TimeDriver

属性：

```rust
// src/driver_std.rs

struct TimeDriver {
    signaler: Signaler,
    inner: Mutex<Inner>,
}

struct Inner {
    zero_instant: Option<StdInstant>,
    queue: Queue,
}
```

方法：

```rust
// src/driver_std.rs

impl Inner {
    fn init(&mut self) -> StdInstant {
        *self.zero_instant.get_or_insert_with(|| {
            thread::spawn(alarm_thread);
            StdInstant::now()
        })
    }
}

impl Driver for TimeDriver {
    fn now(&self) -> u64 {
        let mut inner = self.inner.lock().unwrap();
        let zero = inner.init();
        StdInstant::now().duration_since(zero).as_micros() as u64
    }

    fn schedule_wake(&self, at: u64, waker: &core::task::Waker) {
        let mut inner = self.inner.lock().unwrap();
        inner.init();
        if inner.queue.schedule_wake(at, waker) {
            self.signaler.signal();
        }
    }
}
```

在 ``Inner::init()`` 中，如果 ``Inner::zero_instant`` 为 ``None`` 则开启一个守护线程 ``alarm_thread``，并将当前时间值赋值给 ``Inner::zero_instant``。所以 ``zero_instant`` 记录的是 ``alarm_thread`` 开始运行的时间，作为基准时间。关于这个 ``alarm_thread()`` 函数：

```rust
fn alarm_thread() {
    let zero = DRIVER.inner.lock().unwrap().zero_instant.unwrap();
    loop {
        let now = DRIVER.now();

        let next_alarm = DRIVER.inner.lock().unwrap().queue.next_expiration(now);

        // Ensure we don't overflow
        let until = zero
            .checked_add(StdDuration::from_micros(next_alarm))
            .unwrap_or_else(|| StdInstant::now() + StdDuration::from_secs(1));

        DRIVER.signaler.wait_until(until);
    }
}
```

``alarm_thread`` 是一个调度线程，负责循环唤醒等待中的任务。每个循环：

- 唤醒 time queue 中所有过期的任务
- 默认休眠 1 秒，（如果不溢出的话）休眠到最近的任务过期时

综上所述，``TimeDriver::signaler`` 基于 std 互斥锁和条件变量实现线程间等待-通知机制；而 ``Inner::zero_instant`` 作为时间基准，记录了 ``alarm_thread`` 开始运行的时间；``Inner::queue`` 是一个 time queue；而 ``alarm_thread`` 作为调度线程，负责定时唤醒任务。

## 3. driver_wasm

TODO

## 补充

### 关于 critical_section::with

critical section 意为“临界区”。这是在 Rust 嵌入式开发中（尤其是使用 ``cortex-m`` 等微控制器时）常见的**临界区保护**模式。``with()`` 会接收一个闭包，这个闭包中的代码可以**不被中断或抢占地**安全地执行。 

[官方文档](https://docs.rs/critical-section/latest/critical_section/#usage-in-no-std-binaries)中记载了这么一个 example：

```rust
use core::cell::Cell;
use critical_section::Mutex;

static MY_VALUE: Mutex<Cell<u32>> = Mutex::new(Cell::new(0));

critical_section::with(|cs| {
    // This code runs within a critical section.

    // `cs` is a token that you can use to "prove" that to some API,
    // for example to a `Mutex`:
    MY_VALUE.borrow(cs).set(42);
});
```

### [关于 std::sync::Mutex](https://doc.rust-lang.org/std/sync/struct.Mutex.html)

互斥锁

- ``Mutex::lock()``

   ```rust
   pub fn lock(&self) -> LockResult<MutexGuard<'_, T>>
   ```

   **阻塞**当前线程，直到获取到锁。

   ``MutexGuard`` 提供对内部数据的独占访问（``&mut T``），且会在离开当前作用域时通过 ``Drop`` trait **自动**释放锁。

### [关于 std::sync::Condvar](https://doc.rust-lang.org/std/sync/struct.Condvar.htmls)

条件变量

- [``Condvar::wait_timeout()``](https://doc.rust-lang.org/std/sync/struct.Condvar.html#method.wait_timeout)

   ```rust
   pub fn wait_timeout<'a, T>(
       &self,
       guard: MutexGuard<'a, T>,
       dur: Duration,
   ) -> LockResult<(MutexGuard<'a, T>, WaitTimeoutResult)>
   ```

   ``wait_timeout`` 允许线程在等待条件变量通知时，设置一个最大等待时间。它会：
   - **阻塞**当前线程，直到
       - 收到 ``notify_one()`` 或 ``notify_all()`` 的唤醒信号，**或**
       - 达到指定的超时时间
   - 返回一个**元组 (MutexGuard, WaitTimeoutResult)**，其中：
       - ``MutexGuard``：重新获取的互斥锁（与 ``Condvar`` 关联的锁）
       - ``WaitTimeoutResult``：指示是否因超时而返回（通过 ``.timed_out()`` 检查）

- ``Condvar::notify_one()``

### 关于 [Option::get_or_insert_with()](https://doc.rust-lang.org/std/option/enum.Option.html#method.get_or_insert_with)

```rust
pub fn get_or_insert_with<F>(&mut self, f: F) -> &mut T
where
    F: FnOnce() -> T,
```

如果 ``Option`` 为 ``Some(value)`` 则返回 ``value`` 的可变引用；否则调用闭包生成值，插入后返回其可变引用。

### 关于 [std::time::Instant](https://doc.rust-lang.org/std/time/struct.Instant.html)（即这里的 StdInstant）

```rust
pub fn duration_since(&self, earlier: Instant) -> Duration
```

返回从 earlier 到 self 的 duration，如果 earlier 比 self 更晚则返回 zero duration。

```rust
pub fn checked_add(&self, duration: Duration) -> Option<Instant>
```
如果 self + duration 超出最大表示范围则返回 ``None``，否则返回 ``Some(t: Instant)``，``t`` 为 self + duration。

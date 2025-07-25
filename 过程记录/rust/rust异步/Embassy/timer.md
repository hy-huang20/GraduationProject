# embassy_time::Timer 源码

我在 [embassy-learning](https://github.com/hy-huang20/rust-learning/tree/embassy-learning/embassy-learning) 中使用到了 embassy_time 包中 Timer 中的函数。这里分析一下使用的 Timer 中函数的调用过程，以 ``Timer::after_secs()`` 为例。

```rust
// src/main.rs

use embassy_executor::{Spawner, task, main};
use embassy_time::{Timer, Duration};

#[task]
async fn run() {
    loop {
        info!("tick");
        Timer::after_secs(1).await;
    }
}
```

首先查看一下 ``Timer::after_secs()`` 的实现：

```rust
// embassy/embassy-time/src/timer.rs

pub struct Timer {
    expires_at: Instant,
    yielded_once: bool,
}

impl Timer {
    pub fn after(duration: Duration) -> Self {
        Self {
            expires_at: Instant::now() + duration,
            yielded_once: false,
        }
    }

    #[inline]
    pub fn after_secs(secs: u64) -> Self {
        Self::after(Duration::from_secs(secs))
    }
}
```

所以 ``Timer::after_secs()`` 返回了一个 ``Timer`` 类型。由于能够对该函数返回值调用 ``.await``，可见该类型实现了 ``Future`` trait:

```rust
// embassy/embassy-time/src/timer.rs

impl Future for Timer {
    type Output = ();
    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        if self.yielded_once && self.expires_at <= Instant::now() {
            Poll::Ready(())
        } else {
            embassy_time_driver::schedule_wake(self.expires_at.as_ticks(), cx.waker());
            self.yielded_once = true;
            Poll::Pending
        }
    }
}
```

以上的 ``poll()`` 方法只有在显式调用或者在 future 上 ``.await`` 时才会被执行：关于 [await](https://os.phil-opp.com/zh-TW/async-await/#the-async-await-pattern)。第一次调用 ``poll()`` 由于 ``yielded_once`` 初值为 false，所以一定执行 else 分支，并调用了 ``embassy_time_driver`` 包中的 ``schedule_wake()``：

```rust
// embassy/embassy-time-driver/src/lib.rs

extern "Rust" {
    fn _embassy_time_schedule_wake(at: u64, waker: &Waker);
}

/// Schedule the given waker to be woken at `at`.
#[inline]
pub fn schedule_wake(at: u64, waker: &Waker) {
    unsafe { _embassy_time_schedule_wake(at, waker) }
}

#[macro_export]
macro_rules! time_driver_impl {
    (static $name:ident: $t: ty = $val:expr) => {
        static $name: $t = $val;

        #[no_mangle]
        #[inline]
        fn _embassy_time_schedule_wake(at: u64, waker: &core::task::Waker) {
            <$t as $crate::Driver>::schedule_wake(&$name, at, waker);
        }
    };
}
```

关于如何使用这个宏 ``time_driver_impl`` 的详细信息，可以查看同文件中的**注释**。简单来说，需要在外部模块实现类似于以下的内容：

```rust
use core::task::Waker;

use embassy_time_driver::Driver;

struct MyDriver{} // not public!

impl Driver for MyDriver {
    // ...

    fn schedule_wake(&self, at: u64, waker: &Waker) { // 会调用这个函数
        todo!()
    }
}

embassy_time_driver::time_driver_impl!(static DRIVER: MyDriver = MyDriver{});
```

之后，``_embassy_time_schedule_wake()`` 中实际就会调用 ``MyDriver`` 中的 ``schedule_wake()``。

在 ``embassy_time`` 包中提供了三种这样的 driver，它们位于 ``src/driver_*.rs`` 文件中；而具体启用哪种，在 ``src/lib.rs`` 中被决定，可以看到这样的条件编译代码：

```rust
// embassy/embassy-time/src/lib.rs

#[cfg(feature = "mock-driver")]
mod driver_mock;

#[cfg(feature = "mock-driver")]
pub use driver_mock::MockDriver;

#[cfg(feature = "std")]
mod driver_std;
#[cfg(feature = "wasm")]
mod driver_wasm;
```

关于 ``driver_mock``, ``driver_std``, ``driver_wasm`` 这三种 time driver 的介绍可以参考[这里](./time-driver.md)。

例子是在 wsl 中运行的，使用的是 ``driver_std``。[验证](https://github.com/hy-huang20/rust-learning/blob/embassy-learning/embassy-learning/tests/use_driver_std.rs)
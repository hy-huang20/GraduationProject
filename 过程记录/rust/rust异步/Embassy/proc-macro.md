# embassy-executor 的过程宏

Rust 的``过程宏（Procedural Macro）``是一种在编译期间操作和生成 Rust 代码的强大工具。

在[例子](https://github.com/hy-huang20/rust-learning/blob/embassy-learning/embassy-learning/src/main.rs)中见到的 ``#[task]`` 和 ``#[main]`` 实际是过程宏中的属性宏: ``#[name_of_macro(attribute1, attribute2, ...)]``，用于添加或修改代码结构中的元数据信息。

## 代码定位

位于 ``embassy-executor-macros`` package。[访问](https://github.com/embassy-rs/embassy/tree/main/embassy-executor-macros)。使用 embassy 的用户并不是直接使用这个 package，这个 package 在 ``embassy-executor`` 中被 re-exported，从而能被用户访问到。在 ``embassy-executor`` package 中的 ``src/lib.rs`` 中：

```rust
pub use embassy_executor_macros::task;

#[cfg(feature = "_arch")]
#[cfg_attr(feature = "arch-avr", path = "arch/avr.rs")]
#[cfg_attr(feature = "arch-cortex-m", path = "arch/cortex_m.rs")]
#[cfg_attr(feature = "arch-cortex-ar", path = "arch/cortex_ar.rs")]
#[cfg_attr(feature = "arch-riscv32", path = "arch/riscv32.rs")]
#[cfg_attr(feature = "arch-std", path = "arch/std.rs")]
#[cfg_attr(feature = "arch-wasm", path = "arch/wasm.rs")]
#[cfg_attr(feature = "arch-spin", path = "arch/spin.rs")]
mod arch;

#[cfg(feature = "_arch")]
#[allow(unused_imports)] // don't warn if the module is empty.
pub use arch::*;
#[cfg(not(feature = "_arch"))]
pub use embassy_executor_macros::main_unspecified as main;
```

可见，``#[task]`` 相关代码平台无关所以直接 pub use，而 ``#[main]`` 会根据平台的不同而使用不同的代码。这些不同的代码在 ``embassy-executor`` 包的 ``src/arch/`` 文件夹下。这里以 ``src/arch/std.rs`` 举例。主要代码：

```rust
pub use thread::*;
mod thread {
    pub use embassy_executor_macros::main_std as main;

    #[export_name = "__pender"]
    fn __pender(context: *mut ()) {
        let signaler: &'static Signaler = unsafe { std::mem::transmute(context) };
        signaler.signal()
    }

    /// Single-threaded std-based executor.
    pub struct Executor {
        inner: raw::Executor,
        not_send: PhantomData<*mut ()>,
        signaler: &'static Signaler,
    }
    
    impl Executor {
        /// Create a new Executor.
        pub fn new() -> Self { /* ... */ }

        /// Run the executor.
        pub fn run(&'static mut self, init: impl FnOnce(Spawner)) -> ! { /* ... */ }
    }
}
```

所以在 std 平台使用 ``#[main]`` 会在编译期执行 ``embassy-executor-macros`` 包中的一个名为 ``main_std`` 的函数。[测试在 wsl 上使用 embassy-executor 是否确实会在编译期执行到 main_std()](https://github.com/hy-huang20/rust-learning/tree/embassy-learning/embassy-learning/tests)。

这里实现了 ``__pender()``，和[embassy-executor 中关于 mod.rs 源码的分析](https://github.com/hy-huang20/rust-os-learning/blob/main/%E8%BF%87%E7%A8%8B%E8%AE%B0%E5%BD%95/rust/rust%E5%BC%82%E6%AD%A5/Embassy/executor/raw/mod.md#%E5%85%B3%E4%BA%8E-__pender)相对应。

## 编译期代码转换

主要工作由 ``embassy-executor-macros`` 中的相关代码完成。顺着之前的思路找到 ``main_std`` 函数：

```rust
// src/lib.rs

#[proc_macro_attribute]
pub fn task(args: TokenStream, item: TokenStream) -> TokenStream {
    task::run(args.into(), item.into()).into()
}

#[proc_macro_attribute]
pub fn main_std(args: TokenStream, item: TokenStream) -> TokenStream {
    main::run(args.into(), item.into(), &main::ARCH_STD).into()
}
```

这里的 ``task`` 函数是平台无关的。

``src/lib.rs`` 中还有很多以 ``main`` 开头的函数，分别对应不同平台。这些 ``main_*`` 函数的差异仅仅是 ``main::run()`` 第 3 个参数的不同，像这里 std 对应的第 3 个参数便是 ``&main::ARCH_STD``。``task::run()`` 和 ``main::run()`` 分别定义在 ``src/macros/task.rs`` 和 ``src/macros/main.rs`` 中。

### task::run() 源码

TODO

### main::run() 源码

TODO

## 宏展开结果

[一个简单的属性宏使用例](https://github.com/hy-huang20/rust-learning/tree/attribute-macros/attribute-macros)

可以通过对上述源码进行分析得出展开后的代码，不过这里直接使用 cargo-expand 工具自动展开宏。[例子](https://github.com/hy-huang20/rust-learning/blob/embassy-learning/embassy-learning/src/main.rs)中的代码会被转换成[这样](https://github.com/hy-huang20/rust-learning/blob/main/embassy-learning/src/expand.rs)。
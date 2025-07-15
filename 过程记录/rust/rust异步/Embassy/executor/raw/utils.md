# src/raw/utils.rs 源码

## UninitCell

### 定义

一个**单一字段**的元组结构体（Tuple Struct），称为 ``Newtype 模式``。[Effective Rust 第 6 条：拥抱 newtype 模式](https://colobu.com/effective-rust/chapter_1/item6-newtype.html)

```rust
pub(crate) struct UninitCell<T>(MaybeUninit<UnsafeCell<T>>);
```

>#### [关于 core::mem::MaybeUninit](https://doc.rust-lang.org/core/mem/union.MaybeUninit.html)
>
>用于实现``延迟初始化``。对 ``MaybeUninit::uninit()``（类型为 ``MaybeUninit<T>``）调用 ``write()`` 后，方可通过 ``assume_init()`` 获取已初始化的值（类型为 ``T``）。

>#### [关于 core::cell::UnsafeCell](https://doc.rust-lang.org/core/cell/struct.UnsafeCell.html)
>
> ``&T`` 类型是``内部不可变``的，即不能修改其指向的数据；而 ``&UnsafeCell<T>`` 是``内部可变``的。
>
>``UnsafeCell::get()`` 可获取一个 wrapped value 的可变指针 ``*mut T``。

### 实现

```rust
impl<T> UninitCell<T> {
    pub const fn uninit() -> Self {
        Self(MaybeUninit::uninit())
    }

    pub unsafe fn as_mut_ptr(&self) -> *mut T {
        (*self.0.as_ptr()).get()
    }

    #[allow(clippy::mut_from_ref)]
    pub unsafe fn as_mut(&self) -> &mut T {
        &mut *self.as_mut_ptr()
    }

    #[inline(never)]
    pub unsafe fn write_in_place(&self, func: impl FnOnce() -> T) {
        ptr::write(self.as_mut_ptr(), func())
    }

    pub unsafe fn drop_in_place(&self) {
        ptr::drop_in_place(self.as_mut_ptr())
    }
}
```

>#### 关于 #[allow(clippy::mut_from_ref)]
>
>这是一个 Rust ``属性（attribute）``，用于临时禁用 Clippy（Rust 的官方 lint 工具）对从 ref（``&T``）获取 mut（``&mut T``）的**警告**。

>#### 关于 [ptr::drop_in_place()](https://doc.rust-lang.org/core/ptr/fn.drop_in_place.html)
>
>```rust
>pub unsafe fn drop_in_place<T: ?Sized>(to_drop: *mut T)
>```
>调用 ``T`` 类型的 destructor。

## SyncUnsafeCell

### 定义

```rust
#[repr(transparent)]
pub struct SyncUnsafeCell<T> {
    value: UnsafeCell<T>,
}
```

>#### 关于 #[repr(transparent)]
>
>Rust 的一种``内存布局属性（representation attribute）``。用于确保 ``newtype`` 或者 ``wrapper type`` 与其内部单一字段在内存中的表示完全相同，没有任何额外开销。

### 实现

```rust
unsafe impl<T: Sync> Sync for SyncUnsafeCell<T> {}

impl<T> SyncUnsafeCell<T> {
    #[inline]
    pub const fn new(value: T) -> Self {
        Self {
            value: UnsafeCell::new(value),
        }
    }

    pub unsafe fn set(&self, value: T) {
        *self.value.get() = value;
    }

    pub unsafe fn get(&self) -> T
    where
        T: Copy,
    {
        *self.value.get()
    }
}
```

>#### 关于 [Sync trait](https://doc.rust-lang.org/core/marker/trait.Sync.html)
>
>编译器会在自己认为合适时（比如，这个类型的所有字段都是 ``Sync``）自动实现这个 triat。但是：
>
>- ``UnsafeCell`` 会阻止自动实现 ``Sync``
>   - ``UnsafeCell<T>`` 是 Rust 内部可变性的基础类型，编译器会默认其不是 ``Sync``
>   - 如果类型包含 ``UnsafeCell``，编译器会保守地不自动推导 ``Sync``，即使开发者已经通过锁等方法保证了线程安全
>- 这里手动实现了 ``Sync`` 的含义：
>   - 虽然这个类型包含 ``UnsafeCell``，但是我**以开发者的名义保证**，它的共享引用 ``&T`` 可以安全地跨线程使用
>
>这里实现 ``Sync`` 是``零成本抽象（Zero-Cost Abstractions）``思想的体现：
>
>- ``SyncUnsafeCell<T>`` 本身只是作为 ``UnsafeCell<T>`` 的透明封装而存在，手动实现 ``Sync`` 告诉编译器不要引入额外的内存分配或生成额外的同步代码，而且这个类型在 ``T: Sync`` 条件下是线程安全的，无需引入运行时检查。同时 ``unsafe`` 说明了该类型的线程安全性依赖调用者的正确使用。
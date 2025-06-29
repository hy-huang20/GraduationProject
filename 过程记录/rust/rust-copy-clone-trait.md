# Copy trait 和 Clone trait

[Copy trait doc](https://doc.rust-lang.org/std/marker/trait.Copy.html) 和 [Clone trait doc](https://doc.rust-lang.org/std/clone/trait.Clone.html)


## Copy trait

>Types whose values can be duplicated simply by copying bits.

- 不可被重载
    - 行为固定：简单按位复制
    - 原因
        - ``Copy`` 是一个 marker trait
            - 作用仅仅是**标记**类型可以通过简单的位拷贝安全复制
        - ``Copy`` 不包含任何方法，无法自定义实现逻辑
- 行为是隐式发生的
    - 比如：赋值 ``=`` 时自动发生，无需显式调用
- 一个类型可以实现 ``Copy`` **当且仅当**其所有成员实现了 ``Copy``
    - 不管 ``T`` 是否实现 ``Copy``
        - ``&T`` 总是实现 ``Copy`` 的，``&mut T`` 总不是 ``Copy`` 的
            - ``&mut T`` 遵循**独占性**
- 如果一个类型实现了 ``Drop`` 则其必然不能实现为 ``Copy``
    - 比如：``String``    
    >... any type implementing Drop can’t be Copy, because it’s managing some resource besides its own size_of::\<T> bytes.
    - 说人话就是 ``size_of`` 计入不了**编译期**无法确定大小的**动态数据**


## Clone trait

>A common trait for the ability to explicitly duplicate an object.

- 可以被重新实现
- 行为是显式发生的
    - 用户必须手动 ``.clone()``
- ``Clone`` 是 ``Copy`` 的父 trait
    - 原文是 ``supertrait``
    - 只要实现了 ``Copy`` 就自动实现了 ``Clone``
        - 此时 ``.clone()`` 的行为**默认且必须**等同于 ``*self``
        - 如果用户尝试手动实现 ``clone`` 且行为与上述不一致，编译器会拒绝
- 一个类型可以**自动**实现 ``Clone`` **当且仅当**其所有属性都实现了 ``Clone``
    - 通过 ``#[derive(Clone)]`` 来**自动**实现 ``Clone``
    - 但如果选择**手动**实现 ``Clone`` 则可以**放宽**该限制
        - 比如在自定义 ``clone()`` 逻辑时忽略那些**非** ``Clone`` 属性

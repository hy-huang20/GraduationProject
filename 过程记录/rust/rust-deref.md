# rust 中 String 类型和 &str 类型相加

## 结论

在 Rust 中：

- ``String`` 类型不能和 ``String`` 类型直接相加
- ``&str`` 类型不能和 ``&str`` 类型直接相加。
- ``+`` 运算符**强制**要求运算符左侧是 ``String`` 右侧是 ``&str``

究其原理，会涉及到 ``Deref`` trait，下面作简单介绍。

## Deref 简介

### 解引用

*以下内容来自于 [Rust 官方文档](https://doc.rust-lang.org/std/ops/trait.Deref.html)*

```rust
pub trait Deref {
    type Target: ?Sized;

    // Required method
    const fn deref(&self) -> &Self::Target;
}
```

- 关于 ``?Sized``
    - ``?Sized`` 其实就是**允许动态大小**
    - [官方文档]((https://doc.rust-lang.org/std/marker/trait.Sized.html))中的示例代码
    ```rust
    struct Foo<T>(T);
    struct Bar<T: ?Sized>(T);

    // struct FooUse(Foo<[i32]>); // error: Sized is not implemented for [i32]
    struct BarUse(Bar<[i32]>); // OK
    ```
    因为 ``[i32]`` 是动态大小，所以 ``FooUse`` 通不过编译，因为**默认**不允许动态大小。如果修改为
    ```rust
    struct FooUse(Foo<[i32; 3]>);
    ```
    则由于 ``[i32; 3]`` 固定大小数组在**编译期**便能确定大小，编译可以通过。

- 注意到 deref **必须**返回一个值的**引用**，原因见下文

*以下内容来自于 [Rust 程序设计语言](https://kaisery.github.io/trpl-zh-cn/ch15-02-deref.html#%E4%BD%BF%E7%94%A8-deref-trait-%E5%B0%86%E6%99%BA%E8%83%BD%E6%8C%87%E9%92%88%E5%BD%93%E4%BD%9C%E5%B8%B8%E8%A7%84%E5%BC%95%E7%94%A8%E5%A4%84%E7%90%86)*

``Deref`` trait 顾名思义就是解引用。实现 ``Deref`` trait 允许我们定制**解引用运算符**（*dereference operator*）``*``。

当你使用 ``*`` 对一个实现了 ``Deref`` trait 的类型进行解引用时，rust 会先调用 ``deref()`` 方法，然后再进行普通的解引用操作。也就是当输入

```rust
*obj
```

时，rust 实际会在底层运行

```rust
*(obj.deref())
```

上文中提到了 ``deref()`` 被规定必须返回一个值的引用；而且，为什么调用 ``deref()`` 后括号外还要再有一次**普通解引用**呢？以上两点是因为 rust 的**所有权**。首先，如果 ``deref`` 方法直接返回值而不是值的引用，其值将被移出 ``self``；其次，由于 ``deref()`` 返回值的引用，而最终我们需要的是解引用后的东西，所以还要在此基础上多加一次普通解引用。（注意 rust 和 C++ 关于引用的区别，C++ 引用其实是一个别名，访问其值不需要解引用操作符；而 rust 引用相当于一个指针，一个地址，需要解引用才能访问引用的值 [关于 rust 引用](./rust-diff-ref-ptr.md)）

还要注意的是，上面提到的**括号外的普通解引用**并不会引起 ``deref()`` 的调用从而陷入无限递归。也即，这个普通解引用只会在 ``deref()`` 后发生一次。

### Deref 隐式强制类型转换

**规则**：如果类型 ``U`` 实现了 ``Deref<Target = T>``，那么 ``&U`` 会自动转换为 ``&T``（rustc 隐式调用 ``deref()``），使得 ``&U`` 可以直接用在需要 ``&T`` 的地方。

到这里可能会困惑于强制类型转换和解引用有什么关系。个人理解是 ``deref()`` 定义了一种如何去解引用的方式，比如 ``deref()`` **返回什么类型的引用**也属于这个范畴。

*这里的 ``Deref`` 简介只是为了快速回忆，其实关于 ``Deref`` 还有很多内容，详情见参考部分。*

## 参考

- 关于 ``Deref`` trait
    - [Rust 官方文档](https://doc.rust-lang.org/std/ops/trait.Deref.html)
    - [Rust 程序设计语言](https://kaisery.github.io/trpl-zh-cn/ch15-02-deref.html#%E4%BD%BF%E7%94%A8-deref-trait-%E5%B0%86%E6%99%BA%E8%83%BD%E6%8C%87%E9%92%88%E5%BD%93%E4%BD%9C%E5%B8%B8%E8%A7%84%E5%BC%95%E7%94%A8%E5%A4%84%E7%90%86)
        - 提一嘴文档里面的 ``Box<T>`` 是 Rust 标准库提供的智能指针
- [关于隐式 Deref 强制转换](https://kaisery.github.io/trpl-zh-cn/ch15-02-deref.html#%E5%87%BD%E6%95%B0%E5%92%8C%E6%96%B9%E6%B3%95%E7%9A%84%E9%9A%90%E5%BC%8F-deref-%E5%BC%BA%E5%88%B6%E8%BD%AC%E6%8D%A2)
- [关于 String 的 Deref](https://doc.rust-lang.org/std/string/struct.String.html#deref)
- [String 类型和 &str 类型相加](https://doc.rust-lang.org/std/string/struct.String.html#impl-Add%3C%26str%3E-for-String)
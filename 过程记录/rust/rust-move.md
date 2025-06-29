# rust 中的 move 关键字

## 概念

[闭包](./rust-closure.md)

## 关于 move

除非变量实现了 ``Copy`` [trait](./rust-copy-clone-trait.md)，否则 ``move`` 会**强制**闭包获取所有捕获变量的所有权（按值捕获），无论闭包如何使用这些变量。之后外部不能再使用这些变量。

## 参考

- [Rust 官方文档](https://doc.rust-lang.org/std/keyword.move.html)
- [The Rust Programming Language](https://doc.rust-lang.org/book/ch13-01-closures.html)
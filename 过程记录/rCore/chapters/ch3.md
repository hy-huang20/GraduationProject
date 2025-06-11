# ch3

## 编程实验

最简单的一章

只需要实现一个 ``sys_trace`` 系统调用

首先是熟悉一下如何利用 rust 实现从指定地址读/向指定地址写单字节数据：

```rust
// os/src/syscall/process.rs

use core::ptr;

unsafe { ptr::read(addr) }; // 读
unsafe { ptr::write(addr, data) }; // 写
```

读写每次均只涉及一个字节，因此如果遇到写的情形需要首先将形参中的 _data 转为 u8 以获取低位单字节。

``sys_trace`` 还有一个功能就是获取当前 task 调用任意编号系统调用的次数，因此需要在每次 syscall 的时候记录在某个地方。我的做法是为每个 tcb 增加一个 usize **定长数组**属性用来记录。定长数组的大小我设置为 500，即 syscall id 的一个上界。

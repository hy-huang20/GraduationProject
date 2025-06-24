# rust 生命周期

## 简明介绍

*以下内容来自于 [Rust Course](https://course.rs/about-book.html)*

### [初级](https://course.rs/basic/lifetime.html)

其中一个小地方一开始不太理解，原文是这么说的：

#### 原文节选

在结束这块儿内容之前，再来做一个有趣的修改，将方法返回的生命周期改为 ``'b``：

```rust
impl<'a> ImportantExcerpt<'a> {
    fn announce_and_return_part<'b>(&'a self, announcement: &'b str) -> &'b str {
        println!("Attention please: {}", announcement);
        self.part
    }
}
```

此时，编译器会报错，因为编译器无法知道 ``'a`` 和 ``'b`` 的关系。 ``&self`` 生命周期是 ``'a``，那么 ``self.part`` 的生命周期也是 ``'a``，但是好巧不巧的是，我们手动为返回值 ``self.part`` 标注了生命周期 ``'b``，因此编译器需要知道 ``'a`` 和 ``'b`` 的关系。

有一点很容易推理出来：由于 ``&'a self`` 是被引用的一方，因此引用它的 ``&'b str`` 必须要活得比它短，否则会出现悬垂引用。因此说明生命周期 ``'b`` 必须要比 ``'a`` 小，只要满足了这一点，编译器就不会再报错：

```rust
impl<'a: 'b, 'b> ImportantExcerpt<'a> {
    fn announce_and_return_part(&'a self, announcement: &'b str) -> &'b str {
        println!("Attention please: {}", announcement);
        self.part
    }
}
```

#### 原文节选分析

其实这里合理的写法就是将返回值类型写为 ``&'a str``，而不是像文中那样改成 ``&'b str``，因为 ``self.part`` 本身就是 ``&'a str``。但如果非要这么写，还想要编译通过的话，就得保证 ``'a`` 至少和 ``'b`` 活得一样长。否则如果 ``'b`` 活得更长的话，这里由于该方法的调用者将一个 ``'a`` 的引用当成 ``'b`` 的引用使用，也就是导致了把一个实际活得短的引用当成了能够活得长的引用去使用，便有可能造成悬垂引用。

### [进阶](https://course.rs/advance/lifetime/intro.html)

TODO

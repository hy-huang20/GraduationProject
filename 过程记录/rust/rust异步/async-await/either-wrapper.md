# Either 包装器

## Enum futures::future::Either

``Either`` 作为一个 ``Enum`` 的声明如下：

```rust
pub enum Either<A, B> {
    Left(A), // 第一种可能的值
    Right(B), // 第二种可能的值
}
```

作用是将两个具有相同``关联类型``的不同 ``Future``, ``Stream``, ``Sink`` 组合为单一类型
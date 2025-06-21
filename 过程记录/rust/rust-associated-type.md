# rust 中的关联类型

关联类型允许在 trait 中声明一个类型占位符，这个类型将由 trait 的实现者来指定。关联类型提供了一种在 trait 中定义类型的方式。

## 基本语法

```rust
trait MyTrait {
    type MyType;  // 关联类型声明
    
    fn do_something(&self) -> Self::MyType;
}

struct MyStruct;

impl MyTrait for MyStruct {
    type MyType = i32;  // 关联类型定义
    
    fn do_something(&self) -> Self::MyType {
        0
    }
}
```

## 对比

其实同样的功能也可以使用``泛型参数``来实现，模板代码如下：

```rust
trait MyTrait<T> {
    fn do_something(&self) -> T;
}

struct MyStruct;

impl MyTrait<i32> for MyStruct {
    fn do_something(&self) -> i32 {
        0
    }
}
```

## 作用

简化代码，明确表示该类型与 trait 的实现相关，增加代码可读性

## 泛型参数 vs 关联类型

|场景|泛型参数|关联类型|
|---|---|---|
|类型由调用方决定|适合|不适合|
|类型由实现方决定|导致冗余|适合（如 ``Iterator``）|
|需要多套类型组合|适合（如 ``HashMap<K, V>``）|不适合|
|类型关系固定|代码冗余|适合（如 ``Deref::Target``）|

## 标准库中的关联类型

经典案例：

- ``Iterator`` trait
    ```rust
    trait Iterator {
        type Item;
        fn next(&mut self) -> Option<Self::Item>;
    }
    ```
- ``Deref`` trait
    ```rust
    trait Deref {
        type Target: ?Sized;
        fn deref(&self) -> &Self::Target;
    }
    ```
# rust 中引用和 raw pointer 的区别

在 rust 中，``&T``（不可变引用/共享引用）, ``&mut T``（可变引用）, ``*T``（裸指针/原始指针/raw pointer） 都属于指针类型，在内存中没有区别，都是存储目标地址的整数值。不过，引用是一种安全的指针，裸指针是一种不安全的指针。

## 区别

|特性|引用|裸指针|
|---|---|---|
|安全性|受编译器严格检查，安全|不安全，需手动管理（在 ``unsafe`` 块中使用）|
|是否可为空|永远不为空，没有空引用|可以为 ``null``|
|生命周期|有明确的生命周期标注（``'a``）|无生命周期约束|
|别名规则|可变引用独占，无可变引用时可允许多个共享引用|无限制，可能引发数据竞争|
|解引用 ``*``|部分自动（见[下文](#关于引用的解引用操作)）|必须显式解引用（且需 ``unsafe``）|
|用途|安全代码中的默认选择|不安全的底层操作|

## 安全性

|安全特性|引用|裸指针|
|---|---|---|
|空指针|永远不为空|可能为 ``null``|
|悬垂指针|编译器通过生命周期检查禁止|可能指向已释放的内存|
|数据竞争|编译时强制读写规则|可能同时读写导致未定义行为|
|作用域逃逸|编译器确保引用不超出数据生命周期|可能逃逸作用域导致非法访问|

## 示例

```rust
// 安全引用（编译器保护）
fn safe_ref() {
    let x = 42;
    let r = &x; // r 是安全指针
    println!("{}", r); // 合法
} // r 和 x 的作用域同步结束，无悬垂问题

// 不安全裸指针（手动管理风险）
fn unsafe_raw() {
    let p: *const i32;
    {
        let x = 42;
        p = &x as *const i32; // 获取裸指针
    } // x 被释放，p 成为悬垂指针！
    unsafe { println!("{}", *p) }; // 未定义行为！
}
```

## 关于引用的解引用操作

### 隐式（自动）

当引用在特定上下文中使用时，rust 会自动调用 ``Deref`` trait 进行隐式解引用，无需手动写 ``*``。主要发生在以下场景：

#### 方法调用

```rust
let s = String::from("hello");
// 自动解引用 &String → &str（因为 String 实现了 Deref<Target=str>）
println!("{}", s.len()); // 等价于 (*s).len() 或 s.deref().len()
```

注意：``s`` 是 ``String`` 类型，由于 ``String`` 实现了 ``Deref<Target = str>``，因此**继承**了 ``str`` 类型的所有方法，因此可以调用 ``.len()``。

在这里可能会好奇为什么 ``(*s).len()`` 这种写法是可行的。以下是其执行步骤：

- ``*s`` 触发 ``Deref`` trait
    - 等价于 ``*(s.deref())``，即先调用 ``s.deref()`` 得到 ``&str``
    - 然后对 ``&str`` 解引用得到 ``str``（注意：``str`` 是动态大小类型，通常只能通过引用操作）
- ``.len()`` 调用时，rust 会自动对 ``str`` 借用为 ``&str``，最终调用 ``str::len()``。这里关于**自动对切片借用为切片引用**的表述，[这篇博客](https://zhuanlan.zhihu.com/p/708113253)里有一句似乎相关的表述：**``.``操作可以自动引用/解引用**。此外，切片基础知识可参考[关于切片和切片引用](https://course.rs/difficulties/slice.html)

#### 函数传参

```rust
fn print_len(s: &str) {
    println!("{}", s.len());
}

let s = String::from("hello");
print_len(&s); // 自动 &String → &str
```

#### 链式调用

```rust
struct MyBox<T>(T);
impl<T> std::ops::Deref for MyBox<T> {
    type Target = T;
    fn deref(&self) -> &T { &self.0 }
}

let my_box = MyBox(42);
println!("{}", my_box.abs()); // 自动 MyBox<i32> → &i32
```

### 显式

当需要直接操作**值**（而非引用）时，必须显式使用 ``*``

#### 赋值或表达式

```rust
let x = 42;
let r = &x;
let y = *r + 1; // 显式解引用获取 i32 值
```

#### 修改数据（可变引用）

```rust
let mut x = 42;
let r = &mut x;
*r += 1; // 必须用 * 解引用以修改值
```

#### 匹配所有权

```rust
let s = String::from("hello");
let s_ref = &s;
// let s_val = *s_ref; // 错误！String 不是 Copy 类型，不能直接移动
let s_val = (*s_ref).clone(); // 显式解引用后克隆
```
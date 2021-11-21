# 为什么要有 Associated Type？

## 背景

什么是 Associated Type？

Associated Type 是泛型的一个子概念。在 Rust 里 Associated Type 和 Trait 绑定在一起，指定输出类型 (output type)。

Rust Book 里的一个例子是：

```rust
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}

impl Iterator for Counter {
    type Item = u32;

    fn next(&mut self) -> Option<Self::Item> {
        // --snip--
```

这样写，`Counter`里的`Self::Item`就指代的是`u32`。

而更详细的介绍请看[Rust Book 对应章节](https://doc.rust-lang.org/book/ch19-03-advanced-traits.html)。

## 答案

短答案是：

* 带有 Associated Type 的 Trait 只能被一个类型`impl` 一次，所以可以避免一个类型有多个`impl`。
* Associated Type 可以当做 Output Type
* Associated Type 带来工程上的便利



如果短答案总觉得是隔靴搔痒，那么我们需要问以下子问题：

1. 既然 Associated Type 是泛型的子概念，那么 Associated Type 和 Rust 泛型有什么不同？
2. 什么是 Input Type 和 Output Type？
3. 为什么带有 Associated Type 的 Trait 只能被一个类型`impl` 一次？
3. 普通泛型可以代替 Associated Type 吗？如果可以，那为什么还要 Associated Type？

对于第一个问题，它们的不同主要有两点：

* 普通泛型可以用于 Trait, Struct和函数，但是 Associated Type 只能与 Trait 绑定。

* 带有 Associated Type 的 Trait 只能被一个类型`impl` 一次，但是带有普通泛型的 Trait 可以有多个`impl`，例如

    ```rust
    // 普通泛型 + Trait
    pub trait GiveMeSomething<T: Clone> {
        fn get_something(&self) -> T;
    }
    
    // Associated Type + Trait
    pub trait GiveMeData {
        type Data;
        fn get_some_data(&self) -> Self::Data;
    }
    
    // 普通泛型 + Struct
    pub struct Something<T: Clone> {
        data: T,
    }
    
    // 一个 struct 可以有多个 GiveMeSomething<T> 的 impl
    impl<T: Clone> GiveMeSomething<u8> for Something<T> {
        fn get_something(&self) -> u8 {
            1
        }
    }
    
    impl<T: Clone> GiveMeSomething<i32> for Something<T> {
        fn get_something(&self) -> i32 {
            -1
        }
    }
    
    
    impl<T: Clone> Something<T> {
        // 普通泛型 + 函数
        pub fn get_data(&self) -> T {
            self.data.clone()
        }
    }
    
    // 一个 struct 只能有一个 GiveMeData 的 impl
    impl GiveMeData for Something<u8> {
        type Data = u8;
    
        fn get_some_data(&self) -> Self::Data {
            self.data
        }
    }
    ```

    

如果我们把泛型的类型（也就是`<T>`中的`T`）也作为输入的话，那么我们可以把`get_data()`的输入看成是`(T, &self)`，把它的输出看成是`(T, T)`，第一个`T`是类型，第二个`T`是指属于这个类型的值。

假如说我们有一个`Something<u8>`的示例，它存着`data = 1`，那么它的`get_data()`的输入就是`(u8, &self)`，输出就是`(u8, 1)`。

那么，很简单的，输入的类型就是 Input Type，输出的类型就是 Output Type，也就是，出现在参数的类型是 Input Type，出现在返回值的类型是 Output Type。



至于第三个问题，为什么带有 Associated Type 的 Trait 只能被一个类型`impl` 一次，我们需要一些实际的例子。典型的例子就是上面的`pub trait Iterator`。对于`Iterator`，很自然的，我们想要遍历某个实例的所有的值，那么这些值只有一个类型，所以对于一个类型来说，这个 Trait 只能被`impl`一次。



第四个问题和第三个问题紧密相关。实际上，如果我们能人为地保证带有普通泛型的 Trait 只被一个类型 `impl` 一次，那么我们完全可以用普通泛型代替 Associated Type。那么为什么我们还要发明一个 Associated Type 呢？



第一个理由是最简单的，你不能人为地保证带有普通泛型的 Trait 只被一个类型 `impl` 一次。万一一个实习生不懂，为了方便就随手加多了一个`impl`呢？假如说这个乌龙发生在`Iterator`这个 Trait 上面，而恰好你也在用着它（例如`for item in my_custom_list {}`），那么`rustc`会抱怨不知道用哪个`impl`，然后一个编译错误就产生了。如果你想`rustc`帮你检查并保证这个带有普通泛型的 Trait 只被一个类型 `impl` 一次，那么这就实际上变成了 Associated Type 了。

更多的理由我们需要举个例子。Rust RFC Book 里的[例子](https://rust-lang.github.io/rfcs/0195-associated-items.html)已经非常好了，所以我在这里简要翻译一下，并且补充一些内容。

假设我们需要建图，并且表达成一个 Trait `Graph`。如果仅仅使用普通泛型，我们可以写成

```rust
// N 和 E 是节点和边的类型
trait Graph<N, E> {
    fn has_edge(&self, &N, &N) -> bool;
    ...
}
```

但是如果这样写的话，一个类型可以有多个`Graph`的`impl`，并且每个`impl` 的 `N` 和 `E` 都不一样。这就有点诡异了，因为对于一个实际的图，它的节点和边的类型是唯一确定的。并且很自然地，`N`和`E`这两个类型就应该和`Graph`绑定在一起，有个从属的关系。

而且，假如说我们要写个函数，计算两节点间的距离，使用普通泛型，我们要这样写

```rust
fn distance<N, E, G: Graph<N, E>>(graph: &G, start: &N, end: &N) -> uint { ... }
```

前面泛型的部分就会有一长串，就是`<N, E, G<N, E>>`，如果一个 Trait 有好几个泛型，那么情况就更糟了，可能`<>`里的内容就要写一行。

而如果我们用 Associated Type，我们可以这样写

```rust
trait Graph {
    type N;
    type E;
    fn has_edge(&self, &N, &N) -> bool;
}
```

然后计算距离的函数就可以写成

```rust
fn distance<G: Graph>(graph: &G, start: &G::N, end: &G::N) -> uint { ... }
```

这样代码就变得更加简洁了。

另外，对比两个`distance`函数，第一个函数根本就没用到类型`E`，但是因为语法要求必须要写上，而第二个用`G::N`就行，可以不用写用不上的`G::E`。

除了少写用不上的代码这个好处之外，我们写代码的时候还有更多灵活性和可扩展性。假如说我们要拓展我们的`Graph`，变成

```rust
trait Graph{
    type N;
    type E;
    type A;
    type B;
    type C;
    fn has_edge(&self, &N, &N) -> bool;
}
```

如果使用普通泛型的话，第一个`distance`函数就不可避免地要加上`A, B, C`这些泛型标识，否则编译就会出问题，但是使用 Associated Type 的话，第二个`distance`函数根本就不用动。



综上，Rust 的 Associated Type 不是普通泛型的语法糖，而是经过深思熟虑的、能够解决实际问题并且优化代码的实现。

## 相关链接及参考链接

* Associated Type 的 [Rust RFC](https://rust-lang.github.io/rfcs/0195-associated-items.html)，里面有更加详细的解释，并且还列举了Associated Type 另外两个优点（虽然对应用开发者影响不大）
* Rust Forum 里的[相关讨论帖子](https://users.rust-lang.org/t/type-parameter-versus-type-alias-in-trait/67418)
* Rust Book 里的[相关章节](https://doc.rust-lang.org/book/ch19-03-advanced-traits.html)

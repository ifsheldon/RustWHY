# 为什么没有继承？

## 背景

继承常常用于面向对象的编程语言之中，通过继承，一个子类可以获得父类的部分（或全部）的属性、数据和方法。继承又可以分成单继承和多继承。例如，在 Java 中，子类只能有一个父类，所以叫单继承，而在 C++等语言中，一个子类可以有多个父类，所以叫多继承。

## 答案

为什么 Rust 没有继承？

太长不看版就是（从[Rust Book](https://doc.rust-lang.org/book/ch17-01-what-is-oo.html)摘录）：

* 继承有时候会共享过多东西（如用不上的属性、数据、方法等）。有时候子类并不需要父类的所有东西（人也是一样）。
* 继承有一些问题难以解决。
* Rust 不需要继承也可以达到类似继承的目的（代码共享、多态）。



长答案就是短答案的完全展开，但是与其问为什么不要 （“Why not?”），我们先来问一下自己，为什么要（“Why?”）。

所以，为什么要有继承？

有的人说，这是面向对象编程（OOP）的特性。但是，在 Erich Gamma, Richard Helm, Ralph Johnson 和 John Vlissides (AKA gang of four) 的书《Design Patterns: Elements of Reusable Object-Oriented Software》中，他们这样定义OOP：

>Object-oriented programs are made up of objects. An *object* packages both data and the procedures that operate on that data. The procedures are typically called *methods* or *operations*.

他们压根没提到继承。（如果有人质疑权威性的话，可以查查 Gang of Four）

引用 Rust Book 的观点，继承的目的只有两点：

1. 代码复用
2. 多态

其实多态在**一定程度**上也可以说是为了代码复用，所以终极的原因还是代码可复用。所以这也表现在继承的具体实现上了。通过继承，程序员可以不用写父类已经有的方法，也可以不用声明父类已经有的属性，这就让代码更加能被复用了。

那么看起来继承全是好处，所以程序员和继承 forever and forever 100 years!? 

这时候学过软件工程的同学应该会搬出一个词来反驳了，那就是“耦合”(Coupling)。“耦合”说的是组件之间的依赖程度，那么，如果把父类和子类看成是两个组件，那么他们之间的耦合应该是最高的，也就是依赖程度最高。而高耦合一般是认为不好的，就不展开讲了。举个不恰当的例子，小时候买东西都要爸妈出钱，而爸妈 diss 你到了最后总是会回到一句话“你吃我的穿我的还这么不听话“，而就算占理的是你，也只能认怂。这就是高耦合的坏处🐶

所以顺着高耦合的坏处，我们来到了继承机制的坏处，也就是为什么不（”Why not“）。

首先，第一个理由就是高耦合。引申出来的问题就是，有时候子类并不需要父类的所有东西，但是继承一股脑都把父类的东西塞给子类了。大家都知道，没有免费的午餐，所以继承的东西也不是越多越好。例如，继承方法，万一子类需要写一个同名方法，但是有着不同的输入输出怎么办？有人说可以用重载，但是复杂度又增加了。

第二个，继承方法还有一些难堪的情形不能解决。例如，经典的圆-椭圆问题（[Circle-Ellipse Problem](https://en.wikipedia.org/wiki/Circle–ellipse_problem)）。大家都知道，圆是椭圆，那么，按照继承的思路，圆就应该是椭圆的子类。同时我们知道，无论椭圆怎么在主轴上伸缩（除了零变换），它依旧是个椭圆。那么椭圆就应该有一个方法叫

`Ellipse::stretch(stretch_principal_direction0:f32, stretch_principal_direction1:f32)`

然而，如果圆也继承了这个方法，那么就会有圆伸缩了之后还是圆的奇怪情形。这在实践中微不足道，抛个 Exception 就完了，但是这是使用继承带来的理论上的根本问题之一。

第三个理由是，如果一个编程语言允许多继承，那么还会有[菱形继承问题](https://en.wikipedia.org/wiki/Multiple_inheritance#The_diamond_problem)，这里就不展开了。

最后一个理由是，当继承的层级过多，记忆、管理继承下来的属性和方法就非常麻烦，使得代码更难读懂，更难理解。或许有人也曾遇到过我这样的情况，接手或学习别人的代码的时候，看到别人在代码里用了一个属性或者方法，但是在当前类中找不到，最后发现是定义在了父类中（甚至父类的父类，甚至曾祖父类中）。而如果想要定义一个子类，写一个方法，没准就是在重复造轮子，别人早早就在你继承的父类（或者祖父类，曾祖父类）中写好了，只是用了一个不同的名字。

另外一个，值得反思的是，为什么继承就一定会把属性和方法都继承了？属性是一个类需要的数据，方法是一个类的行为，数据和行为可能有关也可能无关，为什么要捆绑一起作为继承大礼包？

所以，基于上面的理由，Rust 选择完全抛开继承的概念，**重新审视**如何更好地复用代码。

首先，对于代码复用，Rust Book 中提到可以使用`trait`，因为`trait`可以有默认实现，如果不想重新写代码，那么就使用默认实现就好了。另外`trait`还可以有Super Trait（介绍见 [Rust Book - super trait](https://doc.rust-lang.org/book/ch19-03-advanced-traits.html#using-supertraits-to-require-one-traits-functionality-within-another-trait)），就可以定义`trait`之间的层级。有人说，那 Super Trait 不就是像继承了吗？这个问题我们在附录中讨论。

对于多态，Rust Book 里提出可以使用泛型加 Trait Bound。

>To many people, polymorphism is synonymous with inheritance. But it’s actually a more general concept that refers to code that can work with data of multiple types. For inheritance, those types are generally subclasses.
>
>Rust instead uses generics to abstract over different possible types and trait bounds to impose constraints on what those types must provide. This is sometimes called *bounded parametric polymorphism*.

也就是多态可以由泛型加上 Trait 的限制来实现。

需要补充的是，Rust Book 说的多态更多的是静态多态，就是编译时可以实现的多态。而如果想要实现像 Java 一样的多态（例如，向一个 Vector 里添加有着共同父类的不同子类的对象），我们需要运行时多态。而在 Rust 中，”运行时多态“是由 Dynamic Dispatch 和 [Trait Object](https://doc.rust-lang.org/book/ch17-02-trait-objects.html#using-trait-objects-that-allow-for-values-of-different-types) 实现的。

这样做，Rust 降低了继承的高耦合。一个类型实现一个`trait`并不会把不需要的方法也继承下来，而因为`trait`不带属性，所以也不会继承属性。

而没有了继承，菱形继承的数据问题当然也不复存在了。而 Super Trait 带来的方法上的菱形继承问题我们也会在附录讨论。

对于圆-椭圆问题，在 Rust 中`struct Circle`可以包含一个`Ellipse`的属性，但是可以选择不把一些椭圆的方法暴露出来，这就解决了圆-椭圆问题。就像这样:

```rust
pub struct Ellipse{
    // zip
}
impl Ellipse{
    pub fn area(&self){
        // zip
    }
    pub fn stretch(&mut self){
        // zip
    }
}
pub struct Circle{
    e: Ellipse
}

impl Circle{
    pub fn area(&self){
        self.e.area()
    }
}
```

事实上，Rust 的编程建模模式更像是组合（Composition） 而不是继承（Inheritance）。继承使用的是隐式”复制代码“，而在这个例子中，程序员要显式地在`Circle::area`中调用`Ellipse::area`。虽然多余的代码写多了一点点，但是可以清晰地指出依赖层级。

这样，就算 Composition 的层级再多，看代码的时候总能追踪回原来的实现，就像是上面`Circle::area`到`Ellipse::area`一样。虽然层级过多时，难以管理代码的问题依旧存在，但是至少可以减缓一点。

综上，Rust 抛弃了继承，选择了结合`trait`、泛型和Trait Bound 的组合（Composition）模式，解决了大部分继承带来的问题，同时尽量不引入太多的新问题、新限制，提高灵活度的同时也尽量保持了代码的可复用性。



有人说，有道理，但是我就想要继承，我连上面`Circle::area`这样的委托函数（delegate）都不想写，那可以怎么办？参见附录中的`Deref` Anti-pattern。

## 相关链接

* Rust 社区中的讨论 [how to implement inheritance-like feature for rust](https://users.rust-lang.org/t/how-to-implement-inheritance-like-feature-for-rust/31159)
* Rust Book相关链接，请看正文中的链接

## 附录

### Super Trait 和继承有什么不同？

* Super Trait 还是属于 Trait，所以它不能定义属性，所以也不会”继承“属性。

* Super Trait 定义的是行为，也就是一个类型应该有的方法，由此形成的 Trait 的层级是行为上的层级。而继承不仅会继承方法，还会继承属性，引入了更多耦合，还会引入菱形继承中的数据问题。

* 虽然 super trait 同样会有菱形继承问题，但是不像数据的问题那样难以解决，方法上的冲突用[Fully qualified syntax](https://doc.rust-lang.org/book/ch19-03-advanced-traits.html#fully-qualified-syntax-for-disambiguation-calling-methods-with-the-same-name)就可以解决。例如:

    ```rust
    pub trait Say {
        fn say_something(&self) {
            println!("yummy")
        }
    }
    
    pub trait EatApple: Eat + Say {
        fn eat_something(&self) {
            println!("Eat something, maybe Apple?");
            self.say_something()
        }
        fn eat_apple(&self) {
            println!("Eat Apple")
        }
    }
    
    pub trait EatPear: Eat + Say {
        fn eat_something(&self) {
            println!("Eat something, maybe Pear?");
            self.say_something()
        }
        fn eat_pear(&self) {
            println!("Eat pear")
        }
    }
    
    struct People;
    
    impl Eat for People {}
    
    impl Say for People {}
    
    impl EatPear for People {}
    
    fn main() {
        let p = People;
        Eat::eat_something(&p); // OK
        p.eat_something() // compile error
    }
    ```

    

### `Deref` Anti-pattern

参考自 [Rust Design Patterns](https://rust-unofficial.github.io/patterns/anti_patterns/deref.html)。

太长不看版：

Rust 中`Deref`是这样定义的，一般用来实现智能指针，比如说`Box<T>`就可以实现`Deref`并且让`Target=T`。

```rust
pub trait Deref {
    type Target: ?Sized;
    fn deref(&self) -> &Self::Target;
}
```

但是既然可以随便指定`Target`，那么我们可以滥用这个来模拟继承，像这样（例子来自于参考）：

```rust
struct Foo {}

impl Foo {
    fn m(&self) {
        //..
    }
}

struct Bar {
    f: Foo,
}

impl Deref for Bar {
    type Target = Foo;
    fn deref(&self) -> &Foo {
        &self.f
    }
}

fn main() {
    let b = Bar { f: Foo {} };
    b.m();
}
```

那么`b.m()`就实际上是`b.deref().m()`，从而实际上实现了继承方法的作用。

但是这种用法属于滥用，**强烈建议不要使用**。在一个 `crate` 里，`impl` 和 `struct` 定义块常常可以分开，这通常问题不大，因为 IDE 可以帮忙搜索出一个类型的方法定义在哪个 `impl` 块里。然而，`Deref` 是一个微妙的语法糖，IDE 不一定能从 `b.m()` 帮忙找到 `Foo::m` ，而你可以把 `struct Foo` 和 `struct Bar` 的定义分开放并且把 `impl Deref for Bar` 代码块放在项目里十万八千里外的地方。相信没有人想遇上这样的状况，所以己所不欲勿施于人，Love and Peace❤️


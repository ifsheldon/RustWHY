# 为什么没有重载？

## 背景

我们在说重载（Overloading）的时候到底在说什么？重载指的是函数重载，就是多个函数签名（Function Signature）不一样的函数可以使用同一个函数名。这个与方法重写（Method Overriding）在概念上有本质的不同。至于函数签名是什么，各个语言里重载的要求是什么，就留给感兴趣的读者自行搜索。在这里我们讨论最简单的函数重载，也就是下面的形式：

```c++
int plus(int x, int y) {  // 1
  return x + y;
}

float plus(float x, float y) { // 2
  return x + y;
}

float plus(float x, float y, float z) { // 3
  return x + y;
}
```

对比 1 和 2，我们发现同名函数可以有不同的参数类型和返回值类型；对比 2 和 3，我们发现同名函数可以有不同的参数数量。

枚举的话，我们可以发现会用到重载的情形一共有 6 种。比如说 1 和 2 就属于情形 E，而 2 和 3 属于情形 B。

| 分类 | 参数类型 | 参数数量 | 返回值类型 |
| :--: | :------: | :------: | :--------: |
|  A   |   相同   |   相同   |    不同    |
|  B   |   相同   |   不同   |    相同    |
|  C   |   相同   |   不同   |    不同    |
|  D   |   不同   |   相同   |    相同    |
|  E   |   不同   |   相同   |    不同    |
|  F   |   不同   |   不同   |    相同    |



## 答案

### 太长不看版

* Rust 的设计哲学之一是 Be Explicit，翻译过来就是不要藏着掖着的，越直白越好。重载违背了这一条设计哲学，所以没有引入重载。
* 一些情况下，重载的功能可以使用泛型替代，而 Rust 的 Trait 和泛型系统让去掉泛型变得容易而且更符合直觉。
* 重载可以变相实现函数参数默认值，Rust 没有引入参数默认值的特性，所以也不会有重载。
* 重载不是百利无一害的，去掉重载可以让代码变得更加容易理解。
* 重载会让 Rust 的自动类型推断出现小问题，去掉重载可以规避这些问题。



### 长答案

如果读者是从 Java 或者 C++之类支持函数或者方法重载的语言迁移过来的，有可能对重载已经习以为常了，甚至觉得理所当然，以致于没有思考过为什么需要重载。

所以，让我们扪心自问，为什么需要重载？

一个最简单的答案是，因为方便啊！像上面的 `plus`，我想写一个接受两个整数的版本，也写一个接受两个浮点数的版本，如果不能重载的话，那我就要写一个`plus_int`和`plus_float`，我就要记两个函数名，即使它们在本质上做的是一样的事。但是按照奥卡姆剃刀准则，“方便”并不是必要，所以 Rust 设计者完全可以“任性”去掉这个特性。

如果剃刀这个理由还觉得差点意思，那么我们来想想 `plus` 到底想干嘛？`plus` 有两个（双参数）实现，一个接受整数，一个接受浮点数，那么浮点数和整数的共同点是什么？它们都是数（废话）！那`plus` 的意思非常直接，就是把两个数加在一起，所以我们在 Rust 里可以这样写：

```rust
use num::Num;

fn plus<T: Num>(a: T, b: T) -> T {
    a + b
}
```

使用泛型和 Trait Bound，我们完全可以精准表示出原来想要表达的意思，甚至还更加精确了。因为这里`plus`的意思是把两个**相同**类型的**数**相加，并且返回一个值，这个值的类型跟参数是**一样的**。如果是比较熟悉 Rust 类型的老油条应该还会注意到加号。加号在 Rust 里也是一个 Trait，通过实现这个 Trait 来重载加法运算符。

那么一个附加题就是，为什么编译器可以放行呢？也就是，我这里定义了`T: Num`，只是说明了`T` 是数字类型，但是好像没有告诉编译器`T`是可加的（实现了`Add`这个 Trait），但是为什么编译器知道 `T` 是可加的，并且加出来的结果是同一个类型的呢？

回归正题，举这个例子的意图是用来说明一个观点：

> Logically if we assume that function name is describing a behavior, there is no reason, that the same behavior may be performed on two completely independent, not similar types - which is what overloading actually is. If there is a situation, where operation may be performed on set of types, it‘s clear, that they have something in common, so this commonness should be extracted to trait, and then generalizing function over trait is straight way to go.  
>
> – Hashedone on Rust Forum

翻译过来就是：假如我们同意函数是用来描述一个行为的，那么（在重载的情况里）一个行为竟然可以作用于完全不相干的类型，这完全没有道理。那如果这个行为可以作用于一些类型上，那么就说明这些类型是有共通之处的。那在 Rust 里表达不同类型的共通之处的方法就是 Trait，那么合理的办法就是使用泛化函数（带泛型的函数），就像 Rust 的 `plus` 那样做。这样，我们就解决了表格中的 D 和 E 两种情况。

而至于情况 A，两个函数输入参数相同，输入参数数量相同，只有输出类型不同，那编译器很难推导返回值类型，就会出问题。而这于常理也解释不同，为什么同一个行为，接受同样的输入，会输出不同类型的结果呢？所以以下例子应该在大多数语言里都是行不通的。

```
function plus(x: float, y: float) -> float{...}
function plus(x: float, y: float) -> int{...}
```

而熟悉 Rust 的人应该会知道在 Rust 里也有情况 A 出现的，但是为什么编译器可以放行呢？这有两种情况，一种跟普通泛型相关，而另一种跟关联类型（Associated Type）有关（关联类型也是泛型的一种），例如：

```rust
let v = Vec::new(); // 泛型相关，现在调用的是 Vec<T> 的 new()，但是不知道 T 是什么类型
// let v = Vec::<i8>::new() // OK，编译通过，因为我们指定了使用 Vec<i8>的new
println!("{:?}", v);
```

```rust
let a = [1, 2];
let v = a.into_iter().collect(); // 关联类型相关
// let v = let v = a.into_iter().collect::<Vec<_>>(); // OK
println!("{:?}", v);
// 感兴趣的人可以按这个路径翻一下标注库 Iterator::collect -> FromIteration -> IntoIterator
```

这里不深入讨论，这两种情况都或多或少跟以下例子的问题同源

```rust
fn plus<T: Num>(a: T, b: T) -> T {
    a + b
}
fn main() {
    let f = plus; // f 的类型是 fn(?, ?) -> ? 编译器不知道? 到底是什么，所以 fn(?, ?) -> ? 这个类型也是不确定的，编译出错
}
```

简而言之就是纯函数的情况 A 是不允许的，因为于常理不通；而跟泛型相关的情况下，确实也会有情况 A 出现，但是这个时候跟重载没关系，那么这个时候我们可以通过特定语法，或者直接标注上变量类型来帮助类型推断器。那么情况 A 也解决了。没有重载的时候尚且会有这些小问题，如果加入重载的话那显然问题就更多了。

那么我们只剩下 B、C、F 这三种情况没有考虑。那么，我们什么时候会写有不同参数数量的同名函数呢？

一种可能是，我们想要实现默认值，用来省去复杂的配置。比如 Java 中的`InputStreamReader`的构造器，我们可以指定编码格式，或者使用默认的 UTF-8 格式。但是 Rust 不允许有默认值（为什么？请看 [为什么没有缺省参数](why_not_parameter_defaults.md)），所以基于不同参数数量的重载（B、C、F 三种情况）是被“不允许有默认值”这个设定一票否决了的。

不过还有另一种可能，我们想要可变参数数量的函数，例如 `print` 就是最常用的需求，有可能是情况 B 或者 F。情况 B 时，有同类型的输入但是不同数量，这在 Java中是用 Varargs 实现的。讲道理 Rust 也可以实现内置Varargs，比如说将多个参数放进一个数组或者 `Vec` 里，但是 It is what it is (maybe for now)。但是情况 F 是完全不可能的，因为这就代表着要将不同的类型的数据放进同一个容器中，这个是绝对不应该放进 Rust 标准库中的，但是 Rust 的 `println!`是怎么实现的呢？这好办，既然不允许函数重载，那不用函数就是了，`println!`就是一个宏，相当于是给不同的输入生成了不同的函数，这是属于元编程的暴力优雅。

到这，表格中的 6 种情况就讨论完了。其实除了情况 B 可以处理得更加方便之外，别的情况（A、D、E）可以使用泛型系统解决，不用引入重载，避免语言变得太繁杂，而其他情况，则是 Rust 的理念决定的，也就是越直白越好，从而就否决了函数重载，进而鼓励程序员写不同的函数名字来突出函数行为的区别。

## 引用

* [is-there-a-simple-way-to-overload-functions](https://users.rust-lang.org/t/is-there-a-simple-way-to-overload-functions/30937)


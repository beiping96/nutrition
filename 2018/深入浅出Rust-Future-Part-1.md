# 深入浅出Rust-Future-Part-1

>本文译自[Rust futures: an uneducated, short and hopefully not boring tutorial - Part 1](https://dev.to/mindflavor/rust-futures-an-uneducated-short-and-hopefully-not-boring-tutorial---part-1-3k3)，时间：2018-12-02，译者:
[motecshine](https://github.com/motecshine), 简介：motecshine

欢迎向Rust中文社区投稿,**[投稿地址](https://github.com/rustlang-cn/articles)**,好文将在以下地方直接展示

- 1 [Rust中文社区首页](https://rustlang-cn.org/)
- 2 Rust中文社区**Rust阅读文章栏目**
- 3 知乎专栏[Rust中文社区](https://zhuanlan.zhihu.com/rustlang-cn)
- 4 思否专栏[Rust中文社区](https://segmentfault.com/blog/rust-lang)
- 5 微博[Rustlang-cn](https://weibo.com/kriry)
- 6 简书专题[Rust中文社区](https://www.jianshu.com/c/2efae7198ea3)

## 介绍 

如果你是一个程序员并且也喜欢Rust这门语言, 那么你应该经常在社区听到讨论`Future` 这个库的声音, 一些很优秀的`Rust Crates`都使用了`Future` 所以我们也应该对它有足够的了解并且使用它. 但是大多数程序员很难理解`Future`到底是怎么工作的, 当然有官方 `Crichton's tutorial`这样的教程, 虽然很完善, 但我还是很难理解并把它付诸实践. 

我猜测我并不是唯一一个遇到这样问题的程序员, 所以我将分享我自己的最佳实践, 希望这能帮助你理解这个话题. 

## 一言以蔽之

 `Future` 是一个不会立即执行的特殊`functions`. 他会在将来执行(这也是他被命名为`future`的原因).我们有很多理由让`future functions`来替代`std functions`，例如: `优雅`，`性能`，`可组合性`.`future`的缺点也很明显: 很难用代码去实现. 当你不知道何时会执行某个函数时, 你又怎么能理解他们之间的因果关系呢？ 

处于这个原因， Rust会试图帮助我们这些菜鸟程序员去理解和使用`future`这个特性。

## Rust 的 future

Rust 的`futures` 总是一个`Results`: 这意味着你必须同时指定期待的返回值和备用的错误类型。 让我们先简单的实现一个方法，然后把它改造成`future`. 我们设计的这个方法返回值是 `u32` 或者是一个 被`Box`包围着的`Error trait`， 代码如下所示:

```rust
fn my_fn() -> Result<u32, Box<Error>> { 
    Ok(100) 
} 
``` 

这段代码很简单，看起来并没有涉及到`future`. 接下来让我们看看下面的代码: 

```rust
fn my_fut() -> impl Future<Item = u32, Error = Box<Error>> { 
    ok(100) 
} 
``` 

注意这两段代码不同的地方: 

1. 返回的类型不再是`Result`而是一个`impl Future`. `Rust Nightly`版本是允许我们返回一个`future`的。 

2. 第二个函数返回值的参量`Item = u32, Error = Box<Error>`较第一个函数来看更加详细明确。 

> 为了能让第二段代码工作 你需要使用拥有`conservative_impl_trait`特性的`nightly`版本。当然，如果不嫌麻烦,你可以使用`boxed trait`来替代。 

另请注意第一个函数返回值使用的是大写的`Ok(100)`。 在`Result`函数中，我们使用大写的`Ok`枚举，而`future`我们使用小写的ok方法. 

> 规则: 在Rust`future`中使用小写返回方法`ok(100)`. 

好了现在我们改造完毕了，但是我们该怎样执行第二个我们改造好的方法？标准方法我们可以直接调用，但是这里需要注意的是地一个方法返回值是一个`Result`, 所以我们需要使用`unwrap()`来获取我们期待的值。 

```rust
let retval = my_fn().unwrap(); 
println!("{:?}", retval); 
``` 

由于`future`在实际执行之前返回(或者更准确的说, 返回的是我们将来要执行的代码), 我们需要一种途径去执行`future`。为此我们使用`Reactor`。我们只需要创建一个`Reactor`并且调用他的`run`方法就可以执行`future`. 就像下面的代码： 

```rust
let mut reactor = Core::new().unwrap(); 
let retval = reactor.run(my_fut()).unwrap(); 
println!("{:?}", retval); 
``` 

注意这里我们`unwrap`的是`run`方法，而不是`my_fut`. 看起来真的很简单。 

## 链式调用 

`future`一个很重要的特性就是能够把其他的`future`组织起来形成一个调用链. 举个栗子: 

> 你通过email邀请你的父母一起吃晚饭. 
> 你在电脑前等待他们的回复 
> 父母同意与你一起吃晚饭(或者因为一些原因拒绝了)。 

链式调用就是这样的，让我们看一个简单的例子： 

```rust
fn my_fn_squared(i: u32) -> Result<u32, Box<Error>> { 
     Ok(i * i) 
} 

fn my_fut_squared(i: u32) -> impl Future<Item = u32, Error = Box<Error>> { 
    ok(i * i) 
} 
``` 

现在我们可以使用下面的方式去调用这两个函数： 

```rust 
let retval = my_fn().unwrap(); 
println!("{:?}", retval); 
let retval2 = my_fn_squared(retval).unwrap(); 
println!("{:?}", retval2); 
``` 

当然我们也可以模拟`Reactor`来执行相同的代码: 

```rust
let mut reactor = Core::new().unwrap(); 
let retval = reactor.run(my_fut()).unwrap(); 
println!("{:?}", retval); 
let retval2 = reactor.run(my_fut_squared(retval)).unwrap(); 
println!("{:?}", retval2); 
``` 

但还有更好的方法，在Rust中`future`也是一个`trait`他有很多种方法(这里我们会介绍些)，其中一个名为`and_then`的方法，在语义上完全符合我们最后写的代码片段。但是没有显式的执行`Reactor Run`, 让我们一起来看看下面的代码： 

```rust
let chained_future = my_fut().and_then(|retval| my_fut_squared(retval));
let retval2 = reactor.run(chained_future).unwrap(); 
println!("{:?}", retval2); 
``` 

让我们看看第一行:创建一个被叫做`chained_future`的`future`， 它把`my_fut`与`mu_fut_squared``future`串联了起来。 这里让人难以理解的部分是: 我们如何将上一个`future`的结果传递给下一个`future`? 
> 在Rust中我们可以通过闭包来捕获外部变量来传递`future`的值。 可以这样想： 

1. 调度并且执行`my_fut()` 
2. 当`my_fut()`执行完毕后，创建一个`retval`变量并且将`my_fut()`的返回值存到其中。 
3. 现在将`retval`作为`my_fn_squared(i: u32)`的参数传递进去，并且调度执行`my_fn_squared`。 
4. 把上面一些列的操作打包成一个名为`chained_future`的调用链。 

第二行代码,与之前的相同: 我们调用`Reactor run()`, 要求执行`chained_future`并给出结果。 当然我们可以通过这种方式将无数个`future`打包成一个调用链, 不要去担心性能问题, 因为future调用链是零成本的. 

> RUST 的借用检查可能让你的`future` 调用链写起来不是那么的轻松，所以你可以尝试`move`你的参数变量. 

## future 和普通函数一起使用

你也可以把普通的函数加入`future`调用链, 这很有用, 因为不是每个功能都需要使用`future`. 此外， 你也有可能希望调用你无法控制的外部函数。 如果函数没有返回Result，你只需在闭包中添加函数调用即可。 例如，如果我们有这个普通函数：

```rust
fn fn_plain(i: u32) -> u32 { 
    i - 50  
} 

let chained_future = my_fut().and_then(|retval| { 
    let retval2 = fn_plain(retval); 
    my_fut_squared(retval2) 
}); 
let retval3 = reactor.run(chained_future).unwrap(); 
println!("{:?}", retval3); 
``` 

如果你的函数返回`Result`则有更好的办法。我们一起来尝试将`my_fn_squared(i: u32) -> Result<u32, Box<Error>`方法打包进`future`调用链。 

在这里由于返回值是`Result`所以你无法调用`and_then`, 但是`future`有一个方法`done()`可以将`Result`转换为`impl Future`.这意味着我们可以将普通的函数通过`done`方法把它包装成一个`future`. 

```rust
let chained_future = my_fut().and_then(|retval| { 
    done(my_fn_squared(retval)).and_then(|retval2| my_fut_squared(retval2)) 
}); 
let retval3 = reactor.run(chained_future).unwrap(); 
println!("{:?}", retval3); 
``` 

注意第二：`done(my_fn_squared(retval))`允许我们在链式调用的原因是: 我们将普通函数通过`done`方法转换成一个`impl Future`. 现在我们不使用`done`方法试试: 

```rust
let chained_future = my_fut().and_then(|retval| {
    my_fn_squared(retval).and_then(|retval2| my_fut_squared(retval2)) 
}); 
let retval3 = reactor.run(chained_future).unwrap(); 
println!("{:?}", retval3); 
``` 

编译不通过! 

```rust
Compiling tst_fut2 v0.1.0 (file:///home/MINDFLAVOR/mindflavor/src/rust/tst_future_2) 
error[E0308]: mismatched types 
--> src/main.rs:136:50 | 136 | my_fn_squared(retval).and_then(|retval2| my_fut_squared(retval2)) | ^^^^^^^^^^^^^^^^^^^^^^^ expected enum `std::result::Result`, found anonymized type | = note: expected type `std::result::Result<_, std::boxed::Box<std::error::Error>>` found type `impl futures::Future` 
error: aborting due to previous error 
error: Could not compile `tst_fut2`. 
``` 

`expected type std::result::Result<_, std::boxed::Box<std::error::Error>> found type impl futures::Future`,这个错误有点让人困惑. 我们将会在第二部分讨论它。 

## 泛型 

最后但并非最不重要的， `future` 与泛型一起工作不需要任何黑魔法. 

```rust
fn fut_generic_own<A>(a1: A, a2: A) -> impl Future<Item = A, Error = Box<Error>> where A: std::cmp::PartialOrd, { 
    if a1 < a2 { ok(a1) } else { ok(a2) } 
} 
``` 

这个函数返回的是 a1 与 a2之间的较小的值。但是即便我们很确定这个函数没有错误也需要给出`Error`，此外，返回值在这种情况下是小写的`ok`(因为它是一个函数， 而不是枚举) 

现在我们调用这个`future`: 

```rust
let future = fut_generic_own("Sampdoria", "Juventus"); 
let retval = reactor.run(future).unwrap(); 
println!("fut_generic_own == {}", retval); 
``` 

阅读到现在你可能对`future`应该有所了解了， 在这边文章里你可能注意到我没有使用`&`, 并且仅使用函数自身的值。这是因为使用`impl Future`，生命周期的行为并不相同，我将在下一篇文章中解释如何使用它们。在下一篇文章中我们也会讨论如何在`future`链处理错误和使用await!()宏。


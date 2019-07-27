# 深入浅出Rust-Future-Part-2

>译自[Rust futures: an uneducated, short and hopefully not boring tutorial - Part 2](https://dev.to/mindflavor/rust-futures-an-uneducated-short-and-hopefully-not-boring-tutorial---part-2-8dd)时间：2018-12-03，译者:
[motecshine](https://github.com/motecshine), 简介：motecshine

欢迎向Rust中文社区投稿,**[投稿地址](https://github.com/rustlang-cn/articles)**,好文将在以下地方直接展示

- 1 [Rust中文社区首页](https://rustlang-cn.org/)
- 2 Rust中文社区**Rust阅读文章栏目**
- 3 知乎专栏[Rust中文社区](https://zhuanlan.zhihu.com/rustlang-cn)
- 4 思否专栏[Rust中文社区](https://segmentfault.com/blog/rust-lang)
- 5 微博[Rustlang-cn](https://weibo.com/kriry)
- 6 简书专题[Rust中文社区](https://www.jianshu.com/c/2efae7198ea3)

## Intro

在这个系列的第一篇文章我们了解了如何使用`Rust Future`.但是只有我们彻底的了解`Future`并且操作得当才能发挥它真正的作用。这个系列的第二篇文章，我们将介绍如何避免`Future`里常见的陷阱。

## Error troubles

我们将`Future`组织成一个`链`很简单，只要通过`Rust Future`提供的`and_then`函数就可以了。但是在上一篇文章中我们使用了`Box<Error> trait`作为错误类型，绕过了编译器的检查。为什么我们没有使用更为详细的错误类型？原因很简单， 每个`Future`函数的错误返回都有可能不同.

> 原则1: 当我们将不同的`Future`组织成一个调用`链`时，每个`Future`都应该返回相同的`Error Type`.

让我们一起来证明一下这一点.

我们有两个被叫做`ErrorA`和`ErrorB`的`Error`类型,  我们将会实现`error::Error trait`,尽管这并不是编译器必须让我们做的(但是这是一个好习惯[在我看来这应该算是一个最佳实践]),在我们实现`error::Error trait`的同时还需要实现`std::fmt::Display`,现在就让我们一起实现他吧!

```rust
#[derive(Debug, Default)]
pub struct ErrorA {}

impl fmt::Display for ErrorA {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "ErrorA!")
    }
}

impl error::Error for ErrorA {
    fn description(&self) -> &str {
        "Description for ErrorA"
    }

    fn cause(&self) -> Option<&error::Error> {
        None
    }
}

// Error B
#[derive(Debug, Default)]
pub struct ErrorB {}

impl fmt::Display for ErrorB {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "ErrorB!")
    }
}

impl error::Error for ErrorB {
    fn description(&self) -> &str {
        "Description for ErrorB"
    }

    fn cause(&self) -> Option<&error::Error> {
        None
    }
}
```

我尽量用简单的方式去实现`Error Trait`,这样可以排除别的干扰来证明我的观点. 现在让我们在`Future`中使用`ErrorA`与`ErrorB`.

```rust
fn fut_error_a() -> impl Future<Item = (), Error = ErrorA> {
    err(ErrorA {})
}

fn fut_error_b() -> impl Future<Item = (), Error = ErrorB> {
    err(ErrorB {})
}
```

现在让我们在`main`函数里调用它.

```rust
let retval = reactor.run(fut_error_a()).unwrap_err();
println!("fut_error_a == {:?}", retval);

let retval = reactor.run(fut_error_b()).unwrap_err();
println!("fut_error_b == {:?}", retval);
```

跟我们所预见的结果一致:

```RUST
fut_error_a == ErrorA
fut_error_b == ErrorB
```

到现在为止还挺好的，让我们把`ErrorA`与`ErrorB`打包成一个调用链:

```rust
let future = fut_error_a().and_then(|_| fut_error_b());
```

我们先调用`fut_error_a`然后再调用`fut_error_b`,我们不用关心`fut_error_a`的返回值所以我们用`_`省略不用. 用更复杂的术语解释就是: 我们将`impl Future<Item=(), Error=ErrorA>` 和 `impl Future<Item=(), Error=ErrorB>`打包成调用链.

现在让我们尝试编译这段代码:

```rust

Compiling tst_fut2 v0.1.0 (file:///home/MINDFLAVOR/mindflavor/src/rust/tst_future_2)
error[E0271]: type mismatch resolving `<impl futures::Future as futures::IntoFuture>::Error == errors::ErrorA`
   --> src/main.rs:166:32
    |
166 |     let future = fut_error_a().and_then(|_| fut_error_b());
    |                                ^^^^^^^^ expected struct `errors::ErrorB`, found struct `errors::ErrorA`
    |
    = note: expected type `errors::ErrorB`
               found type `errors::ErrorA`
```

这个报错非常明显, 编译器期待我们使用`ErrorB`但是我们给了一个`ErrorA`。

>原则2: 当我们组织`Future Chain`时，第一个错误类型必须与最后一个`future`返回的错误类型一致.(When chaining futures, the first function error type must be the same as the chained one.)

`rustc`已经非常明确的告诉我们了. 这个`Future chain`最终返回的是`ErrorB`所以我们第一个函数也应该返回`ErrorB`. 在上述代码我们返回了`ErrorA`, 所以导致编译失败.

我们改如何处理这个问题?非常幸运的是, 我们可以使用`Rust Future`给我们提供的`map_err`方法. 在我们的示例中，我们想要把`ErrorA`转换成`ErrorB`,所以我们只需要在`ErrorA`与`ErrorB`之间调用这个函数就行了.

```rust
let future = fut_error_a()
    .map_err(|e| {
        println!("mapping {:?} into ErrorB", e);
        ErrorB::default()
    })
    .and_then(|_| fut_error_b());

let retval = reactor.run(future).unwrap_err();
println!("error chain == {:?}", retval);
```

如果我们现在编译并运行示例，将会输出：

```rust
mapping ErrorA into ErrorB
error chain == ErrorB
```

让我们进一步推动这个例子.假设我们想连接ErrorA，然后是ErrorB，然后再连接ErrorA。 就像是：

```rust
let future = fut_error_a()
  .and_then(|_| fut_error_b())
    .and_then(|_| fut_error_a());
```

我们最初的解决方式只适合成对的`future`， 并没有考虑其他的情况。所以在上面代码中我们不得不这么做:
`ErrorA => ErrorB => ErrorA`.就像这样:

```rust
let future = fut_error_a()
    .map_err(|_| ErrorB::default())
    .and_then(|_| fut_error_b())
    .map_err(|_| ErrorA::default())
    .and_then(|_| fut_error_a());
```

看上去不那么优雅但是还是解决了多个`Future`的错误处理.

## "From" to the rescue

简化上述代码的一种简单的方式就是利用`std::covert::From`. 当我们实现`From`, 这样编译器就可以自动的将一个结构软换为另一个结构.现在让我们实现`From<ErrorA> for ErrorB`和`From<ErrorB> for ErrorA`.

```rust
impl From<ErrorB> for ErrorA {
    fn from(e: ErrorB) -> ErrorA {
        ErrorA::default()
    }
}

impl From<ErrorA> for ErrorB {
    fn from(e: ErrorA) -> ErrorB {
        ErrorB::default()
    }
}
```

通过上述的实现我们只需要用`from_err`函数来代替`map_err`就好了。

```rust
let future = fut_error_a()
   .from_err()
   .and_then(|_| fut_error_b())
   .from_err()
   .and_then(|_| fut_error_a());
```

现在的代码仍然与错误转换混合, 但转换代码不再是内联的，而且代码可读性也提高了。`Futrue Crate`非常聪明:只有在错误的情况下才会调用`from_err`代码， 因此在不使用`from_err`时, 也不会在`Runtime`时产生额外的开销.

##Lifetimes

Rust签名功能是引用的显式生命周期注释. 但是，大多数情况下，Rust允许我们避免使用生命周期省略来指定生命周期.让我们看看它的实际效果. 我们想编写一个带字符串引用的函数，如果成功则返回相同的字符串引用：

```rust
fn my_fn_ref<'a>(s: &'a str) -> Result<&'a str, Box<Error>> {
    Ok(s)
}
```

注意代码中 `<'a>` 的部分, 意思是我们显示的声明一个`生命周期`. 接着我们声明了一个引用形参`s: &'a str`, 这个参数必须在`'a`生命周期有效的情况下使用.使用`Result <＆'str，Box <Error >>`，我们告诉`Rust`我们的返回值将包含一个字符串引用.只要'a有效，该字符串引用必须有效.换句话说，传递的字符串引用和返回的对象必须具有相同的生命周期.这会导致我们的语法非常冗长，以至于Rust允许我们避免在常见情况下指定生命周期。 所以我们可以这样重写函数：

```rust
fn my_fn_ref(s: &str) -> Result<&str, Box<Error>> {
    Ok(s)
}
```

但是在`Future`中你不能这样写， 让我们来尝试用`Future`方式复写这个函数:

```rust
fn my_fut_ref_implicit(s: &str) -> impl Future<Item = &str, Error = Box<Error>> {
    ok(s)
}
```

编译将会失败(`rustc 1.23.0-nightly (2be4cc040 2017-11-01`)：

```rust
Compiling tst_fut2 v0.1.0 (file:///home/MINDFLAVOR/mindflavor/src/rust/tst_future_2)
error: internal compiler error: /checkout/src/librustc_typeck/check/mod.rs:633: escaping regions in predicate Obligation(predicate=Binder(ProjectionPredicate(ProjectionTy { substs: Slice([_]), item_def_id: DefId { krate: CrateNum(15), index: DefIndex(0:330) => futures[59aa]::future[0]::Future[0]::Item[0] } }, &str)),depth=0)
  --> src/main.rs:39:36
   |
39 | fn my_fut_ref_implicit(s: &str) -> impl Future<Item = &str, Error = Box<Error>> {
   |                                    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

note: the compiler unexpectedly panicked. this is a bug.

note: we would appreciate a bug report: https://github.com/rust-lang/rust/blob/master/CONTRIBUTING.md#bug-reports

note: rustc 1.23.0-nightly (2be4cc040 2017-11-01) running on x86_64-unknown-linux-gnu

thread 'rustc' panicked at 'Box<Any>', /checkout/src/librustc_errors/lib.rs:450:8
note: Run with `RUST_BACKTRACE=1` for a backtrace.
```

当然也有解决方式，我们只要显示声明一个有效的生命周期就行了:

```rust
fn my_fut_ref<'a>(s: &'a str) -> impl Future<Item = &'a str, Error = Box<Error>> {
    ok(s)
}
```

## impl Future with lifetimes

在`Future`中如果有引用传参我们必须要显示的注释生命周期. 举个例子, 我们希望使用`&s`的值并且返回的是一个没有引用的`String`.我们必须显示的注释生命周期:

```rust
fn my_fut_ref_chained<'a>(s: &'a str) -> impl Future<Item = String, Error = Box<Error>> {
    my_fut_ref(s).and_then(|s| ok(format!("received == {}", s)))
}
```

上面的代码将会报错:

```rust
error[E0564]: only named lifetimes are allowed in `impl Trait`, but `` was found in the type `futures::AndThen<impl futures::Future, futures::FutureResult<std::string::String, std::boxed::Box<std::error::Error + 'static>>, [closure@src/main.rs:44:28: 44:64]>`
  --> src/main.rs:43:42
   |
43 | fn my_fut_ref_chained<'a>(s: &'a str) -> impl Future<Item = String, Error = Box<Error>> {
   |                                          ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
```

为了解决这个错误我们必须为`impl Future`追加一个`'a`生命周期：

```rust
fn my_fut_ref_chained<'a>(s: &'a str) -> impl Future<Item = String, Error = Box<Error>> + 'a {
    my_fut_ref(s).and_then(|s| ok(format!("received == {}", s)))
}
```

现在你可以运行这段代码了:

```rust
let retval = reactor
    .run(my_fut_ref_chained("str with lifetime"))
    .unwrap();
println!("my_fut_ref_chained == {}", retval);
```

## Closing remarks

在下一篇文章中，我们将介绍`Reactor`。 我们还将从头开始编写未来的实现结构。

# 深入浅出Rust-Future-Part-3

>译自[Rust futures: an uneducated, short and hopefully not boring tutorial - Part 3 - The reactor](https://dev.to/mindflavor/rust-futures-an-uneducated-short-and-hopefully-not-boring-tutorial---part-3---the-reactor-433)
>本文时间：2018-12-03，译者:
[motecshine](https://github.com/motecshine), 简介：motecshine

欢迎向Rust中文社区投稿,**[投稿地址](https://github.com/rustlang-cn/articles)**,好文将在以下地方直接展示

- 1 [Rust中文社区首页](https://rustlang-cn.org/)
- 2 Rust中文社区**Rust阅读文章栏目**
- 3 知乎专栏[Rust中文社区](https://zhuanlan.zhihu.com/rustlang-cn)
- 4 思否专栏[Rust中文社区](https://segmentfault.com/blog/rust-lang)
- 5 微博[Rustlang-cn](https://weibo.com/kriry)
- 6 简书专题[Rust中文社区](https://www.jianshu.com/c/2efae7198ea3)

## Intro

在这篇文章中我们将会讨论和阐释`Reactor`是如何工作的.在上篇文章中我们，我们频繁的使用`Reactor`来执行我们的`Future`，但是并没有阐述它是如何工作的。现在是时候阐明它了。

## Reactor? Loop?

如果用一句话来描述`Reactor`，那应该是:
> `Reactor`是一个环(`Loop`)

举个栗子：
你决定通过Email邀请你喜欢的女孩或者男孩(emmm, 这个栗子听起来很老套), 你怀着忐忑的心将这份邮件发送出去，心里焦急着等待着， 不停的一遍又一遍的检查你的邮箱是否有新的回复. 直到收到回复。
`Rust's Reactor`就是这样， 你给他一个`future`, 他会不断的检查，直到这个`future`完成(或者返回错误). `Reactor`通过调用程序员实现的`Poll`函数，来检查`Future`是否已完成。你所要做的就是实现`future poll` 并且返回`Poll<T, E>`结构。但是 `Reactor`也不会无休止的对你的`future function`轮询。

## A future from scratch

为了让我们能更容易理解`Reactor`知识，我们还是从零开始实现一个`Future`. 换句话说就是，我们将动手实现`Future Trait`.

```rust
#[derive(Debug)]
struct WaitForIt {
    message: String,
    until: DateTime<Utc>,
    polls: u64,
}
```

我们的结构体字段也很简单:

- message： 自定义字符串消息体
- polls: 轮循次数
- util: 等待时间

我们还会实现 `WaitFotIt`结构体的`new`方法.这个方法作用是初始化`WaitForIt`

```rust
impl WaitForIt {
    pub fn new(message: String, delay: Duration) -> WaitForIt {
        WaitForIt {
            polls: 0,
            message: message,
            until: Utc::now() + delay,
        }
    }
}

impl Future for WaitForIt {
    type Item = String;
    type Error = Box<Error>;

    fn poll(&mut self) -> Poll<Self::Item, Self::Error> {
        let now = Utc::now();
        if self.until < now {
            Ok(Async::Ready(
                format!("{} after {} polls!", self.message, self.polls),
            ))
        } else {
            self.polls += 1;

            println!("not ready yet --> {:?}", self);
            Ok(Async::NotReady)
        }
    }
}
```

让我们逐步解释

```rust
    type Item = String;
    type Error = Box<Error>;
```

上面两行在`RUST`里被叫做`associated types`, 意思就是`Future`在将来完成时返回的值(或者错误).

```rust
    fn poll(&mut self) -> Poll<Self::Item, Self::Error> {}
```

定义轮询的方法。`Self::Item, Self::Error` 是我们定义的`associated types`站位符。在我们的例子中，该方法如下：

```rust
fn poll（＆mut self） - > Poll <String，Box <Error >>
```

现在看看我们的逻辑代码:

```rust
let now = Utc::now();
if self.until < now {
// 告诉reactor `Future` 已经完成了！
} else {
// 告诉 reactor `Future` 还没准备好，过会儿再来。
}
```

在`Rust`里我们该怎样告诉`Reactor`某个`Future`已经完成了？很简单使用枚举

```rust
Ok(Async::NotReady(.......)) // 还没完成
Ok(Async::Ready(......)) // 完成了
```

让我们来实现上述的方法：

```rust
impl Future for WaitForIt {
    type Item = String;
    type Error = Box<Error>;

    fn poll(&mut self) -> Poll<Self::Item, Self::Error> {
        let now = Utc::now();
        if self.until < now {
            Ok(Async::Ready(
                format!("{} after {} polls!", self.message, self.polls),
            ))
        } else {
            self.polls += 1;

            println!("not ready yet --> {:?}", self);
            Ok(Async::NotReady)
        }
    }
}
```

为了让这段代码运行起来我们还需要：

```Rust
extern crate chrono;
extern crate futures;

extern crate tokio_core;

use futures::done;
use futures::prelude::*;
use futures::future::{err, ok};
use tokio_core::reactor::Core;
use std::error::Error;
use futures::prelude::*;
use futures::*;
use chrono::prelude::*;
use chrono::*;

fn main() {
    let mut reactor = Core::new().unwrap();

    let wfi_1 = WaitForIt::new("I'm done:".to_owned(), Duration::seconds(1));
    println!("wfi_1 == {:?}", wfi_1);

    let ret = reactor.run(wfi_1).unwrap();
    println!("ret == {:?}", ret);
}
```

运行!! 等待一秒我们将会看到结果：

```rust
Running `target/debug/tst_fut_create`
wfi_1 == WaitForIt { message: "I\'m done:", until: 2017-11-07T16:07:06.382232234Z, polls: 0 }
not ready yet --> WaitForIt { message: "I\'m done:", until: 2017-11-07T16:07:06.382232234Z, polls: 1 }
```

emmm~, 只运行一次就被卡住了, 但是没有额外的消耗`CPU`.但是为什么会这样？
> 如果不明确告诉`Reactor`， `Reactor`是不会再次轮询停放(park)给它的`Future`.

(- 译注： Park: 翻译成停放其实也挺好的，就像车场的停车位一样.)

在我们的例子里， `Reactor`会立即执行我们停放的`Future`方法， 当我们返回`Async::NotReady`, 它就会认为当前停放的`Future`还未完成。如果我们不主动去解除停放，`Reactor`永远也不会再次调用。

空闲中的`Reactor`是不会消耗CPU的。这样看起来`Reactor`效率还是很高的。
在我们的电子邮件示例中，我们可以避免手动检查邮件并等待通知。 所以我们可以在此期间自由玩Doom。(emm～看来作者很喜欢这款游戏).

另一个更有意义的示例可能是从网络接收数据。 我们可以阻止我们的线程等待网络数据包，或者我们等待时可以做其他事情。 您可能想知道为什么这种方法比使用OS线程更好？

## Unparking

我们该如何纠正我们例子？我们需要以某种方式取消我们的`Future`。 理想情况下，我们应该有一些外部事件来取消我们的`Future`（例如按键或网络数据包),但是对于我们的示例，我们将使用这个简单的行手动取消停放

```rust
futures::task::current().notify();
```

像这样:

```RUST
impl Future for WaitForIt {
    type Item = String;
    type Error = Box<Error>;

    fn poll(&mut self) -> Poll<Self::Item, Self::Error> {
        let now = Utc::now();
        if self.until < now {
            Ok(Async::Ready(
                format!("{} after {} polls!", self.message, self.polls),
            ))
        } else {
            self.polls += 1;

            println!("not ready yet --> {:?}", self);
            futures::task::current().notify();
            Ok(Async::NotReady)
        }
    }
}
```

现在代码完成了。 请注意，在我的情况下，该函数已被调用超过50k次， CPU占用也很高！
这是严重的浪费，也清楚地说明你为什么需要在某个合理的时间点去`Unpark Future`.( That's a waste of resources and clearly demonstrates why you should unpark your future only when something happened. )

另请注意循环如何仅消耗单个线程。 这是设计和效率的来源之一。 当然，如果需要，您可以使用更多线程。


## Joining

`Reactor`可以同时运行多个`Future`，这也是他为什么如此有https://github.com/rustlang-cn/forum/edit/master/translate/futures/深入浅出Rust-Future-Part-3.md效率的原因. 那么我们该如何充分利用单线程: 当一个`Future`被停放的时候, 另一个可以继续工作。

对于这个例子，我们将重用我们的WaitForIt结构。 我们只是同时调用两次。 我们开始创建两个`Future`的实例：

```rust
let wfi_1 = WaitForIt::new("I'm done:".to_owned(), Duration::seconds(1));
println!("wfi_1 == {:?}", wfi_1);
let wfi_2 = WaitForIt::new("I'm done too:".to_owned(), Duration::seconds(1));
println!("wfi_2 == {:?}", wfi_2);
```

现在我们来调用`futures::future::join_all`， 他需要一个`vec![]`迭代器， 并且返回枚举过的`Future`

```rust
let v = vec![wfi_1, wfi_2];
let sel = join_all(v);
```

我们重新实现的代码像这样:

```rust
fn main() {
    let mut reactor = Core::new().unwrap();

    let wfi_1 = WaitForIt::new("I'm done:".to_owned(), Duration::seconds(1));
    println!("wfi_1 == {:?}", wfi_1);
    let wfi_2 = WaitForIt::new("I'm done too:".to_owned(), Duration::seconds(1));
    println!("wfi_2 == {:?}", wfi_2);

    let v = vec![wfi_1, wfi_2];

    let sel = join_all(v);

    let ret = reactor.run(sel).unwrap();
    println!("ret == {:?}", ret);
}
```

这里的关键点是两个请求是交错的：第一个`Future`被调用，然后是第二个，然后是第一个，依此类推，直到两个完成。 如上图所示，第一个`Future`在第二个之前完成。 第二个在完成之前被调用两次。


## Select

`Future`的特性还有很多功能。 这里值得探讨的另一件事是select函数。 select函数运行两个（或者在select_all的情况下更多）`Future`，并返回第一个完成。 这对于实现超时很有用。 我们的例子可以简单：

```rust
fn main() {
    let mut reactor = Core::new().unwrap();

    let wfi_1 = WaitForIt::new("I'm done:".to_owned(), Duration::seconds(1));
    println!("wfi_1 == {:?}", wfi_1);
    let wfi_2 = WaitForIt::new("I'm done too:".to_owned(), Duration::seconds(2));
    println!("wfi_2 == {:?}", wfi_2);

    let v = vec![wfi_1, wfi_2];

    let sel = select_all(v);

    let ret = reactor.run(sel).unwrap();
    println!("ret == {:?}", ret);
}
```

## Closing remarks

下篇将会创建一个更`Real`的`Future`.

## 可运行的代码

```rust
extern crate chrono;
extern crate futures;

extern crate tokio_core;

use futures::done;
use futures::prelude::*;
use futures::future::{err, ok};
use tokio_core::reactor::Core;
use std::error::Error;
use futures::prelude::*;
use futures::*;
use chrono::prelude::*;
use chrono::*;
use futures::future::join_all;
#[derive(Debug)]
struct WaitForIt {
    message: String,
    until: DateTime<Utc>,
    polls: u64,
}

impl WaitForIt {
    pub fn new(message: String, delay: Duration) -> WaitForIt {
        WaitForIt {
            polls: 0,
            message: message,
            until: Utc::now() + delay,
        }
    }
}

iml Future for WaitForIt {
    type Item = String;
    type Error = Box<Error>;

    fn poll(&mut self) -> Poll<Self::Item, Self::Error> {
        let now = Utc::now();
        if self.until < now {
            Ok(Async::Ready(
                format!("{} after {} polls!", self.message, self.polls),
            ))
        } else {
            self.polls += 1;

            println!("not ready yet --> {:?}", self);
            futures::task::current().notify();
            Ok(Async::NotReady)
        }
    }
}

fn main() {
    let mut reactor = Core::new().unwrap();

    let wfi_1 = WaitForIt::new("I'm done:".to_owned(), Duration::seconds(1));
    println!("wfi_1 == {:?}", wfi_1);
    let wfi_2 = WaitForIt::new("I'm done too:".to_owned(), Duration::seconds(1));
    println!("wfi_2 == {:?}", wfi_2);

    let v = vec![wfi_1, wfi_2];

    let sel = join_all(v);

    let ret = reactor.run(sel).unwrap();
    println!("ret == {:?}", ret);
}
```

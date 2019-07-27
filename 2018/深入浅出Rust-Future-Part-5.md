# 深入浅出Rust-Future-Part-5

>[原文出处](https://dev.to/mindflavor/rust-futures-an-uneducated-short-and-hopefully-not-boring-tutorial---part-5---streams-5i8)
>本文时间：2018-12-09，译者:
[motecshine](https://github.com/motecshine), 简介：motecshine

欢迎向Rust中文社区投稿,[投稿地址](https://github.com/rustlang-cn/articles) ,好文将在以下地方直接展示

1. [Rust中文社区首页](https://rustlang-cn.org)

2. Rust中文社区阅读Rust文章栏目

3. 知乎专栏[Rust中文社区](https://zhuanlan.zhihu.com/rustlang-cn)

4. 思否专栏[Rust中文社区](https://segmentfault.com/blog/rust-lang)

5. 简书专题[Rust中文社区](https://www.jianshu.com/c/2efae7198ea3)

6. 微博[Rustlang-cn](https://weibo.com/kriry)

## Intro

在上篇文章中我们学习了如何实现一个高效率的`Future`(尽量不阻塞, 只有在需要时才会`Unpark`我们的`Task`).  今天继续扩展我们的`Future`: 实现一个`Stream Trait`.
`Stream`跟`Iterators`看起来很像: 他们随着时间的推移产生多个相同类型的输出, 与`Iterators`唯一的区别就是消费的方式不同. 让我们一起尝试使用`Reactor`来处理`Streams`吧.

## ForEach combinator

我们使用一个名为`for_each`的组合器, 来代替我们手动迭代消费`Stream`. 查询文档不难发现`future::stream`实现了`ForEach`, 所以我们不仅可以迭代, 也可以把`stream`放入`Reactor`, 把它作为`Future Chain`的一部分. 这看起来简直太酷了.现在让我们一步一步来实现一个简单的`Stream`.

## impl Stream

`Stream Trait` 与 `Future Trait`很像:

```RUST
pub trait Future {
    type Item;
    type Error;
    fn poll(&mut self) -> Poll<Self::Item, Self::Error>;

    // <-- CUT -->
}

pub trait Stream {
    type Item;
    type Error;
    fn poll(&mut self) -> Poll<Option<Self::Item>, Self::Error>;

    // <-- CUT -->
}
```

这两个`Trait`都有很多的函数, 由于这些函数都有默认值, 因此如果你不需要它, 就无需实现他们. 在本篇文章里我们只关注`poll`这个方法.

```RUST
    // Future
    fn poll(&mut self) -> Poll<Self::Item, Self::Error>;
    // Stream
    
    fn poll(&mut self) -> Poll<Option<Self::Item>, Self::Error>;
```

对比下`Future`与`Stream`两者`poll`函数的区别:

| Situation        | Future      | Stream  |
|---|---|---|
|Item to return ready| Ok(Async::Ready(t))|Ok(Async::Ready(Some(t)))|
|Item to return not ready|Ok(Async::NotReady)|Ok(Async::NotReady)|
|No more items to return|N.A.|Ok(Async::Ready(None))|
|Error|Err(e)|Err(e)|

## Simple stream

让我们一起实现一个简单的`stream`:

```RUST
struct MyStream {
    current: u32,
    max: u32,
}



impl MyStream {
    pub fn new(max: u32) -> MyStream {
        MyStream {
            current: 0,
            max: max,
        }
    }
}

impl Stream for MyStream {
    type Item = u32;
    type Error = Box<Error>;

    fn poll(&mut self) -> Poll<Option<Self::Item>, Self::Error> {
        match self.current {
            ref mut x if *x < self.max => {
                *x = *x + 1;
                Ok(Async::Ready(Some(*x)))
            }
            _ => Ok(Async::Ready(None)),
        }
    }
}
```

我们重点关注下`poll`函数, 形参传递了一个可变引用, 所以我们可以改变`MyStream`内部的值. 这段代码理解起来很容易:
> 检查`MyStream.current`是否大于 `MyStream.max` 如果大于: 返回`Ok(Async::Ready(None))`, 否则`MyStream.current`自增`1`并且返回当前的值.

## Consume a stream

```RUST
let mut reactor = Core::new().unwrap();
let my_stream = MyStream::new(5);

let fut = my_stream.for_each(|num| {
    println!("num === {}", num);
    ok(())
});
```

注意`ok(())`, 这段代码意味着我们返回的是个`Future`, 所以我们不仅可以使用`Reactor`执行`fut`, 也可以跟别的`Future`, 组合成`Future Chain`.

## Spawn futures during the event loop

我们在处理`Stream`时, 有时候想创建(spawn:派生)新的`Future`, 这样做理由有很多, 比如不想阻塞当前的`Future Task`, `Rust` 是允许我们使用`Reactor`的`execute`函数将创建的`Future`加入现有的事件循环中的. 然而这有一个陷阱: `execute` 返回的是`Result<(), ExecuteError<f>>`, 可以看出这个函数正常返回时,没有任何的值.

```RUST
impl Stream for MyStream {
    type Item = u32;
    type Error = Box<Error>;

    fn poll(&mut self) -> Poll<Option<Self::Item>, Self::Error> {
        use futures::future::Executor;

        match self.current {
            ref mut x if *x < self.max => {
                *x = *x + 1;

                self.core_handle.execute(WaitInAnotherThread::new(
                    Duration::seconds(2),
                    format!("WAIT {:?}", x),
                ));
                Ok(Async::Ready(Some(*x)))
            }
            _ => Ok(Async::Ready(None)),
        }
    }
}

```

这里需要关注的是`execute`这段代码, 它产生一个新的`Future`(等待两秒, 然后打印x), 不过请记住, 这个`future`将不会返回任何值(除了`Error`), 所以我们当且仅当把他是一个`Daemon-like`线程.

测试Code:

```RUST

fn main() {
    let mut reactor = Core::new().unwrap();

    // create a Stream returning 5 items
    // Each item will spawn an "inner" future
    // into the same reactor loop
    let my_stream = MyStream::new(5, reactor.handle());

    // we use for_each to consume
    // the stream
    let fut = my_stream.for_each(|num| {
        println!("num === {:?}", num);
        ok(())
    });

    // this is a manual future. it's the same as the
    // future spawned into our stream
    let wait = WaitInAnotherThread::new(Duration::seconds(3), "Manual3".to_owned());

    // we join the futures to let them run concurrently
    let future_joined = fut.map_err(|err| {}).join(wait);

    // let's run the future
    let ret = reactor.run(future_joined).unwrap();
    println!("ret == {:?}", ret);
}
```

上面代码给我们展示了如何连接`Stream`和`Future`. 现在让我尝试跑一下我们的代码:

```RUST

num === 1
num === 2
num === 3
num === 4
num === 5
"Manual3" starting the secondary thread!
"Manual3" not ready yet! parking the task.
"WAIT 1" starting the secondary thread!
"WAIT 1" not ready yet! parking the task.
"WAIT 2" starting the secondary thread!
"WAIT 2" not ready yet! parking the task.
"WAIT 3" starting the secondary thread!
"WAIT 3" not ready yet! parking the task.
"WAIT 4" starting the secondary thread!
"WAIT 4" not ready yet! parking the task.
"WAIT 5" starting the secondary thread!
"WAIT 5" not ready yet! parking the task.
"WAIT 1" the time has come == 2017-12-06T10:23:30.853796527Z!
"WAIT 1" ready! the task will complete.
"WAIT 2" the time has come == 2017-12-06T10:23:30.853831227Z!
"WAIT 2" ready! the task will complete.
"WAIT 3" the time has come == 2017-12-06T10:23:30.853842927Z!
"WAIT 3" ready! the task will complete.
"WAIT 5" the time has come == 2017-12-06T10:23:30.853856927Z!
"WAIT 5" ready! the task will complete.
"WAIT 4" the time has come == 2017-12-06T10:23:30.853850427Z!
"WAIT 4" ready! the task will complete.
"Manual3" the time has come == 2017-12-06T10:23:31.853775627Z!
"Manual3" ready! the task will complete.
ret == ((), ())

```

这个结果不是唯一的, 你和我的输出也许有所不同, 如果我们没有派生等待3s的`Future`, 结果是否就会有所不同?

```RUST
fn main() {
    let mut reactor = Core::new().unwrap();

    // create a Stream returning 5 items
    // Each item will spawn an "inner" future
    // into the same reactor loop
    let my_stream = MyStream::new(5, reactor.handle());

    // we use for_each to consume
    // the stream
    let fut = my_stream.for_each(|num| {
        println!("num === {:?}", num);
        ok(())
    });

    // let's run the future
    let ret = reactor.run(fut).unwrap();
    println!("ret == {:?}", ret);
}

```

我们会注意到这段代码几乎会立即返回下面的值:

```RUST
num === 1
num === 2
num === 3
num === 4
num === 5
ret == ()

```

`poll` 函数里派生出的`Future`没有机会运行.


## Next steps

下一篇我们将会使用`await!()`来精简我们的`Future`.

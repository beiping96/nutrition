# 深入浅出Rust-Future-Part-4

>译自[Rust futures: an uneducated, short and hopefully not boring tutorial - Part 4 - A "real" future from scratch](https://dev.to/mindflavor/rust-futures-an-uneducated-short-and-hopefully-not-boring-tutorial---part-4---a-real-future-from-scratch-734)
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

上三篇文章我们阐述如何处理`Future`的基础知识， 我们现在能组织多个`Future`成为一个`Future chain`, 执行他们,甚至创建他们.但是到现在我们的`Future`还没有贴近我们日常的使用场景。(But, so far, our futures are not really delegating the execution to another thing.)
在Part-3中我们用了粗暴的方法来`Unpark Future`。虽然解决了问题，并且使`Reactor`变得相对高效，但是这不是最佳实践。今天就让我们换种更好的方式去实现`Future`。

## A timer future

我们可以创建一个最简单的`Timer Future`(就像我们在Part-3章节所做的那样). 但这一次，我们不会立即`Unpark Future Task`, 而是一直`Parked`， 直到这个`Future`准备完为止. 我们该怎样实现？ 最简单的方式就是再委派一个线程。这个线程将等待一段时间， 然后`Unpark`我们的`Future Task`.

这就像在模拟一个`AsyncIO`的使用场景。当我们异步做的一些事情已经完成我们会收到与之相应的通知。为了简单起见，我们认为`Reactor`是单线程的，在等待通知的时候可以做其他的事情。

## Timer revised

我们的结构体非常简单.他包含结束日期和任务是否在运行。

```RUST
pub struct WaitInAnotherThread {
    end_time: DateTime<Utc>,
    running: bool,
}

impl WaitInAnotherThread {
    pub fn new(how_long: Duration) -> WaitInAnotherThread {
        WaitInAnotherThread {
            end_time: Utc::now() + how_long,
            running: false,
        }
    }
}
```

`DateTime`类型和`Duration`持续时间来自`chronos crate`.

## Spin wait

实现等待时间的函数:

```RUST
pub fn wait_spin(&self) {
    while Utc::now() < self.end_time {}
    println!("the time has come == {:?}!", self.end_time);
}

fn main() {
    let wiat = WaitInAnotherThread::new(Duration::seconds(30));
    println!("wait spin started");
    wiat.wait_spin();
    println!("wait spin completed");
}
```

在这种情况下，我们基本上会根据到期时间检查当前时间。 这很有效，而且非常精确。 这种方法的缺点是我们浪费了大量的CPU周期。 在我的电脑上, CPU一个核心完全被占用，这和我们`Part-3`遇到的情况一致。

> Spin wait这种方式只适用于等待时间非常短的场景, 或者你没有别的选择的情况下使用它。

## Sleep wait

系统通常会允许你的线程`Park`一段特定的时间.这通常被称为线程睡眠。睡眠线程X秒，换据换的意思是: 告诉系统X秒内，不需要调度我。这样的好处是，CPU可以在这段时间内干别的事情。在`Rust`中我们使用`std::thread::sleep()`.

```RUST
pub fn wait_blocking(&self) {
    while Utc::now() < self.end_time {
        let delta_sec = self.end_time.timestamp() - Utc::now().timestamp();
        if delta_sec > 0 {
            thread::sleep(::std::time::Duration::from_secs(delta_sec as u64));
        }
    }
    println!("the time has come == {:?}!", self.end_time);
}

let wiat = WaitInAnotherThread::new(Duration::seconds(30));
println!("wait blocking started");
wiat.wait_blocking();
println!("wait blocking completed");
```

尝试运行我们的代码会发现, 改进过的代码再也不会完全占用一个CPU核心了。改进过的代码比我们该开始写的性能好多了，但是这就是`Future`了吗？

## Future

当然不是，我们还没有实现`Future Trait`, 所以，我们现在实现它。

```RUST
impl Future for WaitInAnotherThread {
    type Item = ();
    type Error = Box<Error>;

    fn poll(&mut self) -> Poll<Self::Item, Self::Error> {
        while Utc::now() < self.end_time {
            let delta_sec = self.end_time.timestamp() - Utc::now().timestamp();
            if delta_sec > 0 {
                thread::sleep(::std::time::Duration::from_secs(delta_sec as u64));
            }
        }
        println!("the time has come == {:?}!", self.end_time);
        Ok(Async::Ready(())
}
```

emmm，这块代码我们是不是以前见过， 跟上篇我们写的一个很像，它会阻塞`Reactor`，这样做实在是太糟糕了。

> `Future` 应该尽可能的不要阻塞。

一个`Reactor`的最佳实践应该至少包含下面几条：

- 当主`Task`需要等待别的`Task`时，应该停止它。
- 不要阻塞当前线程。
- 任务完成时向`Reactor`发送信号。

我们要做的是创建另一个睡眠线程. 睡眠的线程是不会占用CPU资源。所以在另一个线程里`Reactor`还像往常那样，高效的工作着。当这个`Sleep Thread`醒来后, 它会`Unpark`这个任务, 并且通知`Reactor`。

让我们一步一步完善我们的想法:

```RUST
impl Future for WaitInAnotherThread {
    type Item = ();
    type Error = Box<Error>;

    fn poll(&mut self) -> Poll<Self::Item, Self::Error> {
        if Utc::now() < self.end_time {
            println!("not ready yet! parking the task.");

            if !self.running {
                println!("side thread not running! starting now!");
                self.run(task::current());
                self.running = true;
            }

            Ok(Async::NotReady)
        } else {
            println!("ready! the task will complete.");
            Ok(Async::Ready(()))
        }
    }
}
```

我们只需要创建一个并行线程, 所以我们需要有个字段来判断(`WaitInAnotherThread.runing`)，当前需不需要创建这个线程。这里需要注意的是当`Future`被轮询之前，这些代码是不会被执行的。当然我们还会检测当前时间是否大于过期时间，如果大于，也不会产生另外一个线程。

如果`end_time`大于当前的时间并且另一个线程没有被创建，程序就会立即创建一个新的线程。然后程序会返回`Ok(Async::NotReady())`, 与我们`Part-3`中所做的相反，我们不会在这里`Unpark Task`. 这是另一个线程应该做的事情。在别的实现中，例如IO，唤醒我们的主线程的应该是操作系统。

```RUST
fn run(&mut self, task: task::Task) {
    let lend = self.end_time;
    
    thread::spawn(move || {
        while Utc::now() < lend {
            let delta_sec = lend.timestamp() - Utc::now().timestamp();
            if delta_sec > 0 {
                thread::sleep(::std::time::Duration::from_secs(delta_sec as u64));
            }
            task.notify();
        }
        println!("the time has come == {:?}!", lend);
    });
}
```

这里有两件事情需要注意下.

- 我们将`Task`引用传递给，另一个并行的线程。这很重要，因为我们不能在单独的线程里使用`task::current`.

- 我们不能将self移动到闭包中，所以我们需要转移所有权至`lend`变量.为啥这样做？
> Rust中的线程需要实现具有`'static`生命周期的`Send Trait`。

`task`自身实现了上述的要求所以可以引用传递。但是我们的结构体没有实现，所以这也是为什么要移动`end_time`的所有权。这就意味着当线程被创建后你不能更改`end_time`.

让我们尝试运行下:

```rust
fn main() {
    let mut reactor = Core::new().unwrap();

    let wiat = WaitInAnotherThread::new(Duration::seconds(3));
    println!("wait future started");
    let ret = reactor.run(wiat).unwrap();
    println!("wait future completed. ret == {:?}", ret);
}
```

运行结果：

```RUST
Finished dev [unoptimized + debuginfo] target(s) in 0.96 secs
Running `target/debug/tst_fut_complete`
wait future started
not ready yet! parking the task.
side thread not running! starting now!
the time has come == 2017-11-21T12:55:23.397862771Z!
ready! the task will complete.
wait future completed. ret == ()
```

让我们总结下流程：

- 我们让`Reactor`执行我们的`Future`.
- `Future`发现`end_time`大于当前时间：
   1. `Park Task`
   2. 开启另一个线程

- 副线程在一段时间后被唤醒:
  1. 告诉`Reactor`我们的`Task`可以`Unpark`了。
  2. 销毁自身

- `Reactor`唤醒被`Park`的`Task`

- `Future(Task)`完成了自身的任务：
   1. 通知`Reactor`
   2. 返回相应的结果

- reactor将任`Task`的输出值返回给run函数的调用者。

## Code

```rust
extern crate chrono;
extern crate futures;

extern crate tokio_core;

use chrono::prelude::*;
use chrono::*;
use futures::prelude::*;
use futures::*;
use std::error::Error;
use std::thread::{sleep, spawn};
use tokio_core::reactor::Core;

pub struct WaitInAnotherThread {
    end_time: DateTime<Utc>,
    running: bool,
}

impl WaitInAnotherThread {
    pub fn new(how_long: Duration) -> WaitInAnotherThread {
        WaitInAnotherThread {
            end_time: Utc::now() + how_long,
            running: false,
        }
    }

    fn run(&mut self, task: task::Task) {
       let lend = self.end_time;

        spawn(move || {
            while Utc::now() < lend {
                let delta_sec = lend.timestamp() - Utc::now().timestamp();
                if delta_sec > 0 {
                    sleep(::std::time::Duration::from_secs(delta_sec as u64));
                }
                task.notify();
            }
            println!("the time has come == {:?}!", lend);
        });
    }
}

impl Future for WaitInAnotherThread {
    type Item = ();
    type Error = Box<Error>;

    fn poll(&mut self) -> Poll<Self::Item, Self::Error> {
        if Utc::now() < self.end_time {
            println!("not ready yet! parking the task.");

            if !self.running {
                println!("side thread not running! starting now!");
                self.run(task::current());
                self.running = true;
            }
            
            Ok(Async::NotReady)
        } else {
            println!("ready! the task will complete.");
            Ok(Async::Ready(()))
        }
    }
}

fn main() {
    let mut reactor = Core::new().unwrap();

    let wiat = WaitInAnotherThread::new(Duration::seconds(3));
    println!("wait future started");
    let ret = reactor.run(wiat).unwrap();
    println!("wait future completed. ret == {:?}", ret);
}
```

## Conclusion

到目前未知我们完整的实现了没有阻塞的`real life future`. 所以也没有浪费CPU资源。除了这个例子你还能想到与此相同的应用场景吗？
尽管`RUST`早都有现成的`Crate`帮我们实现好了。但是了解其中的工作原理还是对我们有很大的帮助。

下一个主题将是Streams，目标是：创建一个不会阻塞`Reactor`的`Iterators`.

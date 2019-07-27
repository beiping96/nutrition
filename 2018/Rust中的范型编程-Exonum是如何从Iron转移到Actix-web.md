# Rust中的范型编程-Exonum是如何从Iron转移到Actix-web

> 本文同步与[Rust中文社区](https://rustlang-cn.org/read/rust/2018/Rust中的范型编程-Exonum是如何从Iron转移到Actix-web.html)
> 时间：2018-11-23，作者：[krircc](https://krircc.github.io/)， 简介：天青色  

欢迎向Rust中文社区投稿,**[投稿地址](https://github.com/rustlang-cn/articles)**,好文将在以下地方直接展示

- 1 [Rust中文社区首页](https://rustlang-cn.org/)
- 2 Rust中文社区**Rust阅读文章栏目**
- 3 知乎专栏[Rust中文社区](https://zhuanlan.zhihu.com/rustlang-cn)
- 4 思否专栏[Rust中文社区](https://segmentfault.com/blog/rust-lang)
- 5 微博[Rustlang-cn](https://weibo.com/kriry)
- 6 简书专题[Rust中文社区](https://www.jianshu.com/c/2efae7198ea3)

Rust生态系统仍在增长。因此，具有改进功能的新库经常被发布到开发人员社区中，而较旧的库变得过时。当我们最初设计Exonum时，我们使用了Iron Web框架。我们现在决定将[Exonum平台](https://github.com/exonum/exonum)转移到actix-web框架。

在本文中，我们将介绍如何使用泛型编程将Exonum框架移植到[actix-web](https://github.com/actix/actix-web)

## Exonum on Iron

在Exonum平台中，Iron框架在没有任何抽象的情况下使用。我们为某些资源安装了handlers，并通过使用辅助方法解析URL来获取请求参数; 结果只是以字符串的形式返回。

另外，我们以CORS头的形式使用了一些中间件插件。我们使用mount将所有处理程序合并到一个API中。

## 我们决定摆脱 Iron

[Iron](https://github.com/iron)是一个很好的库，有很多插件。然而，它是在`future`和`tokio`等项目不存在的日子写的。

Iron的体系结构涉及同步请求处理，这很容易受到大量同时打开的连接(并发)的影响。为了实现可扩展性，Iron需要变为异步，这将涉及重新思考和重写整个框架。结果，我们看到软件工程师逐渐不使用Iron。

## 为什么我们选择Actix-Web

Actix-web是一个受欢迎的框架，在[TechEmpower基准测试](https://www.techempower.com/benchmarks/#section=data-r17&hw=ph&test=update)中排名很高。它拥有一个活跃的开发人员社区，与Iron不同，它拥有精心设计的API和基于actix actor框架的高质量实现。线程池异步处理请求; 如果请求处理panics，则actor会自动重启。

以前，人们担心actix-web包含许多不安全的代码。但是，当框架逐渐以安全的编程语言（Rust）重写时，不安全代码的数量显着减少。Bitfury的工程师(Exonum框架的工程师)自己对这些代码进行了审查，并对其长期稳定性充满信心。

对于Exonum框架，转向actix解决了操作稳定性问题。如果存在大量连接，则Iron框架可能会失败。我们还发现actix-web API更简单，更高效，更统一。我们相信，用户和开发人员可以更轻松地使用Exonum编程界面，由于采用了actix-web设计，现在可以更快地运行。

## 我们对Web框架的要求

在此过程中，我们意识到，不仅要简单地转换框架，而且要设计独立于任何特定Web框架的新API体系结构，这对我们非常重要。这种架构允许创建处理程序，几乎不关心Web细节，并将它们转移到任何后端。这个概念可以通过编写一个应用基本类型和`trait`的前端来实现。

要了解这个前端需要看起来像什么，让我们定义任何HTTP API的真正含义：

- 请求仅由客户提出; 服务器只响应它们（服务器不发起请求）。

- 请求读取数据或更改数据。

- 作为请求处理的结果，服务器在成功的情况下返回包含所需数据的响应; 如果失败，或者有关错误的信息。

如果我们要分析所有抽象层，事实证明任何HTTP请求只是一个函数调用：

```rust
fn  request（context：＆ ServiceContext，query：Query） - > Result <Response，ServiceError>
```

其他一切都可以被视为这个基本实体的延伸。因此，为了独立于Web框架的特定实现，我们需要以类似于上面示例的样式编写处理程序。

用于HTTP请求的范型处理的 trait“Endpoint”

最简单直接的方法是声明`Endpoint` trait，它描述了特定请求的实现：

```rust
// A trait describing GET request handlers. It should be possible to call each of the handlers from any freed
// thread. This requirement imposes certain restrictions on the trait. Parameters and request results are
// configured using associated types.
trait Endpoint: Sync + Send + 'static {
    type Request: DeserializeOwned + 'static;
    type Response: Serialize + 'static;

    fn handle(&self, context: &Context, request: Self::Request) -> Result<Self::Response, io::Error>;
}
```

现在我们需要在特定的框架中实现这个处理程序。例如，在actix-web中，它看起来如下所示：

```rust
// Response type in actix-web. Note that they are asynchronous, even though `Endpoint` assumes that 
// processing is synchronous.
type FutureResponse = actix_web::FutureResponse<HttpResponse, actix_web::Error>;

// A raw request handler for actix-web. This is what the framework ultimately works with. The handler 
// receives parameters from an arbitrary context, through which the request parameters are passed.
type RawHandler = dyn Fn(HttpRequest<Context>) -> FutureResponse + 'static + Send + Sync;

// For convenience, let’s put everything we need from the handler into a single structure.
#[derive(Clone)]
struct RequestHandler {
    /// The name of the resource.
    pub name: String,
    /// HTTP method.
    pub method: actix_web::http::Method,
    /// The raw handler. Note that it will be used from multiple threads.
    pub inner: Arc<RawHandler>,
}
```

我们可以使用结构通过上下文传递请求参数。Actix-web可以使用serde自动反序列化参数。例如，a = 15＆b = hello被反序列化为如下结构：

```rust
#[derive(Deserialize)]
struct SimpleQuery {
    a: i32,
    b: String,
}
```

这种反序列化功能与来自“Endpoint” trait的相关类型请求很好地吻合。

接下来，让我们设计一个适配器，它将特定的`Endpoint`实现包装到`actix-web`的`RequestHandler`中。请注意，在执行此操作时，请求和响应类型的信息将消失。这种技术称为类型擦除 - 它将静态调度转换为动态调度。

```rust
impl RequestHandler {
    fn from_endpoint<E: Endpoint>(name: &str, endpoint: E) -> RequestHandler {
        let index = move |request: HttpRequest<Context>| -> FutureResponse {
            let context = request.state();
            let future = Query::from_request(&request, &())
                .map(|query: Query<E::Request>| query.into_inner())
                .and_then(|query| endpoint.handle(context, query).map_err(From::from))
                .and_then(|value| Ok(HttpResponse::Ok().json(value)))
                .into_future();
            Box::new(future)
        };

        Self {
            name: name.to_owned(),
            method: actix_web::http::Method::GET,
            inner: Arc::from(index) as Arc<RawHandler>,
        }
    }
}
```

在这个阶段，仅为POST请求添加处理程序就足够了，因为我们创建了一个独立于实现细节的trait。 但是，我们发现这个解决方案还不够先进。

## “Endpoint” trait的缺点

编写处理程序时会生成大量辅助代码：

```rust
// A structure with the context of the handler.
struct ElementCountEndpoint {
    elements: Rc<RefCell<Vec<Something>>>,
}

// Implementation of the `Endpoint` trait.
impl Endpoint for ElementCountEndpoint {
    type Request = ();
    type Result = usize;

    fn handle(&self, context: &Context, _request: ()) -> Result<usize, io::Error> {
        Ok(self.elements.borrow().len())
    }
}

// Installation of the handler in the backend.
let endpoint = ElementCountEndpoint::new(elements.clone());
let handler = RequestHandler::from_endpoint("/v1/element_count", endpoint);
actix_backend.endpoint(handler);
```

理想情况下，我们需要能够将一个简单的闭包作为处理程序传递，从而显着减少语法噪声的数量。

```rust
let elements = elements.clone();
actix_backend.endpoint("/v1/elements_count", move || {
    Ok(elements.borrow().len())
});
```

下面我们将讨论如何做到这一点。

## 深入范型编程

我们需要添加自动生成适配器的功能，该适配器使用正确的关联类型实现`Endpoint` trait。 输入将仅包含具有HTTP请求处理程序的闭包。

参数和闭包的结果可以有不同的类型，因此我们必须在这里使用方法重载。 Rust不支持直接重载，但允许使用`Into`和`From` trait进行模拟。

此外，闭包值的返回类型不必与`Endpoint`实现的返回值匹配。 要操纵此类型，必须从接收到的闭包的类型中提取它。

## 从“Fn”  trait中获取类型

在Rust中，每个闭包都有自己独特的类型，无法在程序中明确指出。 对于带闭包的操作，我们使用`Fn` trait。  trait包含函数的签名以及参数类型和返回值，但是，分别检索这些元素并不容易。

主要思想是使用以下形式的辅助结构：

```rust
/// Simplified example of extracting types from an F closure: Fn(A) -> B.
struct SimpleExtractor<A, B, F>
{
    // The original function.
    inner: F,
    _a: PhantomData<A>,
    _b: PhantomData<B>,
}
```

我们必须使用PhantomData，因为Rust要求所有范型参数都在结构的定义中指出。 但是，闭包或函数F本身的类型不是范型的（尽管它实现了范型的`Fn` trait）。 类型参数A和B不直接使用。

正是Rust类型系统的这种限制使得我们无法通过直接为闭包实现`Endpoint` trait来应用更简单的策略：

```rust
impl<A, B, F> Endpoint for F where F: Fn(&Context, A) -> B {
    type Request = A;
    type Response = B;

    fn handle(&self, context: &Context, request: A) -> Result<B, io::Error> {
        // ...
    }
}
```

在上面的例子中，编译器返回一个错误：

```rust
error[E0207]: the type parameter `A` is not constrained by the impl trait, self type, or predicates
  --> src/main.rs:10:6
   |
10 | impl<A, B, F> Endpoint for F where F: Fn(&Context, A) -> B {
   |      ^ unconstrained type parameter

```

辅助结构SimpleExtractor可以描述“From”的转换。 这种转换允许我们保存任何函数并提取其参数的类型：

```rust
impl<A, B, F> From<F> for SimpleExtractor<A, B, F>
where
    F: Fn(&Context, A) -> B,
    A: DeserializeOwned,
    B: Serialize,
{
    fn from(inner: F) -> Self {
        SimpleExtractor {
            inner,
            _a: PhantomData,
            _b: PhantomData,
        }
    }
}
```

以下代码成功编译：

```rust
#[derive(Deserialize)]
struct Query {
    a: i32,
    b: String,
};

// Verification of the ordinary structure.
fn my_handler(_: &Context, q: Query) -> String {
    format!("{} has {} apples.", q.b, q.a)
}
let fn_extractor = SimpleExtractor::from(my_handler);

// Verification of the closure.
let c = 15;
let my_closure = |_: &Context, q: Query| -> String {
    format!("{} has {} apples, but Alice has {}", q.b, q.a, c)
};
let closure_extractor = SimpleExtractor::from(my_closure);
```

## 专业化和标记类型

现在我们有一个带有显式参数化参数类型的函数，可以使用它来代替`Endpoint` trait。 例如，我们可以轻松实现从`SimpleExtractor`到`RequestHandler`的转换。 不过，这不是一个完整的解决方案。 我们需要以某种方式区分类型级别（以及同步和异步处理程序之间）的`GET`和`POST`请求的处理程序。 在此任务中，标记类型可以帮助我们。

首先，让我们重写`SimpleExtractor`，以便它可以区分同步和异步结果。 同时，我们将为每个案例实施“From” trait。 注意，可以针对范型结构的特定变体实现 trait。

```rust
/// Generic handler for HTTP-requests.
pub struct With<Q, I, R, F> {
    /// A specific handler function.
    pub handler: F,
    /// Structure type containing the parameters of the request.
    _query_type: PhantomData<Q>,
    /// Type of the request result.
    _item_type: PhantomData<I>,
    /// Type of the value returned by the handler.
    /// Note that this value can differ from the result of the request.
    _result_type: PhantomData<R>,
}

// Implementation of an ordinary synchronous returned value.
impl<Q, I, F> From<F> for With<Q, I, Result<I>, F>
where
    F: Fn(&ServiceApiState, Q) -> Result<I>,
{
    fn from(handler: F) -> Self {
        Self {
            handler,
            _query_type: PhantomData,
            _item_type: PhantomData,
            _result_type: PhantomData,
        }
    }
}

// Implementation of an asynchronous request handler.
impl<Q, I, F> From<F> for With<Q, I, FutureResult<I>, F>
where
    F: Fn(&ServiceApiState, Q) -> FutureResult<I>,
{
    fn from(handler: F) -> Self {
        Self {
            handler,
            _query_type: PhantomData,
            _item_type: PhantomData,
            _result_type: PhantomData,
        }
    }
}
```

现在我们需要声明将请求处理程序与其名称和类型组合在一起的结构：

```rust
#[derive(Debug)]
pub struct NamedWith<Q, I, R, F, K> {
    /// The name of the handler.
    pub name: String,
    /// The handler with the extracted types.
    pub inner: With<Q, I, R, F>,
    /// The type of the handler.
    _kind: PhantomData<K>,
}
```

接下来，我们声明几个将用作标记类型的空结构。 标记将允许我们为每个处理程序实现自己的代码，以将处理程序转换为先前描述的`RequestHandler`。

```rust
/// A handler that does not change the state of the service. In HTTP, GET-requests correspond to this 
// handler.
pub struct Immutable;
/// A handler that changes the state of the service. In HTTP, POST, PUT, UPDATE and other similar 
//requests correspond to this handler, but for the current case POST will suffice.
pub struct Mutable;
```

现在我们可以为模板参数R和K的所有组合（处理程序的返回值和请求的类型）定义“From”特征的四种不同实现。

```rust
// Implementation of a synchronous handler of GET requests.
impl<Q, I, F> From<NamedWith<Q, I, Result<I>, F, Immutable>> for RequestHandler
where
    F: Fn(&ServiceApiState, Q) -> Result<I> + 'static + Send + Sync + Clone,
    Q: DeserializeOwned + 'static,
    I: Serialize + 'static,
{
    fn from(f: NamedWith<Q, I, Result<I>, F, Immutable>) -> Self {
        let handler = f.inner.handler;
        let index = move |request: HttpRequest| -> FutureResponse {
            let context = request.state();
            let future = Query::from_request(&request, &())
                .map(|query: Query<Q>| query.into_inner())
                .and_then(|query| handler(context, query).map_err(From::from))
                .and_then(|value| Ok(HttpResponse::Ok().json(value)))
                .into_future();
            Box::new(future)
        };

        Self {
            name: f.name,
            method: actix_web::http::Method::GET,
            inner: Arc::from(index) as Arc<RawHandler>,
        }
    }
}
// Implementation of a synchronous handler of POST requests.
impl<Q, I, F> From<NamedWith<Q, I, Result<I>, F, Mutable>> for RequestHandler
where
    F: Fn(&ServiceApiState, Q) -> Result<I> + 'static + Send + Sync + Clone,
    Q: DeserializeOwned + 'static,
    I: Serialize + 'static,
{
    fn from(f: NamedWith<Q, I, Result<I>, F, Mutable>) -> Self {
        let handler = f.inner.handler;
        let index = move |request: HttpRequest| -> FutureResponse {
            let handler = handler.clone();
            let context = request.state().clone();
            request
                .json()
                .from_err()
                .and_then(move |query: Q| {
                    handler(&context, query)
                        .map(|value| HttpResponse::Ok().json(value))
                        .map_err(From::from)
                })
                .responder()
        };

        Self {
            name: f.name,
            method: actix_web::http::Method::POST,
            inner: Arc::from(index) as Arc<RawHandler>,
        }
    }
}
// Implementation of an asynchronous handler of GET requests.
impl<Q, I, F> From<NamedWith<Q, I, FutureResult<I>, F, Immutable>> for RequestHandler
where
    F: Fn(&ServiceApiState, Q) -> FutureResult<I> + 'static + Clone + Send + Sync,
    Q: DeserializeOwned + 'static,
    I: Serialize + 'static,
{
    fn from(f: NamedWith<Q, I, FutureResult<I>, F, Immutable>) -> Self {
        let handler = f.inner.handler;
        let index = move |request: HttpRequest| -> FutureResponse {
            let context = request.state().clone();
            let handler = handler.clone();
            Query::from_request(&request, &())
                .map(move |query: Query<Q>| query.into_inner())
                .into_future()
                .and_then(move |query| handler(&context, query).map_err(From::from))
                .map(|value| HttpResponse::Ok().json(value))
                .responder()
        };

        Self {
            name: f.name,
            method: actix_web::http::Method::GET,
            inner: Arc::from(index) as Arc<RawHandler>,
        }
    }
}
// Implementation of an asynchronous handler of POST requests.
impl<Q, I, F> From<NamedWith<Q, I, FutureResult<I>, F, Mutable>> for RequestHandler
where
    F: Fn(&ServiceApiState, Q) -> FutureResult<I> + 'static + Clone + Send + Sync,
    Q: DeserializeOwned + 'static,
    I: Serialize + 'static,
{
    fn from(f: NamedWith<Q, I, FutureResult<I>, F, Mutable>) -> Self {
        let handler = f.inner.handler;
        let index = move |request: HttpRequest| -> FutureResponse {
            let handler = handler.clone();
            let context = request.state().clone();
            request
                .json()
                .from_err()
                .and_then(move |query: Q| {
                    handler(&context, query)
                        .map(|value| HttpResponse::Ok().json(value))
                        .map_err(From::from)
                })
                .responder()
        };

        Self {
            name: f.name,
            method: actix_web::http::Method::POST,
            inner: Arc::from(index) as Arc<RawHandler>,
        }
    }
}
```

## 处理后端

最后一步是设计一个可以接受闭包并将它们添加到相应后端的外观。 在给定的情况下，我们有一个后端--actix-web。 但是，幕墙背后还有可能进行额外的实施。 例如：`Swagger`规范的生成器。

```rust
pub struct ServiceApiScope {
    actix_backend: actix::ApiBuilder,
}

impl ServiceApiScope {
    /// This method adds an Immutable handler to all backends.
    pub fn endpoint<Q, I, R, F, E>(&mut self, name: &'static str, endpoint: E) -> &mut Self
    where
        // Here we list the typical restrictions which we have encountered earlier:
        Q: DeserializeOwned + 'static,
        I: Serialize + 'static,
        F: Fn(&ServiceApiState, Q) -> R + 'static + Clone,
        E: Into<With<Q, I, R, F>>,
        // Note that the list of restrictions includes the conversion from NamedWith into RequestHandler 
        // we have implemented earlier.
        RequestHandler: From<NamedWith<Q, I, R, F, Immutable>>,
    {
        self.actix_backend.endpoint(name, endpoint);
        self
    }

    /// A similar method for Mutable handlers.
    pub fn endpoint_mut<Q, I, R, F, E>(&mut self, name: &'static str, endpoint: E) -> &mut Self
    where
        Q: DeserializeOwned + 'static,
        I: Serialize + 'static,
        F: Fn(&ServiceApiState, Q) -> R + 'static + Clone,
        E: Into<With<Q, I, R, F>>,
        RequestHandler: From<NamedWith<Q, I, R, F, Mutable>>,
    {
        self.actix_backend.endpoint_mut(name, endpoint);
        self
    }
}
```

请注意请求参数的类型，请求结果的类型以及处理程序的同步/异步是如何从其签名自动派生的。 此外，我们需要明确指定请求的名称和类型。

## 此方式的缺点

上述方法虽然非常有效，但也有其缺点。 特别是，`endpoint`和`endpoint_mut`方法应该考虑特定后端的实现 trait。 此限制阻止我们在旅途中添加后端，但很少需要此功能。

另一个问题是我们无法在没有其他参数的情况下定义处理程序的特化。 换句话说，如果我们编写以下代码，它将不会被编译，因为它与现有的范型实现相冲突：

```rust
impl<(), I, F> From<F> for With<(), I, Result<I>, F>
    where
        F: Fn(&ServiceApiState) -> Result<I>,
    {
        fn from(handler: F) -> Self {
            Self {
                handler,
                _query_type: PhantomData,
                _item_type: PhantomData,
                _result_type: PhantomData,
            }
        }
    }
```

因此，没有任何参数的请求仍必须接受JSON字符串`null`，该字符串被反序列化为`（）`。 这个问题可以通过C ++风格的专业化来解决，但是现在它只在编译器的夜间版本中可用，并且不清楚何时它将成为一个稳定的 trait。

同样，返回值的类型也不能专门化。 即使请求没有暗示某种类型的返回值，它仍然会传递带有`null`的JSON。

在`GET`请求中解码`URL`查询也对参数类型施加了一些不明显的限制，但是这个问题与`serde-urlencoded`实现的 trait有关。

## 结论

如上所述，我们已经实现了一个改进的API，它允许简单而清晰地创建处理程序，而无需担心Web细节。 这些处理程序可以与任何后端一起使用，甚至可以同时使用多个后端。

[英文原文](https://medium.com/meetbitfury/generic-methods-in-rust-how-exonum-shifted-from-iron-to-actix-web-7a2752171388)
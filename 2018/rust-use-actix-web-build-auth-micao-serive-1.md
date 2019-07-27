# Rust使用Actix-Web验证Auth Web微服务-整教程第1部分

>[本文同步于Rust中文社区](http://rustlang-cn.org/read/rust/rust-use-actix-web-build-auth-micao-serive-1)

欢迎向Rust中文社区投稿,**[投稿地址](https://github.com/rustlang-cn/articles)**,好文将在以下地方直接展示

- 1 [Rust中文社区首页](https://rustlang-cn.org/)
- 2 Rust中文社区**Rust阅读文章栏目**
- 3 知乎专栏[Rust中文社区](https://zhuanlan.zhihu.com/rustlang-cn)
- 4 思否专栏[Rust中文社区](https://segmentfault.com/blog/rust-lang)
- 5 微博[Rustlang-cn](https://weibo.com/kriry)
- 6 简书专题[Rust中文社区](https://www.jianshu.com/c/2efae7198ea3)

我们将创建一个rust仅处理用户注册和身份验证的Web服务器。我将在逐步解释每个文件中的步骤。完整的项目代码在[这里](https://gitlab.com/mygnu/rust-auth-server/tree/part_one)。

## 事件的流程如下所示：

- 使用电子邮件地址注册➡接收带有链接的📨进行验证
- 点击链接➡使相同的电子邮件和密码注册
- 使用电子邮件和密码登录➡获取验证并接收jwt令牌

## 我们打算使用的包

- actix         &emsp;&emsp;&emsp;&emsp;// Actix是一个Rust actor框架。
- actix-web     &emsp;&emsp;// Actix web是Rust的一个简单，实用且极其快速的Web框架。
- brcypt        &emsp;&emsp;&emsp;&emsp;//使用bcrypt轻松散列和验证密码。
- chrono        &emsp;&emsp;&emsp;&emsp;// Rust的日期和时间库。
- diesel        &emsp;&emsp;&emsp;&emsp;//用于PostgreSQL，SQLite和MySQL的安全，可扩展的ORM和查询生成器。
- dotenv        &emsp;&emsp;&emsp;&emsp;// Rust的dotenv实现。
- env_logger    &emsp;&emsp;//通过环境变量配置的日志记录实现。
- failure       &emsp;&emsp;&emsp;&emsp;//实验性错误处理抽象。
- jsonwebtoken  &emsp;&emsp;//以强类型方式创建和解析JWT。
- futures       &emsp;&emsp;&emsp;&emsp;//future和Stream的实现，具有零分配，可组合性和类似迭代器的接口。
- r2d2         &emsp;&emsp; &emsp;&emsp;//通用连接池。
- serde         &emsp;&emsp;&emsp;&emsp;//通用序列化/反序列化框架。
- serde_json    &emsp;&emsp;// JSON序列化文件格式。
- serde_derive  &emsp;&emsp;//＃[derive（Serialize，Deserialize）]的宏1.1实现。
- sparkpost     &emsp;&emsp;//用于sparkpost电子邮件api v1的Rust绑定。
- uuid          &emsp;&emsp;&emsp;&emsp;//用于生成和解析UUID的库。

我从他们的官方说明中提供了有关正在使用的包的简要信息。如果您想了解更多有关这些板条箱的信息，请转到crates.io。

## 准备

我将在这里假设您对编程有一些了解，最好还有一些Rust。需要进行工作设置rust。查看`https://rustup.rs`用于rust设置。

我们将使用`diesel`来创建模型并处理数据库，查询和迁移。请前往`http://diesel.rs/guides/getting-started/`开始使用并进行设置`diesel_cli`。在本教程中我们将使用postgresql，请按照说明设置postgres。您需要有一个正在运行的postgres服务器，并且可以创建一个数据库来完成本教程。另一个很好的工具是`Cargo Watch`，它允许您在进行任何更改时观看文件系统并重新编译并重新运行应用程序。

如果您的系统上已经没有安装`Curl`，请在本地测试api。

## 让我们开始

检查你的rust和cargo版本并创建一个新的项目

```rust
# at the time of writing this tutorial my setup is 
rustc --version && cargo --version
# rustc 1.29.1 (b801ae664 2018-09-20)
# cargo 1.29.0 (524a578d7 2018-08-05)

cargo new simple-auth-server
# Created binary (application) `simple-auth-server` project

cd simple-auth-server # and then 

# watch for changes re-compile and run
cargo watch -x run 
```

用以下内容填写cargo依赖关系，我将在项目中使用它们。我正在使用crate的显式版本，因为你知道包变旧了并且发生了变化。（如果你在很长一段时间之后阅读本教程）。在本教程的第1部分中，我们不会使用所有这些，但它们在最终的应用程序中都会变得很方便。

```toml
[dependencies]
actix = "0.7.4"
actix-web = "0.7.8"
bcrypt = "0.2.0"
chrono = { version = "0.4.6", features = ["serde"] }
diesel = { version = "1.3.3", features = ["postgres", "uuid", "r2d2", "chrono"] }
dotenv = "0.13.0"
env_logger = "0.5.13"
failure = "0.1.2"
frank_jwt = "3.0"
futures = "0.1"
r2d2 = "0.8.2"
serde_derive="1.0.79"
serde_json="1.0"
serde="1.0"
sparkpost = "0.4"
uuid = { version = "0.6.5", features = ["serde", "v4"] }
```

## 设置基本APP

创建新文件src/models.rs与src/app.rs。

```rust
// models.rs
use actix::{Actor, SyncContext};
use diesel::pg::PgConnection;
use diesel::r2d2::{ConnectionManager, Pool};

/// This is db executor actor. can be run in parallel
pub struct DbExecutor(pub Pool<ConnectionManager<PgConnection>>);


// Actors communicate exclusively by exchanging messages. 
// The sending actor can optionally wait for the response. 
// Actors are not referenced directly, but by means of addresses.
// Any rust type can be an actor, it only needs to implement the Actor trait.
impl Actor for DbExecutor {
    type Context = SyncContext<Self>;
}
```

要使用此Actor，我们需要设置actix-web服务器。我们有以下内容src/app.rs。我们暂时将资源构建者留空。这就是路由的核心所在。

```rust
// app.rs
use actix::prelude::*;
use actix_web::{http::Method, middleware, App};
use models::DbExecutor;

pub struct AppState {
    pub db: Addr<DbExecutor>,
}

// helper function to create and returns the app after mounting all routes/resources
pub fn create_app(db: Addr<DbExecutor>) -> App<AppState> {
    App::with_state(AppState { db })
        // setup builtin logger to get nice logging for each request
        .middleware(middleware::Logger::new("\"%r\" %s %b %Dms"))

         // routes for authentication
        .resource("/auth", |r| {
        })
        // routes to invitation
        .resource("/invitation/", |r| {
        })
        // routes to register as a user after the
        .resource("/register/", |r| {
        })
}
```

`main.rs`

```rust
// main.rs
// to avoid the warning from diesel macros
#![allow(proc_macro_derive_resolution_fallback)]

extern crate actix;
extern crate actix_web;
extern crate serde;
extern crate chrono;
extern crate dotenv;
extern crate futures;
extern crate r2d2;
extern crate uuid;
#[macro_use] extern crate diesel;
#[macro_use] extern crate serde_derive;
#[macro_use] extern crate failure;

mod app;
mod models;
mod schema;
// mod errors;
// mod invitation_handler;
// mod invitation_routes;

use models::DbExecutor;
use actix::prelude::*;
use actix_web::server;
use diesel::{r2d2::ConnectionManager, PgConnection};
use dotenv::dotenv;
use std::env;


fn main() {
    dotenv().ok();
    let database_url = env::var("DATABASE_URL").expect("DATABASE_URL must be set");
    let sys = actix::System::new("Actix_Tutorial");

    // create db connection pool
    let manager = ConnectionManager::<PgConnection>::new(database_url);
    let pool = r2d2::Pool::builder()
        .build(manager)
        .expect("Failed to create pool.");

    let address :Addr<DbExecutor>  = SyncArbiter::start(4, move || DbExecutor(pool.clone()));

    server::new(move || app::create_app(address.clone()))
        .bind("127.0.0.1:3000")
        .expect("Can not bind to '127.0.0.1:3000'")
        .start();

    sys.run();
}
```

在此阶段，您的服务器应该编译并运行127.0.0.1:3000。让我们创建一些模型。

## 设置diesel并创建我们的用户模型

我们首先为用户创建模型。假设您已经完成postgres并diesel-cli安装并正常工作。在您的终端中`echo DATABASE_URL=postgres://username:password@localhost/database_name > .env`，在设置时替换`database_name，username和password`。然后我们在终端跑`diesel setup`。这将创建我们的数据库并设置迁移目录等。

我们来写一些吧SQL。通过`diesel migration generate users`和创建迁移`diesel migration generate invitations`。在`migrations`文件夹中打开`up.sql`和`down.sql`文件，并分别添加以下sql。

```sql
--migrations/TIMESTAMP_users/up.sql
CREATE TABLE users (
  email VARCHAR(100) NOT NULL PRIMARY KEY,
  password VARCHAR(64) NOT NULL, --bcrypt hash
  created_at TIMESTAMP NOT NULL
);

--migrations/TIMESTAMP_users/down.sql
DROP TABLE users;

--migrations/TIMESTAMP_invitations/up.sql
CREATE TABLE invitations (
  id UUID NOT NULL PRIMARY KEY,
  email VARCHAR(100) NOT NULL,
  expires_at TIMESTAMP NOT NULL
);

--migrations/TIMESTAMP_invitations/down.sql
DROP TABLE invitations;
```

在您的终端中 `diesel migration run`将在DB和src/schema.rs文件中创建表。这将进行diesel和migrations。请阅读他们的文档以了解更多信息。

在这个阶段，我们已经在db中创建了表，让我们编写一些代码来创建`users`和`invitations`的表示。在models.rs我们添加以下内容。

```rust
// models.rs
...
// --- snip
use chrono::NaiveDateTime;
use uuid::Uuid;
use schema::{users,invitations};

#[derive(Debug, Serialize, Deserialize, Queryable, Insertable)]
#[table_name = "users"]
pub struct User {
    pub email: String,
    pub password: String,
    pub created_at: NaiveDateTime, // only NaiveDateTime works here due to diesel limitations
}

impl User {
    // this is just a helper function to remove password from user just before we return the value out later
    pub fn remove_pwd(mut self) -> Self {
        self.password = "".to_string();
        self
    }
}

#[derive(Debug, Serialize, Deserialize, Queryable, Insertable)]
#[table_name = "invitations"]
pub struct Invitation {
    pub id: Uuid,
    pub email: String,
    pub expires_at: NaiveDateTime,
}
```

检查您的实现是否没有错误/警告，并密切关注终端中的`cargo watch -x run`命令。

## 我们自己的错误响应类型

在我们开始为应用程序的各种路由实现处理程序之前，我们首先设置一般错误响应。它不是强制性要求，但随着您的应用程序的增长，将来可能会有用。

>Actix-web提供与`failure`库的自动兼容性，以便错误导出失败将自动转换为actix错误。请记住，这些错误将使用默认的500状态代码呈现，除非您还为它们提供了自己的`error_response（）`实现。

这将允许我们使用自定义消息发送http错误响应。创建`errors.rs`包含以下内容的文件。

```rust
// errors.rs
use actix_web::{error::ResponseError, HttpResponse};


#[derive(Fail, Debug)]
pub enum ServiceError {
    #[fail(display = "Internal Server Error")]
    InternalServerError,

    #[fail(display = "BadRequest: {}", _0)]
    BadRequest(String),
}

// impl ResponseError trait allows to convert our errors into http responses with appropriate data
impl ResponseError for ServiceError {
    fn error_response(&self) -> HttpResponse {
        match *self {
            ServiceError::InternalServerError => {
                HttpResponse::InternalServerError().json("Internal Server Error")
            },
            ServiceError::BadRequest(ref message) => HttpResponse::BadRequest().json(message),
        }
    }
}
```

不要忘记添加`mod errors`;到您的`main.rs`文件中以便能够使用自定义错误消息。

## 实现`handler`处理程序

我们希望我们的服务器从客户端收到一封电子邮件，并在数据库中的`invitations`表中创建。在此实现中，我们将向用户发送电子邮件。如果您没有设置电子邮件服务，则可以忽略电子邮件功能，只需使用服务器上的响应数据。

从actix文档：

>Actor通过发送消息与其他actor通信。在actix中，所有消息具有类型。消息可以是实现`Message trait`的任何Rust类型。

并且

>请求处理程序可以是实现`Handler trait`的任何对象。请求处理分两个阶段进行。首先调用handler对象，返回实现`Responder trait`的任何对象。然后，在返回的对象上调用`respond_to（）`，将自身转换为`AsyncResult`或`Error`。

让我们实现Handler这样的请求。首先创建一个新文件`src/invitation_handler.rs`并在其中创建以下结构。

```rust
// invitation_handler.rs
use actix::{Handler, Message};
use chrono::{Duration, Local};
use diesel::result::{DatabaseErrorKind, Error::DatabaseError};
use diesel::{self, prelude::*};
use errors::ServiceError;
use models::{DbExecutor, Invitation};
use uuid::Uuid;

#[derive(Debug, Deserialize)]
pub struct CreateInvitation {
    pub email: String,
}

// impl Message trait allows us to make use if the Actix message system and
impl Message for CreateInvitation {
    type Result = Result<Invitation, ServiceError>;
}

impl Handler<CreateInvitation> for DbExecutor {
    type Result = Result<Invitation, ServiceError>;

    fn handle(&mut self, msg: CreateInvitation, _: &mut Self::Context) -> Self::Result {
        use schema::invitations::dsl::*;
        let conn: &PgConnection = &self.0.get().unwrap();

        // creating a new Invitation object with expired at time that is 24 hours from now
        // this could be any duration from current time we will use it later to see if the invitation is still valid
        let new_invitation = Invitation {
            id: Uuid::new_v4(),
            email: msg.email.clone(),
            expires_at: Local::now().naive_local() + Duration::hours(24),
        };

        diesel::insert_into(invitations)
            .values(&new_invitation)
            .execute(conn)
            .map_err(|error| {
                println!("{:#?}",error); // for debugging purposes
                ServiceError::InternalServerError
            })?;

        let mut items = invitations
            .filter(email.eq(&new_invitation.email))
            .load::<Invitation>(conn)
            .map_err(|_| ServiceError::InternalServerError)?;

        Ok(items.pop().unwrap())
    }
}
```

不要忘记在`main.rs`文件中添加`mod invitation_handler`。

现在我们有一个处理程序来插入和返回DB的`invitations`。使用以下内容创建另一个文件。`register_email()`接收`CreateInvitation`结构和保存DB地址的状态。我们通过调用`into_inner（）`发送实际的`signup_invitation`结构。此函数以异步方式返回`invitations`或我们的`Handler`处理程序中定义的错误.

```rust
// invitation_routes.rs

use actix_web::{AsyncResponder, FutureResponse, HttpResponse, Json, ResponseError, State};
use futures::future::Future;

use app::AppState;
use invitation_handler::CreateInvitation;

pub fn register_email((signup_invitation, state): (Json<CreateInvitation>, State<AppState>))
    -> FutureResponse<HttpResponse> {
    state
        .db
        .send(signup_invitation.into_inner())
        .from_err()
        .and_then(|db_response| match db_response {
            Ok(invitation) => Ok(HttpResponse::Ok().json(invitation)),
            Err(err) => Ok(err.error_response()),
        }).responder()
}
```

## 测试你的服务器

你应该能够使用以下curl命令测试`http://localhost:3000/invitation`路由。

```rust
curl --request POST \
  --url http://localhost:3000/invitation \
  --header 'content-type: application/json' \
  --data '{"email":"test@test.com"}'
# result would look something like
{
    "id": "67a68837-a059-43e6-a0b8-6e57e6260f0d",
    "email": "test@test.com",
    "expires_at": "2018-10-23T09:49:12.167510"
}
```

## 结束第1部分

在下一部分中，我们将扩展我们的应用程序以生成电子邮件并将其发送给注册用户进行验证。我们还允许用户在验证后注册和验证。

[英文原文](https://hgill.io/posts/auth-microservice-rust-actix-web-diesel-complete-tutorial-part-1)
# Rust使用Actix-Web验证Auth Web微服务 - 第2部分

> [本文同步于Rust中文社区](http://rustlang-cn.org/read/rust/rust-use-actix-web-build-auth-micao-serive-2.html)

欢迎向Rust中文社区投稿,**[投稿地址](https://github.com/rustlang-cn/articles)**,好文将在以下地方直接展示

- 1 [Rust中文社区首页](https://rustlang-cn.org/)
- 2 Rust中文社区**Rust阅读文章栏目**
- 3 知乎专栏[Rust中文社区](https://zhuanlan.zhihu.com/rustlang-cn)
- 4 思否专栏[Rust中文社区](https://segmentfault.com/blog/rust-lang)
- 5 微博[Rustlang-cn](https://weibo.com/kriry)
- 6 简书专题[Rust中文社区](https://www.jianshu.com/c/2efae7198ea3)

## 验证用户电子邮件

从第一部分开始，我们现在拥有一个服务器，它从请求中获取一个电子邮件地址，并使用`invitation`(邀请)对象发出JSON响应。在第一部分中，我说我们将向用户发送一封电子邮件，经过一番思考和反馈，我们现在将跳过这一部分（请注意第3部分）。我使用的服务是sparkpost，你作为本教程的读者可能没有他们的帐户（免费用于小用途）。

**警告：如果没有正确的电子邮件验证，请不要在任何真实应用中使用此解决方法**

## 解决方法

现在我们将使用来自服务器的http响应来验证电子邮件。创建电子邮件验证的最简单方法是让我们的服务器使用通过电子邮件发送到用户电子邮件的某种秘密，并让他们单击带有秘密的链接进行验证。我们可以使用`UUID`邀请对象作为秘密。假设客户在使用uuid输入电子邮件后收到邀请`67a68837-a059-43e6-a0b8-6e57e6260f0d`。

我们可以发送请求`UUID`在网址中注册具有上述内容的新用户。我们的服务器可以获取该id并在数据库中找到Invitation对象，然后将到期日期与当前时间进行比较。如果所有这些条件都成立，我们将让用户注册，否则返回错误响应。`现在我们将邀请对象作为解决方法返回给客户端`。电子邮件支持将在`第3部分`中实现。

我们可以发送请求UUID在网址中注册具有上述内容的新用户。我们的服务器可以获取该id并在数据库中找到Invitation对象，然后将到期日期与当前时间进行比较。如果所有这些条件都成立，我们将让用户注册，否则返回错误响应。现在我们将邀请对象作为解决方法返回给客户端。电子邮件支持将在第3部分中实现。

## 错误处理和`FROM`Trait

Rust提供了非常强大的工具，我们可以使用它们将一种错误转换为另一种错误。在这个应用程序中，我们将使用不同的插入操作进行一些操作，即使用柴油保存数据，使用bcrypt保存密码等。这些操作可能会返回错误，但我们需要将它们转换为我们的自定义错误类型。`std::convert::From`是一个Trait，允许我们转换它。在[这里](https://doc.rust-lang.org/std/convert/trait.From.html)阅读更多有关`From`特征的信息。通过实现`From`特征，我们可以使用`?`运算符来[传播](https://doc.rust-lang.org/book/second-edition/ch09-02-recoverable-errors-with-result.html#a-shortcut-for-propagating-errors-the--operator)将转换为我们的`ServiceError`类型的许多不同类型的错误。

我们的错误定义在`errors.rs`，让我们通过为`uuid`和`diesel`错误添加impl `From`来实现一些`From`特性，我们还将为`ServiceError`枚举添加一个`Unauthorized`变量。该文件如下所示：

```rust
// errors.rs
use actix_web::{error::ResponseError, HttpResponse};
use std::convert::From;
use diesel::result::{DatabaseErrorKind, Error};
use uuid::ParseError;


#[derive(Fail, Debug)]
pub enum ServiceError {
    #[fail(display = "Internal Server Error")]
    InternalServerError,

    #[fail(display = "BadRequest: {}", _0)]
    BadRequest(String),

    #[fail(display = "Unauthorized")]
    Unauthorized,
}

// impl ResponseError trait allows to convert our errors into http responses with appropriate data
impl ResponseError for ServiceError {
    fn error_response(&self) -> HttpResponse {
        match *self {
            ServiceError::InternalServerError => HttpResponse::InternalServerError().json("Internal Server Error, Please try later"),
            ServiceError::BadRequest(ref message) => HttpResponse::BadRequest().json(message),
            ServiceError::Unauthorized => HttpResponse::Unauthorized().json("Unauthorized")
        }
    }
}

// we can return early in our handlers if UUID provided by the user is not valid
// and provide a custom message
impl From<ParseError> for ServiceError {
    fn from(_: ParseError) -> ServiceError {
        ServiceError::BadRequest("Invalid UUID".into())
    }
}

impl From<Error> for ServiceError {
    fn from(error: Error) -> ServiceError {
        // Right now we just care about UniqueViolation from diesel
        // But this would be helpful to easily map errors as our app grows
        match error {
            Error::DatabaseError(kind, info) => {
                if let DatabaseErrorKind::UniqueViolation = kind {
                    let message = info.details().unwrap_or_else(|| info.message()).to_string();
                    return ServiceError::BadRequest(message);
                }
                ServiceError::InternalServerError
            }
            _ => ServiceError::InternalServerError
        }
    }
}
```

这一切都将让我们做事变得方便。

## 得到一些帮助

我们有时需要一些帮助。在将密码存储到数据库之前，我们需要对密码进行哈希处理。在Reddit rust community有一个建议可以使用什么算法。在这里建议`argon2`。但为了简单起见，我决定使用`bcrypt`。bcrypt算法在生产中被广泛使用，并且`bcrypt` crate提供了一个非常好的接口来散列和验证密码。

为了将一些问题分开，我们创建一个新文件src/utils.rs并定义一个帮助程序哈希函数，如下所示。

```rust
//utils.rs
use bcrypt::{hash, DEFAULT_COST};
use errors::ServiceError;
use std::env;

pub fn hash_password(plain: &str) -> Result<String, ServiceError> {
    // get the hashing cost from the env variable or use default
    let hashing_cost: u32 = match env::var("HASH_ROUNDS") {
        Ok(cost) => cost.parse().unwrap_or(DEFAULT_COST),
        _ => DEFAULT_COST,
    };
    hash(plain, hashing_cost).map_err(|_| ServiceError::InternalServerError)
}
```

您可能已经注意到我们返回一个Result并使用`map_error（）`来返回我们的自定义错误。这是为了允许稍后在我们调用此函数时使用`?`运算符（另一种转换错误的方法是为`Frombcrypt`函数返回的错误实现特征）。

当我们在这里时，让我们为上一个教程`models.rs`中定义的`User`结构添加一个方便的方法。我们还删除了`remove_pwd（）`方法，而是定义了另一个SlimUser没有密码字段的结构。我们实现`From`trait来从`User`生成SlimUser。当我们使用它时，一切都会变得清晰。

```rust
use chrono::{NaiveDateTime, Local};
use std::convert::From;
//... snip
impl User {
    pub fn with_details(email: String, password: String) -> Self {
        User {
            email,
            password,
            created_at: Local::now().naive_local(),
        }
    }
}
//--snip
#[derive(Debug, Serialize, Deserialize)]
pub struct SlimUser {
    pub email: String,
}

impl From<User> for SlimUser {
    fn from(user: User) -> Self {
        SlimUser {
           email: user.email
        }
    }
}
```

不要忘记添加`extern crate bcrypt`;并`mod utils`在您的main.rs。我在第一部分忘记了另一个是登录到控制台。要启用它，请将以下内容添加到main.rs

```rust
extern crate env_logger;
// --snip

fn main(){
    dotenv().ok();
    std::env::set_var("RUST_LOG", "simple-auth-server=debug,actix_web=info");
    env_logger::init();
    //--snip
}
```

## 注册用户

如果您还记得上一个教程，我们为`Invitation`创建了一个`handler `程序，现在让我们创建一个注册用户的处理程序。我们将创建一个RegisterUser包含一些数据的结构，允许我们验证邀请，然后从数据库创建并返回一个用户。

创建一个新文件`src/register_handler.rs`并添加`mod register_handler`到您的文件中main.rs。

```rust
// register_handler.rs
use actix::{Handler, Message};
use chrono::Local;
use diesel::prelude::*;
use errors::ServiceError;
use models::{DbExecutor, Invitation, User, SlimUser};
use uuid::Uuid;
use utils::hash_password;

// UserData is used to extract data from a post request by the client
#[derive(Debug, Deserialize)]
pub struct UserData {
    pub password: String,
}

// to be used to send data via the Actix actor system
#[derive(Debug)]
pub struct RegisterUser {
    pub invitation_id: String,
    pub password: String,
}


impl Message for RegisterUser {
    type Result = Result<SlimUser, ServiceError>;
}


impl Handler<RegisterUser> for DbExecutor {
    type Result = Result<SlimUser, ServiceError>;
    fn handle(&mut self, msg: RegisterUser, _: &mut Self::Context) -> Self::Result {
        use schema::invitations::dsl::{invitations, id};
        use schema::users::dsl::users;
        let conn: &PgConnection = &self.0.get().unwrap();

        // try parsing the string provided by the user as url parameter
        // return early with error that will be converted to ServiceError
        let invitation_id = Uuid::parse_str(&msg.invitation_id)?;

        invitations.filter(id.eq(invitation_id))
            .load::<Invitation>(conn)
            .map_err(|_db_error| ServiceError::BadRequest("Invalid Invitation".into()))
            .and_then(|mut result| {
                if let Some(invitation) = result.pop() {
                    // if invitation is not expired
                    if invitation.expires_at > Local::now().naive_local() {
                        // try hashing the password, else return the error that will be converted to ServiceError
                        let password: String = hash_password(&msg.password)?;
                        let user = User::with_details(invitation.email, password);
                        let inserted_user: User = diesel::insert_into(users)
                            .values(&user)
                            .get_result(conn)?;

                        return Ok(inserted_user.into()); // convert User to SlimUser
                    }
                }
                Err(ServiceError::BadRequest("Invalid Invitation".into()))
            })
    }
}
```

## 解析URL参数

actix-web有许多简单的方法可以从请求中提取数据。其中一种方法是使用Path提取器。

[Path](https://actix.rs/actix-web/actix_web/struct.Path.html)提供可从Request的路径中提取的信息。您可以从路径反序列化任何变量段。

这将允许我们为每个要注册为用户的邀请创建唯一路径。

让我们修改`app.rs`文件中的寄存器路由，并添加一个稍后我们将实现的处理函数。

```rust
// app.rs
/// creates and returns the app after mounting all routes/resources
// add use statement at the top.
use register_routes::register_user;
//...snip
pub fn create_app(db: Addr<DbExecutor>) -> App<AppState> {
    App::with_state(AppState { db })
        //... snip
        // routes to register as a user
        .resource("/register/{invitation_id}", |r| {
           r.method(Method::POST).with(register_user);
        })

}
```

您可能希望暂时注释更改，因为事情未实现并保持您的应用程序已编译并运行。（我尽可能地做，持续反馈）。

我们现在需要的是实现`register_user（）`函数，该函数从客户端发送的请求中提取数据，通过向`RegisterUserActor` 发送消息来调用处理程序。除了url参数，我们还需要从客户端提取密码。我们已经为此创建了一个`UserData`结构体在`register_handler.rs`。我们将使用类型`Json`来创建`UserData`结构。

>Json允许将请求主体反序列化为结构体。要从请求的主体中提取类型信息，类型T必须实现serde的反序列化特征。

创建一个新文件`src/register_routes.rs`并添加`mod register_routes`到您的文件中main.rs。

```rust
use actix_web::{AsyncResponder, FutureResponse, HttpResponse, ResponseError, State, Json, Path};
use futures::future::Future;

use app::AppState;
use register_handler::{RegisterUser, UserData};


pub fn register_user((invitation_id, user_data, state): (Path<String>, Json<UserData>, State<AppState>))
                     -> FutureResponse<HttpResponse> {
    let msg = RegisterUser {
        // into_inner() returns the inner string value from Path
        invitation_id: invitation_id.into_inner(),
        password: user_data.password.clone(),
    };

    state.db.send(msg)
        .from_err()
        .and_then(|db_response| match db_response {
            Ok(slim_user) => Ok(HttpResponse::Ok().json(slim_user)),
            Err(service_error) => Ok(service_error.error_response()),
        }).responder()
}
```

## 测试您的实现

如果你有任何错误，在处理完错误后，让我们给它一个测试

```rust
curl --request POST \
  --url http://localhost:3000/invitation \
  --header 'content-type: application/json' \
  --data '{
	"email":"name@domain.com"
}'
```

应该返回类似的东西

```rust
{
  "id": "f87910d7-0e33-4ded-a8d8-2264800d1783",
  "email": "name@domain.com",
  "expires_at": "2018-10-27T13:02:00.909757"
}
```

想象一下，我们通过创建一个链接来向用户发送电子邮件，该链接包含一个供用户填写的表单。从那里我们会让我们的客户向`http：// localhost：3000 / register / f87910d7-0e33-4ded-a8d8-2264800d1783`发送请求。为了演示本演示，您可以使用以下测试命令测试您的应用程序。

```rust
curl --request POST \
  --url http://localhost:3000/register/f87910d7-0e33-4ded-a8d8-2264800d1783 \
  --header 'content-type: application/json' \
  --data '{"password":"password"}'
```

应该返回类似的东西

```rust
{
  "email": "name@domain.com"
}
```

再次运行该命令将导致错误

```rust
"Key (email)=(name@domain.com) already exists."
```

恭喜您现在拥有一个可以邀请，验证和创建用户的Web服务，甚至可以向您发送半有用的错误消息。🎉🎉

## 我们来做AUTH

根据w3.org：

>基于令牌的身份验证系统背后的一般概念很简单。允许用户输入用户名和密码以获取允许他们获取特定资源的令牌 - 而无需使用他们的用户名和密码。一旦获得其令牌，用户就可以向远程站点提供令牌 - 其提供对特定资源的访问一段时间。

现在，您如何选择交换该令牌可能会产生安全隐患。您会在互联网上找到许多讨论/辩论以及人们使用的许多方式。我非常警惕在客户端存储可由客户端JavaScript访问的东西。不幸的是，这种方法在各地成千上万的教程中提出 这是一个很好的阅读停止使用JWT进行会话。

这里我不确定，除了你之外，作为读者还有什么建议你`don't follow online tutorials blindly and do your own research`。本教程的目的是了解Actix-web和rust，而不是如何防止服务器漏洞。为了本教程的目的，我们将仅使用http的cookie来交换令牌。

**请不要在生产中使用。**

现在，这就是😰，让我们看看我们能在这里做些什么。actix-web为我们提供了一种巧妙的方法，作为处理会话cookie的中间件，这里记录了[actix_web :: middleware :: identity](https://actix.rs/actix-web/actix_web/middleware/identity/index.html)。要启用此功能，我们修改app.rs文件如下。

```rust
use actix_web::middleware::identity::{CookieIdentityPolicy, IdentityService};
use chrono::Duration;
//--snip
pub fn create_app(db: Addr<DbExecutor>) -> App<AppState> {
    // secret is a random 32 character long base 64 string
    let secret: String = std::env::var("SECRET_KEY").unwrap_or_else(|_| "0".repeat(32));
    let domain: String = std::env::var("DOMAIN").unwrap_or_else(|_| "localhost".to_string());

    App::with_state(AppState { db })
        .middleware(Logger::default())
        .middleware(IdentityService::new(
            CookieIdentityPolicy::new(secret.as_bytes())
                .name("auth")
                .path("/")
                .domain(domain.as_str())
                .max_age(Duration::days(1)) // just for testing
                .secure(false),
        ))
        //--snip
}
```

很方便的方法，如`req.remember(data)`，`req.identity()`和`req.forget()`等操作HttpRequest的路由参数。这反过来将设置和删除客户端的cookie身份验证。

## JWT

在编写本教程时，我遇到了一些关于要使用什么`JWT` lib的讨论。从一个简单的搜索我发现了一些，并决定与`frank_jwt`一起，然后文森特指出不完整性，建议使用`jsonwebtoken`。使用该lib后遇到问题我得到了很好的响应。现在repo有工作示例，我能够实现以下默认解决方案。请注意，这不是JWT最安全的实现，您可能希望查找资源以使其更好地满足您的需求。

在我们创建auth处理程序和路由函数之前，让我们为util.rs添加一些jwt编码和解码辅助函数。别忘了加入`extern crate jsonwebtoken as jwt`在你的main.rs。

如果有人有更好的实施，我会很乐意接受。

```rust
// utils.rs
use models::SlimUser;
use std::convert::From;
use jwt::{decode, encode, Header, Validation};
use chrono::{Local, Duration};
//--snip

#[derive(Debug, Serialize, Deserialize)]
struct Claims {
    // issuer
    iss: String,
    // subject
    sub: String,
    //issued at
    iat: i64,
    // expiry
    exp: i64,
    // user email
    email: String,
}

// struct to get converted to token and back
impl Claims {
    fn with_email(email: &str) -> Self {
        Claims {
            iss: "localhost".into(),
            sub: "auth".into(),
            email: email.to_owned(),
            iat: Local::now().timestamp(),
            exp: (Local::now() + Duration::hours(24)).timestamp(),
        }
    }
}

impl From<Claims> for SlimUser {
    fn from(claims: Claims) -> Self {
        SlimUser { email: claims.email }
    }
}

pub fn create_token(data: &SlimUser) -> Result<String, ServiceError> {
    let claims = Claims::with_email(data.email.as_str());
    encode(&Header::default(), &claims, get_secret().as_ref())
        .map_err(|_err| ServiceError::InternalServerError)
}

pub fn decode_token(token: &str) -> Result<SlimUser, ServiceError> {
    decode::<Claims>(token, get_secret().as_ref(), &Validation::default())
        .map(|data| Ok(data.claims.into()))
        .map_err(|_err| ServiceError::Unauthorized)?
}

// take a string from env variable
fn get_secret() -> String {
    env::var("JWT_SECRET").unwrap_or("my secret".into())
}
```

## 验证处理

让我们创建一个新文件`src/auth_handler.rs`并给你main.rs添加`mod auth_handler`。

```rust
//auth_handler.rs
use actix::{Handler, Message};
use diesel::prelude::*;
use errors::ServiceError;
use models::{DbExecutor, User, SlimUser};
use bcrypt::verify;
use actix_web::{FromRequest, HttpRequest, middleware::identity::RequestIdentity};

#[derive(Debug, Deserialize)]
pub struct AuthData {
    pub email: String,
    pub password: String,
}

impl Message for AuthData {
    type Result = Result<SlimUser, ServiceError>;
}


impl Handler<AuthData> for DbExecutor {
    type Result = Result<SlimUser, ServiceError>;
    fn handle(&mut self, msg: AuthData, _: &mut Self::Context) -> Self::Result {
        use schema::users::dsl::{users, email};
        let conn: &PgConnection = &self.0.get().unwrap();
        let mismatch_error = Err(ServiceError::BadRequest("Username and Password don't match".into()));

        let mut items = users
            .filter(email.eq(&msg.email))
            .load::<User>(conn)?;

        if let Some(user) = items.pop() {
            match verify(&msg.password, &user.password) {
                Ok(matching) => {
                    if matching { return Ok(user.into()); } else { return mismatch_error; }
                }
                Err(_) => { return mismatch_error; }
            }
        }
        mismatch_error
    }
}
```

上面的处理程序采用`AuthData`包含客户端发送的电子邮件和密码的结构。我们使用该电子邮件从数据库中提取用户并使用bcrypt `verify`函数来匹配密码。如果一切顺利，我们返回用户或我们返回`BadRequest`错误。

现在让我们创建`src/auth_routes.rs`以下内容：

```rust
// auth_routes.rs
use actix_web::{AsyncResponder, FutureResponse, HttpResponse, HttpRequest, ResponseError, Json};
use actix_web::middleware::identity::RequestIdentity;
use futures::future::Future;
use utils::create_token;

use app::AppState;
use auth_handler::AuthData;

pub fn login((auth_data, req): (Json<AuthData>, HttpRequest<AppState>))
             -> FutureResponse<HttpResponse> {
    req.state()
        .db
        .send(auth_data.into_inner())
        .from_err()
        .and_then(move |res| match res {
            Ok(slim_user) => {
                let token = create_token(&slim_user)?;
                req.remember(token);
                Ok(HttpResponse::Ok().into())
            }
            Err(err) => Ok(err.error_response()),
        }).responder()
}

pub fn logout(req: HttpRequest<AppState>) -> HttpResponse {
    req.forget();
    HttpResponse::Ok().into()
}
```

我们的login方法提取`AuthData from请求并向我们在auth_handler.rs中实现的`DbEexcutor`Actor处理程序发送消息。如果一切都很好，我们会让用户返回给我们，我们使用我们之前在`utils.rs`中定义的辅助函数来创建一个令牌和调用`req.remember(token`)。这又设置了一个带有令牌的cookie头，供客户端保存。

我们现在需要做的最后一件事是`app.rs`使用我们的登录/注销功能。将`.rsource("/auth")`闭包更改为以下内容：

```rust
.resource("/auth", |r| {
            r.method(Method::POST).with(login);
            r.method(Method::DELETE).with(logout);
        })
```

不要忘记在文件的顶部添加`use auth_routes::{login, logout};`

## 试运行AUTH

如果您一直关注本教程，那么您已经创建了一个使用电子邮件和密码的用户。使用以下curl命令测试我们的服务器。

```rust
curl -i --request POST \
  --url http://localhost:3000/auth \
  --header 'content-type: application/json' \
  --data '{
        "email": "name@domain.com",
        "password":"password"
}'

## response
HTTP/1.1 200 OK
set-cookie: auth=iqsB4KUUjXUjnNRl1dVx9lKiRfH24itiNdJjTAJsU4CcaetPpaSWfrNq6IIoVR5+qKPEVTrUeg==; HttpOnly; Path=/; Domain=localhost; Max-Age=86400
content-length: 0
date: Sun, 28 Oct 2018 12:36:43 GMT
```

如果你收到了如上所述的带有set-cookie标头的200响应，恭喜你已成功登录。

为了测试注销，我们向/auth它发送一个DELETE请求，确保你得到带有空数据和即时到期日的set-cookie头。

```rust
curl -i --request DELETE \
  --url http://localhost:3000/auth

## response
HTTP/1.1 200 OK
set-cookie: auth=; HttpOnly; Path=/; Domain=localhost; Max-Age=0; Expires=Fri, 27 Oct 2017 13:01:52 GMT
content-length: 0
date: Sat, 27 Oct 2018 13:01:52 GMT
```

## 实现受保护的路由

使Auth的全部意义在于验证请求是否来自经过身份验证的客户端。Actix-web有一个特性`FromRequest`，我们可以在任何类型上实现，然后使用它从请求中提取数据。见文档[这里](https://actix.rs/actix-web/actix_web/trait.FromRequest.html)。我们将在`auth_handler.rs`底部添加以下内容。

```rust
//auth_handler.rs
//--snip
use actix_web::FromRequest;
use utils::decode_token;
//--snip

// we need the same data as SlimUser
// simple aliasing makes the intentions clear and its more readable
pub type LoggedUser = SlimUser;

impl<S> FromRequest<S> for LoggedUser {
    type Config = ();
    type Result = Result<LoggedUser, ServiceError>;
    fn from_request(req: &HttpRequest<S>, _: &Self::Config) -> Self::Result {
        if let Some(identity) = req.identity() {
            let user: SlimUser = decode_token(&identity)?;
            return Ok(user as LoggedUser);
        }
        Err(ServiceError::Unauthorized)
    }
}
```

我们选择使用类型别名而不是创建一个全新的类型。当我们LoggedUser从请求中提取时，读者会知道它是经过身份验证的用户。`FromRequest` trait只是尝试将cookie中的字符串反序列化为我们的结构，如果失败则只返回`Unauthorized`错误。为了测试这个，我们需要添加一个实际路由或app。我们只是`auth_routes.rs`添加另一个函数

```rust
//auth_routes.rs
//--snip

pub fn get_me(logged_user: LoggedUser) -> HttpResponse {
    HttpResponse::Ok().json(logged_user)
}
```

要调用它，我们在app.rs资源中注册此方法。它看起来像是以下。

```rust
//app.rs
use auth_routes::{login, logout, get_me};
//--snip

.resource("/auth", |r| {
    r.method(Method::POST).with(login);
    r.method(Method::DELETE).with(logout);
    r.method(Method::GET).with(get_me);
})
//--snip
```

## 测试登录用户

在终端中尝试以下Curl命令。

```rust
curl -i --request POST \
  --url http://localhost:3000/auth \
  --header 'content-type: application/json' \
  --data '{
        "email": "name@domain.com",
        "password":"password"
}'
# result would be something like
HTTP/1.1 200 OK
set-cookie: auth=HdS0iPKTBL/4MpTmoUKQ5H7wft5kP7OjP6vbyd05Ex5flLvAkKd+P2GchG1jpvV6p9GQtzPEcg==; HttpOnly; Path=/; Domain=localhost; Max-Age=86400
content-length: 0
date: Sun, 28 Oct 2018 19:16:12 GMT

## and then pass the cookie back for a get request
curl -i --request GET \
  --url http://localhost:3000/auth \
  --cookie auth=HdS0iPKTBL/4MpTmoUKQ5H7wft5kP7OjP6vbyd05Ex5flLvAkKd+P2GchG1jpvV6p9GQtzPEcg==
## result
HTTP/1.1 200 OK
content-length: 27
content-type: application/json
date: Sun, 28 Oct 2018 19:21:04 GMT

{"email":"name@domain.com"}
```

它应该以json的形式成功返回您的电子邮件。只有登录的用户或具有有效cookie身份验证和令牌的请求才会通过您提取的`LoggedUser`路由。

## 下一步是什么

在本教程的第3部分中，我们将为此应用程序创建**电子邮件验证和前端**。我希望使用某种rust的html模板系统。与此同时，我正在学习Angular，所以我可能会在它前面做一个小应用程序。

[英文原文](https://hgill.io/posts/auth-microservice-rust-actix-web-diesel-complete-tutorial-part-2/)

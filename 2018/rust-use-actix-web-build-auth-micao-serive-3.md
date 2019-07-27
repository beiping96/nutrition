# Rust使用Actix-Web验证Auth Web微服务-整教程第3部分

> 本文同步与[Rust中文社区](https://rustlang-cn.org/read/rust/2018/rust-use-actix-web-build-auth-micao-serive-3.html)
> 时间：2018-11-27，作者：[krircc](https://krircc.github.io/)， 简介：天青色  

欢迎向Rust中文社区投稿,**[投稿地址](https://github.com/rustlang-cn/articles)**,好文将在以下地方直接展示

- 1 [Rust中文社区首页](https://rustlang-cn.org/)
- 2 Rust中文社区**Rust阅读文章栏目**
- 3 知乎专栏[Rust中文社区](https://zhuanlan.zhihu.com/rustlang-cn)
- 4 思否专栏[Rust中文社区](https://segmentfault.com/blog/rust-lang)
- 5 微博[Rustlang-cn](https://weibo.com/kriry)
- 6 简书专题[Rust中文社区](https://www.jianshu.com/c/2efae7198ea3)

## 更新我们的CARGO.TOML

Rust生态系统正在快速发展，在几个星期内，我们的许多箱子都在上游更新，包括我的sparkpost箱子（稍后会详细介绍）。没有任何延迟，我们只需更新cargo.toml文件中的以下条件箱。

```rust
[dependencies]
actix = "0.7.7"
actix-web = "0.7.14"
env_logger = "0.6.0"
r2d2 = "0.8.3"
sparkpost = "0.5.2"
```

## 使用SPARKPOST发送注册电子邮件

请随意使用您喜欢的任何电子邮件服务（除个人使用外，我与`sparkpost`没有关联），只要您能够复制已发送的电子邮件即可。现在，您需要在`.env`文件中添加以下内容。

```rust
SPARKPOST_API_KEY='yourapikey'
SENDING_EMAIL_ADDRESS='register@yourdomain.com'
```

Api密钥是从sparkpost帐户获得的，只要您拥有可以控制的域名，就可以免费创建一个。要处理电子邮件，我们创建一个文件`email_service.rs`并添加以下内容。

```rust
use models::Invitation;
use sparkpost::transmission::{
    EmailAddress, Message, Options, Recipient, Transmission, TransmissionResponse,
};

fn get_api_key() -> String {
    std::env::var("SPARKPOST_API_KEY").expect("SPARKPOST_API_KEY must be set")
}

#[allow(unused)]
pub fn send_invitation(invitation: &Invitation) {
    let tm = Transmission::new_eu(get_api_key());
    let sending_email =
        std::env::var("SENDING_EMAIL_ADDRESS").expect("SENDING_EMAIL_ADDRESS must be set");
    // new email message with sender name and email
    let mut email = Message::new(EmailAddress::new(sending_email, "Let's Organise"));

    // set options for a transactional email for now
    let options = Options {
        open_tracking: false,
        click_tracking: false,
        transactional: true,
        sandbox: false,
        inline_css: false,
        start_time: None,
    };

    // recipient from the invitation email
    let recipient: Recipient = invitation.email.as_str().into();

    let email_body = format!(
        "Please click on the link below to complete registration. <br/>
         <a href=\"http://localhost:3000/register.html?id={}&email={}\">
         http://localhost:3030/register</a> <br>
         your Invitation expires on <strong>{}</strong>",
        invitation.id,
        invitation.email,
        invitation
            .expires_at
            .format("%I:%M %p %A, %-d %B, %C%y")
            .to_string()
    );


    // complete the email message with details
    email
        .add_recipient(recipient)
        .options(options)
        .subject("You have been invited to join Simple-Auth-Server Rust")
        .html(email_body);

    let result = tm.send(&email);

    match result {
        Ok(res) => {
            // println!("{:?}", &res);
            match res {
                TransmissionResponse::ApiResponse(api_res) => {
                    println!("API Response: \n {:#?}", api_res);
                    //   assert_eq!(1, api_res.total_accepted_recipients);
                    //   assert_eq!(0, api_res.total_rejected_recipients);
                }
                TransmissionResponse::ApiError(errors) => {
                    println!("Response Errors: \n {:#?}", &errors);
                }
            }
        }
        Err(error) => {
            println!("error \n {:#?}", error);
        }
    }
}
```

为了能够在我们的应用程序中使用此服务，我们在`main.rs`文件中添加`extern crate sparkpost;`和`mod email_service;`。请注意，我们不会从`send_invitation`函数返回任何内容。这取决于您在真实应用中想要做什么，现在我们只需登录终端。

## 调整您的邀请处理

在上一个教程中，我们以一种返回对象本身的方式实现了我们的邀请。让我们改变它并将电子邮件发送给用户。在我们`invitation_routes.rs`中我们邀请通过调用`send_invitation`函数。代码如下所示。

```rust
use actix_web::{AsyncResponder, FutureResponse, HttpResponse, Json, ResponseError, State};
use futures::future::Future;

use app::AppState;
use email_service::send_invitation;
use invitation_handler::CreateInvitation;

pub fn register_email(
 (signup_invitation, state): (Json<CreateInvitation>, State<AppState>),
) -> FutureResponse<HttpResponse> {
 state
     .db
     .send(signup_invitation.into_inner())
     .from_err()
     .and_then(|db_response| match db_response {
         Ok(invitation) => {
             send_invitation(&invitation);
             Ok(HttpResponse::Ok().into())
         }
         Err(err) => Ok(err.error_response()),
     }).responder()
}
```

## 设置模拟前端进行测试

我不打算详细介绍前端设置。在本教程中，我创建用以作为静态文件的一些`HTML / CSS / JS`[文件](https://gitlab.com/mygnu/rust-auth-server/tree/master/static)的目的为了方便。你可以简单地将这个文件夹复制到你的代码的根目录作为'static /'，或者狂野地做一个`react / vue / angular`前端。

我们将稍微改变我们的路由以满足我们的需求，并将静态文件路由从业务逻辑路由中分离出来。我决定使用在根级别提供的静态文件，并将所有应用程序路径移动到`/api`前缀。为此，我们将`app.rs`文件修改为以下内容。

```rust
use actix::prelude::*;
use actix_web::middleware::identity::{CookieIdentityPolicy, IdentityService};
use actix_web::{fs, http::Method, middleware::Logger, App};
use auth_routes::{get_me, login, logout};
use chrono::Duration;
use invitation_routes::register_email;
use models::DbExecutor;
use register_routes::register_user;

pub struct AppState {
    pub db: Addr<DbExecutor>,
}

/// creates and returns the app after mounting all routes/resources
pub fn create_app(db: Addr<DbExecutor>) -> App<AppState> {
    // secret is a random minimum 32 bytes long base 64 string
    let secret: String = std::env::var("SECRET_KEY").unwrap_or_else(|_| "0123".repeat(8));
    let domain: String = std::env::var("DOMAIN").unwrap_or_else(|_| "localhost".to_string());

    App::with_state(AppState { db })
        .middleware(Logger::default())
        .middleware(IdentityService::new(
            CookieIdentityPolicy::new(secret.as_bytes())
                .name("auth")
                .path("/")
                .domain(domain.as_str())
                .max_age(Duration::days(1))
                .secure(false), // this can only be true if you have https
        ))
        // everything under '/api/' route
        .scope("/api", |api| {
            // routes for authentication
            api.resource("/auth", |r| {
                r.method(Method::POST).with(login);
                r.method(Method::DELETE).with(logout);
                r.method(Method::GET).with(get_me);
            })
            // routes to invitation
            .resource("/invitation", |r| {
                r.method(Method::POST).with(register_email);
            })
            // routes to register as a user after the
            .resource("/register/{invitation_id}", |r| {
                r.method(Method::POST).with(register_user);
            })
        })
        // serve static files
        .handler(
            "/",
            fs::StaticFiles::new("./static/")
                .unwrap()
                .index_file("index.html"),
        )
}
```

请注意封闭所有路由的`scope`方法。这就是设置电子邮件验证和简单前端所需的全部内容。

[英文原文](https://hgill.io/posts/auth-microservice-rust-actix-web-diesel-complete-tutorial-part-3)

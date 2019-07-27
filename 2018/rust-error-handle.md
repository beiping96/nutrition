![img](https://images.wallpaperscraft.com/image/artist_waves_colorful_129158_1366x768.jpg)

# Rust错误处理

> 本文同步于[Rust中文社区专栏文章：Rust错误处理](https://rustlang-cn.org/read/rust/2018/rust-error-handle.html) ,本文时间：2018-12-14, 译者：[krircc](https://krircc.github.io/)，简介：天青色，[原文出处](https://medium.com/learning-rust/rust-error-handling-72a8e036dd3)
> 
[欢迎加入](https://github.com/rustlang-cn/Important/issues/1)Rust中文社区,共建Rust语言中文网络！欢迎向Rust中文社区专栏投稿,[投稿地址](https://github.com/rustlang-cn/articles) ,好文在以下地方直接展示, 欢迎访问[Rust中文论坛](https://github.com/rustlang-cn/forum/issues)，QQ群：570065685

1. [Rust中文社区首页](https://rustlang-cn.org)
2. Rust中文社区[文章栏目](https://rustlang-cn.org/read/rust/)
3. 知乎专栏[Rust中文社区](https://zhuanlan.zhihu.com/rustlang-cn)
4. 思否专栏[Rust中文社区](https://segmentfault.com/blog/rust-lang)
5. 简书专题[Rust中文社区](https://www.jianshu.com/c/2efae7198ea3)
6. 微博[Rustlang-cn](https://weibo.com/kriry)

## 智能编译器

Rust编译器最重要的工作是防止Rust程序中的错误。如果代码没有正确遵循内存管理规则或生命周期注释，它会在编译时分析代码并发出警告。例如，

```rust
#[allow(unused_variables)] //💡 A lint attribute used to suppress the warning; unused variable: `b`
fn main() {
    let a = vec![1, 2, 3];
    let b = a;

    println!("{:?}", a);
}


// ------ Compile time error ------
error[E0382]: use of moved value: `a`
 --> src/main.rs:6:22
  |
3 |     let b = a;
  |         - value moved here
4 |
5 |     println!("{:?}", a);
  |                      ^ value used here after move
  |
  = note: move occurs because `a` has type `std::vec::Vec<i32>`, which does not implement the `Copy` trait

error: aborting due to previous error
For more information about this error, try `rustc --explain E0382`.

// ⭐ instead using #[allow(unused_variables)], consider using "let _b = a;" in line 4. 
// Also you can use "let _ =" to completely ignore return values
```

Rust编译器不仅检查与生命周期或内存管理相关的问题，还检查常见的编码错误，如下面的代码。

```rust
struct Color {
    r: u8,
    g: u8,
    b: u8,
}

fn main() {
    let yellow = Color {
        r: 255,
        g: 255,
        d: 0,
    };

    println!("Yellow = rgb({},{},{})", yellow.r, yellow.g, yellow.b);
}


// ------------ Compile time error ------------
error[E0560]: struct `Color` has no field named `d`
  --> src/main.rs:11:9
   |
11 |         d: 0,
   |         ^ field does not exist - did you mean `b`?

error: aborting due to previous error
For more information about this error, try `rustc --explain E0560`.
```

以上错误消息非常具有描述性，我们可以很容易地看出错误在哪里。但是，虽然我们无法通过错误消息识别问题，但`rustc --explain` 命令通过显示表达相同问题的简单代码示例以及我们必须使用的解决方案来帮助我们**识别错误类型以及如何解决它**。例如，在控制台中显示以下输出。`rustc --explain E0571`

```rust
// A `break` statement with an argument appeared in a non-`loop` loop.

// Example of erroneous code:

let result = while true {
    if satisfied(i) {
        break 2*i; // error: `break` with value from a `while` loop
    }
    i += 1;
};

// The `break` statement can take an argument (which will be the value of the loop expression if the `break` statement is executed) in `loop` loops, but not `for`, `while`, or `while let` loops.

Make sure `break value;` statements only occur in `loop` loops:
let result = loop { // ok!
    if satisfied(i) {
        break 2*i;
    }
    i += 1;
};
```

💡您也可以通过`Rust Compiler Error Index`阅读相同的解释 。例如，要检查`E0571`错误的解释，您可以使用`https://doc.rust-lang.org/error-index.html#E0571 `

## Panicking 

### panic!()

▸ 在某些情况下，当发生错误时，我们无法做任何事情来处理它，如果错误是某种情况，那就不应该发生。换句话说，如果这是一个不可恢复的错误。

▸ 当我们不使用功能丰富的调试器或正确的日志时，有时我们需要通过打印特定的消息或变量绑定的值从特定的代码行退出程序来调试代码以了解当前的程序的流程。

对于上述情况，我们可以使用`panic!`宏。让我们看几个例子。

> ⭐ `panic!()` 运行基于线程。一个线程可能会被恐慌，而其他线程正在运行。

01.从特定行退出。

```rust
fn main() {
    // some code

    // if we need to debug in here
    panic!();
}

// -------------- Compile time error --------------
thread 'main' panicked at 'explicit panic', src/main.rs:5:5
```

02.退出并显示自定义错误消息。

```rust
#[allow(unused_mut)] // 💡 A lint attribute used to suppress the warning; username variable does not need to be mutable
fn main() {
    let mut username = String::new();

    // some code to get the name
  
    if username.is_empty() {
        panic!("Username is empty!");
    }

    println!("{}", username);
}

// -------------- Compile time error --------------
thread 'main' panicked at 'Username is empty!', src/main.rs:8:9
```

03.退出附带代码元素的值。

```rust
#[derive(Debug)] // 💡 A lint attribute which use to implement `std::fmt::Debug` to Color
struct Color {
    r: u8,
    g: u8,
    b: u8,
}

#[allow(unreachable_code)] // 💡 A lint attribute used to suppress the warning; unreachable statement
fn main() {
    let some_color: Color;
    
    // some code to get the color. ex
    some_color = Color {r: 255, g: 255, b: 0};

    // if we need to debug in here
    panic!("{:?}", some_color);

    println!(
        "The color = rgb({},{},{})",
        some_color.r, some_color.g, some_color.b
    );
}

// -------------- Compile time error --------------
thread 'main' panicked at 'Color { r: 255, g: 255, b: 0 }', src/main.rs:16:5
```

正如您在上面的示例中所看到的，`panic!()`支持`println!()`类型样式参数  。默认情况下，它会输出错误消息，文件路径以及发生错误的行号和列号。

### unimplemented!()

如果您的代码具有未完成的代码段，则有一个标准化宏`unimplemented!()`来标记这些路径。如果程序通过这些路径运行，程序将`panicked`并返回"not yet implemented"的错误消息。

```rust
// error messages with panic!()
thread 'main' panicked at 'explicit panic', src/main.rs:6:5
thread 'main' panicked at 'Username is empty!', src/main.rs:9:9
thread 'main' panicked at 'Color { r: 255, g: 255, b: 0 }', src/main.rs:17:5

// error messages with unimplemented!()
thread 'main' panicked at 'not yet implemented', src/main.rs:6:5
thread 'main' panicked at 'not yet implemented: Username is empty!', src/main.rs:9:9
thread 'main' panicked at 'not yet implemented: Color { r: 255, g: 255, b: 0 }', src/main.rs:17:5
```

### unreachable!()

这是标记程序不应输入的路径的标准宏。如果程序进入这些路径，程序将`panicked`并返回"'internal error: entered unreachable code'"错误消息。

```rust
fn main() {
    let level = 22;
    let stage = match level {
        1...5 => "beginner",
        6...10 => "intermediate",
        11...20 => "expert",
        _ => unreachable!(),
    };

    println!("{}", stage);
}


// -------------- Compile time error --------------
thread 'main' panicked at 'internal error: entered unreachable code', src/main.rs:7:20
```

我们也可以为此设置自定义错误消息。

```rust
// --- with a custom message ---
_ => unreachable!("Custom message"),
// -------------- Compile time error --------------
thread 'main' panicked at 'internal error: entered unreachable code: Custom message', src/main.rs:7:20


// --- with debug data ---
_ => unreachable!("level is {}", level),
// -------------- Compile time error --------------
thread 'main' panicked at 'internal error: entered unreachable code: level is 22', src/main.rs:7:14
```

### assert!(), assert_eq!(), assert_ne!()

这些是标准宏，通常与测试断言一起使用。

- assert!()确保布尔表达式为true。如果表达式为false，则会发生`panics`。

```rust
fn main() {
    let f = false;

    assert!(f)
}


// -------------- Compile time error --------------
thread 'main' panicked at 'assertion failed: f', src/main.rs:4:5
```

- assert_eq!()确保两个表达式相等。如果表达式不相等则会发生`panics`。

```rust
fn main() {
    let a = 10;
    let b = 20;

    assert_eq!(a, b);
}


// -------------- Compile time error --------------
thread 'main' panicked at 'assertion failed: `(left == right)`
  left: `10`,
 right: `20`', src/main.rs:5:5
```

- assert_ne!()确保两个表达式不相等。如果表达式相等，它会发生`panics`。

```rust
fn main() {
    let a = 10;
    let b = 10;

    assert_ne!(a, b);
}


// -------------- Compile time error --------------
thread 'main' panicked at 'assertion failed: `(left != right)`
  left: `10`,
 right: `10`', src/main.rs:5:5
```

> ⭐使用表达式assert_ne!()和assert_eq!()应返回相同的数据类型。

我们也可以为这些宏设置自定义错误消息。举些例子，

1. 带有自定义消息 assert_eq!()

```rust
fn main() {
    let a = 10;
    let b = 20;

    assert_eq!(a, b, "a and b should be equal");
}


// -------------- Compile time error --------------
thread 'main' panicked at 'assertion failed: `(left == right)`
  left: `10`,
 right: `20`: a and b should be equal', src/main.rs:5:5
```

2. assert_eq!()带有调试数据

```rust
fn main() {
    let a = 10;
    let b = 20;
    
    let c = 40;

    assert_eq!(a+b, c, "a = {} ; b = {}", a, b);
}

// -------------- Compile time error --------------
thread 'main' panicked at 'assertion failed: `(left == right)`
  left: `30`,
 right: `40`: a = 10 ; b = 20', src/main.rs:7:5
```

### debug_assert!(), debug_assert_eq!(), debug_assert_ne!()

🔎这些与上面的`assert`宏类似。但默认情况下，这些语句仅在非优化构建中启用。`debug_assert`除非我们传递`-C debug-assertions`给编译器，否则在发布版本中将省略所有这些宏。

## Option and Result

许多语言使用`null\ nil\ undefined` 类型来表示空输出和Exceptions处理错误。Rust会同时使用两者，特别是为了防止诸如空指针异常，异常等敏感数据泄漏等问题。相反，Rust提供了两个特殊的通用枚举 ; `Option`和`Result`处理上述案件。

如您所知:

 ▸ `Option`可以包含某个值`Some`或没有值/` None`。

 ▸ `Result`可以表示成功/` Ok` 或失败/`Err`。

```rust
// An output can have either Some value or no value/ None.
enum Option<T> { // T is a generic and it can contain any type of value.
    Some(T),
    None,
}

// A result can represent either success/ Ok or failure/ Err.
enum Result<T, E> { // T and E are generics. T can contain any type of value, E can be any error.
    Ok(T),
    Err(E),
}
```

### `Option`的基本用法

编写函数或数据类型时:  
- 如果函数的参数是可选的，
- 如果函数为非空，并且返回的输出可以为空，
- 如果数据类型的属性的值可以是空，我们不得不使用他们的数据类型为Option类型

例如，如果函数输出一个`&str`值并且输出可以为空，则函数的返回类型应设置为`Option<&str>`

```rust
fn get_an_optional_value() -> Option<&str> {

    //if the optional value is not empty
    return Some("Some value");
    
    //else
    None
}
```

同样，如果数据类型的属性值可以为空或者像下面示例中`middle_name`的`Name`数据类型那样可选，我们应该将其数据类型设置为`Option`类型。

```rust
struct Name {
  first_name: String,
  middle_name: Option<String>, // middle_name can be empty
  last_name: String,
}
```

💭如您所知，我们可以使用模式匹配`match`来捕获相关的返回类型`（Some/ None）` 。有一个函数来获取当前用户的主目录在`std::env`为`home_dir()` 。由于所有用户在Linux等系统中都没有主目录，因此用户的主目录可以是可选的。所以它返回一个`Option`类型;  `Option<PathBuf>`.

```rust
use std::env;

fn main() {
    let home_path = env::home_dir();
    match home_path {
        Some(p) => println!("{:?}", p), // This prints "/root", if you run this in Rust playground
        None => println!("Can not find the home directory!"),
    }
}
```

⭐但是，当在函数中使用可选参数时，我们必须None在调用函数时传递空参数的值。

```rust
fn get_full_name(fname: &str, lname: &str, mname: Option<&str>) -> String { // middle name can be empty
  match mname {
    Some(n) => format!("{} {} {}", fname, n, lname),
    None => format!("{} {}", fname, lname),
  }
}

fn main() {
  println!("{}", get_full_name("Galileo", "Galilei", None));
  println!("{}", get_full_name("Leonardo", "Vinci", Some("Da")));
}

// 💡 Better create a struct as Person with fname, lname, mname fields and create a impl function as full_name()
```

> 🔎除此之外，`Option`类型与Rust中的可空指针一起使用。由于Rust中没有空指针，因此指针类型应指向有效位置。因此，如果指针可以为空，我们就可以使用了`Option<Box<T>>` 。

## `Result`的基本用法

如果函数可以产生错误，我们必须`Result`通过组合有效输出的数据类型和错误的数据类型来使用类型。例如，如果有效输出的数据类型为`u64`且错误类型为`String` ，则返回类型应为`Result<u64, String>` 。

```rust
fn function_with_error() -> Result<u64, String> {
  
    //if error happens
    return Err("The error message".to_string());

    // else, return valid output
    Ok(255)
}
```

💭如您所知，我们可以使用模式匹配`match`来捕获相关的返回类型`（Ok/ Err）`。有一个函数可以获取`std::env` 任何环境变量中的值是`var()` 。它的输入是环境变量名称。如果我们传递了错误的环境变量，或者程序在运行时无法提取环境变量的值，则会产生错误。所以它的返回类型是一种`Result`类型;  `Result<String, VarError>`.

```rust
use std::env;

fn main() {
    let key = "HOME";
    match env::var(key) {
        Ok(v) => println!("{}", v), // This prints "/root", if you run this in Rust playground
        Err(e) => println!("{}", e), // This prints "environment variable not found", if you give a nonexistent environment variable
    }
}
```

### is_some(), is_none(), is_ok(), is_err()

除了`match`表情，rust还提供`is_some()` ，`is_none()`并且`is_ok()` ，`is_err()`功能，以确定返回类型。

```rust
fn main() {
    let x: Option<&str> = Some("Hello, world!");
    assert_eq!(x.is_some(), true);
    assert_eq!(x.is_none(), false);

    let y: Result<i8, &str> = Ok(10);
    assert_eq!(y.is_ok(), true);
    assert_eq!(y.is_err(), false);
}
```

### ok(), err() for Result types

rust另外提供`ok()`和`err()`为`Result`类型。它们将`Result`类型的`Ok<T>`值和`Err<E>`值转换为Option类型。

```rust
fn main() {
    let o: Result<i8, &str> = Ok(8);
    let e: Result<i8, &str> = Err("message");
    
    assert_eq!(o.ok(), Some(8)); // Ok(v) ok = Some(v)
    assert_eq!(e.ok(), None);    // Err(v) ok = None
    
    assert_eq!(o.err(), None);            // Ok(v) err = None
    assert_eq!(e.err(), Some("message")); // Err(v) err = Some(v)
}
```

## Unwrap and Expect

## unwrap()

▸如果`Option`类型具有`Some`值或`Result`类型具有`Ok`值，则其中的值将传递到下一步。

▸如果`Option`类型具有`None`值或`Result`类型具有`Err`值，则编程`panics` ; 如果Err，`panics`携带错误消息。

该功能与以下代码类似，使用`match`而不是使用`unwrap()` 。

示例使用`Option`和`match`

```rust
fn main() {
    let x;
    match get_an_optional_value() {
        Some(v) => x = v, // if Some("abc"), set x to "abc"
        None => panic!(), // if None, panic without any message
    }

    println!("{}", x); // "abc" ; if you change line 14 `false` to `true`
}

fn get_an_optional_value() -> Option<&'static str> {

    //if the optional value is not empty
    if false {
        return Some("abc");
    }
    
    //else
    None
}


// --------------- Compile time error ---------------
thread 'main' panicked at 'explicit panic', src/main.rs:5:17
```

示例使用`Result`和`match`

```rust
fn main() {
    let x;
    match function_with_error() {
        Ok(v) => x = v, // if Ok(255), set x to 255
        Err(e) => panic!(e), // if Err("some message"), panic with error message "some message"
    }

    println!("{}", x); // 255 ; if you change line 13 `true` to `false`
}

fn function_with_error() -> Result<u64, String> {
    //if error happens
    if true {
        return Err("some message".to_string());
    }

    // else, return valid output
    Ok(255)
}


// ---------- Compile time error ----------
thread 'main' panicked at 'some message', src/main.rs:5:19
```

上述main函数中的相同代码可以使用`unwrap()`两行来编写。

```rust
// 01. unwrap error message for None
fn main() {
    let x = get_an_optional_value().unwrap();

    println!("{}", x);
}

// --------------- Compile time error ---------------
thread 'main' panicked at 'called `Option::unwrap()` on a `None` value', libcore/option.rs:345:21


// 02. unwrap error message for Err
fn main() {
    let x = function_with_error().unwrap();

    println!("{}", x);
}

// --------------- Compile time error ---------------
thread 'main' panicked at 'called `Result::unwrap()` on an `Err` value: "some message"', libcore/result.rs:945:5
```

⭐但是正如您所看到的，当使用`unwrap()`错误消息时，没有显示发生恐慌的确切行号。

## expect()

类似`unwrap()`但可以为恐慌设置自定义消息。

```rust
// 01. expect error message for None
fn main() {
    let n: Option<i8> = None;
    
    n.expect("empty value returned");
}

// --------------- Compile time error ---------------
thread 'main' panicked at 'empty value returned', libcore/option.rs:989:5


// 02. expect error message for Err
fn main() {
    let e: Result<i8, &str> = Err("some message");

    e.expect("expect error message");
}

// --------------- Compile time error ---------------
thread 'main' panicked at 'expect error message: "some message"', libcore/result.rs:945:5
```

### unwrap_err() and expect_err() for Result types

`unwrap()`和`expect()`相反的情况; `Ok`时恐慌而不是`Err`时。两者都在`Ok`错误消息中打印内部值。

💡通常用于测试。

```rust
// 01. unwrap_err error message for Ok
fn main() {
    let o: Result<i8, &str> = Ok(8);

    o.unwrap_err();
}

// ---------- Compile time error ----------
thread 'main' panicked at 'called `Result::unwrap_err()` on an `Ok` value: 8', libcore/result.rs:945:5


// 02. expect_err error message for Ok
fn main() {
    let o: Result<i8, &str> = Ok(8);

    o.expect_err("Should not get Ok value");
}

// ---------- Compile time error ----------
thread 'main' panicked at 'Should not get Ok value: 8', libcore/result.rs:945:5
```

### unwrap_or(), unwrap_or_default() and unwrap_or_else()

> 💡这些有点类似于`unwrap()`，如果`Option`类型有`Some`值或`Result`类型有`Ok`值，则它们内部的值传递到下一步。但是当有`None `或者 `Err`，功能有点不同。

- `unwrap_or()` ：使用`None`或`Err`，您传递给的`unwrap_or()`值将传递到下一步。但是，您传递的值的数据类型应与相关`Some`或`Ok`的数据类型匹配。

```rust
fn main() {
    let v1 = 8;
    let v2 = 16;

    let s_v1 = Some(8);
    let n = None;

    assert_eq!(s_v1.unwrap_or(v2), v1); // Some(v1) unwrap_or v2 = v1
    assert_eq!(n.unwrap_or(v2), v2);    // None unwrap_or v2 = v2

    let o_v1: Result<i8, &str> = Ok(8);
    let e: Result<i8, &str> = Err("error");

    assert_eq!(o_v1.unwrap_or(v2), v1); // Ok(v1) unwrap_or v2 = v1
    assert_eq!(e.unwrap_or(v2), v2);    // Err unwrap_or v2 = v2
}
```

- `unwrap_or_default(`) ：使用`None`或`Err`，相关的数据类型的默认值`Some`或者`Ok`，传递到下一步。

```rust
fn main() {
    let v = 8;
    let v_default = 0;

    let s_v: Option<i8> = Some(8);
    let n: Option<i8> = None;

    assert_eq!(s_v.unwrap_or_default(), v);       // Some(v) unwrap_or_default = v
    assert_eq!(n.unwrap_or_default(), v_default); // None unwrap_or_default = default value of v

    let o_v: Result<i8, &str> = Ok(8);
    let e: Result<i8, &str> = Err("error");

    assert_eq!(o_v.unwrap_or_default(), v);       // Ok(v) unwrap_or_default = v
    assert_eq!(e.unwrap_or_default(), v_default); // Err unwrap_or_default = default value of v
}
```

- `unwrap_or_else()` ：类似于`unwrap_or()`。唯一的区别是，您必须传递一个闭包，它返回一个具有`Some`或`Ok`相关数据类型的值，而不是传递一个值。

```rust
fn main() {
    let v1 = 8;
    let v2 = 16;

    let s_v1 = Some(8);
    let n = None;
    let fn_v2_for_option = || 16;

    assert_eq!(s_v1.unwrap_or_else(fn_v2_for_option), v1); // Some(v1) unwrap_or_else fn_v2 = v1
    assert_eq!(n.unwrap_or_else(fn_v2_for_option), v2);    // None unwrap_or_else fn_v2 = v2

    let o_v1: Result<i8, &str> = Ok(8);
    let e: Result<i8, &str> = Err("error");
    let fn_v2_for_result = |_| 16;

    assert_eq!(o_v1.unwrap_or_else(fn_v2_for_result), v1); // Ok(v1) unwrap_or_else fn_v2 = v1
    assert_eq!(e.unwrap_or_else(fn_v2_for_result), v2);    // Err unwrap_or_else fn_v2 = v2
}
```

## Error and None Propagation

我们应该使用恐慌`panic!()`，`unwrap()`，`expect()`只有当我们没有一个更好处理办法的情况。此外如果一个函数包含表达式既能产生`None`也能产生`Err`，  

▸我们可以在同一函数中处理

▸我们可以立即返回`None` 和`Err`给调用者。因此调用者可以决定如何处理它们。

💡 None类型无需始终由函数的调用者处理。但Rusts处理`Err`类型的约定是，立即将它们返回给调用者，以便给调用者更多的控制权来决定如何处理它们。

## `？`操作符

▸如果`Option`类型具有`Some`值或`Result`类型具有`Ok`值，则其中的值将传递到下一步。
▸如果`Option`类型具有`None`值或`Result`类型具有`Err`值，则立即将它们返回给函数的调用者。

示例`Option`类型，

```rust
fn main() {
    if complex_function().is_none() {
        println!("X not exists!");
    }
}

fn complex_function() -> Option<&'static str> {
    let x = get_an_optional_value()?; // if None, returns immidiately; if Some("abc"), set x to "abc"

    // some other code, ex
    println!("{}", x); // "abc" ; if you change line 19 `false` to `true` 

    Some("")
}

fn get_an_optional_value() -> Option<&'static str> {

    //if the optional value is not empty
    if false {
        return Some("abc");
    }
    
    //else
    None
}
```

示例`Result`类型，

```rust
fn main() {
    // `main` function is the caller of `complex_function` function
    // So we handle errors of complex_function(), inside main()
    if complex_function().is_err() {
        println!("Can not calculate X!");
    }
}

fn complex_function() -> Result<u64, String> {
    let x = function_with_error()?; // if Err, returns immidiately; if Ok(255), set x to 255

    // some other code, ex
    println!("{}", x); // 255 ; if you change line 20 `true` to `false`

    Ok(0)
}

fn function_with_error() -> Result<u64, String> {
    //if error happens
    if true {
        return Err("some message".to_string());
    }

    // else, return valid output
    Ok(255)
}
```

### 从main（）传播错误

在Rust版本1.26之前，我们无法从main()函数传播`Result`和`Option`。但是现在，我们可以从main()函数中传播`Result`类型，并打印出`Err`的`Debug`表示形式。


```rust
use std::fs::File;

fn main() -> std::io::Result<()> {
    let _ = File::open("not-existing-file.txt")?;

    Ok(()) // Because of the default return value of Rust functions is an empty tuple/ ()
}

// Because of the program can not find not-existing-file.txt , it produces,
//    Err(Os { code: 2, kind: NotFound, message: "No such file or directory" })
// While propagating error, the program prints,
//    Error: Os { code: 2, kind: NotFound, message: "No such file or directory" }
```

## Combinators

让我们看看组合器是什么，

- “组合者”的一个含义是更加非正式的意义，指的是组合模式，一种以组合事物的思想为中心组织图书馆的风格。通常存在一些类型T，一些用于构造类型T的“原始”值的函数，以及一些可以以各种方式组合类型T的值以构建类型T的更复杂值的 “ 组合器 ” 。另一个定义是“ 没有自由变量的函数 ”（wiki.haskell.org）

- 组合子是一个函数，其从程序片段构建程序片段 ; 从某种意义上说，使用组合器的程序员自动构建了大部分所需的程序，而不是手工编写每个细节。

Rust生态系统中“组合子”的确切定义有点不清楚。

▸ `or()`，`and()`，`or_else()`，`and_then()`
 - 组合类型为T的两个值并返回相同类型T。

▸ `filter()`对于Option类型
 - 使用闭包作为条件函数来过滤类型T. 
 - 返回相同的类型T.

▸ `map()`，`map_err()`
 - 通过使用闭包转换类型T。 
 - 可以更改T内部值的数据类型。例如:`Some<&str>`可转化为`Some<usize>`或者`Err<&str>`可转化为`Err<isize>`等

▸ `map_or()`，`map_or_else()` 
 - 通过应用闭包转换类型T并返回类型T内的值。
 - 对`None` 和`Err`，应用默认值或其他闭包。

▸ `ok_or()`，`ok_or_else()`对于Option类型
 - 将Option类型转换为Result类型。

▸ `as_ref()`，`as_mut()` 
 - 将类型T转换为引用或可变引用。

### or() and and()

组合两个表达式返回`Option/ Result`

▸`or() ：如果任何一个得到`Some`或`Ok`，该值立即返回。

▸ and() ：如果两者都得到`Some`或`Ok`，则返回第二个表达式中的值。如果任何一个获得None或Err该值立即返回。

```rust
fn main() {
  let s1 = Some("some1");
  let s2 = Some("some2");
  let n: Option<&str> = None;

  let o1: Result<&str, &str> = Ok("ok1");
  let o2: Result<&str, &str> = Ok("ok2");
  let e1: Result<&str, &str> = Err("error1");
  let e2: Result<&str, &str> = Err("error2");

  assert_eq!(s1.or(s2), s1); // Some1 or Some2 = Some1
  assert_eq!(s1.or(n), s1);  // Some or None = Some
  assert_eq!(n.or(s1), s1);  // None or Some = Some
  assert_eq!(n.or(n), n);    // None1 or None2 = None2

  assert_eq!(o1.or(o2), o1); // Ok1 or Ok2 = Ok1
  assert_eq!(o1.or(e1), o1); // Ok or Err = Ok
  assert_eq!(e1.or(o1), o1); // Err or Ok = Ok
  assert_eq!(e1.or(e2), e2); // Err1 or Err2 = Err2

  assert_eq!(s1.and(s2), s2); // Some1 and Some2 = Some2
  assert_eq!(s1.and(n), n);   // Some and None = None
  assert_eq!(n.and(s1), n);   // None and Some = None
  assert_eq!(n.and(n), n);    // None1 and None2 = None1
  
  assert_eq!(o1.and(o2), o2); // Ok1 and Ok2 = Ok2
  assert_eq!(o1.and(e1), e1); // Ok and Err = Err
  assert_eq!(e1.and(o1), e1); // Err and Ok = Err
  assert_eq!(e1.and(e2), e1); // Err1 and Err2 = Err1
}
```

> 🔎nightly支持Option类型的`xor()`，它返回`Some`当只有一个表达式返回`Some`，而不是两个。

### or_else()

类似于or()。唯一的区别是，第二个表达式应该是一个返回相同类型T 的闭包。

```rust
fn main() {
    // or_else with Option
    let s1 = Some("some1");
    let s2 = Some("some2");
    let fn_some = || Some("some2"); // similar to: let fn_some = || -> Option<&str> { Some("some2") };

    let n: Option<&str> = None;
    let fn_none = || None;

    assert_eq!(s1.or_else(fn_some), s1);  // Some1 or_else Some2 = Some1
    assert_eq!(s1.or_else(fn_none), s1);  // Some or_else None = Some
    assert_eq!(n.or_else(fn_some), s2);   // None or_else Some = Some
    assert_eq!(n.or_else(fn_none), None); // None1 or_else None2 = None2

    // or_else with Result
    let o1: Result<&str, &str> = Ok("ok1");
    let o2: Result<&str, &str> = Ok("ok2");
    let fn_ok = |_| Ok("ok2"); // similar to: let fn_ok = |_| -> Result<&str, &str> { Ok("ok2") };

    let e1: Result<&str, &str> = Err("error1");
    let e2: Result<&str, &str> = Err("error2");
    let fn_err = |_| Err("error2");

    assert_eq!(o1.or_else(fn_ok), o1);  // Ok1 or_else Ok2 = Ok1
    assert_eq!(o1.or_else(fn_err), o1); // Ok or_else Err = Ok
    assert_eq!(e1.or_else(fn_ok), o2);  // Err or_else Ok = Ok
    assert_eq!(e1.or_else(fn_err), e2); // Err1 or_else Err2 = Err2
}
```

### and_then()

类似于and()。唯一的区别是，第二个表达式应该是一个返回相同类型T 的闭包。

```rust
fn main() {
    // and_then with Option
    let s1 = Some("some1");
    let s2 = Some("some2");
    let fn_some = |_| Some("some2"); // similar to: let fn_some = |_| -> Option<&str> { Some("some2") };

    let n: Option<&str> = None;
    let fn_none = |_| None;

    assert_eq!(s1.and_then(fn_some), s2); // Some1 and_then Some2 = Some2
    assert_eq!(s1.and_then(fn_none), n);  // Some and_then None = None
    assert_eq!(n.and_then(fn_some), n);   // None and_then Some = None
    assert_eq!(n.and_then(fn_none), n);   // None1 and_then None2 = None1

    // and_then with Result
    let o1: Result<&str, &str> = Ok("ok1");
    let o2: Result<&str, &str> = Ok("ok2");
    let fn_ok = |_| Ok("ok2"); // similar to: let fn_ok = |_| -> Result<&str, &str> { Ok("ok2") };

    let e1: Result<&str, &str> = Err("error1");
    let e2: Result<&str, &str> = Err("error2");
    let fn_err = |_| Err("error2");

    assert_eq!(o1.and_then(fn_ok), o2);  // Ok1 and_then Ok2 = Ok2
    assert_eq!(o1.and_then(fn_err), e2); // Ok and_then Err = Err
    assert_eq!(e1.and_then(fn_ok), e1);  // Err and_then Ok = Err
    assert_eq!(e1.and_then(fn_err), e1); // Err1 and_then Err2 = Err1
}
```

### filter()

>💡通常在编程语言中，filter函数与数组或迭代器一起使用，通过函数/闭包过滤自己的元素来创建新的数组/迭代器。Rust还提供了一个`filter()`迭代器适配器，用于在迭代器的每个元素上应用闭包，将其转换为另一个迭代器。然而，在这里，我们正在谈论`filter()`函数与`Option`类型。

仅当我们传递一个`Some`值并且给定的闭包为它返回`true`时,返回相同的`Some`类型。如果`None`传递类型或闭包返回`false`,返回`None`。闭包使用`Some`里面的值作为参数。Rust仍然支持`filter()`只支持`Option`的类型。

```rust
fn main() {
    let s1 = Some(3);
    let s2 = Some(6);
    let n = None;

    let fn_is_even = |x: &i8| x % 2 == 0;

    assert_eq!(s1.filter(fn_is_even), n);  // Some(3) -> 3 is not even -> None
    assert_eq!(s2.filter(fn_is_even), s2); // Some(6) -> 6 is even -> Some(6)
    assert_eq!(n.filter(fn_is_even), n);   // None -> no value -> None
}
```

### map() and map_err()

> 💡通常在编程语言中，map()函数与数组或迭代器一起使用，以在数组或迭代器的每个元素上应用闭包。Rust还提供了一个`map()`迭代器适配器，用于在迭代器的每个元素上应用闭包，将其转换为另一个迭代器。但是在这里我们讨论的是`map()`函数与`Option`和`Result`类型。

- `map()` ：通过应用闭包来转换类型T. 可以根据闭包的返回类型更改`Some`或`Ok`块数据类型。转换`Option<T>`为`Option<U>` ，转换`Result<T, E>`为`Result<U, E>`

⭐ `map()`，仅仅 `Some`和`Ok`值改变。对`Err`内部值没有影响（`None`根本不包含任何值）。

```rust
fn main() {
    let s1 = Some("abcde");
    let s2 = Some(5);

    let n1: Option<&str> = None;
    let n2: Option<usize> = None;

    let o1: Result<&str, &str> = Ok("abcde");
    let o2: Result<usize, &str> = Ok(5);
    
    let e1: Result<&str, &str> = Err("abcde");
    let e2: Result<usize, &str> = Err("abcde");
    
    let fn_character_count = |s: &str| s.chars().count();

    assert_eq!(s1.map(fn_character_count), s2); // Some1 map = Some2
    assert_eq!(n1.map(fn_character_count), n2); // None1 map = None2

    assert_eq!(o1.map(fn_character_count), o2); // Ok1 map = Ok2
    assert_eq!(e1.map(fn_character_count), e2); // Err1 map = Err2
}
```

- `map_err()`对于`Result`类型：Err块的数据类型可以根据闭包的返回类型进行更改。转换`Result<T, E>`为`Result<T, F>`。

⭐`map_err()`，只有`Err`值会发生变化。对`Ok`内部的值没有影响。


```rust
fn main() {
    let o1: Result<&str, &str> = Ok("abcde");
    let o2: Result<&str, isize> = Ok("abcde");

    let e1: Result<&str, &str> = Err("404");
    let e2: Result<&str, isize> = Err(404);

    let fn_character_count = |s: &str| -> isize { s.parse().unwrap() }; // convert str to isize

    assert_eq!(o1.map_err(fn_character_count), o2); // Ok1 map = Ok2
    assert_eq!(e1.map_err(fn_character_count), e2); // Err1 map = Err2
}
```

### map_or() and map_or_else()

这些功能也与`unwrap_or()`和`unwrap_or_else()`相似。但是`map_or()`和`map_or_else()`在`Some`，`Ok`值上应用闭包和返回类型T内的值。

- `map_or()` ：仅支持`Option`类型（不支持`Result`）。将闭包应用于`Some`内部值并根据闭包返回输出。为None类型返回给定的默认值。

```rust
fn main() {
    const V_DEFAULT: i8 = 1;

    let s = Some(10);
    let n: Option<i8> = None;
    let fn_closure = |v: i8| v + 2;

    assert_eq!(s.map_or(V_DEFAULT, fn_closure), 12);
    assert_eq!(n.map_or(V_DEFAULT, fn_closure), V_DEFAULT);
}
```

- `map_or_else()` ：支持两种`Option`和`Result`类型（Result仅限nightly）。类似`map_or()`但应该提供另一个闭包而不是第一个参数的默认值。

⭐ `None`类型不包含任何值。所以不需要将任何东西传递给闭包作为输入Option类型。但是`Err`类型在其中包含一些值。因此，默认闭包应该能够将其作为输入读取，同时将其与`Result`类型一起使用。

```rust
#![feature(result_map_or_else)] // enable unstable library feature 'result_map_or_else' on nightly
fn main() {
    let s = Some(10);
    let n: Option<i8> = None;

    let fn_closure = |v: i8| v + 2;
    let fn_default = || 1; // None doesn't contain any value. So no need to pass anything to closure as input.

    assert_eq!(s.map_or_else(fn_default, fn_closure), 12);
    assert_eq!(n.map_or_else(fn_default, fn_closure), 1);

    let o = Ok(10);
    let e = Err(5);
    let fn_default_for_result = |v: i8| v + 1; // Err contain some value inside it. So default closure should able to read it as input

    assert_eq!(o.map_or_else(fn_default_for_result, fn_closure), 12);
    assert_eq!(e.map_or_else(fn_default_for_result, fn_closure), 6);
}
```

### ok_or() and ok_or_else()

如前所述`ok_or(`)，`ok_or_else()`将`Option`类型转换为`Result`类型。`Some`对`Ok`和`None`对`Err`  。

- `ok_or()` ：默认`Err`消息应作为参数传递.

```rust
fn main() {
    const ERR_DEFAULT: &str = "error message";

    let s = Some("abcde");
    let n: Option<&str> = None;

    let o: Result<&str, &str> = Ok("abcde");
    let e: Result<&str, &str> = Err(ERR_DEFAULT);

    assert_eq!(s.ok_or(ERR_DEFAULT), o); // Some(T) -> Ok(T)
    assert_eq!(n.ok_or(ERR_DEFAULT), e); // None -> Err(default)
}
```

- `ok_or_else() `：类似于`ok_or()`。应该将闭包作为参数传递。

```rust
fn main() {
    let s = Some("abcde");
    let n: Option<&str> = None;
    let fn_err_message = || "error message";

    let o: Result<&str, &str> = Ok("abcde");
    let e: Result<&str, &str> = Err("error message");

    assert_eq!(s.ok_or_else(fn_err_message), o); // Some(T) -> Ok(T)
    assert_eq!(n.ok_or_else(fn_err_message), e); // None -> Err(default)
}
```

### as_ref() and as_mut()

🔎如前所述，这些函数用于借用类型T作为引用或作为可变引用。

- `as_ref()` ：转换`Option<T>`到`Option<&T>`和`Result<T, E>`到`Result<&T, &E>`
- `as_mut()` ：转换`Option<T>`到`Option<&mut T>`和`Result<T, E>`到`Result<&mut T, &mut E>`

## 自定义错误类型

Rust允许我们创建自己的Err类型。我们称之为“ 自定义错误类型”。

### Error trait

如您所知，traits定义了类型必须提供的功能。但是我们不需要总是为常用功能定义新的特性，因为Rust 标准库提供了一些可以在我们自己的类型上实现的可重用特性。创建自定义错误类型时，[std::error::Error](https://doc.rust-lang.org/std/error/trait.Error.html) trait可帮助我们将任何类型转换为`Err`类型。

```rust
use std::fmt::{Debug, Display};

pub trait Error: Debug + Display {
    fn source(&self) -> Option<&(Error + 'static)> { ... }
}
```

一个特质可以从另一个特质继承。`trait Error: Debug + Display`意味着`Error`特质继承`fmt::Debug`和`fmt::Display`特质。

```rust
// traits inside Rust standard library core fmt module/ std::fmt
pub trait Display {
    fn fmt(&self, f: &mut Formatter) -> Result<(), Error>;
}

pub trait Debug {
    fn fmt(&self, f: &mut Formatter) -> Result<(), Error>;
}
```

▸ Display 
 - 最终用户应如何将此错误视为面向消息/面向用户的输出。
 - 通常通过`println!("{}")`或打印`eprintln!("{}")`

▸ Debug 
 - 如何显示Errwhile调试/面向程序员的输出。
 - 通常打印`println!("{:?}")`或`eprintln!("{:?}") `
 - - 漂亮打印，可以使用`println!("{:#?}")`或`eprintln!("{:#?}")`。

▸ source() 
 - 此错误的较低级别来源（如果有）。
 - 可选的。

首先，让我们看看如何`std::error::Error`在最简单的自定义错误类型上实现特征。

```rust
use std::fmt;

// Custom error type; can be any type which defined in the current crate
// 💡 In here, we use a simple "unit struct" to simplify the example
struct AppError;

// Implement std::fmt::Display for AppError
impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "An Error Occurred, Please Try Again!") // user-facing output
    }
}

// Implement std::fmt::Debug for AppError
impl fmt::Debug for AppError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "{{ file: {}, line: {} }}", file!(), line!()) // programmer-facing output
    }
}

// A sample function to produce an AppError Err
fn produce_error() -> Result<(), AppError> {
    Err(AppError)
}

fn main() {
    match produce_error() {
        Err(e) => eprintln!("{}", e), // An Error Occurred, Please Try Again!
        _ => println!("No error"),
    }

    eprintln!("{:?}", produce_error()); // Err({ file: src/main.rs, line: 17 })
}
```

希望你理解要点。现在，让我们看一些带有错误代码和错误消息的自定义错误类型。

```rust
use std::fmt;

struct AppError {
    code: usize,
    message: String,
}

// Different error messages according to AppError.code
impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        let err_msg = match self.code {
            404 => "Sorry, Can not find the Page!",
            _ => "Sorry, something is wrong! Please Try Again!",
        };

        write!(f, "{}", err_msg)
    }
}

// A unique format for dubugging output
impl fmt::Debug for AppError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(
            f,
            "AppError {{ code: {}, message: {} }}",
            self.code, self.message
        )
    }
}

fn produce_error() -> Result<(), AppError> {
    Err(AppError {
        code: 404,
        message: String::from("Page not found"),
    })
}

fn main() {
    match produce_error() {
        Err(e) => eprintln!("{}", e), // Sorry, Can not find the Page!
        _ => println!("No error"),
    }

    eprintln!("{:?}", produce_error()); // Err(AppError { code: 404, message: Page not found })

    eprintln!("{:#?}", produce_error());
    // Err(
    //     AppError { code: 404, message: Page not found }
    // )
}
```

⭐️Rust标准库不仅提供了可重用的特性，而且还有助于通过`#[derive]`属性神奇地生成少数特征的实现。Rust支持`derive` `std::fmt::Debug`，为调试消息提供默认格式。因此，我们可以在struct前声明使用`#[derive(Debug)]`跳过实现`std::fmt::Debug` 自定义错误类型。

```rust
use std::fmt;

#[derive(Debug)] // derive std::fmt::Debug on AppError
struct AppError {
    code: usize,
    message: String,
}

impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        let err_msg = match self.code {
            404 => "Sorry, Can not find the Page!",
            _ => "Sorry, something is wrong! Please Try Again!",
        };

        write!(f, "{}", err_msg)
    }
}

fn produce_error() -> Result<(), AppError> {
    Err(AppError {
        code: 404,
        message: String::from("Page not found"),
    })
}

fn main() {
    match produce_error() {
        Err(e) => eprintln!("{}", e), // Sorry, Can not find the Page!
        _ => println!("No error"),
    }

    eprintln!("{:?}", produce_error()); // Err(AppError { code: 404, message: Page not found })

    eprintln!("{:#?}", produce_error());
    // Err(
    //     AppError {
    //         code: 404,
    //         message: "Page not found"
    //     }
    // )
}
```

### From trait

在编写真实的程序时，我们大多数时候必须同时处理不同的模块，不同的std和第三方的板条箱。但是每个包都使用自己的错误类型，如果我们使用自己的错误类型，我们应该将这些错误转换为错误类型。我们可以使用`std::convert::From`标准化特征进行这些转换。

```rust
// traits inside Rust standard library core convert module/ std::convert
pub trait From<T>: Sized {
  fn from(_: T) -> Self;
}
```

>💡如您所知，`String::from()`函数用于创建`String` from `&str`数据类型。实际上这也是`std::convert::From`特质的实现。

让我们看看如何在自定义错误类型上实现`std::convert::From`特征。

```rust
use std::fs::File;
use std::io;

#[derive(Debug)]
struct AppError {
    kind: String,    // type of the error
    message: String, // error message
}

// Implement std::convert::From for AppError; from io::Error
impl From<io::Error> for AppError {
    fn from(error: io::Error) -> Self {
        AppError {
            kind: String::from("io"),
            message: error.to_string(),
        }
    }
}

fn main() -> Result<(), AppError> {
    let _file = File::open("nonexistent_file.txt")?; // This generates an io::Error. But because of return type is Result<(), AppError>, it converts to AppError

    Ok(())
}


// --------------- Run time error ---------------
Error: AppError { kind: "io", message: "No such file or directory (os error 2)" }
```

在上面的例子中，`File::open(“nonexistent.txt”)?`产生`std::io::Error`。但由于返回类型是`Result<(), AppError>`，它转换为`AppError`。因为我们正在从`main()`函数传播错误，所以它会打印出`Err`的Debug表示形式。

在上面的例子中，我们只处理一种std错误类型`std::io::Error`。让我们看一些处理多种std错误类型的例子。

```rust
use std::fs::File;
use std::io::{self, Read};
use std::num;

#[derive(Debug)]
struct AppError {
    kind: String,
    message: String,
}

// Implement std::convert::From for AppError; from io::Error
impl From<io::Error> for AppError {
    fn from(error: io::Error) -> Self {
        AppError {
            kind: String::from("io"),
            message: error.to_string(),
        }
    }
}

// Implement std::convert::From for AppError; from num::ParseIntError
impl From<num::ParseIntError> for AppError {
    fn from(error: num::ParseIntError) -> Self {
        AppError {
            kind: String::from("parse"),
            message: error.to_string(),
        }
    }
}

fn main() -> Result<(), AppError> {
    let mut file = File::open("hello_world.txt")?; // generates an io::Error, if can not open the file and converts to an AppError

    let mut content = String::new();
    file.read_to_string(&mut content)?; // generates an io::Error, if can not read file content and converts to an AppError

    let _number: usize;
    _number = content.parse()?; // generates num::ParseIntError, if can not convert file content to usize and converts to an AppError

    Ok(())
}


// --------------- Few possible run time errors ---------------

// 01. If hello_world.txt is a nonexistent file
Error: AppError { kind: "io", message: "No such file or directory (os error 2)" }

// 02. If user doesn't have relevant permission to access hello_world.txt
Error: AppError { kind: "io", message: "Permission denied (os error 13)" }

// 03. If hello_world.txt contains non-numeric content. ex Hello, world!
Error: AppError { kind: "parse", message: "invalid digit found in string" }
```

> 🔎 搜索有关实现的内容[std::io::ErrorKind](https://doc.rust-lang.org/std/io/enum.ErrorKind.html)，以了解如何进一步组织错误类型。
# The Rust Programming Language - 学习笔记

> [The Rust Programming Language - The Rust Programming Language (rust-lang.org)](https://doc.rust-lang.org/book/)
>
> [Rust 程序设计语言 中文版 (cntofu.com)](https://www.cntofu.com/book/55/index.html)

# 1. Getting Started

## 1.1 Installation

## 1.2 Hello, World!

指令：

+ `rustc main.rs` 编译main.rs文件，产生可执行二进制文件 main。（类似C语言gcc编译）
+ `rustfmt main.rs` 对 main.rs 的代码进行格式化

```rust
fn main() {
    println!("Hello, world!");
}
```

1. Rust代码风格，使用4个空格，而非Tab制表符
2. `println!`是一个Rust macro宏指令。（具体在19章对macro进行介绍）目前只需知道带有`!`的是macro，而不是普通的funciton
3. `"Hello, world!"` 这里是`prinln!`宏指令的参数
4. Rust使用`;`表明表达式的结束位置

Rust is an *ahead-of-time compiled* language.

## 1.3 Hello, Cargo!

Cargo集成了Rust编译、包管理、二进制依赖下载等功能的工具。

> 建议配置一下cargo的环境变量，假如使用zsh：
>
> ```shell
> ## Rust - cargo
> CARGO_HOME=/Users/用户名/.cargo
> export CARGO_HOME/bin
> ```

+ 创建rust项目（会新建和项目名一致的文件夹）

  ```shell
  cargo new 项目名
  ```

  > Note: Git is a common version control system. You can change `cargo new` to use a different version control system or no version control system by using the `--vcs` flag. Run `cargo new --help` to see the available options.

+ 编译项目（生成target目录，里面产生可执行二进制文件）

  ```shell
  cargo build
  ```

  > Running `cargo build` for the first time also causes Cargo to create a new file at the top level: *Cargo.lock*. This file keeps track of the exact versions of dependencies in your project. This project doesn’t have dependencies, so the file is a bit sparse. You won’t ever need to change this file manually; Cargo manages its contents for you.

+ 编译且运行二进制可执行文件

  ```shell
  cargo run
  ```

+ 检查项目代码

  ```shell
  cargo check
  ```

+ 项目代码发版(release，需要更长的编译时间)

  ```shell
  cargo build --release
  ```

Let’s recap what we’ve learned so far about Cargo:

- We can build a project using `cargo build`.
- We can build and run a project in one step using `cargo run`.
- We can build a project without producing a binary to check for errors using `cargo check`.
- Instead of saving the result of the build in the same directory as our code, Cargo stores it in the *target/debug* directory.

> When your project is finally ready for release, you can use `cargo build --release` to compile it with optimizations. This command will create an executable in *target/release* instead of *target/debug*. <u>The optimizations make your Rust code run faster, but turning them on lengthens the time it takes for your program to compile</u>. This is why there are two different profiles: one for development, when you want to rebuild quickly and often, and another for building the final program you’ll give to a user that won’t be rebuilt repeatedly and that will run as fast as possible. If you’re benchmarking your code’s running time, be sure to run `cargo build --release` and benchmark with the executable in *target/release*.

# 2. Programming a Guessing Game

+ [`String`](https://doc.rust-lang.org/std/string/struct.String.html) is a string type provided by the standard library that is a growable, **UTF-8 encoded** bit of text.
+ The `::` syntax in the `::new` line indicates that `new` is an associated function of the `String` type.

+ use `break` to exit `loop`

+ `_`, is a catchall value
+ use `continue` 进入 `loop` 的下一轮循环
+ `mut` 修饰变量时，变量由默认的不可变变成可变
+ `match`关键字的使用，就类似其他语言的 `switch`

---

猜数字-参考代码：

+ main.rs

  ```rust
  use rand::Rng;
  use std::cmp::Ordering;
  use std::io;
  
  fn main() {
      println!("Guess the number!");
  
      let secret_number = rand::thread_rng().gen_range(1..101);
  
      loop {
          println!("Please input your guess.");
  
          let mut guess = String::new();
  
          io::stdin()
              .read_line(&mut guess)
              .expect("Failed to read line");
  
          let guess: u32 = match guess.trim().parse() {
              Ok(num) => num,
              Err(_) => continue,
          };
  
          println!("You guessed: {}", guess);
  
          match guess.cmp(&secret_number) {
              Ordering::Less => println!("Too small!"),
              Ordering::Greater => println!("Too big!"),
              Ordering::Equal => {
                  println!("You win!");
                  break;
              }
          }
      }
  }
  ```

+ Cargo.toml

  ```toml
  [package]
  name = "just_test"
  version = "0.1.0"
  edition = "2021"
  
  # See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html
  
  [dependencies]
  rand="0.8.3"
  ```

# 3. Common Programming Concepts

## 3.1 Variables and Mutability

+ 使用 `const` 声明和定义 不可变的常量

  ```rust
  const THREE_HOURS_IN_SECONDS: u32 = 60 * 60 * 3;
  ```

  > Rust’s naming convention for constants is to use all uppercase with underscores between words.
  >
  > See the [Rust Reference’s section on constant evaluation](https://doc.rust-lang.org/reference/const_eval.html) for more information on what operations can be used when declaring constants.

### Shadowing

Rust允许使用同一个变量名来声明新的变量。

这时候，称第一个使用该变量名的var1被第二个使用该变量名的var2给shaowed了。

```rust
fn main() {
    let x = 5;

    let x = x + 1;

    {
        let x = x * 2;
        println!("The value of x in the inner scope is: {}", x);
    }

    println!("The value of x is: {}", x);
}
```

This program first binds `x` to a value of `5`. Then it shadows `x` by repeating `let x =`, taking the original value and adding `1` so the value of `x` is then `6`. Then, within an inner scope, the third `let` statement also shadows `x`, multiplying the previous value by `2` to give `x` a value of `12`. When that scope is over, the inner shadowing ends and `x` returns to being `6`.

```shell
$ cargo run
The value of x in the inner scope is: 12
The value of x is: 6
```

+ `mut` 声明变量可变，但是一旦赋值后，不可以再赋值成不同的类型
+ shadowing，使用`let`另外声明一个重名的变量，并且可以赋值不同类型

```rust
// shadowing 合法，从字符串类型，到 数值类型
let spaces = "   ";
let spaces = spaces.len();

// mut 非法，赋值成字符串类型后，不得改变类型赋值
let mut spaces = "   ";
spaces = spaces.len();
```

## 3.2 Data Types

Rust是强类型语言，编译器能够在编译时自动判断变量的类型。

当类型无法自动确定时，则需要用户主动声明变量的类型，例如：

`let guess: u32 = "42".parse().expect("Not a number!");`

### Scalar Type

#### Integer Types

| Length  | Signed  | Unsigned |
| ------- | ------- | -------- |
| 8-bit   | `i8`    | `u8`     |
| 16-bit  | `i16`   | `u16`    |
| 32-bit  | `i32`   | `u32`    |
| 64-bit  | `i64`   | `u64`    |
| 128-bit | `i128`  | `u128`   |
| arch    | `isize` | `usize`  |

> Signed numbers are stored using [two’s complement](https://en.wikipedia.org/wiki/Two's_complement) representation.
>
> 有符号数值用二进制补码表示

> Additionally, the `isize` and `usize` types depend on the architecture of the computer your program is running on, which is denoted in the table as “arch”: 64 bits if you’re on a 64-bit architecture and 32 bits if you’re on a 32-bit architecture.

| Number literals  | Example       |
| ---------------- | ------------- |
| Decimal          | `98_222`      |
| Hex              | `0xff`        |
| Octal            | `0o77`        |
| Binary           | `0b1111_0000` |
| Byte (`u8` only) | `b'A'`        |

> **integer types default to `i32`.** 
>
> The primary situation in which you’d use `isize` or `usize` is when indexing some sort of collection.

> To explicitly handle the possibility of overflow, you can use these families of methods provided by the standard library for primitive numeric types:
>
> - Wrap in all modes with the `wrapping_*` methods, such as `wrapping_add`
> - Return the `None` value if there is overflow with the `checked_*` methods
> - Return the value and a boolean indicating whether there was overflow with the `overflowing_*` methods
> - Saturate at the value’s minimum or maximum values with `saturating_*` methods

#### Floating-Point Types

Rust支持两种浮点数类型，`f32`和`f64`。这两种都是有符号的，并且默认使用`f64`作为浮点数类型。（因为现代CPU处理`f64`和`f32`的速度相差无几，并且`f64`精度要更高）

```rust
fn main() {
    let x = 2.0; // f64

    let y: f32 = 3.0; // f32
}
```

> Floating-point numbers are represented according to the IEEE-754 standard. The `f32` type is a single-precision float, and `f64` has double precision.

#### Numeric Operations

```rust
fn main() {
    // addition
    let sum = 5 + 10;

    // subtraction
    let difference = 95.5 - 4.3;

    // multiplication
    let product = 4 * 30;

    // division
    let quotient = 56.7 / 32.2;
    let floored = 2 / 3; // Results in 0

    // remainder
    let remainder = 43 % 5;
}
```

#### The Boolean Type

Boolean，只有`true`和`false`，仅占用1 byte。

```rust
fn main() {
    let t = true;

    let f: bool = false; // with explicit type annotation
}
```

#### The Character Type

Rust’s `char` type is the language’s most primitive alphabetic type. Here’s some examples of declaring `char` values:

```rust
fn main() {
    let c = 'z';
    let z = 'ℤ';
    let heart_eyed_cat = '😻';
}
```

> Rust’s `char` type is **four bytes** in size and represents a **Unicode** Scalar Value, which means it can represent a lot more than just ASCII. 

### Compound Types

*Compound types* can group multiple values into one type. Rust has two primitive compound types: tuples and arrays.

#### The Tuple Type

Tuple类型，可以聚合多种类型为一体，但是一旦声明后，不可修改声明中包含的类型和数量。

```rust
fn main() {
    let tup: (i32, f64, u8) = (500, 6.4, 1);
}
```

```rust
fn main() {
    let tup = (500, 6.4, 1);

    let (x, y, z) = tup;

    println!("The value of y is: {}", y);
}
```

We can also access a tuple element directly by using a period (`.`) followed by the index of the value we want to access. For example:

```rust
fn main() {
    let x: (i32, f64, u8) = (500, 6.4, 1);

    let five_hundred = x.0;

    let six_point_four = x.1;

    let one = x.2;
}
```

> The tuple without any values, `()`, is a special type that has only one value, also written `()`. 
>
> The type is called the *unit type* and the value is called the *unit value*. Expressions implicitly return the unit value if they don’t return any other value.

#### The Array Type

Array，里面的元素类型必须一致，并且数组的长度是固定的。

> A vector is a similar collection type provided by the standard library that *is* allowed to grow or shrink in size. If you’re unsure whether to use an array or a vector, chances are you should use a vector. [Chapter 8](https://doc.rust-lang.org/book/ch08-01-vectors.html) discusses vectors in more detail.

```rust
let a: [i32; 5] = [1, 2, 3, 4, 5];
```

```rust
let a = [3; 5];
//等同于 let a = [3, 3, 3, 3, 3];
```

##### Accessing Array Elements

```rust
fn main() {
    let a = [1, 2, 3, 4, 5];

    let first = a[0];
    let second = a[1];
}
```

##### Invalid Array Element Access

```rust
use std::io;

fn main() {
    let a = [1, 2, 3, 4, 5];

    println!("Please enter an array index.");

    let mut index = String::new();

    io::stdin()
        .read_line(&mut index)
        .expect("Failed to read line");

    let index: usize = index
        .trim()
        .parse()
        .expect("Index entered was not a number");

    let element = a[index];

    println!(
        "The value of the element at index {} is: {}",
        index, element
    );
}
```

如果输入的数字大于4，就会抛出异常。

```shell
thread 'main' panicked at 'index out of bounds: the len is 5 but the index is 10', src/main.rs:19:19
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

## 3.3 Functions

使用`fn`关键字声明函数

```rust
fn 函数名(参数声明) -> 返回值声明 { 函数体 }
```

```rutst
fn main() {
    println!("Hello, world!");

    another_function();
}

fn another_function() {
    println!("Another function.");
}
```

> Rust对函数声明的位置没有严格要求。（不像C需要在之前声明函数，后面的代码才能使用）。

### Parameters

```rust
fn main() {
    another_function(5);
}

fn another_function(x: i32) {
    println!("The value of x is: {}", x);
}
```

Rust中，函数的所有参数的类型，必须显式声明。

### Statements and Expressions

+ *Statements* are instructions that perform some action and do **not return a value**. 
+ *Expressions* evaluate to a resulting value. 

Let’s look at some examples.

```rust
fn main() {
    let y = 6;
}
```

> Listing 3-1: A `main` function declaration containing one statement.
>
> Function definitions are also statements; the entire preceding example is a statement in itself.

```rust
fn main() {
    let x = (let y = 6);
}
```

以上语句会得到Error，因为`let y = 6` 是 statement，而不是expression，没有返回值，所以不能再给`x`赋值。

Expressions evaluate to a value and make up most of the rest of the code that you’ll write in Rust. Consider a math operation, such as `5 + 6`, which is an expression that evaluates to the value `11`. Expressions can be part of statements: in Listing 3-1, the `6` in the statement `let y = 6;` is an expression that evaluates to the value `6`. **Calling a function is an expression**. **Calling a macro is an expression**. **A new scope block created with curly brackets is an expression**, for example:

```rust
fn main() {
    let y = {
        let x = 3;
        x + 1
    };

    println!("The value of y is: {}", y);
}
```

This expression：

```rust
{
    let x = 3;
    x + 1
}
```

is a block that, in this case, evaluates to `4`. That value gets bound to `y` as part of the `let` statement. Note that the `x + 1` line doesn’t have a semicolon at the end, unlike most of the lines you’ve seen so far. **Expressions do not include ending semicolons**. If you add a semicolon to the end of an expression, you turn it into a statement, and it will then not return a value. Keep this in mind as you explore function return values and expressions next.

### Functions with Return Values

Functions can return values to the code that calls them. **We don’t name return values, but we do declare their type after an arrow (`->`)**. 

In Rust, the return value of the function is synonymous with **the value of the final expression** in the block of the body of a function. 

You can return early from a function by using the `return` keyword and specifying a value, but most functions return the last expression implicitly. Here’s an example of a function that returns a value:

```rust
fn five() -> i32 {
    5
}

fn main() {
    let x = five();

    println!("The value of x is: {}", x);
}
```

## 3.4 Comments

使用`//`

## 3.5 Control Flow

### `if` Expressions

```rust
fn main() {
    let number = 3;

    if number < 5 {
        println!("condition was true");
    } else {
        println!("condition was false");
    }
}
```

> It’s also worth noting that **the condition in this code *must* be a `bool`**. If the condition isn’t a `bool`, we’ll get an error. For example, try running the following code:

#### Handling Multiple Conditions with `else if`

You can use multiple conditions by combining `if` and `else` in an `else if` expression. For example:

```rust
fn main() {
    let number = 6;

    if number % 4 == 0 {
        println!("number is divisible by 4");
    } else if number % 3 == 0 {
        println!("number is divisible by 3");
    } else if number % 2 == 0 {
        println!("number is divisible by 2");
    } else {
        println!("number is not divisible by 4, 3, or 2");
    }
}
```

> Using too many `else if` expressions can clutter your code, so if you have more than one, you might want to refactor your code. Chapter 6 describes a powerful Rust branching construct called `match` for these cases.

#### Using `if` in a `let` Statement

**因为`if`是expression**，所以我们可以使用`if`来给`let`变量赋值。

```rust
fn main() {
    let condition = true;
    let number = if condition { 5 } else { 6 };

    println!("The value of number is: {}", number);
}
```

**`if`和`else`的返回值类型必须一致**

```rust
fn main() {
    let condition = true;

    let number = if condition { 5 } else { "six" };

    println!("The value of number is: {}", number);
}
```

When we try to compile this code, we’ll get an error. The `if` and `else` arms have value types that are incompatible, and Rust indicates exactly where to find the problem in the program:

```shell
$ cargo run
   Compiling branches v0.1.0 (file:///projects/branches)
error[E0308]: `if` and `else` have incompatible types
 --> src/main.rs:4:44
  |
4 |     let number = if condition { 5 } else { "six" };
  |                                 -          ^^^^^ expected integer, found `&str`
  |                                 |
  |                                 expected because of this

For more information about this error, try `rustc --explain E0308`.
error: could not compile `branches` due to previous error
```

### Repetition with Loops

Rust has three kinds of loops: `loop`, `while`, and `for`. Let’s try each one.

#### Repeating Code with `loop`

If you have loops within loops, `break` and `continue` apply to the innermost loop at that point. You can optionally specify a *loop label* on a loop that we can then use with `break` or `continue` to specify that those keywords apply to the labeled loop instead of the innermost loop. Here’s an example with two nested loops:

```rust
fn main() {
    let mut count = 0;
    'counting_up: loop {
        println!("count = {}", count);
        let mut remaining = 10;

        loop {
            println!("remaining = {}", remaining);
            if remaining == 9 {
                break;
            }
            if count == 2 {
                break 'counting_up;
            }
            remaining -= 1;
        }

        count += 1;
    }
    println!("End count = {}", count);
}
```

The outer loop has the label `'counting_up`, and it will count up from 0 to 2. The inner loop without a label counts down from 10 to 9. The first `break` that doesn’t specify a label will exit the inner loop only. The `break 'counting_up;` statement will exit the outer loop. This code prints:

```shell
$ cargo run
   Compiling loops v0.1.0 (file:///projects/loops)
    Finished dev [unoptimized + debuginfo] target(s) in 0.58s
     Running `target/debug/loops`
count = 0
remaining = 10
remaining = 9
count = 1
remaining = 10
remaining = 9
count = 2
remaining = 10
End count = 2
```

#### Returning Values from Loops

One of the uses of a `loop` is to retry an operation you know might fail, such as checking whether a thread has completed its job. You might also need to pass the result of that operation out of the loop to the rest of your code. To do this, **you can add the value you want returned after the `break` expression you use to stop the loop**; that value will be returned out of the loop so you can use it, as shown here:

```rust
fn main() {
    let mut counter = 0;

    let result = loop {
        counter += 1;

        if counter == 10 {
            break counter * 2;
        }
    };

    println!("The result is {}", result);
}
```

Finally, we print the value in `result`, which in this case is 20.

#### Conditional Loops with `while`

```rust
fn main() {
    let mut number = 3;

    while number != 0 {
        println!("{}!", number);

        number -= 1;
    }

    println!("LIFTOFF!!!");
}
// 3-4
```

As a more concise alternative, you can use a `for` loop and execute some code for each item in a collection. A `for` loop looks like the code in Listing 3-5.

```rust
fn main() {
    let a = [10, 20, 30, 40, 50];

    for element in a {
        println!("the value is: {}", element);
    }
}
// 3-5
```

Using the `for` loop, you wouldn’t need to remember to change any other code if you changed the number of values in the array, as you would with the method used in Listing 3-4.

# 4. Understanding Ownership

Ownership is Rust’s most unique feature and has deep implications for the rest of the language. **It enables Rust to make memory safety guarantees without needing a garbage collector**, so it’s important to understand how ownership works. In this chapter, we’ll talk about ownership as well as several related features: borrowing, slices, and how Rust lays data out in memory.

## 4.1 What Is OwnerShip?

***Ownership* is a set of rules that governs how a Rust program manages memory**.

All programs have to manage the way they use a computer’s memory while running. 

+ Some languages have garbage collection that constantly looks for no-longer used memory as the program runs; 
+ in other languages, the programmer must explicitly allocate and free the memory. 
+ **Rust uses a third approach: memory is managed through a system of ownership with a set of rules that the compiler checks. If any of the rules are violated, the program won’t compile**. None of the features of ownership will slow down your program while it’s running.

> Because ownership is a new concept for many programmers, it does take some time to get used to. The good news is that the more experienced you become with Rust and the rules of the ownership system, the easier you’ll find it to naturally develop code that is safe and efficient. Keep at it!

### The Stack and the Heap

Many programming languages don’t require you to think about the stack and the heap very often. But in a systems programming language like Rust, whether a value is on the stack or the heap affects how the language behaves and why you have to make certain decisions. Parts of ownership will be described in relation to the stack and the heap later in this chapter, so here is a brief explanation in preparation.

Both the stack and the heap are parts of memory available to your code to use at runtime, but they are structured in different ways. The stack stores values in the order it gets them and removes the values in the opposite order. This is referred to as *last in, first out*. Think of a stack of plates: when you add more plates, you put them on top of the pile, and when you need a plate, you take one off the top. Adding or removing plates from the middle or bottom wouldn’t work as well! Adding data is called *pushing onto the stack*, and removing data is called *popping off the stack*. **All data stored on the stack must have a known, fixed size**. **Data with an unknown size at compile time or a size that might change must be stored on the heap instead**.

The heap is less organized: when you put data on the heap, you request a certain amount of space. The memory allocator finds an empty spot in the heap that is big enough, marks it as being in use, and returns a *pointer*, which is the address of that location. This process is called *allocating on the heap* and is sometimes abbreviated as just *allocating*. Pushing values onto the stack is not considered allocating. Because the pointer to the heap is a known, fixed size, you can store the pointer on the stack, but when you want the actual data, you must follow the pointer. Think of being seated at a restaurant. When you enter, you state the number of people in your group, and the staff finds an empty table that fits everyone and leads you there. If someone in your group comes late, they can ask where you’ve been seated to find you.

**Pushing to the stack is faster than allocating on the heap because the allocator never has to search for a place to store new data; that location is always at the top of the stack**. Comparatively, allocating space on the heap requires more work, because the allocator must first find a big enough space to hold the data and then perform bookkeeping to prepare for the next allocation.

**Accessing data in the heap is slower than accessing data on the stack because you have to follow a pointer to get there**. Contemporary processors are faster if they jump around less in memory. Continuing the analogy, consider a server at a restaurant taking orders from many tables. It’s most efficient to get all the orders at one table before moving on to the next table. Taking an order from table A, then an order from table B, then one from A again, and then one from B again would be a much slower process. By the same token, a processor can do its job better if it works on data that’s close to other data (as it is on the stack) rather than farther away (as it can be on the heap). Allocating a large amount of space on the heap can also take time.

**When your code calls a function, the values passed into the function (including, potentially, pointers to data on the heap) and the function’s local variables get pushed onto the stack. When the function is over, those values get popped off the stack**.

Keeping track of what parts of code are using what data on the heap, minimizing the amount of duplicate data on the heap, and cleaning up unused data on the heap so you don’t run out of space are all problems that ownership addresses. <u>Once you understand ownership, you won’t need to think about the stack and the heap very often, but knowing that **the main purpose of ownership is to manage heap data** can help explain why it works the way it does</u>.

### Ownership Rules

First, let’s take a look at the ownership rules. Keep these rules in mind as we work through the examples that illustrate them:

- **Each value in Rust has a variable that’s called its *owner*.**
- **There can only be one owner at a time**.
- When the owner goes out of scope, the value will be dropped.

#### Variable Scope

```rust
{                      // s is not valid here, it’s not yet declared
  let s = "hello";   // s is valid from this point forward

  // do stuff with s
}                      // this scope is now over, and s is no longer valid
```

#### The `String` Type

You can create a `String` from a string literal using the `from` function, like so：

```rust
let s = String::from("hello");
```

The double colon `::` operator allows us to namespace this particular `from` function under the `String` type rather than using some sort of name like `string_from`. We’ll discuss this syntax more in the [“Method Syntax”](https://doc.rust-lang.org/book/ch05-03-method-syntax.html#method-syntax) section of Chapter 5 and when we talk about namespacing with modules in [“Paths for Referring to an Item in the Module Tree”](https://doc.rust-lang.org/book/ch07-03-paths-for-referring-to-an-item-in-the-module-tree.html) in Chapter 7.

This kind of string *can* be mutated:

```rust
let mut s = String::from("hello");

s.push_str(", world!"); // push_str() appends a literal to a String

println!("{}", s); // This will print `hello, world!`
```

So, what’s the difference here? Why can `String` be mutated but literals cannot? The difference is how these two types deal with memory.

#### Memory and Allocation

With the `String` type, in order to support a mutable, growable piece of text, we need to allocate an amount of memory on the heap, unknown at compile time, to hold the contents. This means:

- The memory must be requested from the memory allocator at runtime.
- We need a way of returning this memory to the allocator when we’re done with our `String`.

That first part is done by us: when we call `String::from`, its implementation requests the memory it needs. This is pretty much universal in programming languages.

However, the second part is different. In languages with a *garbage collector (GC)*, the GC keeps track of and cleans up memory that isn’t being used anymore, and we don’t need to think about it. In most languages without a GC, it’s our responsibility to identify when memory is no longer being used and call code to explicitly return it, just as we did to request it. Doing this correctly has historically been a difficult programming problem. If we forget, we’ll waste memory. If we do it too early, we’ll have an invalid variable. If we do it twice, that’s a bug too. We need to pair exactly one `allocate` with exactly one `free`.

**Rust takes a different path: the memory is automatically returned once the variable that owns it goes out of scope**. Here’s a version of our scope example from Listing 4-1 using a `String` instead of a string literal:

```rust
{
  let s = String::from("hello"); // s is valid from this point forward

  // do stuff with s
}                                  // this scope is now over, and s is no
// longer valid
```

There is a natural point at which we can return the memory our `String` needs to the allocator: when `s` goes out of scope. When a variable goes out of scope, Rust calls a special function for us. This function is called [`drop`](https://doc.rust-lang.org/std/ops/trait.Drop.html#tymethod.drop), and it’s where the author of `String` can put the code to return the memory. **Rust calls `drop` automatically at the closing curly bracket**.

#### Ways Variables and Data Interact: Move

Multiple variables can interact with the same data in different ways in Rust. Let’s look at an example using an integer in Listing 4-2.

```rust
let x = 5;
let y = x;
//Listing 4-2: Assigning the integer value of variable x to y
```

We can probably guess what this is doing: “bind the value `5` to `x`; then **make a copy of the value in `x` and bind it to `y`.**” We now have two variables, `x` and `y`, and both equal `5`. This is indeed what is happening, because integers are simple values with a known, fixed size, and these two `5` values are pushed onto the stack.

Now let’s look at the `String` version:

```rust
let s1 = String::from("hello");
let s2 = s1;
```

<u>This looks very similar, so we might assume that the way it works would be the same: that is, the second line would make a copy of the value in `s1` and bind it to `s2`. But this isn’t quite what happens</u>.

Take a look at Figure 4-1 to see what is happening to `String` under the covers. <u>A `String` is made up of three parts, shown on the left: a pointer to the memory that holds the contents of the string, a length, and a capacity. **This group of data is stored on the stack**. **On the right is the memory on the heap that holds the contents**</u>.

![String in memory](https://doc.rust-lang.org/book/img/trpl04-01.svg)

Figure 4-1: Representation in memory of a String holding the value "hello" bound to s1

> The length is how much memory, in bytes, the contents of the `String` is currently using. The capacity is the total amount of memory, in bytes, that the `String` has received from the allocator. The difference between length and capacity matters, but not in this context, so for now, it’s fine to ignore the capacity.

**When we assign `s1` to `s2`, the `String` data is copied, meaning we copy the pointer, the length, and the capacity that are on the stack. We do not copy the data on the heap that the pointer refers to**. In other words, the data representation in memory looks like Figure 4-2.

![s1 and s2 pointing to the same value](https://doc.rust-lang.org/book/img/trpl04-02.svg)

Figure 4-2: Representation in memory of the variable `s2` that has a copy of the pointer, length, and capacity of `s1`

**The representation does *not* look like Figure 4-3, which is what memory would look like if Rust instead copied the heap data as well. If Rust did this, the operation `s2 = s1` could be very expensive in terms of runtime performance if the data on the heap were large**.

![s1 and s2 to two places](https://doc.rust-lang.org/book/img/trpl04-03.svg)

Figure 4-3: Another possibility for what `s2 = s1` might do if Rust copied the heap data as well

Earlier, we said that when a variable goes out of scope, Rust automatically calls the `drop` function and cleans up the heap memory for that variable. But Figure 4-2 shows both data pointers pointing to the same location. **This is a problem: when `s2` and `s1` go out of scope, they will both try to free the same memory. This is known as a *double free* error and is one of the memory safety bugs we mentioned previously**. Freeing memory twice can lead to memory corruption, which can potentially lead to security vulnerabilities.

**To ensure memory safety, after the line `let s2 = s1`, Rust considers `s1` as no longer valid. Therefore, Rust doesn’t need to free anything when `s1` goes out of scope**. Check out what happens when you try to use `s1` after `s2` is created; it won’t work:

```rust
let s1 = String::from("hello");
let s2 = s1;

println!("{}, world!", s1);
```

You’ll get an error like this because Rust prevents you from using the invalidated reference:

```shell
$ cargo run
   Compiling ownership v0.1.0 (file:///projects/ownership)
error[E0382]: borrow of moved value: `s1`
 --> src/main.rs:5:28
  |
2 |     let s1 = String::from("hello");
  |         -- move occurs because `s1` has type `String`, which does not implement the `Copy` trait
3 |     let s2 = s1;
  |              -- value moved here
4 | 
5 |     println!("{}, world!", s1);
  |                            ^^ value borrowed here after move

For more information about this error, try `rustc --explain E0382`.
error: could not compile `ownership` due to previous error
```

If you’ve heard the terms *shallow copy* and *deep copy* while working with other languages, the concept of copying the pointer, length, and capacity without copying the data probably sounds like making a shallow copy. **But because Rust also invalidates the first variable, instead of calling it a shallow copy, it’s known as a *move***. In this example, we would say that `s1` was *moved* into `s2`. So what actually happens is shown in Figure 4-4.

![s1 moved to s2](https://doc.rust-lang.org/book/img/trpl04-04.svg)

Figure 4-4: Representation in memory after `s1` has been invalidated

That solves our problem! With only `s2` valid, when it goes out of scope, it alone will free the memory, and we’re done.

**In addition, there’s a design choice that’s implied by this: Rust will never automatically create “deep” copies of your data. Therefore, any *automatic* copying can be assumed to be inexpensive in terms of runtime performance**.

#### Ways Variables and Data Interact: Clone

**If we *do* want to deeply copy the heap data of the `String`, not just the stack data, we can use a common method called `clone`**. We’ll discuss method syntax in Chapter 5, but because methods are a common feature in many programming languages, you’ve probably seen them before.

Here’s an example of the `clone` method in action:

```rust
let s1 = String::from("hello");
let s2 = s1.clone();

println!("s1 = {}, s2 = {}", s1, s2);
```

This works just fine and explicitly produces the behavior shown in Figure 4-3, where the heap data *does* get copied.

#### Stack-Only Data: Copy

There’s another wrinkle we haven’t talked about yet. This code using integers – part of which was shown in Listing 4-2 – works and is valid:

```rust
let x = 5;
let y = x;

println!("x = {}, y = {}", x, y);
```

But this code seems to contradict what we just learned: we don’t have a call to `clone`, but `x` is still valid and wasn’t moved into `y`.

**The reason is that types such as integers that have a known size at compile time are stored entirely on the stack, so copies of the actual values are quick to make**. That means there’s no reason we would want to prevent `x` from being valid after we create the variable `y`. I**n other words, there’s no difference between deep and shallow copying here, so calling `clone` wouldn’t do anything different from the usual shallow copying and we can leave it out**.

**Rust has a special annotation called the `Copy` trait that we can place on types that are stored on the stack like integers are (we’ll talk more about traits in Chapter 10). If a type implements the `Copy` trait, a variable is still valid after assignment to another variable**. <u>Rust won’t let us annotate a type with `Copy` if the type, or any of its parts, has implemented the `Drop` trait. If the type needs something special to happen when the value goes out of scope and we add the `Copy` annotation to that type, we’ll get a compile-time error. To learn about how to add the `Copy` annotation to your type to implement the trait, see [“Derivable Traits”](https://doc.rust-lang.org/book/appendix-03-derivable-traits.html) in Appendix C</u>.

So what types implement the `Copy` trait? You can check the documentation for the given type to be sure, but **as a general rule, any group of simple scalar values can implement `Copy`, and nothing that requires allocation or is some form of resource can implement `Copy`**. Here are some of the types that implement `Copy`:

- All the integer types, such as `u32`.
- The Boolean type, `bool`, with values `true` and `false`.
- All the floating point types, such as `f64`.
- The character type, `char`.
- **Tuples, if they only contain types that also implement `Copy`. For example, `(i32, i32)` implements `Copy`, but `(i32, String)` does not**.

#### Ownership and Functions

The semantics for passing a value to a function are similar to those for assigning a value to a variable. **Passing a variable to a function will move or copy, just as assignment does**. Listing 4-3 has an example with some annotations showing where variables go into and out of scope.

```rust
fn main() {
    let s = String::from("hello");  // s comes into scope

    takes_ownership(s);             // s's value moves into the function...
                                    // ... and so is no longer valid here

    let x = 5;                      // x comes into scope

    makes_copy(x);                  // x would move into the function,
                                    // but i32 is Copy, so it's okay to still
                                    // use x afterward

} // Here, x goes out of scope, then s. But because s's value was moved, nothing
  // special happens.

fn takes_ownership(some_string: String) { // some_string comes into scope
    println!("{}", some_string);
} // Here, some_string goes out of scope and `drop` is called. The backing
  // memory is freed.

fn makes_copy(some_integer: i32) { // some_integer comes into scope
    println!("{}", some_integer);
} // Here, some_integer goes out of scope. Nothing special happens.
```

Listing 4-3: Functions with ownership and scope annotated

If we tried to use `s` after the call to `takes_ownership`, Rust would throw a compile-time error. These static checks protect us from mistakes. Try adding code to `main` that uses `s` and `x` to see where you can use them and where the ownership rules prevent you from doing so.

#### Return Values and Scope

**Returning values can also transfer ownership**. Listing 4-4 shows an example of a function that returns some value, with similar annotations as those in Listing 4-3.

```rust
fn main() {
    let s1 = gives_ownership();         // gives_ownership moves its return
                                        // value into s1

    let s2 = String::from("hello");     // s2 comes into scope

    let s3 = takes_and_gives_back(s2);  // s2 is moved into
                                        // takes_and_gives_back, which also
                                        // moves its return value into s3
} // Here, s3 goes out of scope and is dropped. s2 was moved, so nothing
  // happens. s1 goes out of scope and is dropped.

fn gives_ownership() -> String {             // gives_ownership will move its
                                             // return value into the function
                                             // that calls it

    let some_string = String::from("yours"); // some_string comes into scope

    some_string                              // some_string is returned and
                                             // moves out to the calling
                                             // function
}

// This function takes a String and returns one
fn takes_and_gives_back(a_string: String) -> String { // a_string comes into
                                                      // scope

    a_string  // a_string is returned and moves out to the calling function
}
```

**The ownership of a variable follows the same pattern every time: assigning a value to another variable moves it. When a variable that includes data on the heap goes out of scope, the value will be cleaned up by `drop` unless ownership of the data has been moved to another variable**.

While this works, taking ownership and then returning ownership with every function is a bit tedious. What if we want to let a function use a value but not take ownership? It’s quite annoying that anything we pass in also needs to be passed back if we want to use it again, in addition to any data resulting from the body of the function that we might want to return as well.

<u>Rust does let us return multiple values using a tuple</u>, as shown in Listing 4-5.

````rust
fn main() {
    let s1 = String::from("hello");

    let (s2, len) = calculate_length(s1);

    println!("The length of '{}' is {}.", s2, len);
}

fn calculate_length(s: String) -> (String, usize) {
    let length = s.len(); // len() returns the length of a String

    (s, length)
}
````

Listing 4-5: Returning ownership of parameters

But this is too much ceremony and a lot of work for a concept that should be common. Luckily for us, **Rust has a feature for using a value without transferring ownership, called *references***.

## 4.2 References and Borrowing

A *reference* is like a pointer in that it’s an address we can follow to access data stored at that address that is owned by some other variable. **Unlike a pointer, a reference is guaranteed to point to a valid value of a particular type**. Here is how you would define and use a `calculate_length` function that has a reference to an object as a parameter instead of taking ownership of the value:

```rust
fn main() {
    let s1 = String::from("hello");

    let len = calculate_length(&s1);

    println!("The length of '{}' is {}.", s1, len);
}

fn calculate_length(s: &String) -> usize {
    s.len()
}
```

First, notice that all the tuple code in the variable declaration and the function return value is gone. Second, note that we pass `&s1` into `calculate_length` and, in its definition, we take `&String` rather than `String`. These ampersands represent *references*, and they allow you to refer to some value **without taking ownership of it**. Figure 4-5 depicts this concept.

![&String s pointing at String s1](https://doc.rust-lang.org/book/img/trpl04-05.svg)

Figure 4-5: A diagram of `&String s` pointing at `String s1`

> Note: The opposite of referencing by using `&` is *dereferencing*, which is accomplished with the dereference operator, `*`. We’ll see some uses of the dereference operator in Chapter 8 and discuss details of dereferencing in Chapter 15.

Likewise, the signature of the function uses `&` to indicate that the type of the parameter `s` is a reference. Let’s add some explanatory annotations:

```rust
fn calculate_length(s: &String) -> usize { // s is a reference to a String
    s.len()
} // Here, s goes out of scope. But because it does not have ownership of what
  // it refers to, nothing happens.
```

**The scope in which the variable `s` is valid is the same as any function parameter’s scope, but the value pointed to by the reference is not dropped when `s` stops being used because `s` doesn’t have ownership**. When functions have references as parameters instead of the actual values, we won’t need to return the values in order to give back ownership, because we never had ownership.

**We call the action of creating a reference *borrowing***. As in real life, if a person owns something, you can borrow it from them. When you’re done, you have to give it back. You don’t own it.

### Mutable References

```rust
fn main() {
    let mut s = String::from("hello");

    change(&mut s);
}

fn change(some_string: &mut String) {
    some_string.push_str(", world");
}
```

First, we change `s` to be `mut`. Then we create a mutable reference with `&mut s` where we call the `change` function, and update the function signature to accept a mutable reference with `some_string: &mut String`. This makes it very clear that the `change` function will mutate the value it borrows.

**Mutable references have one big restriction: you can have only one mutable reference to a particular piece of data at a time**. This code that attempts to create two mutable references to `s` will fail:

```rust
let mut s = String::from("hello");

let r1 = &mut s;
let r2 = &mut s;

println!("{}, {}", r1, r2);
```

**The restriction preventing multiple mutable references to the same data at the same time allows for mutation but in a very controlled fashion**. It’s something that new Rustaceans struggle with, because most languages let you mutate whenever you’d like. **<u>The benefit of having this restriction is that Rust can prevent data races at compile time</u>**. A *data race* is similar to a race condition and happens when these three behaviors occur:

- Two or more pointers access the same data at the same time.
- At least one of the pointers is being used to write to the data.
- There’s no mechanism being used to synchronize access to the data.

**Data races cause undefined behavior and can be difficult to diagnose and fix when you’re trying to track them down at runtime; Rust prevents this problem by refusing to compile code with data races**!

As always, we can use curly brackets to create a new scope, allowing for multiple mutable references, just not *simultaneous* ones:

```rust
let mut s = String::from("hello");

{
  let r1 = &mut s;
} // r1 goes out of scope here, so we can make a new reference with no problems.

let r2 = &mut s;
```

**Rust enforces a similar rule for combining mutable and immutable references**. This code results in an error:

```rust
    let mut s = String::from("hello");

    let r1 = &s; // no problem
    let r2 = &s; // no problem
    let r3 = &mut s; // BIG PROBLEM

    println!("{}, {}, and {}", r1, r2, r3);
```

Here’s the error:

```shell
$ cargo run
   Compiling ownership v0.1.0 (file:///projects/ownership)
error[E0502]: cannot borrow `s` as mutable because it is also borrowed as immutable
 --> src/main.rs:6:14
  |
4 |     let r1 = &s; // no problem
  |              -- immutable borrow occurs here
5 |     let r2 = &s; // no problem
6 |     let r3 = &mut s; // BIG PROBLEM
  |              ^^^^^^ mutable borrow occurs here
7 | 
8 |     println!("{}, {}, and {}", r1, r2, r3);
  |                                -- immutable borrow later used here

For more information about this error, try `rustc --explain E0502`.
error: could not compile `ownership` due to previous error
```

Whew! **We *also* cannot have a mutable reference while we have an immutable one to the same value.** Users of an immutable reference don’t expect the value to suddenly change out from under them! However, multiple immutable references are allowed because no one who is just reading the data has the ability to affect anyone else’s reading of the data.

Note that a reference’s scope starts from where it is introduced and continues through the last time that reference is used. **For instance, this code will compile because the last usage of the immutable references, the `println!`, occurs before the mutable reference is introduced**:

```rust
let mut s = String::from("hello");

let r1 = &s; // no problem
let r2 = &s; // no problem
println!("{} and {}", r1, r2);
// variables r1 and r2 will not be used after this point

let r3 = &mut s; // no problem
println!("{}", r3);
```

The scopes of the immutable references `r1` and `r2` end after the `println!` where they are last used, which is before the mutable reference `r3` is created. **These scopes don’t overlap, so this code is allowed**. The ability of the compiler to tell that a reference is no longer being used at a point before the end of the scope is called *Non-Lexical Lifetimes* (NLL for short), and you can read more about it in [The Edition Guide](https://doc.rust-lang.org/edition-guide/rust-2018/ownership-and-lifetimes/non-lexical-lifetimes.html).

### Dangling References

In languages with pointers, it’s easy to erroneously create a *dangling pointer*--a pointer that references a location in memory that may have been given to someone else--by freeing some memory while preserving a pointer to that memory. **In Rust, by contrast, the compiler guarantees that references will never be dangling references: if you have a reference to some data, the compiler will ensure that the data will not go out of scope before the reference to the data does**.

Let’s try to create a dangling reference to see how Rust prevents them with a compile-time error:

```rust
fn main() {
    let reference_to_nothing = dangle();
}

fn dangle() -> &String {
    let s = String::from("hello");

    &s
}
```

Here’s the error:

```shell
$ cargo run
   Compiling ownership v0.1.0 (file:///projects/ownership)
error[E0106]: missing lifetime specifier
 --> src/main.rs:5:16
  |
5 | fn dangle() -> &String {
  |                ^ expected named lifetime parameter
  |
  = help: this function's return type contains a borrowed value, but there is no value for it to be borrowed from
help: consider using the `'static` lifetime
  |
5 | fn dangle() -> &'static String {
  |                ^^^^^^^^

For more information about this error, try `rustc --explain E0106`.
error: could not compile `ownership` due to previous error
```

Let’s take a closer look at exactly what’s happening at each stage of our `dangle` code:

```rust
fn dangle() -> &String { // dangle returns a reference to a String

    let s = String::from("hello"); // s is a new String

    &s // we return a reference to the String, s
} // Here, s goes out of scope, and is dropped. Its memory goes away.
  // Danger!
```

Because `s` is created inside `dangle`, when the code of `dangle` is finished, `s` will be deallocated. But we tried to return a reference to it. That means this reference would be pointing to an invalid `String`. That’s no good! Rust won’t let us do this.

The solution here is to return the `String` directly:

```rust
fn no_dangle() -> String {
    let s = String::from("hello");

    s
}
```

This works without any problems. Ownership is moved out, and nothing is deallocated.

### The Rules of References

Let’s recap what we’ve discussed about references:

- **At any given time, you can have *either* one mutable reference *or* any number of immutable references.**
- **References must always be valid**.

Next, we’ll look at a different kind of reference: slices.

## 4.3 The Slice Type

*Slices* let you reference a contiguous sequence of elements in a collection rather than the whole collection. **A slice is a kind of reference, so it does not have ownership**.

### String Slices

A *string slice* is a reference to part of a `String`, and it looks like this:

```rust
let s = String::from("hello world");

let hello = &s[0..5];
let world = &s[6..11];
```

Rather than a reference to the entire `String`, `hello` is a reference to a portion of the `String`, specified in the extra `[0..5]` bit. We create slices using a range within brackets by specifying `[starting_index..ending_index]`, where `starting_index` is the first position in the slice and `ending_index` is one more than the last position in the slice. Internally, **the slice data structure stores the starting position and the length of the slice, which corresponds to `ending_index` minus `starting_index`**. So in the case of `let world = &s[6..11];`, `world` would be a slice that contains a pointer to the byte at index 6 of `s` with a length value of 5.

Figure 4-6 shows this in a diagram.

![world containing a pointer to the byte at index 6 of String s and a length 5](https://doc.rust-lang.org/book/img/trpl04-06.svg)

Figure 4-6: String slice referring to part of a `String`

With Rust’s `..` range syntax, if you want to start at index zero, you can drop the value before the two periods. In other words, these are equal:

```rust
let s = String::from("hello");

let slice = &s[0..2];
let slice = &s[..2];
```

By the same token, if your slice includes the last byte of the `String`, you can drop the trailing number. That means these are equal:

```rust
let s = String::from("hello");

let len = s.len();

let slice = &s[3..len];
let slice = &s[3..];
```

You can also drop both values to take a slice of the entire string. So these are equal:

```rust
let s = String::from("hello");

let len = s.len();

let slice = &s[0..len];
let slice = &s[..];
```

> Note: **String slice range indices must occur at valid UTF-8 character boundaries**. **If you attempt to create a string slice in the middle of a multibyte character, your program will exit with an error**. For the purposes of introducing string slices, we are assuming ASCII only in this section; a more thorough discussion of UTF-8 handling is in the [“Storing UTF-8 Encoded Text with Strings”](https://doc.rust-lang.org/book/ch08-02-strings.html#storing-utf-8-encoded-text-with-strings) section of Chapter 8.

With all this information in mind, let’s rewrite `first_word` to return a slice. The type that signifies “string slice” is written as `&str`:

```rust
fn first_word(s: &String) -> &str {
    let bytes = s.as_bytes();

    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return &s[0..i];
        }
    }

    &s[..]
}
```

We now have a straightforward API that’s much harder to mess up, because the compiler will ensure the references into the `String` remain valid. Remember the bug in the program in Listing 4-8, when we got the index to the end of the first word but then cleared the string so our index was invalid? That code was logically incorrect but didn’t show any immediate errors. The problems would show up later if we kept trying to use the first word index with an emptied string. Slices make this bug impossible and let us know we have a problem with our code much sooner. Using the slice version of `first_word` will throw a compile-time error:

```rust
fn main() {
    let mut s = String::from("hello world");

    let word = first_word(&s);

    s.clear(); // error!

    println!("the first word is: {}", word);
}
```

```shell
$ cargo run
   Compiling ownership v0.1.0 (file:///projects/ownership)
error[E0502]: cannot borrow `s` as mutable because it is also borrowed as immutable
  --> src/main.rs:18:5
   |
16 |     let word = first_word(&s);
   |                           -- immutable borrow occurs here
17 | 
18 |     s.clear(); // error!
   |     ^^^^^^^^^ mutable borrow occurs here
19 | 
20 |     println!("the first word is: {}", word);
   |                                       ---- immutable borrow later used here

For more information about this error, try `rustc --explain E0502`.
error: could not compile `ownership` due to previous error
```

Recall from the borrowing rules that if we have an immutable reference to something, we cannot also take a mutable reference. Because `clear` needs to truncate the `String`, it needs to get a mutable reference. **The `println!` after the call to `clear` uses the reference in `word`, so the immutable reference must still be active at that point. Rust disallows the mutable reference in `clear` and the immutable reference in `word` from existing at the same time, and compilation fails**. Not only has Rust made our API easier to use, but it has also eliminated an entire class of errors at compile time!

#### String Literals Are Slices

Recall that we talked about string literals being stored inside the binary. Now that we know about slices, we can properly understand string literals:

```rust
let s = "Hello, world!";
```

**The type of `s` here is `&str`: it’s a slice pointing to that specific point of the binary. This is also why string literals are immutable; `&str` is an immutable reference**.

#### String Slices as Parameters

Knowing that you can take slices of literals and `String` values leads us to one more improvement on `first_word`, and that’s its signature:

```rust
fn first_word(s: &String) -> &str {
```

A more experienced Rustacean would write the signature shown in Listing 4-9 instead because it allows us to use the same function on both `&String` values and `&str` values.

```rust
fn first_word(s: &str) -> &str {
```

Listing 4-9: Improving the `first_word` function by using a string slice for the type of the `s` parameter

If we have a string slice, we can pass that directly. If we have a `String`, we can pass a slice of the `String` or a reference to the `String`. This flexibility takes advantage of *deref coercions*, a feature we will cover in the [“Implicit Deref Coercions with Functions and Methods”](https://doc.rust-lang.org/book/ch15-02-deref.html#implicit-deref-coercions-with-functions-and-methods) section of Chapter 15. Defining a function to take a string slice instead of a reference to a `String` makes our API more general and useful without losing any functionality:

```rust
fn main() {
    let my_string = String::from("hello world");

    // `first_word` works on slices of `String`s, whether partial or whole
    let word = first_word(&my_string[0..6]);
    let word = first_word(&my_string[..]);
    // `first_word` also works on references to `String`s, which are equivalent
    // to whole slices of `String`s
    let word = first_word(&my_string);

    let my_string_literal = "hello world";

    // `first_word` works on slices of string literals, whether partial or whole
    let word = first_word(&my_string_literal[0..6]);
    let word = first_word(&my_string_literal[..]);

    // Because string literals *are* string slices already,
    // this works too, without the slice syntax!
    let word = first_word(my_string_literal);
}
```

### Other Slices

String slices, as you might imagine, are specific to strings. But there’s a more general slice type, too. Consider this array:

```rust
let a = [1, 2, 3, 4, 5];
```

Just as we might want to refer to a part of a string, we might want to refer to part of an array. We’d do so like this:

```rust
let a = [1, 2, 3, 4, 5];

let slice = &a[1..3];

assert_eq!(slice, &[2, 3]);
```

This slice has the type `&[i32]`. It works the same way as string slices do, by storing a reference to the first element and a length. You’ll use this kind of slice for all sorts of other collections. We’ll discuss these collections in detail when we talk about vectors in Chapter 8.

### Summary

The concepts of ownership, borrowing, and slices ensure memory safety in Rust programs at compile time. The Rust language gives you control over your memory usage in the same way as other systems programming languages, but having the owner of data automatically clean up that data when the owner goes out of scope means you don’t have to write and debug extra code to get this control.

Ownership affects how lots of other parts of Rust work, so we’ll talk about these concepts further throughout the rest of the book. Let’s move on to Chapter 5 and look at grouping pieces of data together in a `struct`.

# 5. Using Structs to Structure Related Data

> [Using Structs to Structure Related Data - The Rust Programming Language (rust-lang.org)](https://doc.rust-lang.org/book/ch05-00-structs.html)

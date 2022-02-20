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

A *struct*, or *structure*, is a custom data type that lets you name and package together multiple related values that make up a meaningful group. If you’re familiar with an object-oriented language, a *struct* is like an object’s data attributes. In this chapter, we’ll compare and contrast tuples with structs. We’ll demonstrate how to define and instantiate structs. We’ll discuss how to define associated functions, especially the kind of associated functions called *methods*, to specify behavior associated with a struct type. Structs and enums (discussed in Chapter 6) are the building blocks for creating new types in your program’s domain to take full advantage of Rust’s compile time type checking.

## 5.1 Defining and Instantiating Structs

we define the names and types of the pieces of data, which we call *fields*.

```rust
struct User {
  active: bool,
  username: String,
  email: String,
  sign_in_count: u64,
}
```

实例化struct时，不需要严格按照声明`fields`的顺序给字段赋值。

```rust
let user1 = User {
  email: String::from("someone@example.com"),
  username: String::from("someusername123"),
  active: true,
  sign_in_count: 1,
};
```

当struct实例声明为可变`mut`时，可以对里面的字段重新赋值。

```rust
let mut user1 = User {
  email: String::from("someone@example.com"),
  username: String::from("someusername123"),
  active: true,
  sign_in_count: 1,
};

user1.email = String::from("anotheremail@example.com");
```

> **注意：Rust不允许声明部分fields可变mutable，只能声明整个实例可变**。

Struct实例作为函数返回值-用例：

```rust
fn build_user(email: String, username: String) -> User {
  User {
    email: email,
    username: username,
    active: true,
    sign_in_count: 1,
  }
}
```

### Using the Field Init Shorthand when Variables and Fields Have the Same Name

 we can use the *field init shorthand* syntax to rewrite `build_user` so that it behaves exactly the same but doesn’t have the repetition of `email` and `username`.

当参数名和Struct字段名一致时，可以使用Struct支持的*filed init shorthand* syntax，直接给Struct实例的字段赋值，如下：

```rust
fn build_user(email: String, username: String) -> User {
  User {
    email,
    username,
    active: true,
    sign_in_count: 1,
  }
}
```

### Creating Instances From Other Instances With Struct Update Syntax

如果创建新的Struct实例，与已有的Struct实例仅部分字段值不同，则可以使用*struct update syntax*。

当我们不使用*struct update syntax*时，申明一个类似user1的Struct实例，需要如下操作：

```rust
let user2 = User {
  active: user1.active,
  username: user1.username,
  email: String::from("another@example.com"),
  sign_in_count: user1.sign_in_count,
};
```

Using struct update syntax, we can achieve the same effect with less code, as shown in Listing 5-7. The syntax `..` specifies that the remaining fields not explicitly set should have the same value as the fields in the given instance.

```rust
let user2 = User {
  email: String::from("another@example.com"),
  ..user1
};
```

Listing 5-7: Using struct update syntax to set a new `email` value for a `User` instance but use the rest of the values from `user1`

注意，由于*move*所有权的原因，这里user2中两个没有实现`Copy` **trait**的字段（email和username），其中一个字段username是从user1那move过来的，所以导致user1不能再访问（所有权让给user2了）。除非这里emaiil和username都是user2自己重新赋值的，那么剩下的两个字段（active、sign_in_count）由于都实现了`Copy` **trait**，所以是copy过来的，user1就还能访问。

> **Note that the struct update syntax is like assignment with `=` because it moves the data**, just as we saw in the [“Ways Variables and Data Interact: Move” section](https://doc.rust-lang.org/book/ch04-01-what-is-ownership.html#ways-variables-and-data-interact-move). **In this example, we can no longer use `user1` after creating `user2` because the `String` in the `username` field of `user1` was moved into `user2`**. If we had given `user2` new `String` values for both `email` and `username`, and thus only used the `active` and `sign_in_count` values from `user1`, then `user1` would still be valid after creating `user2`. **The types of `active` and `sign_in_count` are types that implement the `Copy` trait**, so the behavior we discussed in the [“Stack-Only Data: Copy” section](https://doc.rust-lang.org/book/ch04-01-what-is-ownership.html#stack-only-data-copy) would apply.

### Using Tuple Structs without Named Fields to Create Different Types

To define a tuple struct, start with the `struct` keyword and the struct name followed by the types in the tuple. For example, here are definitions and usages of two tuple structs named `Color` and `Point`:

```rust
struct Color(i32, i32, i32);
struct Point(i32, i32, i32);

let black = Color(0, 0, 0);
let origin = Point(0, 0, 0);
```

注意，这里Color和Point虽然拥有同样的字段类型的定义，但不是同一种类型。一个Struct就是一个类型。

tuple struct和tuple类型，都可以通过`.`和下标来访问字段。

### Unit-Like Structs Without Any Fields

就像Unit Tuple`()`似的，Rust也允许定义没有字段的Struct `struct 名;`。

You can also define structs that don’t have any fields! These are called *unit-like structs* because they behave similarly to `()`, the unit type that we mentioned in [“The Tuple Type”](https://doc.rust-lang.org/book/ch03-02-data-types.html#the-tuple-type) section.

**Unit-like structs can be useful in situations in which you need to implement a trait on some type but don’t have any data that you want to store in the type itself**. We’ll discuss traits in Chapter 10.  

```rust
struct AlwaysEqual;

let subject = AlwaysEqual;
```

Imagine we’ll be implementing behavior for this type that every instance is always equal to every instance of every other type, perhaps to have a known result for testing purposes. We wouldn’t need any data to implement that behavior!

### Ownership of Struct Data

In the `User` struct definition in Listing 5-1, we used the owned `String` type rather than the `&str` string slice type. This is a deliberate choice because we want instances of this struct to own all of its data and for that data to be valid for as long as the entire struct is valid.

**It’s possible for structs to store references to data owned by something else, but to do so requires the use of *lifetimes***, a Rust feature that we’ll discuss in Chapter 10. **Lifetimes ensure that the data referenced by a struct is valid for as long as the struct is.** Let’s say you try to store a reference in a struct without specifying lifetimes, like this, which won’t work:

```rust
struct User {
  username: &str,
  email: &str,
  sign_in_count: u64,
  active: bool,
}

fn main() {
  let user1 = User {
    email: "someone@example.com",
    username: "someusername123",
    active: true,
    sign_in_count: 1,
  };
}
```

```shell
The compiler will complain that it needs lifetime specifiers:


$ cargo run
   Compiling structs v0.1.0 (file:///projects/structs)
error[E0106]: missing lifetime specifier
 --> src/main.rs:2:15
  |
2 |     username: &str,
  |               ^ expected named lifetime parameter
  |
help: consider introducing a named lifetime parameter
  |
1 | struct User<'a> {
2 |     username: &'a str,
  |

error[E0106]: missing lifetime specifier
 --> src/main.rs:3:12
  |
3 |     email: &str,
  |            ^ expected named lifetime parameter
  |
help: consider introducing a named lifetime parameter
  |
1 | struct User<'a> {
2 |     username: &str,
3 |     email: &'a str,
  |

error: aborting due to 2 previous errors

For more information about this error, try `rustc --explain E0106`.
error: could not compile `structs`

To learn more, run the command again with --verbose.
In Chapter 10, we’ll discuss how to fix these errors so you can store references in structs, but for now, we’ll fix errors like these using owned types like String instead of references like &str.
```

## 5.2 An Example Program Using Structs

最原始的计算宽高乘积，直接使用function

```rust
fn main() {
  let width1 = 30;
  let height1 = 50;

  println!(
    "The area of the rectangle is {} square pixels.",
    area(width1, height1)
  );
}

fn area(width: u32, height: u32) -> u32 {
  width * height
}
```

### Refactoring with Tuples

使用1个tuple代替原本的两个入参，然而这样两个参数没有名字，可读性降低。

```rust
fn main() {
  let rect1 = (30, 50);

  println!(
    "The area of the rectangle is {} square pixels.",
    area(rect1)
  );
}

fn area(dimensions: (u32, u32)) -> u32 {
  dimensions.0 * dimensions.1
}
```

### Refactoring with Structs: Adding More Meaning

使用struct实现

```rust
struct Rectangle {
  width: u32,
  height: u32,
}

fn main() {
  let rect1 = Rectangle {
    width: 30,
    height: 50,
  };

  println!(
    "The area of the rectangle is {} square pixels.",
    area(&rect1)
  );
}

fn area(rectangle: &Rectangle) -> u32 {
  rectangle.width * rectangle.height
}
```

### Adding Useful Functionality with Derived Traits

```rust
struct Rectangle {
  width: u32,
  height: u32,
}

fn main() {
  let rect1 = Rectangle {
    width: 30,
    height: 50,
  };

  println!("rect1 is {}", rect1);
}
```

When we compile this code, we get an error with this core message:

```shell
error[E0277]: `Rectangle` doesn't implement `std::fmt::Display`
```

**The `println!` macro can do many kinds of formatting, and by default, the curly brackets tell `println!` to use formatting known as `Display`: output intended for direct end user consumption**. The primitive types we’ve seen so far implement `Display` by default, because there’s only one way you’d want to show a `1` or any other primitive type to a user. But with structs, the way `println!` should format the output is less clear because there are more display possibilities: Do you want commas or not? Do you want to print the curly brackets? Should all the fields be shown? Due to this ambiguity, **Rust doesn’t try to guess what we want, and structs don’t have a provided implementation of `Display`**.

```rust
#[derive(Debug)]
struct Rectangle {
  width: u32,
  height: u32,
}

fn main() {
  let rect1 = Rectangle {
    width: 30,
    height: 50,
  };

  println!("rect1 is {:?}", rect1);
}
```

Now when we run the program, we won’t get any errors, and we’ll see the following output:

```rust
$ cargo run
Compiling rectangles v0.1.0 (file:///projects/rectangles)
  Finished dev [unoptimized + debuginfo] target(s) in 0.48s
  Running `target/debug/rectangles`
  rect1 is Rectangle { width: 30, height: 50 }
```

When we use the `{:#?}` style in the example, the output will look like this:

```shell
$ cargo run
   Compiling rectangles v0.1.0 (file:///projects/rectangles)
    Finished dev [unoptimized + debuginfo] target(s) in 0.48s
     Running `target/debug/rectangles`
rect1 is Rectangle {
    width: 30,
    height: 50,
}
```

Another way to print out a value using the `Debug` format is by using the [`dbg!` macro](https://doc.rust-lang.org/std/macro.dbg.html) . **The `dbg!` macro takes ownership of an expression, prints the file and line number of where that `dbg!` macro call occurs in your code along with the resulting value of that expression, and returns ownership of the value**. Calling the `dbg!` macro prints to the standard error console stream (`stderr`), as opposed to `println!` which prints to the standard output console stream (`stdout`). We’ll talk more about `stderr` and `stdout` in the [“Writing Error Messages to Standard Error Instead of Standard Output” section in Chapter 12](https://doc.rust-lang.org/book/ch12-06-writing-to-stderr-instead-of-stdout.html). Here’s an example where we’re interested in the value that gets assigned to the `width` field, as well as the value of the whole struct in `rect1`:

```rust
#[derive(Debug)]
struct Rectangle {
  width: u32,
  height: u32,
}

fn main() {
  let scale = 2;
  let rect1 = Rectangle {
    width: dbg!(30 * scale),
    height: 50,
  };

  dbg!(&rect1);
}
```

We can put `dbg!` around the expression `30 * scale` and, because `dbg!` returns ownership of the expression’s value, the `width` field will get the same value as if we didn’t have the `dbg!` call there. We don’t want `dbg!` to take ownership of `rect1`, so we use a reference to `dbg!` in the next call. Here’s what the output of this example looks like:

```shell
$ cargo run
   Compiling rectangles v0.1.0 (file:///projects/rectangles)
    Finished dev [unoptimized + debuginfo] target(s) in 0.61s
     Running `target/debug/rectangles`
[src/main.rs:10] 30 * scale = 60
[src/main.rs:14] &rect1 = Rectangle {
    width: 60,
    height: 50,
}
```

We can see the first bit of output came from *src/main.rs* line 10, where we’re debugging the expression `30 * scale`, and its resulting value is 60 (the `Debug` formatting implemented for integers is to print only their value). The `dbg!` call on line 14 of *src/main.rs* outputs the value of `&rect1`, which is the `Rectangle` struct. This output uses the pretty `Debug` formatting of the `Rectangle` type. The `dbg!` macro can be really helpful when you’re trying to figure out what your code is doing!

> In addition to the `Debug` trait, Rust has provided a number of traits for us to use with the `derive` attribute that can add useful behavior to our custom types. Those traits and their behaviors are listed in [Appendix C](https://doc.rust-lang.org/book/appendix-03-derivable-traits.html). We’ll cover how to implement these traits with custom behavior as well as how to create your own traits in Chapter 10. There are also many attributes other than `derive`; for more information, see [the “Attributes” section of the Rust Reference](https://doc.rust-lang.org/reference/attributes.html).

## 5.3 Method Syntax

method和function类似的，都是以`fn`声明，并且接受参数，可以有返回值。不同的是，**methods定义在struct（或enum或trait object，Chapters6 and Chapters17）的上下文中，并且第一个参数永远是self，表示struct实例自身**。

### Defining Methods

Let’s change the `area` function that has a `Rectangle` instance as a parameter and instead make an `area` **method** defined on the `Rectangle` struct, as shown in Listing 5-13.

```rust
#[derive(Debug)]
struct Rectangle {
  width: u32,
  height: u32,
}

impl Rectangle {
  fn area(&self) -> u32 {
    self.width * self.height
  }
}

fn main() {
  let rect1 = Rectangle {
    width: 30,
    height: 50,
  };

  println!(
    "The area of the rectangle is {} square pixels.",
    rect1.area()
  );
}
```

Listing 5-13: Defining an `area` method on the `Rectangle` struct

To define the function within the context of `Rectangle`, we start an `impl` (implementation) block for `Rectangle`. **Everything within this `impl` block will be associated with the `Rectangle` type**. Then we move the `area` function within the `impl` curly brackets and change the first (and in this case, only) parameter to be `self` in the signature and everywhere within the body. In `main`, where we called the `area` function and passed `rect1` as an argument, we can instead use *method syntax* to call the `area` method on our `Rectangle` instance. The method syntax goes after an instance: we add a dot followed by the method name, parentheses, and any arguments.

In the signature for `area`, we use `&self` instead of `rectangle: &Rectangle`. **The `&self` is actually short for `self: &Self`. Within an `impl` block, the type `Self` is an alias for the type that the `impl` block is for**. **<u>Methods must have a parameter named `self` of type `Self` for their first parameter, so Rust lets you abbreviate this with only the name `self` in the first parameter spot</u>**. Note that we still need to use the `&` in front of the `self` shorthand to indicate this method borrows the `Self` instance, just as we did in `rectangle: &Rectangle`. Methods can take ownership of `self`, borrow `self` immutably as we’ve done here, or borrow `self` mutably, just as they can any other parameter.

这里使用`&self`和function版本`&Rectangle`类似的，不获取实例的所有权。**如果我们需要修改实例，则使用`&mut self`作为第一个参数**。

Rust允许声明一个和struct字段名一致的method方法，如下method和field都存在`width`：

```rust
impl Rectangle {
  fn width(&self) -> bool {
    self.width > 0
  }
}

fn main() {
  let rect1 = Rectangle {
    width: 30,
    height: 50,
  };

  if rect1.width() {
    println!("The rectangle has a nonzero width; it is {}", rect1.width);
  }
}
```

>  **Where’s the `->` Operator?**
>
> In C and C++, two different operators are used for calling methods: you use `.` if you’re calling a method on the object directly and `->` if you’re calling the method on a pointer to the object and need to dereference the pointer first. In other words, if `object` is a pointer, `object->something()` is similar to `(*object).something()`.
>
> Rust doesn’t have an equivalent to the `->` operator; **instead, Rust has a feature called *automatic referencing and dereferencing***. Calling methods is one of the few places in Rust that has this behavior.
>
> Here’s how it works: when you call a method with `object.something()`, **Rust automatically adds in `&`, `&mut`, or `*` so `object` matches the signature of the method**. In other words, the following are the same:
>
> ```rust
> p1.distance(&p2);
> (&p1).distance(&p2);
> ```
>
> The first one looks much cleaner. **This automatic referencing behavior works because methods have a clear receiver—the type of `self`**. Given the receiver and name of a method, Rust can figure out definitively whether the method is reading (`&self`), mutating (`&mut self`), or consuming (`self`). **The fact that Rust makes borrowing implicit for method receivers is a big part of making ownership ergonomic in practice**.

### Methods with More Parameters

```rust
impl Rectangle {
  fn area(&self) -> u32 {
    self.width * self.height
  }

  fn can_hold(&self, other: &Rectangle) -> bool {
    self.width > other.width && self.height > other.height
  }
}
```

```rust
fn main() {
  let rect1 = Rectangle {
    width: 30,
    height: 50,
  };
  let rect2 = Rectangle {
    width: 10,
    height: 40,
  };
  let rect3 = Rectangle {
    width: 60,
    height: 45,
  };

  println!("Can rect1 hold rect2? {}", rect1.can_hold(&rect2));
  println!("Can rect1 hold rect3? {}", rect1.can_hold(&rect3));
}
```

### Associated Functions

**All functions defined within an `impl` block are called *associated functions* because they’re associated with the type named after the `impl`**. 

**We can define associated functions that don’t have `self` as their first parameter (and thus are not methods) because they don’t need an instance of the type to work with**. We’ve already used one function like this, the `String::from` function, that’s defined on the `String` type.

Associated functions that aren’t methods are often used for constructors that will return a new instance of the struct.

Associated Functions经常用于对象的构造函数

```rust
impl Rectangle {
  fn square(size: u32) -> Rectangle {
    Rectangle {
      width: size,
      height: size,
    }
  }
}
```

**To call this associated function, we use the `::` syntax with the struct name**; `let sq = Rectangle::square(3);` is an example. This function is namespaced by the struct: **the `::` syntax is used for both associated functions and namespaces created by modules**. We’ll discuss modules in Chapter 7.

### Multiple `impl` Blocks

Each struct is allowed to have multiple `impl` blocks. For example, Listing 5-15 is equivalent to the code shown in Listing 5-16, which has each method in its own `impl` block.

```rust
impl Rectangle {
  fn area(&self) -> u32 {
    self.width * self.height
  }
}

impl Rectangle {
  fn can_hold(&self, other: &Rectangle) -> bool {
    self.width > other.width && self.height > other.height
  }
}
```

Listing 5-16: Rewriting Listing 5-15 using multiple `impl` blocks

There’s no reason to separate these methods into multiple `impl` blocks here, but this is valid syntax. We’ll see a case in which multiple `impl` blocks are useful in Chapter 10, where we discuss generic types and traits.

# 6. Enums and Pattern Matching

In this chapter we’ll look at *enumerations*, also referred to as *enums*. Enums allow you to define a type by enumerating its possible *variants*. First, we’ll define and use an enum to show how an enum can encode meaning along with data. Next, we’ll explore a particularly useful enum, called `Option`, which expresses that a value can be either something or nothing. Then we’ll look at how pattern matching in the `match` expression makes it easy to run different code for different values of an enum. Finally, we’ll cover how the `if let` construct is another convenient and concise idiom available to you to handle enums in your code.

Enums are a feature in many languages, but their capabilities differ in each language. Rust’s enums are most similar to *algebraic data types* in functional languages, such as F#, OCaml, and Haskell.

## 6.1 Defining an Enum

```rust
enum IpAddrKind {
  V4,
  V6,
}
```

`IpAddrKind` is now a custom data type that we can use elsewhere in our code.

### Enum Values

```rust
enum IpAddrKind {
  V4,
  V6,
}

struct IpAddr {
  kind: IpAddrKind,
  address: String,
}

let home = IpAddr {
  kind: IpAddrKind::V4,
  address: String::from("127.0.0.1"),
};

let loopback = IpAddr {
  kind: IpAddrKind::V6,
  address: String::from("::1"),
};
```

Listing 6-1: Storing the data and `IpAddrKind` variant of an IP address using a `struct`

We can represent the same concept in a more concise way using just an enum, rather than an enum inside a struct, by putting data directly into each enum variant. This new definition of the `IpAddr` enum says that both `V4` and `V6` variants will have associated `String` values:

```rust
enum IpAddr {
  V4(String),
  V6(String),
}

let home = IpAddr::V4(String::from("127.0.0.1"));

let loopback = IpAddr::V6(String::from("::1"));
```

**There’s another advantage to using an enum rather than a struct: each variant can have different types and amounts of associated data**. Version four type IP addresses will always have four numeric components that will have values between 0 and 255. If we wanted to store `V4` addresses as four `u8` values but still express `V6` addresses as one `String` value, we wouldn’t be able to with a struct. Enums handle this case with ease:

```rust
enum IpAddr {
  V4(u8, u8, u8, u8),
  V6(String),
}

let home = IpAddr::V4(127, 0, 0, 1);

let loopback = IpAddr::V6(String::from("::1"));
```

We’ve shown several different ways to define data structures to store version four and version six IP addresses. However, as it turns out, wanting to store IP addresses and encode which kind they are is so common that [the standard library has a definition we can use!](https://doc.rust-lang.org/std/net/enum.IpAddr.html) Let’s look at how the standard library defines `IpAddr`: it has the exact enum and variants that we’ve defined and used, but it embeds the address data inside the variants in the form of two different structs, which are defined differently for each variant:

```rust

struct Ipv4Addr {
  // --snip--
}

struct Ipv6Addr {
  // --snip--
}

enum IpAddr {
  V4(Ipv4Addr),
  V6(Ipv6Addr),
}
```

> This code illustrates that you can put any kind of data inside an enum variant: strings, numeric types, or structs, for example. You can even include another enum! Also, standard library types are often not much more complicated than what you might come up with.
>
> **Note that even though the standard library contains a definition for `IpAddr`, we can still create and use our own definition without conflict because we haven’t brought the standard library’s definition into our scope**. We’ll talk more about bringing types into scope in Chapter 7.

Let’s look at another example of an enum in Listing 6-2: this one has a wide variety of types embedded in its variants.

```rust
enum Message {
  Quit,
  Move { x: i32, y: i32 },
  Write(String),
  ChangeColor(i32, i32, i32),
}
```

Listing 6-2: A `Message` enum whose variants each store different amounts and types of values

This enum has four variants with different types:

- `Quit` has no data associated with it at all.
- `Move` has named fields like a struct does.
- `Write` includes a single `String`.
- `ChangeColor` includes three `i32` values.

Defining an enum with variants such as the ones in Listing 6-2 is similar to defining different kinds of struct definitions, <u>except the enum doesn’t use the `struct` keyword and all the variants are grouped together under the `Message` type</u>. The following structs could hold the same data that the preceding enum variants hold:

```rust
struct QuitMessage; // unit struct
struct MoveMessage {
    x: i32,
    y: i32,
}
struct WriteMessage(String); // tuple struct
struct ChangeColorMessage(i32, i32, i32); // tuple struct
```

<u>But if we used the different structs, which each have their own type, we couldn’t as easily define a function to take any of these kinds of messages as we could with the `Message` enum defined in Listing 6-2, which is a single type</u>.

There is one more similarity between enums and structs: just as we’re able to define methods on structs using `impl`, **we’re also able to define methods on enums**. Here’s a method named `call` that we could define on our `Message` enum:

```rust
impl Message {
  fn call(&self) {
    // method body would be defined here
  }
}

let m = Message::Write(String::from("hello"));
m.call();
```

Let’s look at another enum in the standard library that is very common and useful: `Option`.

### The `Option` Enum and Its Advantages Over Null Values

**This section explores a case study of `Option`, which is another enum defined by the standard library**. The `Option` type is used in many places because it encodes the very common scenario in which a value could be something or it could be nothing.

**Rust doesn’t have the null feature that many other languages have.** *Null* is a value that means there is no value there. In languages with null, variables can always be in one of two states: null or not-null.

The problem isn’t really with the concept but with the particular implementation. As such, **Rust does not have nulls, but it does have an enum that can encode the concept of a value being present or absent. This enum is `Option<T>`**, and it is [defined by the standard library](https://doc.rust-lang.org/std/option/enum.Option.html) as follows:

```rust
enum Option<T> {
  None,
  Some(T),
}
```

The `Option<T>` enum is so useful that it’s even included in the prelude; you don’t need to bring it into scope explicitly. **In addition, so are its variants: you can use `Some` and `None` directly without the `Option::` prefix**. **The `Option<T>` enum is still just a regular enum, and `Some(T)` and `None` are still variants of type `Option<T>`**.

The `<T>` syntax is a feature of Rust we haven’t talked about yet. It’s a generic type parameter, and we’ll cover generics in more detail in Chapter 10. For now, all you need to know is that `<T>` means the `Some` variant of the `Option` enum can hold one piece of data of any type, and that each concrete type that gets used in place of `T` makes the overall `Option<T>` type a different type. Here are some examples of using `Option` values to hold number types and string types:

```rust
let some_number = Some(5);
let some_string = Some("a string");

let absent_number: Option<i32> = None;
```

**The type of `some_number` is `Option<i32>`. The type of `some_string` is `Option<&str>`, which is a different type. Rust can infer these types because we’ve specified a value inside the `Some` variant**. For `absent_number`, Rust requires us to annotate the overall `Option` type: the compiler can’t infer the type that the corresponding `Some` variant will hold by looking only at a `None` value. Here, we tell Rust that we mean for `absent_number` to be of type `Option<i32>`.

When we have a `Some` value, we know that a value is present and the value is held within the `Some`. When we have a `None` value, in some sense, it means the same thing as null: we don’t have a valid value. So why is having `Option<T>` any better than having null?

**In short, because `Option<T>` and `T` (where `T` can be any type) are different types, the compiler won’t let us use an `Option<T>` value as if it were definitely a valid value**. For example, this code won’t compile because it’s trying to add an `i8` to an `Option<i8>`:

```rust
let x: i8 = 5;
let y: Option<i8> = Some(5);

let sum = x + y;
```

```shell
$ cargo run
   Compiling enums v0.1.0 (file:///projects/enums)
error[E0277]: cannot add `Option<i8>` to `i8`
 --> src/main.rs:5:17
  |
5 |     let sum = x + y;
  |                 ^ no implementation for `i8 + Option<i8>`
  |
  = help: the trait `Add<Option<i8>>` is not implemented for `i8`

For more information about this error, try `rustc --explain E0277`.
error: could not compile `enums` due to previous error
```

Not having to worry about incorrectly assuming a not-null value helps you to be more confident in your code. In order to have a value that can possibly be null, you must explicitly opt in by making the type of that value `Option<T>`. Then, when you use that value, you are required to explicitly handle the case when the value is null. **<u>Everywhere that a value has a type that isn’t an `Option<T>`, you *can* safely assume that the value isn’t null</u>**. This was a deliberate design decision for Rust to limit null’s pervasiveness and increase the safety of Rust code.

> So, how do you get the `T` value out of a `Some` variant when you have a value of type `Option<T>` so you can use that value? The `Option<T>` enum has a large number of methods that are useful in a variety of situations; you can check them out in [its documentation](https://doc.rust-lang.org/std/option/enum.Option.html). Becoming familiar with the methods on `Option<T>` will be extremely useful in your journey with Rust.

## 6.2 The `match` Control Flow Operator

**Rust has an extremely powerful control flow operator called `match` that allows you to compare a value against a series of patterns and then execute code based on which pattern matches**. <u>Patterns can be made up of literal values, variable names, wildcards, and many other things; Chapter 18 covers all the different kinds of patterns and what they do</u>. The power of `match` comes from the expressiveness of the patterns and the fact that the compiler confirms that all possible cases are handled.

```rust
enum Coin {
  Penny,
  Nickel,
  Dime,
  Quarter,
}

fn value_in_cents(coin: Coin) -> u8 {
  match coin {
    Coin::Penny => 1,
    Coin::Nickel => 5,
    Coin::Dime => 10,
    Coin::Quarter => 25,
  }
}
```

Listing 6-3: An enum and a `match` expression that has the variants of the enum as its patterns

When the `match` expression executes, it compares the resulting value against the pattern of each arm, in order. **If a pattern matches the value, the code associated with that pattern is executed. If that pattern doesn’t match the value, execution continues to the next arm**, much as in a coin-sorting machine. We can have as many arms as we need: in Listing 6-3, our `match` has four arms.

*(类似其他语言的case switch，但是不用手动break)*

**The code associated with each arm is an expression**, and the resulting value of the expression in the matching arm is the value that gets returned for the entire `match` expression.

Curly brackets typically aren’t used if the match arm code is short, as it is in Listing 6-3 where each arm just returns a value. **If you want to run multiple lines of code in a match arm, you can use curly brackets.** For example, the following code would print “Lucky penny!” every time the method was called with a `Coin::Penny` but would still return the last value of the block, `1`:

```rust
fn value_in_cents(coin: Coin) -> u8 {
  match coin {
    Coin::Penny => {
      println!("Lucky penny!");
      1
    }
    Coin::Nickel => 5,
    Coin::Dime => 10,
    Coin::Quarter => 25,
  }
}
```

### Patterns that Bind to Values

Another useful feature of match arms is that **they can bind to the parts of the values that match the pattern**. This is how we can extract values out of enum variants.

As an example, let’s change one of our enum variants to hold data inside it. From 1999 through 2008, the United States minted quarters with different designs for each of the 50 states on one side. No other coins got state designs, so only quarters have this extra value. We can add this information to our `enum` by changing the `Quarter` variant to include a `UsState` value stored inside it, which we’ve done here in Listing 6-4.

```rust
#[derive(Debug)] // so we can inspect the state in a minute
enum UsState {
  Alabama,
  Alaska,
  // --snip--
}

enum Coin {
  Penny,
  Nickel,
  Dime,
  Quarter(UsState),
}
```

Listing 6-4: A `Coin` enum in which the `Quarter` variant also holds a `UsState` value

In the match expression for this code, we add a variable called `state` to the pattern that matches values of the variant `Coin::Quarter`. **When a `Coin::Quarter` matches, the `state` variable will bind to the value of that quarter’s state**. Then we can use `state` in the code for that arm, like so:

```rust
fn value_in_cents(coin: Coin) -> u8 {
  match coin {
    Coin::Penny => 1,
    Coin::Nickel => 5,
    Coin::Dime => 10,
    Coin::Quarter(state) => {
      println!("State quarter from {:?}!", state);
      25
    }
  }
}
```

### Matching with `Option<T>`

In the previous section, we wanted to get the inner `T` value out of the `Some` case when using `Option<T>`; we can also handle `Option<T>` using `match` as we did with the `Coin` enum! Instead of comparing coins, we’ll compare the variants of `Option<T>`, but the way that the `match` expression works remains the same.

Let’s say we want to write a function that takes an `Option<i32>` and, if there’s a value inside, adds 1 to that value. If there isn’t a value inside, the function should return the `None` value and not attempt to perform any operations.

This function is very easy to write, thanks to `match`, and will look like Listing 6-5.

```rust
fn plus_one(x: Option<i32>) -> Option<i32> {
  match x {
    None => None,
    Some(i) => Some(i + 1),
  }
}

let five = Some(5);
let six = plus_one(five);
let none = plus_one(None);
```

Listing 6-5: A function that uses a `match` expression on an `Option<i32>`

Combining `match` and enums is useful in many situations. You’ll see this pattern a lot in Rust code: `match` against an enum, bind a variable to the data inside, and then execute code based on it. It’s a bit tricky at first, but once you get used to it, you’ll wish you had it in all languages. It’s consistently a user favorite.

### Matches Are Exhaustive

There’s one other aspect of `match` we need to discuss. Consider this version of our `plus_one` function that has a bug and won’t compile:

```rust
    fn plus_one(x: Option<i32>) -> Option<i32> {
        match x {
            Some(i) => Some(i + 1),
        }
    }
```

We didn’t handle the `None` case, so this code will cause a bug. Luckily, it’s a bug Rust knows how to catch. If we try to compile this code, we’ll get this error:

```shell
$ cargo run
   Compiling enums v0.1.0 (file:///projects/enums)
error[E0004]: non-exhaustive patterns: `None` not covered
   --> src/main.rs:3:15
    |
3   |         match x {
    |               ^ pattern `None` not covered
    |
    = help: ensure that all possible cases are being handled, possibly by adding wildcards or more match arms
    = note: the matched value is of type `Option<i32>`

For more information about this error, try `rustc --explain E0004`.
error: could not compile `enums` due to previous error
```

Rust knows that we didn’t cover every possible case and even knows which pattern we forgot! Matches in Rust are *exhaustive*: we must exhaust every last possibility in order for the code to be valid. Especially in the case of `Option<T>`, when Rust prevents us from forgetting to explicitly handle the `None` case, it protects us from assuming that we have a value when we might have null, thus making the billion-dollar mistake discussed earlier impossible.

### Catch-all Patterns and the `_` Placeholder

Let’s look at an example where we want to take special actions for a few particular values, but for all other values take one default action. Imagine we’re implementing a game where if you get a value of 3 on a dice roll, your player doesn’t move, but instead gets a new fancy hat. If you roll a 7, your player loses a fancy hat. For all other values, your player moves that number of spaces on the game board. Here’s a `match` that implements that logic, with the result of the dice roll hardcoded rather than a random value, and all other logic represented by functions without bodies because actually implementing them is out of scope for this example:

```rust
let dice_roll = 9;
match dice_roll {
  3 => add_fancy_hat(),
  7 => remove_fancy_hat(),
  other => move_player(other),
}

fn add_fancy_hat() {}
fn remove_fancy_hat() {}
fn move_player(num_spaces: u8) {}
```

For the first two arms, the patterns are the literal values 3 and 7. **For the last arm that covers every other possible value, the pattern is the variable we’ve chosen to name `other`.** The code that runs for the `other` arm uses the variable by passing it to the `move_player` function.

This code compiles, even though we haven’t listed all the possible values a `u8` can have, because **the last pattern will match all values not specifically listed**. This catch-all pattern meets the requirement that `match` must be exhaustive. **<u>Note that we have to put the catch-all arm last because the patterns are evaluated in order. Rust will warn us if we add arms after a catch-all because those later arms would never match!</u>**

**Rust also has a pattern we can use when we don’t want to use the value in the catch-all pattern: `_`, which is a special pattern that matches any value and does not bind to that value. This tells Rust we aren’t going to use the value, so Rust won’t warn us about an unused variable**.

Let’s change the rules of the game to be that if you roll anything other than a 3 or a 7, you must roll again. **We don’t need to use the value in that case, so we can change our code to use `_` instead of the variable named `other`**:

```rust
let dice_roll = 9;
match dice_roll {
  3 => add_fancy_hat(),
  7 => remove_fancy_hat(),
  _ => reroll(),
}

fn add_fancy_hat() {}
fn remove_fancy_hat() {}
fn reroll() {}
```

If we change the rules of the game one more time, so that nothing else happens on your turn if you roll anything other than a 3 or a 7, we can express that by using **the unit value** (the empty tuple type we mentioned in [“The Tuple Type”](https://doc.rust-lang.org/book/ch03-02-data-types.html#the-tuple-type) section) as the code that goes with the `_` arm:

```rust
let dice_roll = 9;
match dice_roll {
  3 => add_fancy_hat(),
  7 => remove_fancy_hat(),
  _ => (),
}

fn add_fancy_hat() {}
fn remove_fancy_hat() {}
```

There’s more about patterns and matching that we’ll cover in [Chapter 18](https://doc.rust-lang.org/book/ch18-00-patterns.html). For now, we’re going to move on to the `if let` syntax, which can be useful in situations where the `match` expression is a bit wordy.

## 6.3 Concise Control Flow with `if let`

**The `if let` syntax lets you combine `if` and `let` into a less verbose way to handle values that match one pattern while ignoring the rest.** Consider the program in Listing 6-6 that matches on an `Option<u8>` value in the `config_max` variable but only wants to execute code if the value is the `Some` variant.

```rust
fn main() {
  let config_max = Some(3u8);
  match config_max {
    Some(max) => println!("The maximum is configured to be {}", max),
    _ => (),
  }
}
```

Listing 6-6: A `match` that only cares about executing code when the value is `Some`

If the value is `Some`, we want to print out the value in the `Some` variant, which we do by binding the value to the variable `max` in the pattern. We don’t want to do anything with the `None` value. To satisfy the `match` expression, we have to add `_ => ()` after processing just one variant, which is annoying boilerplate code to add.

Instead, we could write this in a shorter way using `if let`. The following code behaves the same as the `match` in Listing 6-6:

```rust
fn main() {
  let config_max = Some(3u8);
  if let Some(max) = config_max {
    println!("The maximum is configured to be {}", max);
  }
}
```

**The syntax `if let` takes a pattern and an expression separated by an equal sign. It works the same way as a `match`, where the expression is given to the `match` and the pattern is its first arm**. In this case, the pattern is `Some(max)`, and the `max` binds to the value inside the `Some`. We can then use `max` in the body of the `if let` block in the same way as we used `max` in the corresponding `match` arm. The code in the `if let` block isn’t run if the value doesn’t match the pattern.

> Using `if let` means less typing, less indentation, and less boilerplate code. However, you lose the exhaustive checking that `match` enforces. Choosing between `match` and `if let` depends on what you’re doing in your particular situation and whether gaining conciseness is an appropriate trade-off for losing exhaustive checking.

**In other words, you can think of `if let` as syntax sugar for a `match` that runs code when the value matches one pattern and then ignores all other values**.

**We can include an `else` with an `if let`. The block of code that goes with the `else` is the same as the block of code that would go with the `_` case in the `match` expression that is equivalent to the `if let` and `else`**. Recall the `Coin` enum definition in Listing 6-4, where the `Quarter` variant also held a `UsState` value. If we wanted to count all non-quarter coins we see while also announcing the state of the quarters, we could do that with a `match` expression like this:

```rust
let mut count = 0;
match coin {
  Coin::Quarter(state) => println!("State quarter from {:?}!", state),
  _ => count += 1,
}
```

Or we could use an `if let` and `else` expression like this:

```rust
let mut count = 0;
if let Coin::Quarter(state) = coin {
  println!("State quarter from {:?}!", state);
} else {
  count += 1;
}
```

If you have a situation in which your program has logic that is too verbose to express using a `match`, remember that `if let` is in your Rust toolbox as well.

# 7. Managing Growing Projects with Packages, Crates, and Modules

The programs we’ve written so far have been in one module in one file. As a project grows, you can organize code by splitting it into multiple modules and then multiple files. A package can contain multiple binary crates and optionally one library crate. As a package grows, you can extract parts into separate crates that become external dependencies. This chapter covers all these techniques. For very large projects of a set of interrelated packages that evolve together, Cargo provides workspaces, which we’ll cover in the [“Cargo Workspaces”](https://doc.rust-lang.org/book/ch14-03-cargo-workspaces.html) section in Chapter 14.

Rust has a number of features that allow you to manage your code’s organization, including which details are exposed, which details are private, and what names are in each scope in your programs. These features, sometimes collectively referred to as the *module system*, include:

- **Packages:** A Cargo feature that lets you build, test, and share crates
- **Crates:** A tree of modules that produces a library or executable
- **Modules** and **use:** Let you control the organization, scope, and privacy of paths
- **Paths:** A way of naming an item, such as a struct, function, or module

## 7.1 Packages and Crates

The first parts of the module system we’ll cover are packages and crates. A crate is a binary or library. The *crate root* is a source file that the Rust compiler starts from and makes up the root module of your crate (we’ll explain modules in depth in the [“Defining Modules to Control Scope and Privacy”](https://doc.rust-lang.org/book/ch07-02-defining-modules-to-control-scope-and-privacy.html) section). A *package* is one or more crates that provide a set of functionality. A package contains a *Cargo.toml* file that describes how to build those crates.

Several rules determine what a package can contain. **A package can contain at most one library crate. It can contain as many binary crates as you’d like, but it must contain at least one crate (either library or binary)**.

Let’s walk through what happens when we create a package. First, we enter the command `cargo new`:

```shell
$ cargo new my-project
     Created binary (application) `my-project` package
$ ls my-project
Cargo.toml
src
$ ls my-project/src
main.rs
```

When we entered the command, Cargo created a *Cargo.toml* file, giving us a **package**. Looking at the contents of *Cargo.toml*, there’s no mention of *src/main.rs* because Cargo follows a convention that ***src/main.rs* is the crate root of a binary crate with the same name as the package**. Likewise, Cargo knows that **if the package directory contains *src/lib.rs*, the package contains a library crate with the same name as the package, and *src/lib.rs* is its crate root**. Cargo passes the crate root files to `rustc` to build the library or binary.

Here, we have a package that only contains *src/main.rs*, meaning it only contains a binary crate named `my-project`. <u>If a package contains *src/main.rs* and *src/lib.rs*, it has two crates: a library and a binary, both with the same name as the package</u>. **A package can have multiple binary crates by placing files in the *src/bin* directory: each file will be a separate binary crate**.

A crate will group related functionality together in a scope so the functionality is easy to share between multiple projects. For example, the `rand` crate we used in [Chapter 2](https://doc.rust-lang.org/book/ch02-00-guessing-game-tutorial.html#generating-a-random-number) provides functionality that generates random numbers. We can use that functionality in our own projects by bringing the `rand` crate into our project’s scope. All the functionality provided by the `rand` crate is accessible through the crate’s name, `rand`.

<u>Keeping a crate’s functionality in its own scope clarifies whether particular functionality is defined in our crate or the `rand` crate and prevents potential conflicts</u>. For example, the `rand` crate provides a trait named `Rng`. We can also define a `struct` named `Rng` in our own crate. Because a crate’s functionality is namespaced in its own scope, when we add `rand` as a dependency, the compiler isn’t confused about what the name `Rng` refers to. <u>In our crate, it refers to the `struct Rng` that we defined. We would access the `Rng` trait from the `rand` crate as `rand::Rng`</u>.

Let’s move on and talk about the module system!

## 7.2 Defining Modules to Control Scope and Privacy

In this section, we’ll talk about modules and other parts of the module system, namely *paths* that allow you to name items; the `use` keyword that brings a path into scope; and the `pub` keyword to make items public. We’ll also discuss the `as` keyword, external packages, and the glob operator. For now, let’s focus on modules!

*Modules* let us organize code within a crate into groups for readability and easy reuse. **Modules also control the *privacy* of items, which is whether an item can be used by outside code (*public*) or is an internal implementation detail and not available for outside use (*private*)**.

Create a new library named `restaurant` by running `cargo new --lib restaurant`; then put the code in Listing 7-1 into *src/lib.rs* to define some modules and function signatures.

Filename: src/lib.rs

```rust
mod front_of_house {
  mod hosting {
    fn add_to_waitlist() {}

    fn seat_at_table() {}
  }

  mod serving {
    fn take_order() {}

    fn serve_order() {}

    fn take_payment() {}
  }
}
```

Listing 7-1: A `front_of_house` module containing other modules that then contain functions

We define a module by starting with the `mod` keyword and then specify the name of the module (in this case, `front_of_house`) and place curly brackets around the body of the module. Inside modules, we can have other modules, as in this case with the modules `hosting` and `serving`. Modules can also hold definitions for other items, such as structs, enums, constants, traits, or—as in Listing 7-1—functions.

By using modules, we can group related definitions together and name why they’re related. Programmers using this code would have an easier time finding the definitions they wanted to use because they could navigate the code based on the groups rather than having to read through all the definitions. Programmers adding new functionality to this code would know where to place the code to keep the program organized.

**Earlier, we mentioned that *src/main.rs* and *src/lib.rs* are called crate roots. The reason for their name is that the contents of either of these two files form a module named `crate` at the root of the crate’s module structure, known as the *module tree***.

Listing 7-2 shows the module tree for the structure in Listing 7-1.

```shell
crate
 └── front_of_house
     ├── hosting
     │   ├── add_to_waitlist
     │   └── seat_at_table
     └── serving
         ├── take_order
         ├── serve_order
         └── take_payment
```

Listing 7-2: The module tree for the code in Listing 7-1

This tree shows how some of the modules nest inside one another (for example, `hosting` nests inside `front_of_house`). The tree also shows that some modules are *siblings* to each other, meaning they’re defined in the same module (`hosting` and `serving` are defined within `front_of_house`). To continue the family metaphor, if module A is contained inside module B, we say that module A is the *child* of module B and that module B is the *parent* of module A. **Notice that the entire module tree is rooted under the implicit module named `crate`**.

The module tree might remind you of the filesystem’s directory tree on your computer; this is a very apt comparison! Just like directories in a filesystem, you use modules to organize your code. And just like files in a directory, we need a way to find our modules.

## 7.3 Paths for Referring to an Item in the Module Tree

To show Rust where to find an item in a module tree, we use a path in the same way we use a path when navigating a filesystem. If we want to call a function, we need to know its path.

A path can take two forms:

- An ***absolute path*** starts from a crate root by using a crate name or a literal `crate`.
- A ***relative path*** starts from the current module and uses `self`, `super`, or an identifier in the current module.

Both absolute and relative paths are followed by one or more identifiers separated by double colons (`::`).

Let’s return to the example in Listing 7-1. How do we call the `add_to_waitlist` function? This is the same as asking, what’s the path of the `add_to_waitlist` function? In Listing 7-3, we simplified our code a bit by removing some of the modules and functions. We’ll show two ways to call the `add_to_waitlist` function from a new function `eat_at_restaurant` defined in the crate root. The `eat_at_restaurant` function is part of our library crate’s public API, so we mark it with the `pub` keyword. In the [”Exposing Paths with the `pub` Keyword”](https://doc.rust-lang.org/book/ch07-03-paths-for-referring-to-an-item-in-the-module-tree.html#exposing-paths-with-the-pub-keyword) section, we’ll go into more detail about `pub`. Note that this example won’t compile just yet; we’ll explain why in a bit.

Filename: src/lib.rs

```rust
mod front_of_house {
  mod hosting {
    fn add_to_waitlist() {}
  }
}

pub fn eat_at_restaurant() {
  // Absolute path
  crate::front_of_house::hosting::add_to_waitlist();

  // Relative path
  front_of_house::hosting::add_to_waitlist();
}
```

Listing 7-3: Calling the `add_to_waitlist` function using absolute and relative paths

The first time we call the `add_to_waitlist` function in `eat_at_restaurant`, we use an absolute path. The `add_to_waitlist` function is defined in the same crate as `eat_at_restaurant`, which means we can use the `crate` keyword to start an absolute path.

After `crate`, we include each of the successive modules until we make our way to `add_to_waitlist`. You can imagine a filesystem with the same structure, and we’d specify the path `/front_of_house/hosting/add_to_waitlist` to run the `add_to_waitlist` program; using the `crate` name to start from the crate root is like using `/` to start from the filesystem root in your shell.

The second time we call `add_to_waitlist` in `eat_at_restaurant`, we use a relative path. The path starts with `front_of_house`, the name of the module defined at the same level of the module tree as `eat_at_restaurant`. Here the filesystem equivalent would be using the path `front_of_house/hosting/add_to_waitlist`. Starting with a name means that the path is relative.

Choosing whether to use a relative or absolute path is a decision you’ll make based on your project. The decision should depend on whether you’re more likely to move item definition code separately from or together with the code that uses the item. For example, if we move the `front_of_house` module and the `eat_at_restaurant` function into a module named `customer_experience`, we’d need to update the absolute path to `add_to_waitlist`, but the relative path would still be valid. However, if we moved the `eat_at_restaurant` function separately into a module named `dining`, the absolute path to the `add_to_waitlist` call would stay the same, but the relative path would need to be updated. <u>Our preference is to specify absolute paths because it’s more likely to move code definitions and item calls independently of each other</u>.

Let’s try to compile Listing 7-3 and find out why it won’t compile yet! The error we get is shown in Listing 7-4.

```shell
$ cargo build
   Compiling restaurant v0.1.0 (file:///projects/restaurant)
error[E0603]: module `hosting` is private
 --> src/lib.rs:9:28
  |
9 |     crate::front_of_house::hosting::add_to_waitlist();
  |                            ^^^^^^^ private module
  |
note: the module `hosting` is defined here
 --> src/lib.rs:2:5
  |
2 |     mod hosting {
  |     ^^^^^^^^^^^

error[E0603]: module `hosting` is private
  --> src/lib.rs:12:21
   |
12 |     front_of_house::hosting::add_to_waitlist();
   |                     ^^^^^^^ private module
   |
note: the module `hosting` is defined here
  --> src/lib.rs:2:5
   |
2  |     mod hosting {
   |     ^^^^^^^^^^^

For more information about this error, try `rustc --explain E0603`.
error: could not compile `restaurant` due to 2 previous errors
```

Listing 7-4: Compiler errors from building the code in Listing 7-3

The error messages say that module `hosting` is private. In other words, we have the correct paths for the `hosting` module and the `add_to_waitlist` function, but **Rust won’t let us use them because it doesn’t have access to the private sections**.

Modules aren’t useful only for organizing your code. They also define Rust’s *privacy boundary*: the line that encapsulates the implementation details external code isn’t allowed to know about, call, or rely on. **So, if you want to make an item like a function or struct private, you put it in a module**.

**The way privacy works in Rust is that all items (functions, methods, structs, enums, modules, and constants) are private by default**. <u>Items in a parent module can’t use the private items inside child modules, but items in child modules can use the items in their ancestor modules. The reason is that child modules wrap and hide their implementation details, but the child modules can see the context in which they’re defined</u>. To continue with the restaurant metaphor, think of the privacy rules as being like the back office of a restaurant: what goes on in there is private to restaurant customers, but office managers can see and do everything in the restaurant in which they operate.

Rust chose to have the module system function this way so that hiding inner implementation details is the default. That way, you know which parts of the inner code you can change without breaking outer code. But **you can expose inner parts of child modules’ code to outer ancestor modules by using the `pub` keyword to make an item public**.

### Exposing Paths with the `pub` Keyword

Let’s return to the error in Listing 7-4 that told us the `hosting` module is private. We want the `eat_at_restaurant` function in the parent module to have access to the `add_to_waitlist` function in the child module, so we mark the `hosting` module with the `pub` keyword, as shown in Listing 7-5.

Filename: src/lib.rs

```rust
mod front_of_house {
  pub mod hosting {
    fn add_to_waitlist() {}
  }
}

pub fn eat_at_restaurant() {
  // Absolute path
  crate::front_of_house::hosting::add_to_waitlist();

  // Relative path
  front_of_house::hosting::add_to_waitlist();
}
```

Listing 7-5: Declaring the `hosting` module as `pub` to use it from `eat_at_restaurant`

Unfortunately, the code in Listing 7-5 still results in an error, as shown in Listing 7-6.

```rust
$ cargo build
   Compiling restaurant v0.1.0 (file:///projects/restaurant)
error[E0603]: function `add_to_waitlist` is private
 --> src/lib.rs:9:37
  |
9 |     crate::front_of_house::hosting::add_to_waitlist();
  |                                     ^^^^^^^^^^^^^^^ private function
  |
note: the function `add_to_waitlist` is defined here
 --> src/lib.rs:3:9
  |
3 |         fn add_to_waitlist() {}
  |         ^^^^^^^^^^^^^^^^^^^^

error[E0603]: function `add_to_waitlist` is private
  --> src/lib.rs:12:30
   |
12 |     front_of_house::hosting::add_to_waitlist();
   |                              ^^^^^^^^^^^^^^^ private function
   |
note: the function `add_to_waitlist` is defined here
  --> src/lib.rs:3:9
   |
3  |         fn add_to_waitlist() {}
   |         ^^^^^^^^^^^^^^^^^^^^

For more information about this error, try `rustc --explain E0603`.
error: could not compile `restaurant` due to 2 previous errors
```

Listing 7-6: Compiler errors from building the code in Listing 7-5

What happened? Adding the `pub` keyword in front of `mod hosting` makes the module public. With this change, if we can access `front_of_house`, we can access `hosting`. But the *contents* of `hosting` are still private; **making the module public doesn’t make its contents public. The `pub` keyword on a module only lets code in its ancestor modules refer to it**.

The errors in Listing 7-6 say that the `add_to_waitlist` function is private. **The privacy rules apply to structs, enums, functions, and methods as well as modules**.

Let’s also make the `add_to_waitlist` function public by adding the `pub` keyword before its definition, as in Listing 7-7.

Filename: src/lib.rs

```rust
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

pub fn eat_at_restaurant() {
    // Absolute path
    crate::front_of_house::hosting::add_to_waitlist();

    // Relative path
    front_of_house::hosting::add_to_waitlist();
}
```

Listing 7-7: Adding the `pub` keyword to `mod hosting` and `fn add_to_waitlist` lets us call the function from `eat_at_restaurant`

Now the code will compile! Let’s look at the absolute and the relative path and double-check why adding the `pub` keyword lets us use these paths in `add_to_waitlist` with respect to the privacy rules.

In the absolute path, we start with `crate`, the root of our crate’s module tree. Then the `front_of_house` module is defined in the crate root. **The `front_of_house` module isn’t public, but because the `eat_at_restaurant` function is defined in the same module as `front_of_house` (that is, `eat_at_restaurant` and `front_of_house` are siblings), we can refer to `front_of_house` from `eat_at_restaurant`.** Next is the `hosting` module marked with `pub`. We can access the parent module of `hosting`, so we can access `hosting`. Finally, the `add_to_waitlist` function is marked with `pub` and we can access its parent module, so this function call works!

In the relative path, the logic is the same as the absolute path except for the first step: rather than starting from the crate root, the path starts from `front_of_house`. The `front_of_house` module is defined within the same module as `eat_at_restaurant`, so the relative path starting from the module in which `eat_at_restaurant` is defined works. Then, because `hosting` and `add_to_waitlist` are marked with `pub`, the rest of the path works, and this function call is valid!

### Starting Relative Paths with `super`

We can also construct relative paths that begin in the parent module by using `super` at the start of the path. This is like starting a filesystem path with the `..` syntax. Why would we want to do this?

Consider the code in Listing 7-8 that models the situation in which a chef fixes an incorrect order and personally brings it out to the customer. The function `fix_incorrect_order` calls the function `serve_order` by specifying the path to `serve_order` starting with `super`:

```rust
fn serve_order() {}

mod back_of_house {
    fn fix_incorrect_order() {
        cook_order();
        super::serve_order();
    }

    fn cook_order() {}
}
```

Listing 7-8: Calling a function using a relative path starting with `super`

The `fix_incorrect_order` function is in the `back_of_house` module, so we can use `super` to go to the parent module of `back_of_house`, which in this case is `crate`, the root. From there, we look for `serve_order` and find it. Success! We think the `back_of_house` module and the `serve_order` function are likely to stay in the same relationship to each other and get moved together should we decide to reorganize the crate’s module tree. Therefore, we used `super` so we’ll have fewer places to update code in the future if this code gets moved to a different module.

### Making Structs and Enums Public

+ **struct声明pub时，内部的所有fields默认还是private，需要单独声明pub；**

+ **enum声明pub时，内部所有enum variants默认pub**

We can also use `pub` to designate structs and enums as public, but there are a few extra details. **If we use `pub` before a struct definition, we make the struct public, but the struct’s fields will still be private**. We can make each field public or not on a case-by-case basis. In Listing 7-9, we’ve defined a public `back_of_house::Breakfast` struct with a public `toast` field but a private `seasonal_fruit` field. This models the case in a restaurant where the customer can pick the type of bread that comes with a meal, but the chef decides which fruit accompanies the meal based on what’s in season and in stock. The available fruit changes quickly, so customers can’t choose the fruit or even see which fruit they’ll get.

Filename: src/lib.rs

```rust
mod back_of_house {
    pub struct Breakfast {
        pub toast: String,
        seasonal_fruit: String,
    }

    impl Breakfast {
        pub fn summer(toast: &str) -> Breakfast {
            Breakfast {
                toast: String::from(toast),
                seasonal_fruit: String::from("peaches"),
            }
        }
    }
}

pub fn eat_at_restaurant() {
    // Order a breakfast in the summer with Rye toast
    let mut meal = back_of_house::Breakfast::summer("Rye");
    // Change our mind about what bread we'd like
    meal.toast = String::from("Wheat");
    println!("I'd like {} toast please", meal.toast);

    // The next line won't compile if we uncomment it; we're not allowed
    // to see or modify the seasonal fruit that comes with the meal
    // meal.seasonal_fruit = String::from("blueberries");
}
```

Listing 7-9: A struct with some public fields and some private fields

Because the `toast` field in the `back_of_house::Breakfast` struct is public, in `eat_at_restaurant` we can write and read to the `toast` field using dot notation. Notice that we can’t use the `seasonal_fruit` field in `eat_at_restaurant` because `seasonal_fruit` is private. Try uncommenting the line modifying the `seasonal_fruit` field value to see what error you get!

Also, note that because `back_of_house::Breakfast` has a private field, the struct needs to provide a public associated function that constructs an instance of `Breakfast` (we’ve named it `summer` here). If `Breakfast` didn’t have such a function, we couldn’t create an instance of `Breakfast` in `eat_at_restaurant` because we couldn’t set the value of the private `seasonal_fruit` field in `eat_at_restaurant`.

**In contrast, if we make an enum public, all of its variants are then public**. We only need the `pub` before the `enum` keyword, as shown in Listing 7-10.

Filename: src/lib.rs

```rust
mod back_of_house {
    pub enum Appetizer {
        Soup,
        Salad,
    }
}

pub fn eat_at_restaurant() {
    let order1 = back_of_house::Appetizer::Soup;
    let order2 = back_of_house::Appetizer::Salad;
}
```

Listing 7-10: Designating an enum as public makes all its variants public

Because we made the `Appetizer` enum public, we can use the `Soup` and `Salad` variants in `eat_at_restaurant`. Enums aren’t very useful unless their variants are public; **it would be annoying to have to annotate all enum variants with `pub` in every case, so the default for enum variants is to be public. Structs are often useful without their fields being public, so struct fields follow the general rule of everything being private by default unless annotated with `pub`**.

There’s one more situation involving `pub` that we haven’t covered, and that is our last module system feature: the `use` keyword. We’ll cover `use` by itself first, and then we’ll show how to combine `pub` and `use`.

## 7.4 Bringing Paths into Scope with the `use` Keyword

It might seem like the paths we’ve written to call functions so far are inconveniently long and repetitive. For example, in Listing 7-7, whether we chose the absolute or relative path to the `add_to_waitlist` function, every time we wanted to call `add_to_waitlist` we had to specify `front_of_house` and `hosting` too. **Fortunately, there’s a way to simplify this process. We can bring a path into a scope once and then call the items in that path as if they’re local items with the `use` keyword**.

In Listing 7-11, we bring the `crate::front_of_house::hosting` module into the scope of the `eat_at_restaurant` function so we only have to specify `hosting::add_to_waitlist` to call the `add_to_waitlist` function in `eat_at_restaurant`.

Filename: src/lib.rs

```rust
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

use crate::front_of_house::hosting;

pub fn eat_at_restaurant() {
    hosting::add_to_waitlist();
    hosting::add_to_waitlist();
    hosting::add_to_waitlist();
}
```

Listing 7-11: Bringing a module into scope with `use`

**Adding `use` and a path in a scope is similar to creating a symbolic link in the filesystem**. By adding `use crate::front_of_house::hosting` in the crate root, `hosting` is now a valid name in that scope, just as though the `hosting` module had been defined in the crate root. Paths brought into scope with `use` also check privacy, like any other paths.

**You can also bring an item into scope with `use` and a relative path**. Listing 7-12 shows how to specify a relative path to get the same behavior as in Listing 7-11.

Filename: src/lib.rs

```rust
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

use self::front_of_house::hosting;

pub fn eat_at_restaurant() {
    hosting::add_to_waitlist();
    hosting::add_to_waitlist();
    hosting::add_to_waitlist();
}
```

Listing 7-12: Bringing a module into scope with `use` and a relative path

### Creating Idiomatic `use` Paths

In Listing 7-11, you might have wondered why we specified `use crate::front_of_house::hosting` and then called `hosting::add_to_waitlist` in `eat_at_restaurant` rather than specifying the `use` path all the way out to the `add_to_waitlist` function to achieve the same result, as in Listing 7-13.

Filename: src/lib.rs

```rust
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

use crate::front_of_house::hosting::add_to_waitlist;

pub fn eat_at_restaurant() {
    add_to_waitlist();
    add_to_waitlist();
    add_to_waitlist();
}
```

Listing 7-13: Bringing the `add_to_waitlist` function into scope with `use`, which is unidiomatic

Although both Listing 7-11 and 7-13 accomplish the same task, Listing 7-11 is the idiomatic way to bring a function into scope with `use`. **Bringing the function’s parent module into scope with `use` means we have to specify the parent module when calling the function. Specifying the parent module when calling the function makes it clear that the function isn’t locally defined while still minimizing repetition of the full path**. <u>The code in Listing 7-13 is unclear as to where `add_to_waitlist` is defined.</u>

**On the other hand, when bringing in structs, enums, and other items with `use`, it’s idiomatic to specify the full path**. Listing 7-14 shows the idiomatic way to bring the standard library’s `HashMap` struct into the scope of a binary crate.

Filename: src/main.rs

```rust
use std::collections::HashMap;

fn main() {
    let mut map = HashMap::new();
    map.insert(1, 2);
}
```

Listing 7-14: Bringing `HashMap` into scope in an idiomatic way

There’s no strong reason behind this idiom: it’s just the convention that has emerged, and folks have gotten used to reading and writing Rust code this way.

**The exception to this idiom is if we’re bringing two items with the same name into scope with `use` statements, because Rust doesn’t allow that**. Listing 7-15 shows how to bring two `Result` types into scope that have the same name but different parent modules and how to refer to them.

Filename: src/lib.rs

```rust
use std::fmt;
use std::io;

fn function1() -> fmt::Result {
    // --snip--
}

fn function2() -> io::Result<()> {
    // --snip--
}
```

Listing 7-15: Bringing two types with the same name into the same scope requires using their parent modules.

As you can see, **using the parent modules distinguishes the two `Result` types**. If instead we specified `use std::fmt::Result` and `use std::io::Result`, we’d have two `Result` types in the same scope and Rust wouldn’t know which one we meant when we used `Result`.

### Providing New Names with the `as` Keyword

**There’s another solution to the problem of bringing two types of the same name into the same scope with `use`: after the path, we can specify `as` and a new local name, or alias, for the type**. Listing 7-16 shows another way to write the code in Listing 7-15 by renaming one of the two `Result` types using `as`.

Filename: src/lib.rs

```rust
use std::fmt::Result;
use std::io::Result as IoResult;

fn function1() -> Result {
    // --snip--
}

fn function2() -> IoResult<()> {
    // --snip--
}
```

Listing 7-16: Renaming a type when it’s brought into scope with the `as` keyword

In the second `use` statement, we chose the new name `IoResult` for the `std::io::Result` type, which won’t conflict with the `Result` from `std::fmt` that we’ve also brought into scope. Listing 7-15 and Listing 7-16 are considered idiomatic, so the choice is up to you!

### Re-exporting Names with `pub use`

**When we bring a name into scope with the `use` keyword, the name available in the new scope is private**. To enable the code that calls our code to refer to that name as if it had been defined in that code’s scope, we can combine `pub` and `use`. **This technique is called *re-exporting* because we’re bringing an item into scope but also making that item available for others to bring into their scope**.

Listing 7-17 shows the code in Listing 7-11 with `use` in the root module changed to `pub use`.

Filename: src/lib.rs

```rust
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

pub use crate::front_of_house::hosting;

pub fn eat_at_restaurant() {
    hosting::add_to_waitlist();
    hosting::add_to_waitlist();
    hosting::add_to_waitlist();
}
```

Listing 7-17: Making a name available for any code to use from a new scope with `pub use`

By using `pub use`, external code can now call the `add_to_waitlist` function using `hosting::add_to_waitlist`. If we hadn’t specified `pub use`, the `eat_at_restaurant` function could call `hosting::add_to_waitlist` in its scope, but external code couldn’t take advantage of this new path.

Re-exporting is useful when the internal structure of your code is different from how programmers calling your code would think about the domain. For example, in this restaurant metaphor, the people running the restaurant think about “front of house” and “back of house.” But customers visiting a restaurant probably won’t think about the parts of the restaurant in those terms. With `pub use`, we can write our code with one structure but expose a different structure. Doing so makes our library well organized for programmers working on the library and programmers calling the library.

### Using External Packages

In Chapter 2, we programmed a guessing game project that used an external package called `rand` to get random numbers. To use `rand` in our project, we added this line to *Cargo.toml*:

Filename: Cargo.toml

```toml
rand = "0.8.3"
```

Adding `rand` as a dependency in *Cargo.toml* tells Cargo to download the `rand` package and any dependencies from [crates.io](https://crates.io/) and make `rand` available to our project.

Then, to bring `rand` definitions into the scope of our package, we added a `use` line starting with the name of the crate, `rand`, and listed the items we wanted to bring into scope. Recall that in the [“Generating a Random Number”](https://doc.rust-lang.org/book/ch02-00-guessing-game-tutorial.html#generating-a-random-number) section in Chapter 2, we brought the `Rng` trait into scope and called the `rand::thread_rng` function:

```rust
use rand::Rng;

fn main() {
    let secret_number = rand::thread_rng().gen_range(1..101);
}
```

**Members of the Rust community have made many packages available at [crates.io](https://crates.io/), and pulling any of them into your package involves these same steps: listing them in your package’s *Cargo.toml* file and using `use` to bring items from their crates into scope.**

**Note that the standard library (`std`) is also a crate that’s external to our package**. Because the standard library is shipped with the Rust language, we don’t need to change *Cargo.toml* to include `std`. But we do need to refer to it with `use` to bring items from there into our package’s scope. For example, with `HashMap` we would use this line:

```rust
use std::collections::HashMap;
```

**This is an absolute path starting with `std`, the name of the standard library crate**.

### Using Nested Paths to Clean Up Large `use` Lists

If we’re using multiple items defined in the same crate or same module, listing each item on its own line can take up a lot of vertical space in our files. For example, these two `use` statements we had in the Guessing Game in Listing 2-4 bring items from `std` into scope:

Filename: src/main.rs

```rust
// --snip--
use std::cmp::Ordering;
use std::io;
// --snip--
```

Instead, **we can use nested paths to bring the same items into scope in one line**. We do this by specifying the common part of the path, followed by two colons, and then curly brackets around a list of the parts of the paths that differ, as shown in Listing 7-18.

Filename: src/main.rs

```rust
// --snip--
use std::{cmp::Ordering, io};
// --snip--
```

Listing 7-18: Specifying a nested path to bring multiple items with the same prefix into scope

In bigger programs, bringing many items into scope from the same crate or module using nested paths can reduce the number of separate `use` statements needed by a lot!

**We can use a nested path at any level in a path, which is useful when combining two `use` statements that share a subpath**. For example, Listing 7-19 shows two `use` statements: one that brings `std::io` into scope and one that brings `std::io::Write` into scope.

Filename: src/lib.rs

```rust
use std::io;
use std::io::Write;
```

Listing 7-19: Two `use` statements where one is a subpath of the other

The common part of these two paths is `std::io`, and that’s the complete first path. **To merge these two paths into one `use` statement, we can use `self` in the nested path**, as shown in Listing 7-20.

Filename: src/lib.rs

```rust
use std::io::{self, Write};
```

Listing 7-20: Combining the paths in Listing 7-19 into one `use` statement

This line brings `std::io` and `std::io::Write` into scope.

### The Glob Operator

**If we want to bring *all* public items defined in a path into scope, we can specify that path followed by `*`**, the glob operator:

```rust
use std::collections::*;
```

This `use` statement brings all public items defined in `std::collections` into the current scope. Be careful when using the glob operator! Glob can make it harder to tell what names are in scope and where a name used in your program was defined.

**The glob operator is often used when testing to bring everything under test into the `tests` module**; we’ll talk about that in the [“How to Write Tests”](https://doc.rust-lang.org/book/ch11-01-writing-tests.html#how-to-write-tests) section in Chapter 11. The glob operator is also sometimes used as part of the prelude pattern: see [the standard library documentation](https://doc.rust-lang.org/std/prelude/index.html#other-preludes) for more information on that pattern.

## 7.5 Separating Modules into Different Files

> [Rust:mod、crate、super、self、pub use等模块系统用法梳理_Julia & Rust & Python-CSDN博客_rust super](https://blog.csdn.net/wowotuo/article/details/107591501)

So far, all the examples in this chapter defined multiple modules in one file. When modules get large, you might want to move their definitions to a separate file to make the code easier to navigate.

For example, let’s start from the code in Listing 7-17 and move the `front_of_house` module to its own file *src/front_of_house.rs* by changing the crate root file so it contains the code shown in Listing 7-21. **In this case, the crate root file is *src/lib.rs*, but this procedure also works with binary crates whose crate root file is *src/main.rs***.

Filename: src/lib.rs

```rust
mod front_of_house;

pub use crate::front_of_house::hosting;

pub fn eat_at_restaurant() {
    hosting::add_to_waitlist();
    hosting::add_to_waitlist();
    hosting::add_to_waitlist();
}
```

Listing 7-21: Declaring the `front_of_house` module whose body will be in *src/front_of_house.rs*

And *src/front_of_house.rs* gets the definitions from the body of the `front_of_house` module, as shown in Listing 7-22.

Filename: src/front_of_house.rs

```rust
pub mod hosting {
    pub fn add_to_waitlist() {}
}
```

Listing 7-22: Definitions inside the `front_of_house` module in *src/front_of_house.rs*

**Using a semicolon after `mod front_of_house` rather than using a block tells Rust to load the contents of the module from another file with the same name as the module**. To continue with our example and extract the `hosting` module to its own file as well, we change *src/front_of_house.rs* to contain only the declaration of the `hosting` module:

Filename: src/front_of_house.rs

```rust
pub mod hosting;
```

Then we create a *src/front_of_house* directory and a file *src/front_of_house/hosting.rs* to contain the definitions made in the `hosting` module:

Filename: src/front_of_house/hosting.rs

```rust
pub fn add_to_waitlist() {}
```

The module tree remains the same, and the function calls in `eat_at_restaurant` will work without any modification, even though the definitions live in different files. This technique lets you move modules to new files as they grow in size.

Note that the `pub use crate::front_of_house::hosting` statement in *src/lib.rs* also hasn’t changed, nor does `use` have any impact on what files are compiled as part of the crate. **The `mod` keyword declares modules, and Rust looks in a file with the same name as the module for the code that goes into that module**.

### Summary

Rust lets you split a package into multiple crates and a crate into modules so you can refer to items defined in one module from another module. You can do this by specifying absolute or relative paths. These paths can be brought into scope with a `use` statement so you can use a shorter path for multiple uses of the item in that scope. Module code is private by default, but you can make definitions public by adding the `pub` keyword.

In the next chapter, we’ll look at some collection data structures in the standard library that you can use in your neatly organized code.

# 8. Common Collections

Rust’s standard library includes a number of very useful data structures called *collections*. Most other data types represent one specific value, but collections can contain multiple values. **Unlike the built-in array and tuple types, the data these collections point to is stored on the heap**, which means the amount of data does not need to be known at compile time and can grow or shrink as the program runs. Each kind of collection has different capabilities and costs, and choosing an appropriate one for your current situation is a skill you’ll develop over time. In this chapter, we’ll discuss three collections that are used very often in Rust programs:

- A *vector* allows you to store a variable number of values next to each other.
- A *string* is a collection of characters. We’ve mentioned the `String` type previously, but in this chapter we’ll talk about it in depth.
- A *hash map* allows you to associate a value with a particular key. It’s a particular implementation of the more general data structure called a *map*.

To learn about the other kinds of collections provided by the standard library, see [the documentation](https://doc.rust-lang.org/std/collections/index.html).

We’ll discuss how to create and update vectors, strings, and hash maps, as well as what makes each special.

## 8.1 Storing Lists of Values with Vectors

The first collection type we’ll look at is `Vec<T>`, also known as a *vector*. Vectors allow you to store more than one value in a single data structure that puts all the values next to each other in memory. **Vectors can only store values of the same type**. They are useful when you have a list of items, such as the lines of text in a file or the prices of items in a shopping cart.

### Creating a New Vector

To create a new, empty vector, we can call the `Vec::new` function, as shown in Listing 8-1.

```rust
let v: Vec<i32> = Vec::new();
```

Listing 8-1: Creating a new, empty vector to hold values of type `i32`

<u>Note that we added a type annotation here. Because we aren’t inserting any values into this vector, Rust doesn’t know what kind of elements we intend to store</u>. This is an important point. Vectors are implemented using generics; we’ll cover how to use generics with your own types in Chapter 10. For now, know that the `Vec<T>` type provided by the standard library can hold any type, and when a specific vector holds a specific type, the type is specified within angle brackets. In Listing 8-1, we’ve told Rust that the `Vec<T>` in `v` will hold elements of the `i32` type.

**In more realistic code, Rust can often infer the type of value you want to store once you insert values, so you rarely need to do this type annotation. It’s more common to create a `Vec<T>` that has initial values, and Rust provides the `vec!` macro for convenience**. The macro will create a new vector that holds the values you give it. Listing 8-2 creates a new `Vec<i32>` that holds the values `1`, `2`, and `3`. The integer type is `i32` because that’s the default integer type, as we discussed in the [“Data Types”](https://doc.rust-lang.org/book/ch03-02-data-types.html#data-types) section of Chapter 3.

```rust
let v = vec![1, 2, 3];
```

Listing 8-2: Creating a new vector containing values

Because we’ve given initial `i32` values, Rust can infer that the type of `v` is `Vec<i32>`, and the type annotation isn’t necessary. Next, we’ll look at how to modify a vector.

### Updating a Vector

To create a vector and then add elements to it, we can use the `push` method, as shown in Listing 8-3.

```rust
let mut v = Vec::new();

v.push(5);
v.push(6);
v.push(7);
v.push(8);
```

Listing 8-3: Using the `push` method to add values to a vector

As with any variable, if we want to be able to change its value, we need to make it mutable using the `mut` keyword, as discussed in Chapter 3. **The numbers we place inside are all of type `i32`, and Rust infers this from the data, so we don’t need the `Vec<i32>` annotation**.

### Dropping a Vector Drops Its Elements

**Like any other `struct`, a vector is freed when it goes out of scope**, as annotated in Listing 8-4.

```rust
{
  let v = vec![1, 2, 3, 4];

  // do stuff with v
} // <- v goes out of scope and is freed here
```

Listing 8-4: Showing where the vector and its elements are dropped

**When the vector gets dropped, all of its contents are also dropped, meaning those integers it holds will be cleaned up**. This may seem like a straightforward point but can get a bit more complicated when you start to introduce references to the elements of the vector. Let’s tackle that next!

### Reading Elements of Vectors

Now that you know how to create, update, and destroy vectors, knowing how to read their contents is a good next step. There are two ways to reference a value stored in a vector. In the examples, we’ve annotated the types of the values that are returned from these functions for extra clarity.

Listing 8-5 shows both methods of **accessing a value in a vector, either with indexing syntax or the `get` method.**

```rust
let v = vec![1, 2, 3, 4, 5];

let third: &i32 = &v[2];
println!("The third element is {}", third);

match v.get(2) {
  Some(third) => println!("The third element is {}", third),
  None => println!("There is no third element."),
}
```

Listing 8-5: Using indexing syntax or the `get` method to access an item in a vector

Note two details here. First, we use the index value of `2` to get the third element: vectors are indexed by number, starting at zero. **Second, the two ways to get the third element are by using `&` and `[]`, which gives us a reference, or by using the `get` method with the index passed as an argument, which gives us an `Option<&T>`**.

Rust has two ways to reference an element so you can choose how the program behaves when you try to use an index value that the vector doesn’t have an element for. As an example, let’s see what a program will do if it has a vector that holds five elements and then tries to access an element at index 100, as shown in Listing 8-6.

```rust
let v = vec![1, 2, 3, 4, 5];

let does_not_exist = &v[100];
let does_not_exist = v.get(100);
```

Listing 8-6: Attempting to access the element at index 100 in a vector containing five elements

**When we run this code, the first `[]` method will cause the program to panic because it references a nonexistent element**. This method is best used when you want your program to crash if there’s an attempt to access an element past the end of the vector.

**When the `get` method is passed an index that is outside the vector, it returns `None` without panicking**. You would use this method if accessing an element beyond the range of the vector happens occasionally under normal circumstances. <u>Your code will then have logic to handle having either `Some(&element)` or `None`, as discussed in Chapter 6</u>. For example, the index could be coming from a person entering a number. <u>If they accidentally enter a number that’s too large and the program gets a `None` value, you could tell the user how many items are in the current vector and give them another chance to enter a valid value</u>. That would be more user-friendly than crashing the program due to a typo!

When the program has a valid reference, the borrow checker enforces the ownership and borrowing rules (covered in Chapter 4) to ensure this reference and any other references to the contents of the vector remain valid. **<u>Recall the rule that states you can’t have mutable and immutable references in the same scope</u>**. That rule applies in Listing 8-7, where we hold an immutable reference to the first element in a vector and try to add an element to the end, which won’t work if we also try to refer to that element later in the function:

```rust
let mut v = vec![1, 2, 3, 4, 5];

let first = &v[0];

v.push(6);

println!("The first element is: {}", first);
```

Listing 8-7: Attempting to add an element to a vector while holding a reference to an item

Compiling this code will result in this error:

```console
$ cargo run
   Compiling collections v0.1.0 (file:///projects/collections)
error[E0502]: cannot borrow `v` as mutable because it is also borrowed as immutable
 --> src/main.rs:6:5
  |
4 |     let first = &v[0];
  |                  - immutable borrow occurs here
5 | 
6 |     v.push(6);
  |     ^^^^^^^^^ mutable borrow occurs here
7 | 
8 |     println!("The first element is: {}", first);
  |                                          ----- immutable borrow later used here

For more information about this error, try `rustc --explain E0502`.
error: could not compile `collections` due to previous error
```

The code in Listing 8-7 might look like it should work: why should a reference to the first element care about what changes at the end of the vector? **This error is due to the way vectors work: adding a new element onto the end of the vector might require allocating new memory and copying the old elements to the new space, if there isn’t enough room to put all the elements next to each other where the vector currently is. <u>In that case, the reference to the first element would be pointing to deallocated memory. The borrowing rules prevent programs from ending up in that situation</u>**.

> Note: For more on the implementation details of the `Vec<T>` type, see [“The Rustonomicon”](https://doc.rust-lang.org/nomicon/vec/vec.html).

### Iterating over the Values in a Vector

If we want to access each element in a vector in turn, we can iterate through all of the elements rather than use indices to access one at a time. Listing 8-8 shows how to **use a `for` loop to get immutable references to each element in a vector** of `i32` values and print them.

```rust
    let v = vec![100, 32, 57];
    for i in &v {
        println!("{}", i);
    }
```

Listing 8-8: Printing each element in a vector by iterating over the elements using a `for` loop

We can also iterate over mutable references to each element in a mutable vector in order to make changes to all the elements. The `for` loop in Listing 8-9 will add `50` to each element.

```rust
    let mut v = vec![100, 32, 57];
    for i in &mut v {
        *i += 50;
    }
```

Listing 8-9: Iterating over mutable references to elements in a vector

**To change the value that the mutable reference refers to, we have to use the dereference operator (`*`) to get to the value in `i` before we can use the `+=` operator.** We’ll talk more about the dereference operator in the [“Following the Pointer to the Value with the Dereference Operator”](https://doc.rust-lang.org/book/ch15-02-deref.html#following-the-pointer-to-the-value-with-the-dereference-operator) section of Chapter 15.

### Using an Enum to Store Multiple Types

At the beginning of this chapter, we said that vectors can only store values that are the same type. This can be inconvenient; there are definitely use cases for needing to store a list of items of different types. **<u>Fortunately, the variants of an enum are defined under the same enum type, so when we need to store elements of a different type in a vector, we can define and use an enum</u>**!

For example, say we want to get values from a row in a spreadsheet in which some of the columns in the row contain integers, some floating-point numbers, and some strings. We can define an enum whose variants will hold the different value types, and then **all the enum variants will be considered the same type: that of the enum.** Then we can create a vector that holds that enum and so, ultimately, holds different types. We’ve demonstrated this in Listing 8-10.

```rust
    enum SpreadsheetCell {
        Int(i32),
        Float(f64),
        Text(String),
    }

    let row = vec![
        SpreadsheetCell::Int(3),
        SpreadsheetCell::Text(String::from("blue")),
        SpreadsheetCell::Float(10.12),
    ];
```

Listing 8-10: Defining an `enum` to store values of different types in one vector

**Rust needs to know what types will be in the vector at compile time so it knows exactly how much memory on the heap will be needed to store each element**. A secondary advantage is that we can be explicit about what types are allowed in this vector. If Rust allowed a vector to hold any type, there would be a chance that one or more of the types would cause errors with the operations performed on the elements of the vector. Using an enum plus a `match` expression means that Rust will ensure at compile time that every possible case is handled, as discussed in Chapter 6.

When you’re writing a program, if you don’t know the exhaustive set of types the program will get at runtime to store in a vector, the enum technique won’t work. Instead, you can use a trait object, which we’ll cover in Chapter 17.

Now that we’ve discussed some of the most common ways to use vectors, be sure to review [the API documentation](https://doc.rust-lang.org/std/vec/struct.Vec.html) for all the many useful methods defined on `Vec<T>` by the standard library. For example, in addition to `push`, a `pop` method removes and returns the last element. Let’s move on to the next collection type: `String`!

## 8.2 Storing UTF-8 Encoded Text with Strings

We talked about strings in Chapter 4, but we’ll look at them in more depth now. New Rustaceans commonly get stuck on strings for a combination of three reasons: Rust’s propensity for exposing possible errors, strings being a more complicated data structure than many programmers give them credit for, and **UTF-8**. These factors combine in a way that can seem difficult when you’re coming from other programming languages.

**It’s useful to discuss strings in the context of collections because strings are implemented as a collection of bytes, plus some methods to provide useful functionality when those bytes are interpreted as text**. In this section, we’ll talk about the operations on `String` that every collection type has, such as creating, updating, and reading. We’ll also discuss the ways in which `String` is different from the other collections, namely how indexing into a `String` is complicated by the differences between how people and computers interpret `String` data.

### What Is a String?

We’ll first define what we mean by the term *string*. **Rust has only one string type in the core language, which is the string slice `str` that is usually seen in its borrowed form `&str`**. In Chapter 4, we talked about *string slices*, which are references to some UTF-8 encoded string data stored elsewhere. String literals, for example, are stored in the program’s binary and are therefore string slices.

**The `String` type, which is provided by Rust’s standard library rather than coded into the core language, is a growable, mutable, owned, UTF-8 encoded string type**. When Rustaceans refer to “strings” in Rust, they usually mean the `String` and the string slice `&str` types, not just one of those types. Although this section is largely about `String`, both types are used heavily in Rust’s standard library, and both `String` and string slices are UTF-8 encoded.

Rust’s standard library also includes a number of other string types, such as `OsString`, `OsStr`, `CString`, and `CStr`. Library crates can provide even more options for storing string data. See how those names all end in `String` or `Str`? They refer to owned and borrowed variants, just like the `String` and `str` types you’ve seen previously. These string types can store text in different encodings or be represented in memory in a different way, for example. We won’t discuss these other string types in this chapter; see their API documentation for more about how to use them and when each is appropriate.

### Creating a New String

Many of the same operations available with `Vec<T>` are available with `String` as well, starting with the `new` function to create a string, shown in Listing 8-11.

```rust
let mut s = String::new();
```

Listing 8-11: Creating a new, empty `String`

This line creates a new empty string called `s`, which we can then load data into. **Often, we’ll have some initial data that we want to start the string with. For that, we use the `to_string` method, which is available on any type that implements the `Display` trait, as string literals do**. Listing 8-12 shows two examples.

```rust
let data = "initial contents";

let s = data.to_string();

// the method also works on a literal directly:
let s = "initial contents".to_string();
```

Listing 8-12: Using the `to_string` method to create a `String` from a string literal

This code creates a string containing `initial contents`.

We can also use the function `String::from` to create a `String` from a string literal. The code in Listing 8-13 is equivalent to the code from Listing 8-12 that uses `to_string`.

```rust
let s = String::from("initial contents");
```

Listing 8-13: Using the `String::from` function to create a `String` from a string literal

Because strings are used for so many things, we can use many different generic APIs for strings, providing us with a lot of options. Some of them can seem redundant, but they all have their place! <u>In this case, `String::from` and `to_string` do the same thing, so which you choose is a matter of style</u>.

Remember that **strings are UTF-8 encoded**, so we can include any properly encoded data in them, as shown in Listing 8-14.

```rust
let hello = String::from("السلام عليكم");
let hello = String::from("Dobrý den");
let hello = String::from("Hello");
let hello = String::from("שָׁלוֹם");
let hello = String::from("नमस्ते");
let hello = String::from("こんにちは");
let hello = String::from("안녕하세요");
let hello = String::from("你好");
let hello = String::from("Olá");
let hello = String::from("Здравствуйте");
let hello = String::from("Hola");
```

Listing 8-14: Storing greetings in different languages in strings

All of these are valid `String` values.

### Updating a String

A `String` can grow in size and its contents can change, just like the contents of a `Vec<T>`, if you push more data into it. In addition, you can conveniently use the `+` operator or the `format!` macro to concatenate `String` values.

#### Appending to a String with `push_str` and `push`

**We can grow a `String` by using the `push_str` method to append a string slice**, as shown in Listing 8-15.

```rust
let mut s = String::from("foo");
s.push_str("bar");
```

Listing 8-15: Appending a string slice to a `String` using the `push_str` method

After these two lines, `s` will contain `foobar`. <u>The `push_str` method takes a string slice because we don’t necessarily want to take ownership of the parameter</u>. For example, the code in Listing 8-16 shows that it would be unfortunate if we weren’t able to use `s2` after appending its contents to `s1`.

```rust
let mut s1 = String::from("foo");
let s2 = "bar";
s1.push_str(s2);
println!("s2 is {}", s2);
```

Listing 8-16: Using a string slice after appending its contents to a `String`

If the `push_str` method took ownership of `s2`, we wouldn’t be able to print its value on the last line. However, this code works as we’d expect!

**The `push` method takes a single character as a parameter and adds it to the `String`**. Listing 8-17 shows code that adds the letter “l” to a `String` using the `push` method.

```rust
let mut s = String::from("lo");
s.push('l');
```

Listing 8-17: Adding one character to a `String` value using `push`

As a result of this code, `s` will contain `lol`.

#### Concatenation with the `+` Operator or the `format!` Macro

Often, you’ll want to combine two existing strings. One way is to use the `+` operator, as shown in Listing 8-18.

```rust
let s1 = String::from("Hello, ");
let s2 = String::from("world!");
let s3 = s1 + &s2; // note s1 has been moved here and can no longer be used
```

Listing 8-18: Using the `+` operator to combine two `String` values into a new `String` value

The string `s3` will contain `Hello, world!` as a result of this code. The reason `s1` is no longer valid after the addition and the reason we used a reference to `s2` has to do with the signature of the method that gets called when we use the `+` operator. **The `+` operator uses the `add` method**, whose signature looks something like this:

```rust
fn add(self, s: &str) -> String {
```

This isn’t the exact signature that’s in the standard library: in the standard library, `add` is defined using generics. Here, we’re looking at the signature of `add` with concrete types substituted for the generic ones, which is what happens when we call this method with `String` values. We’ll discuss generics in Chapter 10. This signature gives us the clues we need to understand the tricky bits of the `+` operator.

First, `s2` has an `&`, meaning that we’re adding a *reference* of the second string to the first string because of the `s` parameter in the `add` function: we can only add a `&str` to a `String`; we can’t add two `String` values together. But wait—the type of `&s2` is `&String`, not `&str`, as specified in the second parameter to `add`. So why does Listing 8-18 compile?

**The reason we’re able to use `&s2` in the call to `add` is that the compiler can *coerce* the `&String` argument into a `&str`**. When we call the `add` method, <u>Rust uses a *deref coercion*, which here turns `&s2` into `&s2[..]`</u>. We’ll discuss deref coercion in more depth in Chapter 15. Because `add` does not take ownership of the `s` parameter, `s2` will still be a valid `String` after this operation.

Second, we can see in the signature that `add` takes ownership of `self`, <u>because `self` does *not* have an `&`. This means `s1` in Listing 8-18 will be moved into the `add` call and no longer be valid after that</u>. So although `let s3 = s1 + &s2;` looks like it will copy both strings and create a new one, this statement actually takes ownership of `s1`, appends a copy of the contents of `s2`, and then returns ownership of the result. **In other words, it looks like it’s making a lot of copies but isn’t; the implementation is more efficient than copying**.

If we need to concatenate multiple strings, the behavior of the `+` operator gets unwieldy:

```rust
    let s1 = String::from("tic");
    let s2 = String::from("tac");
    let s3 = String::from("toe");

    let s = s1 + "-" + &s2 + "-" + &s3;
```

At this point, `s` will be `tic-tac-toe`. With all of the `+` and `"` characters, it’s difficult to see what’s going on. For more complicated string combining, we can **use the `format!` macro**:

```rust
    let s1 = String::from("tic");
    let s2 = String::from("tac");
    let s3 = String::from("toe");

    let s = format!("{}-{}-{}", s1, s2, s3);
```

This code also sets `s` to `tic-tac-toe`. The `format!` macro works in the same way as `println!`, but instead of printing the output to the screen, it returns a `String` with the contents. The version of the code using `format!` is much easier to read, and **the code generated by the `format!` macro uses references so that this call doesn’t take ownership of any of its parameters**.

### Indexing into Strings

In many other programming languages, accessing individual characters in a string by referencing them by index is a valid and common operation. However, if you try to access parts of a `String` using indexing syntax in Rust, you’ll get an error. Consider the invalid code in Listing 8-19.

```rust
let s1 = String::from("hello");
let h = s1[0];
```

Listing 8-19: Attempting to use indexing syntax with a String

This code will result in the following error:

```console
$ cargo run
   Compiling collections v0.1.0 (file:///projects/collections)
error[E0277]: the type `String` cannot be indexed by `{integer}`
 --> src/main.rs:3:13
  |
3 |     let h = s1[0];
  |             ^^^^^ `String` cannot be indexed by `{integer}`
  |
  = help: the trait `Index<{integer}>` is not implemented for `String`

For more information about this error, try `rustc --explain E0277`.
error: could not compile `collections` due to previous error
```

The error and the note tell the story: Rust strings don’t support indexing. But why not? To answer that question, we need to discuss how Rust stores strings in memory.

#### Internal Representation

**A `String` is a wrapper over a `Vec<u8>`**. Let’s look at some of our properly encoded UTF-8 example strings from Listing 8-14. First, this one:

```rust
let hello = String::from("Hola");
```

**In this case, `len` will be 4, which means the vector storing the string “Hola” is 4 bytes long**. Each of these letters takes 1 byte when encoded in **UTF-8**. But what about the following line? (Note that this string begins with the capital Cyrillic letter Ze, not the Arabic number 3.)

```rust
let hello = String::from("Здравствуйте");
```

Asked how long the string is, you might say 12. However, Rust’s answer is 24: that’s the number of bytes it takes to encode “Здравствуйте” in **UTF-8**, because each Unicode scalar value in that string takes 2 bytes of storage. Therefore, an index into the string’s bytes will not always correlate to a valid Unicode scalar value. To demonstrate, consider this invalid Rust code:

```rust
let hello = "Здравствуйте";
let answer = &hello[0];
```

What should the value of `answer` be? Should it be `З`, the first letter? When encoded in UTF-8, the first byte of `З` is `208` and the second is `151`, so `answer` should in fact be `208`, but `208` is not a valid character on its own. Returning `208` is likely not what a user would want if they asked for the first letter of this string; however, that’s the only data that Rust has at byte index 0. Users generally don’t want the byte value returned, even if the string contains only Latin letters: if `&"hello"[0]` were valid code that returned the byte value, it would return `104`, not `h`. To avoid returning an unexpected value and causing bugs that might not be discovered immediately, **Rust doesn’t compile this code at all and prevents misunderstandings early in the development process**.

#### Bytes and Scalar Values and Grapheme Clusters! Oh My!

**Another point about UTF-8 is that there are actually three relevant ways to look at strings from Rust’s perspective: as bytes, scalar values, and grapheme clusters (the closest thing to what we would call *letters*)**.

If we look at the Hindi word “नमस्ते” written in the Devanagari script, it is stored as a vector of `u8` values that looks like this:

```text
[224, 164, 168, 224, 164, 174, 224, 164, 184, 224, 165, 141, 224, 164, 164,
224, 165, 135]
```

That’s 18 bytes and is how computers ultimately store this data. If we look at them as Unicode scalar values, which are what <u>Rust’s `char` type</u> is, those bytes look like this:

```text
['न', 'म', 'स', '्', 'त', 'े']
```

There are six `char` values here, but the fourth and sixth are not letters: they’re diacritics that don’t make sense on their own. Finally, if we look at them as <u>grapheme clusters</u>, we’d get what a person would call the four letters that make up the Hindi word:

```text
["न", "म", "स्", "ते"]
```

Rust provides different ways of interpreting the raw string data that computers store so that each program can choose the interpretation it needs, no matter what human language the data is in.

**A final reason Rust doesn’t allow us to index into a `String` to get a character is that indexing operations are expected to always take constant time (O(1)). But it isn’t possible to guarantee that performance with a `String`, because Rust would have to walk through the contents from the beginning to the index to determine how many valid characters there were.**

### Slicing Strings

**Indexing into a string is often a bad idea because it’s not clear what the return type of the string-indexing operation should be: a byte value, a character, a grapheme cluster, or a string slice**. Therefore, Rust asks you to be more specific if you really need to use indices to create string slices. To be more specific in your indexing and indicate that you want a string slice, rather than indexing using `[]` with a single number, **you can use `[]` with a range to create a string slice containing particular bytes**:

```rust
let hello = "Здравствуйте";

let s = &hello[0..4];
```

Here, `s` will be a `&str` that contains the first 4 bytes of the string. <u>Earlier, we mentioned that each of these characters was 2 bytes, which means `s` will be `Зд`.</u>

**What would happen if we used `&hello[0..1]`? The answer: Rust would panic at runtime in the same way as if an invalid index were accessed in a vector**:

```console
$ cargo run
   Compiling collections v0.1.0 (file:///projects/collections)
    Finished dev [unoptimized + debuginfo] target(s) in 0.43s
     Running `target/debug/collections`
thread 'main' panicked at 'byte index 1 is not a char boundary; it is inside 'З' (bytes 0..2) of `Здравствуйте`', src/main.rs:4:14
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

You should use ranges to create string slices with caution, because doing so can crash your program.

### Methods for Iterating Over Strings

Fortunately, you can access elements in a string in other ways.

**If you need to perform operations on individual Unicode scalar values, the best way to do so is to use the `chars` method**. Calling `chars` on “नमस्ते” separates out and returns six values of type `char`, and you can iterate over the result to access each element:

```rust
for c in "नमस्ते".chars() {
    println!("{}", c);
}
```

This code will print the following:

```text
न
म
स
्
त
े
```

The `bytes` method returns each raw byte, which might be appropriate for your domain:

```rust
for b in "नमस्ते".bytes() {
    println!("{}", b);
}
```

This code will print the 18 bytes that make up this `String`:

```text
224
164
// --snip--
165
135
```

But be sure to remember that valid Unicode scalar values may be made up of more than 1 byte.

**Getting grapheme clusters from strings is complex, so this functionality is not provided by the standard library. Crates are available on [crates.io](https://crates.io/) if this is the functionality you need**.

### Strings Are Not So Simple

To summarize, strings are complicated. Different programming languages make different choices about how to present this complexity to the programmer. **Rust has chosen to make the correct handling of `String` data the default behavior for all Rust programs, which means programmers have to put more thought into handling UTF-8 data upfront**. This trade-off exposes more of the complexity of strings than is apparent in other programming languages, but it prevents you from having to handle errors involving non-ASCII characters later in your development life cycle.

Let’s switch to something a bit less complex: hash maps!

## 8.3 Storing Keys with Associated Values in Hash Maps

The last of our common collections is the *hash map*. The type `HashMap<K, V>` stores a mapping of keys of type `K` to values of type `V`. It does this via a *hashing function*, which determines how it places these keys and values into memory. Many programming languages support this kind of data structure, but they often use a different name, such as hash, map, object, hash table, dictionary, or associative array, just to name a few.

> We’ll go over the basic API of hash maps in this section, but many more goodies are hiding in the functions defined on `HashMap<K, V>` by the standard library. As always, check the standard library documentation for more information.

### Creating a New Hash Map

You can create an empty hash map with `new` and add elements with `insert`. In Listing 8-20, we’re keeping track of the scores of two teams whose names are Blue and Yellow. The Blue team starts with 10 points, and the Yellow team starts with 50.

```rust
use std::collections::HashMap;

let mut scores = HashMap::new();

scores.insert(String::from("Blue"), 10);
scores.insert(String::from("Yellow"), 50);
```

Listing 8-20: Creating a new hash map and inserting some keys and values

**Note that we need to first `use` the `HashMap` from the collections portion of the standard library.** Of our three common collections, this one is the least often used, so it’s not included in the features brought into scope automatically in the prelude. **Hash maps also have less support from the standard library; there’s no built-in macro to construct them, for example**.

**Just like vectors, hash maps store their data on the heap**. This `HashMap` has keys of type `String` and values of type `i32`. **Like vectors, hash maps are homogeneous: all of the keys must have the same type, and all of the values must have the same type**.

Another way of constructing a hash map is by using iterators and the `collect` method on a vector of tuples, where each tuple consists of a key and its value. We’ll be going into more detail about iterators and their associated methods in the [”Processing a Series of Items with Iterators” section of Chapter 13](https://doc.rust-lang.org/book/ch13-02-iterators.html). The `collect` method gathers data into a number of collection types, including `HashMap`. For example, if we had the team names and initial scores in two separate vectors, we could use the `zip` method to create an iterator of tuples where “Blue” is paired with 10, and so forth. Then we could use the `collect` method to turn that iterator of tuples into a hash map, as shown in Listing 8-21.

```rust
    use std::collections::HashMap;

    let teams = vec![String::from("Blue"), String::from("Yellow")];
    let initial_scores = vec![10, 50];

    let mut scores: HashMap<_, _> =
        teams.into_iter().zip(initial_scores.into_iter()).collect();
```

Listing 8-21: Creating a hash map from a list of teams and a list of scores

**The type annotation `HashMap<_, _>` is needed here because it’s possible to `collect` into many different data structures and Rust doesn’t know which you want unless you specify**. For the parameters for the key and value types, however, we use underscores, and Rust can infer the types that the hash map contains based on the types of the data in the vectors. In Listing 8-21, the key type will be `String` and the value type will be `i32`, just as the types were in Listing 8-20.

### Hash Maps and Ownership

**For types that implement the `Copy` trait, like `i32`, the values are copied into the hash map. For owned values like `String`, the values will be moved and the hash map will be the owner of those values**, as demonstrated in Listing 8-22.

```rust
use std::collections::HashMap;

let field_name = String::from("Favorite color");
let field_value = String::from("Blue");

let mut map = HashMap::new();
map.insert(field_name, field_value);
// field_name and field_value are invalid at this point, try using them and
// see what compiler error you get!
```

Listing 8-22: Showing that keys and values are owned by the hash map once they’re inserted

We aren’t able to use the variables `field_name` and `field_value` after they’ve been moved into the hash map with the call to `insert`.

<u>If we insert references to values into the hash map, the values won’t be moved into the hash map</u>. **The values that the references point to must be valid for at least as long as the hash map is valid.** We’ll talk more about these issues in the [“Validating References with Lifetimes”](https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html#validating-references-with-lifetimes) section in Chapter 10.

### Accessing Values in a Hash Map

We can get a value out of the hash map by providing its key to the `get` method, as shown in Listing 8-23.

```rust
use std::collections::HashMap;

let mut scores = HashMap::new();

scores.insert(String::from("Blue"), 10);
scores.insert(String::from("Yellow"), 50);

let team_name = String::from("Blue");
let score = scores.get(&team_name);
```

Listing 8-23: Accessing the score for the Blue team stored in the hash map

Here, `score` will have the value that’s associated with the Blue team, and the result will be `Some(&10)`. **The result is wrapped in `Some` because `get` returns an `Option<&V>`; if there’s no value for that key in the hash map, `get` will return `None`**. The program will need to handle the `Option` in one of the ways that we covered in Chapter 6.

We can iterate over each key/value pair in a hash map in a similar manner as we do with vectors, using a `for` loop:

```rust
use std::collections::HashMap;

let mut scores = HashMap::new();

scores.insert(String::from("Blue"), 10);
scores.insert(String::from("Yellow"), 50);

for (key, value) in &scores {
  println!("{}: {}", key, value);
}
```

This code will print each pair in an arbitrary order:

```text
Yellow: 50
Blue: 10
```

### Updating a Hash Map

Although the number of keys and values is growable, each key can only have one value associated with it at a time. When you want to change the data in a hash map, you have to decide how to handle the case when a key already has a value assigned. You could replace the old value with the new value, completely disregarding the old value. You could keep the old value and ignore the new value, only adding the new value if the key *doesn’t* already have a value. Or you could combine the old value and the new value. Let’s look at how to do each of these!

#### Overwriting a Value

If we insert a key and a value into a hash map and then insert that same key with a different value, the value associated with that key will be replaced. Even though the code in Listing 8-24 calls `insert` twice, the hash map will only contain one key/value pair because we’re inserting the value for the Blue team’s key both times.

```rust
use std::collections::HashMap;

let mut scores = HashMap::new();

scores.insert(String::from("Blue"), 10);
scores.insert(String::from("Blue"), 25);

println!("{:?}", scores);
```

Listing 8-24: Replacing a value stored with a particular key

This code will print `{"Blue": 25}`. The original value of `10` has been overwritten.

#### Only Inserting a Value If the Key Has No Value

It’s common to check whether a particular key has a value and, if it doesn’t, insert a value for it. Hash maps have a special API for this called `entry` that takes the key you want to check as a parameter. The return value of the `entry` method is an enum called `Entry` that represents a value that might or might not exist. Let’s say we want to check whether the key for the Yellow team has a value associated with it. If it doesn’t, we want to insert the value 50, and the same for the Blue team. Using the `entry` API, the code looks like Listing 8-25.

```rust
use std::collections::HashMap;

let mut scores = HashMap::new();
scores.insert(String::from("Blue"), 10);

scores.entry(String::from("Yellow")).or_insert(50);
scores.entry(String::from("Blue")).or_insert(50);

println!("{:?}", scores);
```

Listing 8-25: Using the `entry` method to only insert if the key does not already have a value

**The `or_insert` method on `Entry` is defined to return a mutable reference to the value for the corresponding `Entry` key if that key exists, and if not, inserts the parameter as the new value for this key and returns a mutable reference to the new value**. This technique is much cleaner than writing the logic ourselves and, in addition, plays more nicely with the borrow checker.

Running the code in Listing 8-25 will print `{"Yellow": 50, "Blue": 10}`. The first call to `entry` will insert the key for the Yellow team with the value 50 because the Yellow team doesn’t have a value already. The second call to `entry` will not change the hash map because the Blue team already has the value 10.

#### Updating a Value Based on the Old Value

Another common use case for hash maps is to look up a key’s value and then update it based on the old value. For instance, Listing 8-26 shows code that counts how many times each word appears in some text. We use a hash map with the words as keys and increment the value to keep track of how many times we’ve seen that word. If it’s the first time we’ve seen a word, we’ll first insert the value 0.

```rust
use std::collections::HashMap;

let text = "hello world wonderful world";

let mut map = HashMap::new();

for word in text.split_whitespace() {
  let count = map.entry(word).or_insert(0);
  *count += 1;
}

println!("{:?}", map);
```

Listing 8-26: Counting occurrences of words using a hash map that stores words and counts

This code will print `{"world": 2, "hello": 1, "wonderful": 1}`. The `split_whitespace` method iterates over sub-slices, separated by whitespace, of the value in `text`. **The `or_insert` method returns a mutable reference (`&mut V`) to the value for the specified key**. Here we store that mutable reference in the `count` variable, so in order to assign to that value, we must first dereference `count` using the asterisk (`*`). The mutable reference goes out of scope at the end of the `for` loop, so all of these changes are safe and allowed by the borrowing rules.

### Hashing Functions

**By default, `HashMap` uses a hashing function called SipHash that can provide resistance to Denial of Service (DoS) attacks involving hash tables** [1](https://doc.rust-lang.org/book/ch08-03-hash-maps.html#siphash). This is not the fastest hashing algorithm available, but the trade-off for better security that comes with the drop in performance is worth it. If you profile your code and find that the default hash function is too slow for your purposes, you can switch to another function by specifying a different *hasher*. A hasher is a type that implements the `BuildHasher` trait. We’ll talk about traits and how to implement them in Chapter 10. You don’t necessarily have to implement your own hasher from scratch; **[crates.io](https://crates.io/) has libraries shared by other Rust users that provide hashers implementing many common hashing algorithms.**

> [SipHash - wiki](https://en.wikipedia.org/wiki/SipHash)
>
> [漫谈非加密哈希算法 - SegmentFault 思否](https://segmentfault.com/a/1190000010990136)
>
> [什么是哈希洪水攻击（Hash-Flooding Attack）？ - 知乎 (zhihu.com)](https://www.zhihu.com/question/286529973/answer/676981827)

# 9. Error Handling

**Rust groups errors into two major categories: *recoverable* and *unrecoverable* errors**. For a recoverable error, such as a file not found error, it’s reasonable to report the problem to the user and retry the operation. Unrecoverable errors are always symptoms of bugs, like trying to access a location beyond the end of an array.

Most languages don’t distinguish between these two kinds of errors and handle both in the same way, using mechanisms such as exceptions. **Rust doesn’t have exceptions. Instead, it has the type `Result<T, E>` for recoverable errors and the `panic!` macro that stops execution when the program encounters an unrecoverable error**. This chapter covers calling `panic!` first and then talks about returning `Result<T, E>` values. Additionally, we’ll explore considerations when deciding whether to try to recover from an error or to stop execution.

## 9.1 Unrecoverable Errors with `panic!`

Sometimes, bad things happen in your code, and there’s nothing you can do about it. In these cases, Rust has the `panic!` macro. **When the `panic!` macro executes, your program will print a failure message, unwind and clean up the stack, and then quit**. This most commonly occurs when a bug of some kind has been detected and it’s not clear to the programmer how to handle the error.

### Unwinding the Stack or Aborting in Response to a Panic

 默认情况下，发生panic时，Rust进行unwinding步骤，备份stack信息，清除发生panic的function所用的数据，这个备份和清除过程需要进行很多逻辑；另一个可选的方式是值行abort操作，直接终结程序，程序占用的内存将交由操作系统去清除。如果希望Rust生成的可执行文件尽量小，可以选择采用abort策略，这需要在`Cargo.toml`中声明`panic = 'abort'`。

**By default, when a panic occurs, the program starts *unwinding*, which means Rust walks back up the stack and cleans up the data from each function it encounters. But this walking back and cleanup is a lot of work. The alternative is to immediately *abort*, which ends the program without cleaning up. Memory that the program was using will then need to be cleaned up by the operating system**. If in your project you need to make the resulting binary as small as possible, you can switch from unwinding to aborting upon a panic by adding `panic = 'abort'` to the appropriate `[profile]` sections in your *Cargo.toml* file. For example, if you want to abort on panic in release mode, add this:

```toml
[profile.release]
panic = 'abort'
```

Let’s try calling `panic!` in a simple program:

Filename: src/main.rs

```rust
fn main() {
    panic!("crash and burn");
}
```

When you run the program, you’ll see something like this:

```shell
$ cargo run
   Compiling panic v0.1.0 (file:///projects/panic)
    Finished dev [unoptimized + debuginfo] target(s) in 0.25s
     Running `target/debug/panic`
thread 'main' panicked at 'crash and burn', src/main.rs:2:5
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

The call to `panic!` causes the error message contained in the last two lines. The first line shows our panic message and the place in our source code where the panic occurred: *src/main.rs:2:5* indicates that it’s the second line, fifth character of our *src/main.rs* file.

In this case, the line indicated is part of our code, and if we go to that line, we see the `panic!` macro call. In other cases, the `panic!` call might be in code that our code calls, and the filename and line number reported by the error message will be someone else’s code where the `panic!` macro is called, not the line of our code that eventually led to the `panic!` call. We can use the backtrace of the functions the `panic!` call came from to figure out the part of our code that is causing the problem. We’ll discuss what a backtrace is in more detail next.

### Using a `panic!` Backtrace

Let’s look at another example to see what it’s like when a `panic!` call comes from a library because of a bug in our code instead of from our code calling the macro directly. Listing 9-1 has some code that attempts to access an element by index in a vector.

Filename: src/main.rs

```rust
fn main() {
    let v = vec![1, 2, 3];

    v[99];
}
```

Listing 9-1: Attempting to access an element beyond the end of a vector, which will cause a call to `panic!`

Here, we’re attempting to access the 100th element of our vector (which is at index 99 because indexing starts at zero), but it has only 3 elements. In this situation, Rust will panic. **Using `[]` is supposed to return an element, but if you pass an invalid index, there’s no element that Rust could return here that would be correct**.

In C, attempting to read beyond the end of a data structure is undefined behavior. You might get whatever is at the location in memory that would correspond to that element in the data structure, even though the memory doesn’t belong to that structure. <u>This is called a *buffer overread* and can lead to security vulnerabilities if an attacker is able to manipulate the index in such a way as to read data they shouldn’t be allowed to that is stored after the data structure</u>.

**To protect your program from this sort of vulnerability, if you try to read an element at an index that doesn’t exist, Rust will stop execution and refuse to continue**. Let’s try it and see:

```shell
$ cargo run
   Compiling panic v0.1.0 (file:///projects/panic)
    Finished dev [unoptimized + debuginfo] target(s) in 0.27s
     Running `target/debug/panic`
thread 'main' panicked at 'index out of bounds: the len is 3 but the index is 99', src/main.rs:4:5
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

This error points at line 4 of our `main.rs` where we attempt to access index 99. **The next note line tells us that we can set the `RUST_BACKTRACE` environment variable to get a backtrace of exactly what happened to cause the error. A *backtrace* is a list of all the functions that have been called to get to this point. Backtraces in Rust work as they do in other languages: the key to reading the backtrace is to start from the top and read until you see files you wrote**. That’s the spot where the problem originated. The lines above the lines mentioning your files are code that your code called; the lines below are code that called your code. These lines might include core Rust code, standard library code, or crates that you’re using. Let’s try getting a backtrace by setting the `RUST_BACKTRACE` environment variable to any value except 0. Listing 9-2 shows output similar to what you’ll see.

```shell
$ RUST_BACKTRACE=1 cargo run
thread 'main' panicked at 'index out of bounds: the len is 3 but the index is 99', src/main.rs:4:5
stack backtrace:
   0: rust_begin_unwind
             at /rustc/7eac88abb2e57e752f3302f02be5f3ce3d7adfb4/library/std/src/panicking.rs:483
   1: core::panicking::panic_fmt
             at /rustc/7eac88abb2e57e752f3302f02be5f3ce3d7adfb4/library/core/src/panicking.rs:85
   2: core::panicking::panic_bounds_check
             at /rustc/7eac88abb2e57e752f3302f02be5f3ce3d7adfb4/library/core/src/panicking.rs:62
   3: <usize as core::slice::index::SliceIndex<[T]>>::index
             at /rustc/7eac88abb2e57e752f3302f02be5f3ce3d7adfb4/library/core/src/slice/index.rs:255
   4: core::slice::index::<impl core::ops::index::Index<I> for [T]>::index
             at /rustc/7eac88abb2e57e752f3302f02be5f3ce3d7adfb4/library/core/src/slice/index.rs:15
   5: <alloc::vec::Vec<T> as core::ops::index::Index<I>>::index
             at /rustc/7eac88abb2e57e752f3302f02be5f3ce3d7adfb4/library/alloc/src/vec.rs:1982
   6: panic::main
             at ./src/main.rs:4
   7: core::ops::function::FnOnce::call_once
             at /rustc/7eac88abb2e57e752f3302f02be5f3ce3d7adfb4/library/core/src/ops/function.rs:227
note: Some details are omitted, run with `RUST_BACKTRACE=full` for a verbose backtrace.
```

Listing 9-2: The backtrace generated by a call to `panic!` displayed when the environment variable `RUST_BACKTRACE` is set

That’s a lot of output! The exact output you see might be different depending on your operating system and Rust version. **In order to get backtraces with this information, debug symbols must be enabled. Debug symbols are enabled by default when using `cargo build` or `cargo run` without the `--release` flag, as we have here.**

## 9.2 Recoverable Errors with `Result`

Most errors aren’t serious enough to require the program to stop entirely. Sometimes, when a function fails, it’s for a reason that you can easily interpret and respond to. For example, if you try to open a file and that operation fails because the file doesn’t exist, you might want to create the file instead of terminating the process.

Recall from [“Handling Potential Failure with the `Result` Type”](https://doc.rust-lang.org/book/ch02-00-guessing-game-tutorial.html#handling-potential-failure-with-the-result-type) in Chapter 2 that the `Result` enum is defined as having two variants, `Ok` and `Err`, as follows:

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

The `T` and `E` are generic type parameters: we’ll discuss generics in more detail in Chapter 10. What you need to know right now is that **`T` represents the type of the value that will be returned in a success case within the `Ok` variant, and `E` represents the type of the error that will be returned in a failure case within the `Err` variant**. Because `Result` has these generic type parameters, we can use the `Result` type and the functions that the standard library has defined on it in many different situations where the successful value and error value we want to return may differ.

Let’s call a function that returns a `Result` value because the function could fail. In Listing 9-3 we try to open a file.

Filename: src/main.rs

```rust
use std::fs::File;

fn main() {
    let f = File::open("hello.txt");
}
```

Listing 9-3: Opening a file

How do we know `File::open` returns a `Result`? We could look at the [standard library API documentation](https://doc.rust-lang.org/std/index.html), or we could ask the compiler! If we give `f` a type annotation that we know is *not* the return type of the function and then try to compile the code, the compiler will tell us that the types don’t match. The error message will then tell us what the type of `f` *is*. Let’s try it! We know that the return type of `File::open` isn’t of type `u32`, so let’s change the `let f` statement to this:

```rust
let f: u32 = File::open("hello.txt");
```

Attempting to compile now gives us the following output:

```console
$ cargo run
   Compiling error-handling v0.1.0 (file:///projects/error-handling)
error[E0308]: mismatched types
 --> src/main.rs:4:18
  |
4 |     let f: u32 = File::open("hello.txt");
  |            ---   ^^^^^^^^^^^^^^^^^^^^^^^ expected `u32`, found enum `Result`
  |            |
  |            expected due to this
  |
  = note: expected type `u32`
             found enum `Result<File, std::io::Error>`

For more information about this error, try `rustc --explain E0308`.
error: could not compile `error-handling` due to previous error
```

This tells us the return type of the `File::open` function is a `Result<T, E>`. The generic parameter `T` has been filled in here with the type of the success value, `std::fs::File`, which is a file handle. The type of `E` used in the error value is `std::io::Error`.

This return type means the call to `File::open` might succeed and return a file handle that we can read from or write to. The function call also might fail: for example, the file might not exist, or we might not have permission to access the file. The `File::open` function needs to have a way to tell us whether it succeeded or failed and at the same time give us either the file handle or error information. This information is exactly what the `Result` enum conveys.

<u>In the case where `File::open` succeeds, the value in the variable `f` will be an instance of `Ok` that contains a file handle. In the case where it fails, the value in `f` will be an instance of `Err` that contains more information about the kind of error that happened.</u>

We need to add to the code in Listing 9-3 to take different actions depending on the value `File::open` returns. Listing 9-4 shows one way to handle the `Result` using a basic tool, the `match` expression that we discussed in Chapter 6.

Filename: src/main.rs

```rust
use std::fs::File;

fn main() {
    let f = File::open("hello.txt");

    let f = match f {
        Ok(file) => file,
        Err(error) => panic!("Problem opening the file: {:?}", error),
    };
}
```

Listing 9-4: Using a `match` expression to handle the `Result` variants that might be returned

**Note that, like the `Option` enum, the `Result` enum and its variants have been brought into scope by the prelude, so we don’t need to specify `Result::` before the `Ok` and `Err` variants in the `match` arms**.

Here we tell Rust that when the result is `Ok`, return the inner `file` value out of the `Ok` variant, and we then assign that file handle value to the variable `f`. After the `match`, we can use the file handle for reading or writing.

The other arm of the `match` handles the case where we get an `Err` value from `File::open`. In this example, we’ve chosen to call the `panic!` macro. If there’s no file named *hello.txt* in our current directory and we run this code, we’ll see the following output from the `panic!` macro:

```console
$ cargo run
   Compiling error-handling v0.1.0 (file:///projects/error-handling)
    Finished dev [unoptimized + debuginfo] target(s) in 0.73s
     Running `target/debug/error-handling`
thread 'main' panicked at 'Problem opening the file: Os { code: 2, kind: NotFound, message: "No such file or directory" }', src/main.rs:8:23
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

As usual, this output tells us exactly what has gone wrong.

### Matching on Different Errors

The code in Listing 9-4 will `panic!` no matter why `File::open` failed. What we want to do instead is take different actions for different failure reasons: if `File::open` failed because the file doesn’t exist, we want to create the file and return the handle to the new file. If `File::open` failed for any other reason—for example, because we didn’t have permission to open the file—we still want the code to `panic!` in the same way as it did in Listing 9-4. Look at Listing 9-5, which adds an inner `match` expression.

Filename: src/main.rs

```rust
use std::fs::File;
use std::io::ErrorKind;

fn main() {
    let f = File::open("hello.txt");

    let f = match f {
        Ok(file) => file,
        Err(error) => match error.kind() {
            ErrorKind::NotFound => match File::create("hello.txt") {
                Ok(fc) => fc,
                Err(e) => panic!("Problem creating the file: {:?}", e),
            },
            other_error => {
                panic!("Problem opening the file: {:?}", other_error)
            }
        },
    };
}
```

Listing 9-5: Handling different kinds of errors in different ways

The type of the value that `File::open` returns inside the `Err` variant is `io::Error`, which is a struct provided by the standard library. This struct has a method `kind` that we can call to get an `io::ErrorKind` value. The enum `io::ErrorKind` is provided by the standard library and has variants representing the different kinds of errors that might result from an `io` operation. The variant we want to use is `ErrorKind::NotFound`, which indicates the file we’re trying to open doesn’t exist yet. So we match on `f`, but we also have an inner match on `error.kind()`.

The condition we want to check in the inner match is whether the value returned by `error.kind()` is the `NotFound` variant of the `ErrorKind` enum. If it is, we try to create the file with `File::create`. However, because `File::create` could also fail, we need a second arm in the inner `match` expression. When the file can’t be created, a different error message is printed. The second arm of the outer `match` stays the same, so the program panics on any error besides the missing file error.

That’s a lot of `match`! The `match` expression is very useful but also very much a primitive. In Chapter 13, you’ll learn about closures; the `Result<T, E>` type has many methods that accept a closure and are implemented using `match` expressions. Using those methods will make your code more concise. A more seasoned Rustacean might write this code instead of Listing 9-5:

```rust
use std::fs::File;
use std::io::ErrorKind;

fn main() {
    let f = File::open("hello.txt").unwrap_or_else(|error| {
        if error.kind() == ErrorKind::NotFound {
            File::create("hello.txt").unwrap_or_else(|error| {
                panic!("Problem creating the file: {:?}", error);
            })
        } else {
            panic!("Problem opening the file: {:?}", error);
        }
    });
}
```

Although this code has the same behavior as Listing 9-5, it doesn’t contain any `match` expressions and is cleaner to read. Come back to this example after you’ve read Chapter 13, and look up **the `unwrap_or_else` method in the standard library documentation. Many more of these methods can clean up huge nested `match` expressions when you’re dealing with errors**.

### Shortcuts for Panic on Error: `unwrap` and `expect`

Using `match` works well enough, but it can be a bit verbose and doesn’t always communicate intent well. The `Result<T, E>` type has many helper methods defined on it to do various tasks. One of those methods, called `unwrap`, is a shortcut method that is implemented just like the `match` expression we wrote in Listing 9-4. **If the `Result` value is the `Ok` variant, `unwrap` will return the value inside the `Ok`. If the `Result` is the `Err` variant, `unwrap` will call the `panic!` macro for us**. Here is an example of `unwrap` in action:

Filename: src/main.rs

```rust
use std::fs::File;

fn main() {
    let f = File::open("hello.txt").unwrap();
}
```

If we run this code without a *hello.txt* file, we’ll see an error message from the `panic!` call that the `unwrap` method makes:

```text
thread 'main' panicked at 'called `Result::unwrap()` on an `Err` value: Error {
repr: Os { code: 2, message: "No such file or directory" } }',
src/libcore/result.rs:906:4
```

<u>Another method, `expect`, which is similar to `unwrap`, lets us also choose the `panic!` error message. Using `expect` instead of `unwrap` and providing good error messages can convey your intent and make tracking down the source of a panic easier</u>. The syntax of `expect` looks like this:

Filename: src/main.rs

```rust
use std::fs::File;

fn main() {
    let f = File::open("hello.txt").expect("Failed to open hello.txt");
}
```

We use `expect` in the same way as `unwrap`: to return the file handle or call the `panic!` macro. The error message used by `expect` in its call to `panic!` will be the parameter that we pass to `expect`, rather than the default `panic!` message that `unwrap` uses. Here’s what it looks like:

```text
thread 'main' panicked at 'Failed to open hello.txt: Error { repr: Os { code:
2, message: "No such file or directory" } }', src/libcore/result.rs:906:4
```

Because this error message starts with the text we specified, `Failed to open hello.txt`, it will be easier to find where in the code this error message is coming from. If we use `unwrap` in multiple places, it can take more time to figure out exactly which `unwrap` is causing the panic because all `unwrap` calls that panic print the same message.

### Propagating Errors

When you’re writing a function whose implementation calls something that might fail, instead of handling the error within this function, you can return the error to the calling code so that it can decide what to do. This is known as *propagating* the error and gives more control to the calling code, where there might be more information or logic that dictates how the error should be handled than what you have available in the context of your code.

For example, Listing 9-6 shows a function that reads a username from a file. If the file doesn’t exist or can’t be read, this function will return those errors to the code that called this function.

Filename: src/main.rs

```rust
use std::fs::File;
use std::io::{self, Read};

fn read_username_from_file() -> Result<String, io::Error> {
    let f = File::open("hello.txt");

    let mut f = match f {
        Ok(file) => file,
        Err(e) => return Err(e),
    };

    let mut s = String::new();

    match f.read_to_string(&mut s) {
        Ok(_) => Ok(s),
        Err(e) => Err(e),
    }
}
```

Listing 9-6: A function that returns errors to the calling code using `match`

This function can be written in a much shorter way, but we’re going to start by doing a lot of it manually in order to explore error handling; at the end, we’ll show the shorter way. Let’s look at the return type of the function first: `Result<String, io::Error>`. This means the function is returning a value of the type `Result<T, E>` where the generic parameter `T` has been filled in with the concrete type `String` and the generic type `E` has been filled in with the concrete type `io::Error`. If this function succeeds without any problems, the code that calls this function will receive an `Ok` value that holds a `String`—the username that this function read from the file. If this function encounters any problems, the code that calls this function will receive an `Err` value that holds an instance of `io::Error` that contains more information about what the problems were. We chose `io::Error` as the return type of this function because that happens to be the type of the error value returned from both of the operations we’re calling in this function’s body that might fail: the `File::open` function and the `read_to_string` method.

The body of the function starts by calling the `File::open` function. Then we handle the `Result` value with a `match` similar to the `match` in Listing 9-4. If `File::open` succeeds, the file handle in the pattern variable `file` becomes the value in the mutable variable `f` and the function continues. In the `Err` case, instead of calling `panic!`, we use the `return` keyword to return early out of the function entirely and pass the error value from `File::open`, now in the pattern variable `e`, back to the calling code as this function’s error value.

So if we have a file handle in `f`, the function then creates a new `String` in variable `s` and calls the `read_to_string` method on the file handle in `f` to read the contents of the file into `s`. The `read_to_string` method also returns a `Result` because it might fail, even though `File::open` succeeded. So we need another `match` to handle that `Result`: if `read_to_string` succeeds, then our function has succeeded, and we return the username from the file that’s now in `s` wrapped in an `Ok`. If `read_to_string` fails, we return the error value in the same way that we returned the error value in the `match` that handled the return value of `File::open`. However, we don’t need to explicitly say `return`, because this is the last expression in the function.

The code that calls this code will then handle getting either an `Ok` value that contains a username or an `Err` value that contains an `io::Error`. We don’t know what the calling code will do with those values. If the calling code gets an `Err` value, it could call `panic!` and crash the program, use a default username, or look up the username from somewhere other than a file, for example. We don’t have enough information on what the calling code is actually trying to do, so we propagate all the success or error information upward for it to handle appropriately.

**This pattern of propagating errors is so common in Rust that Rust provides the question mark operator `?` to make this easier**.

#### A Shortcut for Propagating Errors: the `?` Operator

Listing 9-7 shows an implementation of `read_username_from_file` that has the same functionality as it had in Listing 9-6, but this implementation uses the `?` operator.

Filename: src/main.rs

```rust
use std::fs::File;
use std::io;
use std::io::Read;

fn read_username_from_file() -> Result<String, io::Error> {
    let mut f = File::open("hello.txt")?;
    let mut s = String::new();
    f.read_to_string(&mut s)?;
    Ok(s)
}
```

Listing 9-7: A function that returns errors to the calling code using the `?` operator

The `?` placed after a `Result` value is defined to work in almost the same way as the `match` expressions we defined to handle the `Result` values in Listing 9-6. **If the value of the `Result` is an `Ok`, the value inside the `Ok` will get returned from this expression, and the program will continue. <u>If the value is an `Err`, the `Err` will be returned from the whole function as if we had used the `return` keyword</u> so the error value gets propagated to the calling code**.

There is a difference between what the `match` expression from Listing 9-6 does and what the `?` operator does: **error values that have the `?` operator called on them go through the `from` function, defined in the `From` trait in the standard library, which is used to convert errors from one type into another. <u>When the `?` operator calls the `from` function, the error type received is converted into the error type defined in the return type of the current function</u>. This is useful when a function returns one error type to represent all the ways a function might fail, even if parts might fail for many different reasons. As long as there’s an `impl From<OtherError> for ReturnedError` to define the conversion in the trait’s `from` function, the `?` operator takes care of calling the `from` function automatically**.

In the context of Listing 9-7, the `?` at the end of the `File::open` call will return the value inside an `Ok` to the variable `f`. If an error occurs, the `?` operator will return early out of the whole function and give any `Err` value to the calling code. The same thing applies to the `?` at the end of the `read_to_string` call.

**The `?` operator eliminates a lot of boilerplate and makes this function’s implementation simpler. We could even shorten this code further by chaining method calls immediately after the `?`**, as shown in Listing 9-8.

Filename: src/main.rs

```rust
use std::fs::File;
use std::io;
use std::io::Read;

fn read_username_from_file() -> Result<String, io::Error> {
    let mut s = String::new();

    File::open("hello.txt")?.read_to_string(&mut s)?;

    Ok(s)
}
```

Listing 9-8: Chaining method calls after the `?` operator

We’ve moved the creation of the new `String` in `s` to the beginning of the function; that part hasn’t changed. Instead of creating a variable `f`, we’ve chained the call to `read_to_string` directly onto the result of `File::open("hello.txt")?`. We still have a `?` at the end of the `read_to_string` call, and we still return an `Ok` value containing the username in `s` when both `File::open` and `read_to_string` succeed rather than returning errors. The functionality is again the same as in Listing 9-6 and Listing 9-7; this is just a different, more ergonomic way to write it.

Speaking of different ways to write this function, Listing 9-9 shows that there’s a way to make this even shorter.

Filename: src/main.rs

```rust
use std::fs;
use std::io;

fn read_username_from_file() -> Result<String, io::Error> {
    fs::read_to_string("hello.txt")
}
```

Listing 9-9: Using `fs::read_to_string` instead of opening and then reading the file

**Reading a file into a string is a fairly common operation, so Rust provides the convenient `fs::read_to_string` function that opens the file, creates a new `String`, reads the contents of the file, puts the contents into that `String`, and returns it**. Of course, using `fs::read_to_string` doesn’t give us the opportunity to explain all the error handling, so we did it the longer way first.

#### Where The `?` Operator Can Be Used

**The `?` operator can only be used in functions that have a return type compatible with the value the `?` is used on**. This is because the `?` operator is defined to perform an early return of a value out of the function, in the same manner as the `match` expression we defined in Listing 9-6 did. In Listing 9-6, the `match` was using a `Result` value, and the early return arm returned an `Err(e)` value. The return type of the function has to be a `Result` to be compatible with this `return`.

In Listing 9-10, let’s look at the error we’ll get if we use the `?` operator in a `main` function with a return type of `()`:

```rust
use std::fs::File;

fn main() {
    let f = File::open("hello.txt")?;
}
```

Listing 9-10: **Attempting to use the `?` in the `main` function that returns `()` won’t compile**

This code opens a file, which might fail. The `?` operator follows the `Result` value returned by `File::open`, but this `main` function has the return type of `()`, not `Result`. When we compile this code, we get the following error message:

```console
$ cargo run
   Compiling error-handling v0.1.0 (file:///projects/error-handling)
error[E0277]: the `?` operator can only be used in a function that returns `Result` or `Option` (or another type that implements `FromResidual`)
   --> src/main.rs:4:36
    |
3   | / fn main() {
4   | |     let f = File::open("hello.txt")?;
    | |                                    ^ cannot use the `?` operator in a function that returns `()`
5   | | }
    | |_- this function should return `Result` or `Option` to accept `?`
    |
    = help: the trait `FromResidual<Result<Infallible, std::io::Error>>` is not implemented for `()`
note: required by `from_residual`

For more information about this error, try `rustc --explain E0277`.
error: could not compile `error-handling` due to previous error
```

**This error points out that we’re only allowed to use the `?` operator in a function that returns `Result`, `Option`, or another type that implements `FromResidual`**. To fix this error, you have two choices. One technique is to change the return type of your function to be `Result<T, E>` if you have no restrictions preventing that. The other technique is to use a `match` or one of the `Result<T, E>` methods to handle the `Result<T, E>` in whatever way is appropriate.

The error message also mentioned that `?` can be used with `Option<T>` values as well. As with using `?` on `Result`, you can only use `?` on `Option` in a function that returns an `Option`. **The behavior of the `?` operator when called on an `Option<T>` is similar to its behavior when called on a `Result<T, E>`: if the value is `None`, the `None` will be returned early from the function at that point. If the value is `Some`, the value inside the `Some` is the resulting value of the expression and the function continues**. Listing 9-11 has an example of a function that finds the last character of the first line in the given text:

```rust
fn last_char_of_first_line(text: &str) -> Option<char> {
    text.lines().next()?.chars().last()
}
```

Listing 9-11: Using the `?` operator on an `Option<T>` value

This function returns `Option<char>` because it might find a character at this position, or there might be no character there. This code takes the `text` string slice argument and calls the `lines` method on it, which returns an iterator over the lines in the string. Because this function wants to examine the first line, it calls `next` on the iterator to get the first value from the iterator. <u>If `text` is the empty string, this call to `next` will return `None`, and here we can use `?` to stop and return `None` from `last_char_of_first_line` if that is the case. If `text` is not the empty string, `next` will return a `Some` value containing a string slice of the first line in `text`.</u>

The `?` extracts the string slice, and we can call `chars` on that string slice to get an iterator of the characters in this string slice. We’re interested in the last character in this first line, so we call `last` to return the last item in the iterator over the characters. This is an `Option` because the first line might be the empty string, if `text` starts with a blank line but has characters on other lines, as in `"\nhi"`. However, if there is a last character on the first line, it will be returned in the `Some` variant. The `?` operator in the middle gives us a concise way to express this logic, and this function can be implemented in one line. If we couldn’t use the `?` operator on `Option`, we’d have to implement this logic using more method calls or a `match` expression.

**Note that you can use the `?` operator on a `Result` in a function that returns `Result`, and you can use the `?` operator on an `Option` in a function that returns `Option`, but you can’t mix and match. <u>The `?` operator won’t automatically convert a `Result` to an `Option` or vice versa</u>; in those cases, there are methods like the `ok` method on `Result` or the `ok_or` method on `Option` that will do the conversion explicitly**.

So far, all the `main` functions we’ve used return `()`. The `main` function is special because it’s the entry and exit point of executable programs, and there are restrictions on what its return type can be for the programs to behave as expected. **Executables written in C return integers when they exit, and Rust executables follow this convention as well: programs that exit successfully return the integer `0`, and programs that error return some integer other than `0`. When `main` returns `()`, <u>Rust executables will return `0` if `main` returns and a nonzero value if the program panics before reaching the end of `main`</u>**.

**Another return type `main` can have is `Result<(), E>`**. Listing 9-12 has the code from Listing 9-10 but we’ve changed the return type of `main` to be `Result<(), Box<dyn Error>>` and added a return value `Ok(())` to the end. This code will now compile:

```rust
use std::error::Error;
use std::fs::File;

fn main() -> Result<(), Box<dyn Error>> {
    let f = File::open("hello.txt")?;

    Ok(())
}
```

Listing 9-12: Changing `main` to return `Result<(), E>` allows the use of the `?` operator on `Result` values

The `Box<dyn Error>` type is called a trait object, which we’ll talk about in the [“Using Trait Objects that Allow for Values of Different Types”](https://doc.rust-lang.org/book/ch17-02-trait-objects.html#using-trait-objects-that-allow-for-values-of-different-types) section in Chapter 17. For now, you can read `Box<dyn Error>` to mean “any kind of error.” Using `?` on a `Result` value in a `main` function with this return type is allowed, because now an `Err` value can be returned early. **When a `main` function returns a `Result<(), E>`, the executable will exit with a value of `0` if `main` returns `Ok(())` and will exit with a nonzero value if `main` returns an `Err` value**.

The types that `main` may return are those that implement [the `std::process::Termination` trait](https://doc.rust-lang.org/std/process/trait.Termination.html). As of this writing, the `Termination` trait is an unstable feature only available in Nightly Rust, so you can’t yet implement it for your own types in Stable Rust, but you might be able to someday!

Now that we’ve discussed the details of calling `panic!` or returning `Result`, let’s return to the topic of how to decide which is appropriate to use in which cases.

## 9.3 To `panic!` or Not to `panic!`

简言之，当不明确定义的function是否应调用`panic!`的时候，默认返回`Result<T,E>`是一个不错的选择（让调用者决定如何处理可能出现的panic）。

So how do you decide when you should call `panic!` and when you should return `Result`? When code panics, there’s no way to recover. You could call `panic!` for any error situation, whether there’s a possible way to recover or not, but then you’re making the decision on behalf of the code calling your code that a situation is unrecoverable. When you choose to return a `Result` value, you give the calling code options rather than making the decision for it. The calling code could choose to attempt to recover in a way that’s appropriate for its situation, or it could decide that an `Err` value in this case is unrecoverable, so it can call `panic!` and turn your recoverable error into an unrecoverable one. Therefore, returning `Result` is a good default choice when you’re defining a function that might fail.

有些情况下，最好是在function中直接直接调用`panic!`，而不是返回Result，下面将讨论这些情况，以及分析为什么这么做。

### Examples, Prototype Code, and Tests

+ 比如编写一个用于说明某些概念的example，通过`unwrap`来处理异常，会更加容易理解。

+ `unwrap`和`expect`在还不明确应该如何处理异常时很有用。
+ 在进行代码测试（诸如单元测试时），比起直接使用`panic!`，可考虑使用`unwrap`或`expect`处理某些具体方法的异常情况。

When you’re writing an example to illustrate some concept, having robust error-handling code in the example as well can make the example less clear. In examples, it’s understood that a call to a method like `unwrap` that could panic is meant as a placeholder for the way you’d want your application to handle errors, which can differ based on what the rest of your code is doing.

Similarly, the `unwrap` and `expect` methods are very handy when prototyping, before you’re ready to decide how to handle errors. They leave clear markers in your code for when you’re ready to make your program more robust.

If a method call fails in a test, you’d want the whole test to fail, even if that method isn’t the functionality under test. Because `panic!` is how a test is marked as a failure, calling `unwrap` or `expect` is exactly what should happen.

### Cases in Which You Have More Information Than the Compiler

简言之，当你明确不会代码出现`Err`时（编译器并不知道这回事），可以考虑直接使用`unwrap`。

**It would also be appropriate to call `unwrap` when you have some other logic that ensures the `Result` will have an `Ok` value, but the logic isn’t something the compiler understands**. You’ll still have a `Result` value that you need to handle: whatever operation you’re calling still has the possibility of failing in general, even though it’s logically impossible in your particular situation. If you can ensure by manually inspecting the code that you’ll never have an `Err` variant, it’s perfectly acceptable to call `unwrap`. Here’s an example:

```rust
use std::net::IpAddr;

let home: IpAddr = "127.0.0.1".parse().unwrap();
```

We’re creating an `IpAddr` instance by parsing a hardcoded string. We can see that `127.0.0.1` is a valid IP address, so it’s acceptable to use `unwrap` here. However, having a hardcoded, valid string doesn’t change the return type of the `parse` method: we still get a `Result` value, and the compiler will still make us handle the `Result` as if the `Err` variant is a possibility because the compiler isn’t smart enough to see that this string is always a valid IP address. <u>If the IP address string came from a user rather than being hardcoded into the program and therefore *did* have a possibility of failure, we’d definitely want to handle the `Result` in a more robust way instead</u>.

### Guidelines for Error Handling

**It’s advisable to have your code panic when it’s possible that your code could end up in a bad state**. In this context, a *bad state* is when some assumption, guarantee, contract, or invariant has been broken, such as when invalid values, contradictory values, or missing values are passed to your code—plus one or more of the following:

- **The bad state is something that is unexpected, as opposed to something that will likely happen occasionally, like a user entering data in the wrong format.**
- Your code after this point needs to rely on not being in this bad state, rather than checking for the problem at every step.
- There’s not a good way to encode this information in the types you use. We’ll work through an example of what we mean in the [“Encoding States and Behavior as Types”](https://doc.rust-lang.org/book/ch17-03-oo-design-patterns.html#encoding-states-and-behavior-as-types) section of Chapter 17.

**If someone calls your code and passes in values that don’t make sense, the best choice might be to call `panic!` and alert the person using your library to the bug in their code so they can fix it during development.** <u>Similarly, `panic!` is often appropriate if you’re calling external code that is out of your control and it returns an invalid state that you have no way of fixing</u>.

However, **when failure is expected, it’s more appropriate to return a `Result` than to make a `panic!` call.** <u>Examples include a parser being given malformed data or an HTTP request returning a status that indicates you have hit a rate limit. In these cases, returning a `Result` indicates that failure is an expected possibility that the calling code must decide how to handle</u>.

下面这段大意：如果你的function用于处理某些值，你应该验证这些值是否“合法”，并且当值不符合要求（比如业务要求值>0）时及时调用`panic!`，然后提示调用者应该按照API文档传参。

**When your code performs operations on values, your code should verify the values are valid first and panic if the values aren’t valid**. This is mostly for safety reasons: attempting to operate on invalid data can expose your code to vulnerabilities. This is the main reason the standard library will call `panic!` if you attempt an out-of-bounds memory access: trying to access memory that doesn’t belong to the current data structure is a common security problem. Functions often have *contracts*: their behavior is only guaranteed if the inputs meet particular requirements. Panicking when the contract is violated makes sense because a contract violation always indicates a caller-side bug and it’s not a kind of error you want the calling code to have to explicitly handle. In fact, there’s no reasonable way for calling code to recover; the calling *programmers* need to fix the code. <u>Contracts for a function, especially when a violation will cause a panic, should be explained in the API documentation for the function</u>.

下面这段大意：代码中进行大量的error check是十分麻烦的，幸运的是Rust已经包揽了大部分的check逻辑。比如永远不需担心function调用时传入的参数是nothing，因为这个情况在编译时就不会通过。

However, having lots of error checks in all of your functions would be verbose and annoying. **Fortunately, you can use Rust’s type system (and thus the type checking the compiler does) to do many of the checks for you**. If your function has a particular type as a parameter, you can proceed with your code’s logic knowing that the compiler has already ensured you have a valid value. For example, if you have a type rather than an `Option`, your program expects to have *something* rather than *nothing*. Your code then doesn’t have to handle two cases for the `Some` and `None` variants: it will only have one case for definitely having a value. **<u>Code trying to pass nothing to your function won’t even compile, so your function doesn’t have to check for that case at runtime</u>**. Another example is using an unsigned integer type such as `u32`, which ensures the parameter is never negative.

### Creating Custom Types for Validation

Let’s take the idea of using Rust’s type system to ensure we have a valid value one step further and look at creating a custom type for validation. Recall the guessing game in Chapter 2 in which our code asked the user to guess a number between 1 and 100. We never validated that the user’s guess was between those numbers before checking it against our secret number; we only validated that the guess was positive. In this case, the consequences were not very dire: our output of “Too high” or “Too low” would still be correct. But it would be a useful enhancement to guide the user toward valid guesses and have different behavior when a user guesses a number that’s out of range versus when a user types, for example, letters instead.

One way to do this would be to parse the guess as an `i32` instead of only a `u32` to allow potentially negative numbers, and then add a check for the number being in range, like so:

```rust
loop {
  // --snip--

  let guess: i32 = match guess.trim().parse() {
    Ok(num) => num,
    Err(_) => continue,
  };

  if guess < 1 || guess > 100 {
    println!("The secret number will be between 1 and 100.");
    continue;
  }

  match guess.cmp(&secret_number) {
    // --snip--
  }
```

The `if` expression checks whether our value is out of range, tells the user about the problem, and calls `continue` to start the next iteration of the loop and ask for another guess. After the `if` expression, we can proceed with the comparisons between `guess` and the secret number knowing that `guess` is between 1 and 100.

However, this is not an ideal solution: if it was absolutely critical that the program only operated on values between 1 and 100, and it had many functions with this requirement, having a check like this in every function would be tedious (and might impact performance).

Instead, we can make a new type and put the validations in a function to create an instance of the type rather than repeating the validations everywhere. That way, it’s safe for functions to use the new type in their signatures and confidently use the values they receive. Listing 9-13 shows one way to define a `Guess` type that will only create an instance of `Guess` if the `new` function receives a value between 1 and 100.

```rust
pub struct Guess {
    value: i32,
}

impl Guess {
    pub fn new(value: i32) -> Guess {
        if value < 1 || value > 100 {
            panic!("Guess value must be between 1 and 100, got {}.", value);
        }

        Guess { value }
    }

    pub fn value(&self) -> i32 {
        self.value
    }
}
```

Listing 9-13: A `Guess` type that will only continue with values between 1 and 100

First, we define a struct named `Guess` that has a field named `value` that holds an `i32`. This is where the number will be stored.

Then we implement an associated function named `new` on `Guess` that creates instances of `Guess` values. The `new` function is defined to have one parameter named `value` of type `i32` and to return a `Guess`. The code in the body of the `new` function tests `value` to make sure it’s between 1 and 100. If `value` doesn’t pass this test, we make a `panic!` call, which will alert the programmer who is writing the calling code that they have a bug they need to fix, because creating a `Guess` with a `value` outside this range would violate the contract that `Guess::new` is relying on. The conditions in which `Guess::new` might panic should be discussed in its public-facing API documentation; we’ll cover documentation conventions indicating the possibility of a `panic!` in the API documentation that you create in Chapter 14. If `value` does pass the test, we create a new `Guess` with its `value` field set to the `value` parameter and return the `Guess`.

Next, we implement a method named `value` that borrows `self`, doesn’t have any other parameters, and returns an `i32`. This kind of method is sometimes called a *getter*, because its purpose is to get some data from its fields and return it. This public method is necessary because the `value` field of the `Guess` struct is private. It’s important that the `value` field be private so code using the `Guess` struct is not allowed to set `value` directly: code outside the module *must* use the `Guess::new` function to create an instance of `Guess`, thereby ensuring there’s no way for a `Guess` to have a `value` that hasn’t been checked by the conditions in the `Guess::new` function.

A function that has a parameter or returns only numbers between 1 and 100 could then declare in its signature that it takes or returns a `Guess` rather than an `i32` and wouldn’t need to do any additional checks in its body.

### Summary

Rust’s error handling features are designed to help you write more robust code. The `panic!` macro signals that your program is in a state it can’t handle and lets you tell the process to stop instead of trying to proceed with invalid or incorrect values. The `Result` enum uses Rust’s type system to indicate that operations might fail in a way that your code could recover from. You can use `Result` to tell code that calls your code that it needs to handle potential success or failure as well. Using `panic!` and `Result` in the appropriate situations will make your code more reliable in the face of inevitable problems.

Now that you’ve seen useful ways that the standard library uses generics with the `Option` and `Result` enums, we’ll talk about how generics work and how you can use them in your code.

# 10. Generic Types, Traits, and Lifetimes

Every programming language has tools for effectively handling the duplication of concepts. In Rust, one such tool is *generics*. Generics are abstract stand-ins for concrete types or other properties. When we’re writing code, we can express the behavior of generics or how they relate to other generics without knowing what will be in their place when compiling and running the code.

Similar to the way a function takes parameters with unknown values to run the same code on multiple concrete values, functions can take parameters of some generic type instead of a concrete type, like `i32` or `String`. In fact, we’ve already used generics in Chapter 6 with `Option<T>`, Chapter 8 with `Vec<T>` and `HashMap<K, V>`, and Chapter 9 with `Result<T, E>`. In this chapter, you’ll explore how to define your own types, functions, and methods with generics!

First, we’ll review how to extract a function to reduce code duplication. Next, we’ll use the same technique to make a generic function from two functions that differ only in the types of their parameters. We’ll also explain how to **use generic types in struct and enum definitions**.

Then you’ll learn how to use *traits* to define behavior in a generic way. You can combine traits with generic types to constrain a generic type to only those types that have a particular behavior, as opposed to just any type.

Finally, we’ll discuss ***lifetimes*, a variety of generics that give the compiler information about how references relate to each other. Lifetimes allow us to borrow values in many situations while still enabling the compiler to check that the references are valid**.

## 10.1 Generic Data Types

### In Function Definitions

When defining a function that uses generics, we place the generics in the signature of the function where we would usually specify the data types of the parameters and return value. Doing so makes our code more flexible and provides more functionality to callers of our function while preventing code duplication.

Continuing with our `largest` function, Listing 10-4 shows two functions that both find the largest value in a slice.

Filename: src/main.rs

```rust
fn largest_i32(list: &[i32]) -> i32 {
    let mut largest = list[0];

    for &item in list {
        if item > largest {
            largest = item;
        }
    }

    largest
}

fn largest_char(list: &[char]) -> char {
    let mut largest = list[0];

    for &item in list {
        if item > largest {
            largest = item;
        }
    }

    largest
}

fn main() {
    let number_list = vec![34, 50, 25, 100, 65];

    let result = largest_i32(&number_list);
    println!("The largest number is {}", result);

    let char_list = vec!['y', 'm', 'a', 'q'];

    let result = largest_char(&char_list);
    println!("The largest char is {}", result);
}
```

Listing 10-4: Two functions that differ only in their names and the types in their signatures

The `largest_i32` function is the one we extracted in Listing 10-3 that finds the largest `i32` in a slice. The `largest_char` function finds the largest `char` in a slice. The function bodies have the same code, so let’s eliminate the duplication by introducing a generic type parameter in a single function.

To parameterize the types in the new function we’ll define, we need to name the type parameter, just as we do for the value parameters to a function. You can use any identifier as a type parameter name. But we’ll use `T` because, by convention, parameter names in Rust are short, often just a letter, and **Rust’s type-naming convention is CamelCase**. Short for “type,” `T` is the default choice of most Rust programmers.

When we use a parameter in the body of the function, we have to declare the parameter name in the signature so the compiler knows what that name means. Similarly, when we use a type parameter name in a function signature, we have to declare the type parameter name before we use it. To define the generic `largest` function, place type name declarations inside angle brackets, `<>`, between the name of the function and the parameter list, like this:

```rust
fn largest<T>(list: &[T]) -> T {
```

We read this definition as: the function `largest` is generic over some type `T`. This function has one parameter named `list`, which is a slice of values of type `T`. The `largest` function will return a value of the same type `T`.

Listing 10-5 shows the combined `largest` function definition using the generic data type in its signature. The listing also shows how we can call the function with either a slice of `i32` values or `char` values. Note that this code won’t compile yet, but we’ll fix it later in this chapter.

Filename: src/main.rs

```rust
fn largest<T>(list: &[T]) -> T {
  let mut largest = list[0];

  for &item in list {
    if item > largest {
      largest = item;
    }
  }

  largest
}

fn main() {
  let number_list = vec![34, 50, 25, 100, 65];

  let result = largest(&number_list);
  println!("The largest number is {}", result);

  let char_list = vec!['y', 'm', 'a', 'q'];

  let result = largest(&char_list);
  println!("The largest char is {}", result);
}
```

Listing 10-5: A definition of the `largest` function that uses generic type parameters but doesn’t compile yet

If we compile this code right now, we’ll get this error:

```shell
$ cargo run
   Compiling chapter10 v0.1.0 (file:///projects/chapter10)
error[E0369]: binary operation `>` cannot be applied to type `T`
 --> src/main.rs:5:17
  |
5 |         if item > largest {
  |            ---- ^ ------- T
  |            |
  |            T
  |
help: consider restricting type parameter `T`
  |
1 | fn largest<T: std::cmp::PartialOrd>(list: &[T]) -> T {
  |             ^^^^^^^^^^^^^^^^^^^^^^

For more information about this error, try `rustc --explain E0369`.
error: could not compile `chapter10` due to previous error
```

The note mentions `std::cmp::PartialOrd`, which is a *trait*. We’ll talk about traits in the next section. For now, this error states that the body of `largest` won’t work for all possible types that `T` could be. <u>Because we want to compare values of type `T` in the body, we can only use types whose values can be ordered</u>. To enable comparisons, the standard library has the `std::cmp::PartialOrd` trait that you can implement on types (see Appendix C for more on this trait). You’ll learn how to specify that a generic type has a particular trait in the [“Traits as Parameters”](https://doc.rust-lang.org/book/ch10-02-traits.html#traits-as-parameters) section, but let’s first explore other ways of using generic type parameters.

### In Struct Definitions

We can also define structs to use a generic type parameter in one or more fields using the `<>` syntax. Listing 10-6 shows how to define a `Point<T>` struct to hold `x` and `y` coordinate values of any type.

Filename: src/main.rs

```rust
struct Point<T> {
    x: T,
    y: T,
}

fn main() {
    let integer = Point { x: 5, y: 10 };
    let float = Point { x: 1.0, y: 4.0 };
}
```

Listing 10-6: A `Point<T>` struct that holds `x` and `y` values of type `T`

Filename: src/main.rs

```rust
struct Point<T> {
  x: T,
  y: T,
}

fn main() {
  let wont_work = Point { x: 5, y: 4.0 };
}
```

Listing 10-7: The fields `x` and `y` must be the same type because both have the same generic data type `T`.

In this example, when we assign the integer value 5 to `x`, we let the compiler know that the generic type `T` will be an integer for this instance of `Point<T>`. Then when we specify 4.0 for `y`, which we’ve defined to have the same type as `x`, we’ll get a type mismatch error like this:

```shell
$ cargo run
   Compiling chapter10 v0.1.0 (file:///projects/chapter10)
error[E0308]: mismatched types
 --> src/main.rs:7:38
  |
7 |     let wont_work = Point { x: 5, y: 4.0 };
  |                                      ^^^ expected integer, found floating-point number

For more information about this error, try `rustc --explain E0308`.
error: could not compile `chapter10` due to previous error
```

To define a `Point` struct where `x` and `y` are both generics but could have different types, we can use multiple generic type parameters. For example, in Listing 10-8, we can change the definition of `Point` to be generic over types `T` and `U` where `x` is of type `T` and `y` is of type `U`.

Filename: src/main.rs

```rust
struct Point<T, U> {
    x: T,
    y: U,
}

fn main() {
    let both_integer = Point { x: 5, y: 10 };
    let both_float = Point { x: 1.0, y: 4.0 };
    let integer_and_float = Point { x: 5, y: 4.0 };
}
```

Listing 10-8: A `Point<T, U>` generic over two types so that `x` and `y` can be values of different types

Now all the instances of `Point` shown are allowed! You can use as many generic type parameters in a definition as you want, but using more than a few makes your code hard to read. When you need lots of generic types in your code, it could indicate that your code needs restructuring into smaller pieces.

### In Enum Definitions

As we did with structs, we can define enums to hold generic data types in their variants. Let’s take another look at the `Option<T>` enum that the standard library provides, which we used in Chapter 6:

```rust
enum Option<T> {
  Some(T),
  None,
}
```

This definition should now make more sense to you. As you can see, `Option<T>` is an enum that is generic over type `T` and has two variants: `Some`, which holds one value of type `T`, and a `None` variant that doesn’t hold any value. By using the `Option<T>` enum, we can express the abstract concept of having an optional value, and because `Option<T>` is generic, we can use this abstraction no matter what the type of the optional value is.

Enums can use multiple generic types as well. The definition of the `Result` enum that we used in Chapter 9 is one example:

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

The `Result` enum is generic over two types, `T` and `E`, and has two variants: `Ok`, which holds a value of type `T`, and `Err`, which holds a value of type `E`. This definition makes it convenient to use the `Result` enum anywhere we have an operation that might succeed (return a value of some type `T`) or fail (return an error of some type `E`). In fact, this is what we used to open a file in Listing 9-3, where `T` was filled in with the type `std::fs::File` when the file was opened successfully and `E` was filled in with the type `std::io::Error` when there were problems opening the file.

When you recognize situations in your code with multiple struct or enum definitions that differ only in the types of the values they hold, you can avoid duplication by using generic types instead.

### In Method Definitions

We can implement methods on structs and enums (as we did in Chapter 5) and use generic types in their definitions, too. Listing 10-9 shows the `Point<T>` struct we defined in Listing 10-6 with a method named `x` implemented on it.

Filename: src/main.rs

```rust
struct Point<T> {
    x: T,
    y: T,
}

impl<T> Point<T> {
    fn x(&self) -> &T {
        &self.x
    }
}

fn main() {
    let p = Point { x: 5, y: 10 };

    println!("p.x = {}", p.x());
}
```

Listing 10-9: Implementing a method named `x` on the `Point<T>` struct that will return a reference to the `x` field of type `T`

Here, we’ve defined a method named `x` on `Point<T>` that returns a reference to the data in the field `x`.

**Note that we have to declare `T` just after `impl` so we can use it to specify that we’re implementing methods on the type `Point<T>`**. **<u>By declaring `T` as a generic type after `impl`, Rust can identify that the type in the angle brackets in `Point` is a generic type rather than a concrete type</u>**. Because this is declaring the generic again, we could have chosen a different name for the generic parameter than the generic parameter declared in the struct definition, but using the same name is conventional. Methods written within an `impl` that declares the generic type will be defined on any instance of the type, no matter what concrete type ends up substituting for the generic type.

**<u>The other option we have is defining methods on the type with some constraint on the generic type</u>**. We could, for example, implement methods only on `Point<f32>` instances rather than on `Point<T>` instances with any generic type. In Listing 10-10 we use the concrete type `f32`, meaning we don’t declare any types after `impl`.

Filename: src/main.rs

```rust
impl Point<f32> {
    fn distance_from_origin(&self) -> f32 {
        (self.x.powi(2) + self.y.powi(2)).sqrt()
    }
}
```

Listing 10-10: An `impl` block that only applies to a struct with a particular concrete type for the generic type parameter `T`

**This code means the type `Point<f32>` will have a method named `distance_from_origin` and other instances of `Point<T>` where `T` is not of type `f32` will not have this method defined**. The method measures how far our point is from the point at coordinates (0.0, 0.0) and uses mathematical operations that are available only for floating point types.

**<u>Generic type parameters in a struct definition aren’t always the same as those you use in that struct’s method signatures</u>**. Listing 10-11 uses the generic types `X1` and `Y1` for the `Point` struct and `X2` `Y2` for the `mixup` method signature to make the example clearer. The method creates a new `Point` instance with the `x` value from the `self` `Point` (of type `X1`) and the `y` value from the passed-in `Point` (of type `Y2`).

Filename: src/main.rs

```rust
struct Point<X1, Y1> {
    x: X1,
    y: Y1,
}

impl<X1, Y1> Point<X1, Y1> {
    fn mixup<X2, Y2>(self, other: Point<X2, Y2>) -> Point<X1, Y2> {
        Point {
            x: self.x,
            y: other.y,
        }
    }
}

fn main() {
    let p1 = Point { x: 5, y: 10.4 };
    let p2 = Point { x: "Hello", y: 'c' };

    let p3 = p1.mixup(p2);

    println!("p3.x = {}, p3.y = {}", p3.x, p3.y);
}
```

Listing 10-11: A method that uses different generic types from its struct’s definition

In `main`, we’ve defined a `Point` that has an `i32` for `x` (with value `5`) and an `f64` for `y` (with value `10.4`). The `p2` variable is a `Point` struct that has a string slice for `x` (with value `"Hello"`) and a `char` for `y` (with value `c`). **Calling `mixup` on `p1` with the argument `p2` gives us `p3`, which will have an `i32` for `x`, because `x` came from `p1`. The `p3` variable will have a `char` for `y`, because `y` came from `p2`. The `println!` macro call will print `p3.x = 5, p3.y = c`.**

The purpose of this example is to demonstrate a situation in which some generic parameters are declared with `impl` and some are declared with the method definition. **<u>Here, the generic parameters `X1` and `Y1` are declared after `impl` because they go with the struct definition. The generic parameters `X2` and `Y2` are declared after `fn mixup`, because they’re only relevant to the method</u>**.

### Performance of Code Using Generics

Rust使用泛型并不会造成运行时的成本。

**You might be wondering whether there is a runtime cost when you’re using generic type parameters. The good news is that Rust implements generics in such a way that your code doesn’t run any slower using generic types than it would with concrete types**.

Rust**仅在编译时使用泛型**（**在编译期间即转换泛型参数为具体的类型**）。

Rust accomplishes this by performing monomorphization of the code that is **<u>using generics at compile time</u>**. <u>*Monomorphization* is the process of turning generic code into specific code by filling in the concrete types that are used when compiled</u>.

In this process, the compiler does the opposite of the steps we used to create the generic function in Listing 10-5: the compiler looks at all the places where generic code is called and generates code for the concrete types the generic code is called with.

Let’s look at how this works with an example that uses the standard library’s `Option<T>` enum:

```rust
let integer = Some(5);
let float = Some(5.0);
```

When Rust compiles this code, it performs monomorphization. During that process, the compiler reads the values that have been used in `Option<T>` instances and identifies two kinds of `Option<T>`: one is `i32` and the other is `f64`. As such, it expands the generic definition of `Option<T>` into `Option_i32` and `Option_f64`, thereby replacing the generic definition with the specific ones.

The monomorphized version of the code looks like the following. The generic `Option<T>` is replaced with the specific definitions created by the compiler:

Filename: src/main.rs

```rust
enum Option_i32 {
    Some(i32),
    None,
}

enum Option_f64 {
    Some(f64),
    None,
}

fn main() {
    let integer = Option_i32::Some(5);
    let float = Option_f64::Some(5.0);
}
```

由于Rust编译时会将泛型解析成具体类型，所以运行时使用泛型没有任何额外成本。

**Because Rust compiles generic code into code that specifies the type in each instance, we pay no runtime cost for using generics. When the code runs, it performs just as it would if we had duplicated each definition by hand. The process of monomorphization makes Rust’s generics extremely efficient at runtime**.

## 10.2 Traits: Defining Shared Behavior

A *trait* tells the Rust compiler about functionality a particular type has and can share with other types. We can use traits to define shared behavior in an abstract way. We can use trait bounds to specify that a generic type can be any type that has certain behavior.

> Note: Traits are similar to a feature often called *interfaces* in other languages, although with some differences.

### Defining a Trait

A type’s behavior consists of the methods we can call on that type. Different types share the same behavior if we can call the same methods on all of those types. Trait definitions are a way to group method signatures together to define a set of behaviors necessary to accomplish some purpose.

For example, let’s say we have multiple structs that hold various kinds and amounts of text: a `NewsArticle` struct that holds a news story filed in a particular location and a `Tweet` that can have at most 280 characters along with metadata that indicates whether it was a new tweet, a retweet, or a reply to another tweet.

We want to make a media aggregator library crate named `aggregator` that can display summaries of data that might be stored in a `NewsArticle` or `Tweet` instance. To do this, we need a summary from each type, and we’ll request that summary by calling a `summarize` method on an instance. Listing 10-12 shows the definition of a public `Summary` trait that expresses this behavior.

Filename: src/lib.rs

```rust
pub trait Summary {
    fn summarize(&self) -> String;
}
```

Listing 10-12: A `Summary` trait that consists of the behavior provided by a `summarize` method

Here, we declare a trait using the `trait` keyword and then the trait’s name, which is `Summary` in this case. <u>We’ve also declared the trait as `pub` so that crates depending on this crate can make use of this trait too</u>, as we’ll see in a few examples. Inside the curly brackets, we declare the method signatures that describe the behaviors of the types that implement this trait, which in this case is `fn summarize(&self) -> String`.

Rust的trait类似Java语言的接口，要求"实现者"需要实现trait中的funciton。

After the method signature, instead of providing an implementation within curly brackets, we use a semicolon. **Each type implementing this trait must provide its own custom behavior for the body of the method. The compiler will enforce that any type that has the `Summary` trait will have the method `summarize` defined with this signature exactly**.

A trait can have multiple methods in its body: the method signatures are listed one per line and each line ends in a semicolon.

### Implementing a Trait on a Type

Now that we’ve defined the desired signatures of the `Summary` trait’s methods, we can implement it on the types in our media aggregator. Listing 10-13 shows an implementation of the `Summary` trait on the `NewsArticle` struct that uses the headline, the author, and the location to create the return value of `summarize`. For the `Tweet` struct, we define `summarize` as the username followed by the entire text of the tweet, assuming that tweet content is already limited to 280 characters.

Filename: src/lib.rs

```rust
pub struct NewsArticle {
    pub headline: String,
    pub location: String,
    pub author: String,
    pub content: String,
}

impl Summary for NewsArticle {
    fn summarize(&self) -> String {
        format!("{}, by {} ({})", self.headline, self.author, self.location)
    }
}

pub struct Tweet {
    pub username: String,
    pub content: String,
    pub reply: bool,
    pub retweet: bool,
}

impl Summary for Tweet {
    fn summarize(&self) -> String {
        format!("{}: {}", self.username, self.content)
    }
}
```

Listing 10-13: Implementing the `Summary` trait on the `NewsArticle` and `Tweet` types

Implementing a trait on a type is similar to implementing regular methods. The difference is that after `impl`, we put the trait name that we want to implement, then use the `for` keyword, and then specify the name of the type we want to implement the trait for. Within the `impl` block, we put the method signatures that the trait definition has defined. Instead of adding a semicolon after each signature, we use curly brackets and fill in the method body with the specific behavior that we want the methods of the trait to have for the particular type.

Now that the library has implemented the `Summary` trait on `NewsArticle` and `Tweet`, users of the crate can call the trait methods on instances of `NewsArticle` and `Tweet` in the same way we call regular methods. <u>The only difference is that the trait has to be brought into scope as well as the types to get the additional trait methods</u>. Here’s an example of how a binary crate could use our `aggregator` library crate:

```rust
use aggregator::{Summary, Tweet};

fn main() {
    let tweet = Tweet {
        username: String::from("horse_ebooks"),
        content: String::from(
            "of course, as you probably already know, people",
        ),
        reply: false,
        retweet: false,
    };

    println!("1 new tweet: {}", tweet.summarize());
}
```

This code prints `1 new tweet: horse_ebooks: of course, as you probably already know, people`.

**Other crates that depend on the `aggregator` crate can also bring the `Summary` trait into scope to implement the trait on their own types**. 

**Rust给struct、enum实现trait的方法时，trait本身、strurt或enum本身，至少需要有一个在本地crate中**。

**<u>One restriction to note with trait implementations is that we can implement a trait on a type only if at least one of the trait or the type is local to our crate</u>**. For example, we can implement standard library traits like `Display` on a custom type like `Tweet` as part of our `aggregator` crate functionality, because the type `Tweet` is local to our `aggregator` crate. We can also implement `Summary` on `Vec<T>` in our `aggregator` crate, because the trait `Summary` is local to our `aggregator` crate.

**在Rust中不能够给外部的type实现外部的trait方法。这能保证项目的“一致性”，这种限制也被称为“孤儿”法则。如果没有这么做，当两个crate都为同一个type以不同方式实现trait中的方法，将导致Rust无法决定该使用哪一种实现。这也保证了用户对某个type实现的trait，不会被其他项目所破坏**。

**<u>But we can’t implement external traits on external types</u>**. For example, <u>we can’t implement the `Display` trait on `Vec<T>` within our `aggregator` crate, because `Display` and `Vec<T>` are defined in the standard library and aren’t local to our `aggregator` crate</u>. **This restriction is part of a property of programs called *coherence*, and more specifically the *orphan rule***, so named because the parent type is not present. <u>This rule ensures that other people’s code can’t break your code and vice versa. Without the rule, two crates could implement the same trait for the same type, and Rust wouldn’t know which implementation to use</u>.

### Default Implementations

Sometimes it’s useful to have default behavior for some or all of the methods in a trait instead of requiring implementations for all methods on every type. Then, as we implement the trait on a particular type, we can keep or override each method’s default behavior.

Listing 10-14 shows how to specify a default string for the `summarize` method of the `Summary` trait instead of only defining the method signature, as we did in Listing 10-12.

Filename: src/lib.rs

```rust
pub trait Summary {
    fn summarize(&self) -> String {
        String::from("(Read more...)")
    }
}
```

Listing 10-14: Definition of a `Summary` trait with a default implementation of the `summarize` method

To use a default implementation to summarize instances of `NewsArticle` instead of defining a custom implementation, we specify an empty `impl` block with `impl Summary for NewsArticle {}`.

Even though we’re no longer defining the `summarize` method on `NewsArticle` directly, we’ve provided a default implementation and specified that `NewsArticle` implements the `Summary` trait. As a result, we can still call the `summarize` method on an instance of `NewsArticle`, like this:

```rust
let article = NewsArticle {
  headline: String::from("Penguins win the Stanley Cup Championship!"),
  location: String::from("Pittsburgh, PA, USA"),
  author: String::from("Iceburgh"),
  content: String::from(
    "The Pittsburgh Penguins once again are the best \
    hockey team in the NHL.",
  ),
};

println!("New article available! {}", article.summarize());
```

This code prints `New article available! (Read more...)`.

Creating a default implementation for `summarize` doesn’t require us to change anything about the implementation of `Summary` on `Tweet` in Listing 10-13. The reason is that the syntax for overriding a default implementation is the same as the syntax for implementing a trait method that doesn’t have a default implementation.

**Default implementations can call other methods in the same trait, even if those other methods don’t have a default implementation**. In this way, a trait can provide a lot of useful functionality and only require implementors to specify a small part of it. For example, we could define the `Summary` trait to have a `summarize_author` method whose implementation is required, and then define a `summarize` method that has a default implementation that calls the `summarize_author` method:

```rust
pub trait Summary {
    fn summarize_author(&self) -> String;

    fn summarize(&self) -> String {
        format!("(Read more from {}...)", self.summarize_author())
    }
}
```

To use this version of `Summary`, we only need to define `summarize_author` when we implement the trait on a type:

```rust
impl Summary for Tweet {
    fn summarize_author(&self) -> String {
        format!("@{}", self.username)
    }
}
```

After we define `summarize_author`, we can call `summarize` on instances of the `Tweet` struct, and the default implementation of `summarize` will call the definition of `summarize_author` that we’ve provided. Because we’ve implemented `summarize_author`, the `Summary` trait has given us the behavior of the `summarize` method without requiring us to write any more code.

```rust
    let tweet = Tweet {
        username: String::from("horse_ebooks"),
        content: String::from(
            "of course, as you probably already know, people",
        ),
        reply: false,
        retweet: false,
    };

    println!("1 new tweet: {}", tweet.summarize());
```

This code prints `1 new tweet: (Read more from @horse_ebooks...)`.

**Note that it isn’t possible to call the default implementation from an overriding implementation of that same method**.

### Traits as Parameters

Now that you know how to define and implement traits, we can explore how to use traits to define functions that accept many different types.

For example, in Listing 10-13, we implemented the `Summary` trait on the `NewsArticle` and `Tweet` types. We can define a `notify` function that calls the `summarize` method on its `item` parameter, which is of some type that implements the `Summary` trait. **To do this, we can use the `impl Trait` syntax**, like this:

```rust
pub fn notify(item: &impl Summary) {
    println!("Breaking news! {}", item.summarize());
}
```

Instead of a concrete type for the `item` parameter, we specify the `impl` keyword and the trait name. This parameter accepts any type that implements the specified trait. In the body of `notify`, we can call any methods on `item` that come from the `Summary` trait, such as `summarize`. We can call `notify` and pass in any instance of `NewsArticle` or `Tweet`. Code that calls the function with any other type, such as a `String` or an `i32`, won’t compile because those types don’t implement `Summary`.

#### Trait Bound Syntax

**The `impl Trait` syntax works for straightforward cases but is actually syntax sugar** for a longer form, which is called a *trait bound*; it looks like this:

```rust
pub fn notify<T: Summary>(item: &T) {
    println!("Breaking news! {}", item.summarize());
}
```

This longer form is equivalent to the example in the previous section but is more verbose. We place trait bounds with the declaration of the generic type parameter after a colon and inside angle brackets.

The `impl Trait` syntax is convenient and makes for more concise code in simple cases. The trait bound syntax can express more complexity in other cases. For example, we can have two parameters that implement `Summary`. Using the `impl Trait` syntax looks like this:

```rust
pub fn notify(item1: &impl Summary, item2: &impl Summary) {
```

If we wanted this function to allow `item1` and `item2` to have different types, using `impl Trait` would be appropriate (as long as both types implement `Summary`). If we wanted to force both parameters to have the same type, that’s only possible to express using a trait bound, like this:

```rust
pub fn notify<T: Summary>(item1: &T, item2: &T) {
```

The generic type `T` specified as the type of the `item1` and `item2` parameters constrains the function such that the concrete type of the value passed as an argument for `item1` and `item2` must be the same.

#### Specifying Multiple Trait Bounds with the `+` Syntax

We can also specify more than one trait bound. Say we wanted `notify` to use display formatting on `item` as well as the `summarize` method: **we specify in the `notify` definition that `item` must implement both `Display` and `Summary`. We can do so using the `+` syntax**:

```rust
pub fn notify(item: &(impl Summary + Display)) {
```

**The `+` syntax is also valid with trait bounds on generic types**:

```rust
pub fn notify<T: Summary + Display>(item: &T) {
```

With the two trait bounds specified, the body of `notify` can call `summarize` and use `{}` to format `item`.

#### Clearer Trait Bounds with `where` Clauses

Using too many trait bounds has its downsides. Each generic has its own trait bounds, so functions with multiple generic type parameters can contain lots of trait bound information between the function’s name and its parameter list, making the function signature hard to read. For this reason, **Rust has alternate syntax for specifying trait bounds inside a `where` clause after the function signature**. So instead of writing this:

```rust
fn some_function<T: Display + Clone, U: Clone + Debug>(t: &T, u: &U) -> i32 {
```

we can use a `where` clause, like this:

```rust
fn some_function<T, U>(t: &T, u: &U) -> i32
    where T: Display + Clone,
          U: Clone + Debug
{
```

This function’s signature is less cluttered: the function name, parameter list, and return type are close together, similar to a function without lots of trait bounds.

### Returning Types that Implement Traits

We can also use the `impl Trait` syntax in the return position to return a value of some type that implements a trait, as shown here:

```rust
fn returns_summarizable() -> impl Summary {
    Tweet {
        username: String::from("horse_ebooks"),
        content: String::from(
            "of course, as you probably already know, people",
        ),
        reply: false,
        retweet: false,
    }
}
```

By using `impl Summary` for the return type, we specify that the `returns_summarizable` function returns some type that implements the `Summary` trait without naming the concrete type. In this case, `returns_summarizable` returns a `Tweet`, but the code calling this function doesn’t know that.

**The ability to return a type that is only specified by the trait it implements is especially useful in the context of closures and iterators, which we cover in Chapter 13**. <u>Closures and iterators create types that **only the compiler knows** or types that are very long to specify</u>. The `impl Trait` syntax lets you concisely specify that a function returns some type that implements the `Iterator` trait without needing to write out a very long type.

**Rust支持返回值为某个implement trait的类型，但是不允许方法中返回多种实现trait的类型**（即必须能在编译阶段解析出实际返回值的类型）

**However, you can only use `impl Trait` if you’re returning a single type**. For example, this code that returns either a `NewsArticle` or a `Tweet` with the return type specified as `impl Summary` wouldn’t work:

```rust
fn returns_summarizable(switch: bool) -> impl Summary {
    if switch {
        NewsArticle {
            headline: String::from(
                "Penguins win the Stanley Cup Championship!",
            ),
            location: String::from("Pittsburgh, PA, USA"),
            author: String::from("Iceburgh"),
            content: String::from(
                "The Pittsburgh Penguins once again are the best \
                 hockey team in the NHL.",
            ),
        }
    } else {
        Tweet {
            username: String::from("horse_ebooks"),
            content: String::from(
                "of course, as you probably already know, people",
            ),
            reply: false,
            retweet: false,
        }
    }
}
```

Returning either a `NewsArticle` or a `Tweet` isn’t allowed due to restrictions around how the `impl Trait` syntax is implemented in the compiler. We’ll cover how to write a function with this behavior in the [“Using Trait Objects That Allow for Values of Different Types”](https://doc.rust-lang.org/book/ch17-02-trait-objects.html#using-trait-objects-that-allow-for-values-of-different-types) section of Chapter 17.

### Fixing the `largest` Function with Trait Bounds

Now that you know how to specify the behavior you want to use using the generic type parameter’s bounds, let’s return to Listing 10-5 to fix the definition of the `largest` function that uses a generic type parameter! Last time we tried to run that code, we received this error:

```console
$ cargo run
   Compiling chapter10 v0.1.0 (file:///projects/chapter10)
error[E0369]: binary operation `>` cannot be applied to type `T`
 --> src/main.rs:5:17
  |
5 |         if item > largest {
  |            ---- ^ ------- T
  |            |
  |            T
  |
help: consider restricting type parameter `T`
  |
1 | fn largest<T: std::cmp::PartialOrd>(list: &[T]) -> T {
  |             ^^^^^^^^^^^^^^^^^^^^^^

For more information about this error, try `rustc --explain E0369`.
error: could not compile `chapter10` due to previous error
```

In the body of `largest` we wanted to compare two values of type `T` using the greater than (`>`) operator. Because that operator is defined as a default method on the standard library trait `std::cmp::PartialOrd`, we need to specify `PartialOrd` in the trait bounds for `T` so the `largest` function can work on slices of any type that we can compare. We don’t need to bring `PartialOrd` into scope because it’s in the prelude. Change the signature of `largest` to look like this:

```rust
fn largest<T: PartialOrd>(list: &[T]) -> T {
```

This time when we compile the code, we get a different set of errors:

```console
$ cargo run
   Compiling chapter10 v0.1.0 (file:///projects/chapter10)
error[E0508]: cannot move out of type `[T]`, a non-copy slice
 --> src/main.rs:2:23
  |
2 |     let mut largest = list[0];
  |                       ^^^^^^^
  |                       |
  |                       cannot move out of here
  |                       move occurs because `list[_]` has type `T`, which does not implement the `Copy` trait
  |                       help: consider borrowing here: `&list[0]`

error[E0507]: cannot move out of a shared reference
 --> src/main.rs:4:18
  |
4 |     for &item in list {
  |         -----    ^^^^
  |         ||
  |         |data moved here
  |         |move occurs because `item` has type `T`, which does not implement the `Copy` trait
  |         help: consider removing the `&`: `item`

Some errors have detailed explanations: E0507, E0508.
For more information about an error, try `rustc --explain E0507`.
error: could not compile `chapter10` due to 2 previous errors
```

The key line in this error is `cannot move out of type [T], a non-copy slice`. With our non-generic versions of the `largest` function, we were only trying to find the largest `i32` or `char`. As discussed in the [“Stack-Only Data: Copy”](https://doc.rust-lang.org/book/ch04-01-what-is-ownership.html#stack-only-data-copy) section in Chapter 4, types like `i32` and `char` that have a known size can be stored on the stack, so they implement the `Copy` trait. **But when we made the `largest` function generic, it became possible for the `list` parameter to have types in it that don’t implement the `Copy` trait**. Consequently, we wouldn’t be able to move the value out of `list[0]` and into the `largest` variable, resulting in this error.

To call this code with only those types that implement the `Copy` trait, we can add `Copy` to the trait bounds of `T`! Listing 10-15 shows the complete code of a generic `largest` function that will compile as long as the types of the values in the slice that we pass into the function implement the `PartialOrd` *and* `Copy` traits, like `i32` and `char` do.

Filename: src/main.rs

```rust
fn largest<T: PartialOrd + Copy>(list: &[T]) -> T {
    let mut largest = list[0];

    for &item in list {
        if item > largest {
            largest = item;
        }
    }

    largest
}

fn main() {
    let number_list = vec![34, 50, 25, 100, 65];

    let result = largest(&number_list);
    println!("The largest number is {}", result);

    let char_list = vec!['y', 'm', 'a', 'q'];

    let result = largest(&char_list);
    println!("The largest char is {}", result);
}
```

Listing 10-15: A working definition of the `largest` function that works on any generic type that implements the `PartialOrd` and `Copy` traits

<u>**If we don’t want to restrict the `largest` function to the types that implement the `Copy` trait, we could specify that `T` has the trait bound `Clone` instead of `Copy`**.</u> Then we could clone each value in the slice when we want the `largest` function to have ownership. **Using the `clone` function means we’re potentially making more heap allocations in the case of types that own heap data like `String`, and heap allocations can be slow if we’re working with large amounts of data**.

**Another way we could implement `largest` is for the function to return a reference to a `T` value in the slice. If we change the return type to `&T` instead of `T`, thereby changing the body of the function to return a reference, we wouldn’t need the `Clone` or `Copy` trait bounds and we could avoid heap allocations**. Try implementing these alternate solutions on your own! If you get stuck with errors having to do with lifetimes, keep reading: the “Validating References with Lifetimes” section coming up will explain, but lifetimes aren’t required to solve these challenges.

### Using Trait Bounds to Conditionally Implement Methods

> [Rust的Blanket Implements(通用实现) - SegmentFault 思否](https://segmentfault.com/a/1190000037510636)

**By using a trait bound with an `impl` block that uses generic type parameters, we can implement methods conditionally for types that implement the specified traits**. For example, the type `Pair<T>` in Listing 10-16 always implements the `new` function to return a new instance of `Pair<T>` (recall from the [”Defining Methods”](https://doc.rust-lang.org/book/ch05-03-method-syntax.html#defining-methods) section of Chapter 5 that `Self` is a type alias for the type of the `impl` block, which in this case is `Pair<T>`). But in the next `impl` block, `Pair<T>` only implements the `cmp_display` method if its inner type `T` implements the `PartialOrd` trait that enables comparison *and* the `Display` trait that enables printing.

Filename: src/lib.rs

```rust
use std::fmt::Display;

struct Pair<T> {
    x: T,
    y: T,
}

impl<T> Pair<T> {
    fn new(x: T, y: T) -> Self {
        Self { x, y }
    }
}

impl<T: Display + PartialOrd> Pair<T> {
    fn cmp_display(&self) {
        if self.x >= self.y {
            println!("The largest member is x = {}", self.x);
        } else {
            println!("The largest member is y = {}", self.y);
        }
    }
}
```

Listing 10-16: Conditionally implement methods on a generic type depending on trait bounds

**<u>We can also conditionally implement a trait for any type that implements another trait. Implementations of a trait on any type that satisfies the trait bounds are called *blanket implementations* and are extensively used in the Rust standard library.</u>** 

For example, the standard library implements the `ToString` trait on any type that implements the `Display` trait. The `impl` block in the standard library looks similar to this code:

```rust
impl<T: Display> ToString for T {
    // --snip--
}
```

Because the standard library has this blanket implementation, we can call the `to_string` method defined by the `ToString` trait on any type that implements the `Display` trait. For example, we can turn integers into their corresponding `String` values like this because integers implement `Display`:

```rust
let s = 3.to_string();
```

Blanket implementations appear in the documentation for the trait in the “Implementors” section.

Traits and trait bounds let us write code that uses generic type parameters to reduce duplication but also specify to the compiler that we want the generic type to have particular behavior. **The compiler can then use the trait bound information to check that all the concrete types used with our code provide the correct behavior**. In dynamically typed languages, we would get an error at runtime if we called a method on a type which didn’t define the method. But Rust moves these errors to compile time so we’re forced to fix the problems before our code is even able to run. **<u>Additionally, we don’t have to write code that checks for behavior at runtime because we’ve already checked at compile time. Doing so improves performance without having to give up the flexibility of generics</u>**.

**<u>Another kind of generic that we’ve already been using is called *lifetimes*. Rather than ensuring that a type has the behavior we want, lifetimes ensure that references are valid as long as we need them to be</u>**. Let’s look at how lifetimes do that.

## 10.3 Validating References with Lifetimes

**在Rust中所有引用都有lifetime（表明在什么作用域内当前reference有效），和类型判断类似，一般reference的lifetime由编译器隐式判断，仅在我们需要不同的lifetime时才需要特地声明**。

One detail we didn’t discuss in the [“References and Borrowing”](https://doc.rust-lang.org/book/ch04-02-references-and-borrowing.html#references-and-borrowing) section in Chapter 4 is that **<u>every reference in Rust has a *lifetime*, which is the scope for which that reference is valid</u>**. Most of the time, lifetimes are implicit and inferred, just like most of the time, types are inferred. We must annotate types when multiple types are possible. <u>In a similar way, we must annotate lifetimes when the lifetimes of references could be related in a few different ways</u>. **Rust requires us to annotate the relationships using generic lifetime parameters to ensure the actual references used at runtime will definitely be valid**.

Annotating lifetimes is not even a concept most other programming languages have, so this is going to feel unfamiliar. Although we won’t cover lifetimes in their entirety in this chapter, we’ll discuss common ways you might encounter lifetime syntax so you can get introduced to the concept.

### Preventing Dangling References with Lifetimes

**<u>The main aim of lifetimes is to prevent dangling references, which cause a program to reference data other than the data it’s intended to reference</u>**. Consider the program in Listing 10-17, which has an outer scope and an inner scope.

```rust
{
  let r;

  {
    let x = 5;
    r = &x;
  }

  println!("r: {}", r);
}
```

Listing 10-17: An attempt to use a reference whose value has gone out of scope

> Note: The examples in Listings 10-17, 10-18, and 10-24 declare variables without giving them an initial value, so the variable name exists in the outer scope. At first glance, this might appear to be in conflict with Rust’s having no null values. However, if we try to use a variable before giving it a value, we’ll get a compile-time error, which shows that Rust indeed does not allow null values.

The outer scope declares a variable named `r` with no initial value, and the inner scope declares a variable named `x` with the initial value of 5. Inside the inner scope, we attempt to set the value of `r` as a reference to `x`. Then the inner scope ends, and we attempt to print the value in `r`. This code won’t compile because the value `r` is referring to has gone out of scope before we try to use it. Here is the error message:

```console
$ cargo run
   Compiling chapter10 v0.1.0 (file:///projects/chapter10)
error[E0597]: `x` does not live long enough
  --> src/main.rs:7:17
   |
7  |             r = &x;
   |                 ^^ borrowed value does not live long enough
8  |         }
   |         - `x` dropped here while still borrowed
9  | 
10 |         println!("r: {}", r);
   |                           - borrow later used here

For more information about this error, try `rustc --explain E0597`.
error: could not compile `chapter10` due to previous error
```

The variable `x` doesn’t “live long enough.” The reason is that `x` will be out of scope when the inner scope ends on line 7. But `r` is still valid for the outer scope; because its scope is larger, we say that it “lives longer.” **<u>If Rust allowed this code to work, `r` would be referencing memory that was deallocated when `x` went out of scope, and anything we tried to do with `r` wouldn’t work correctly. So how does Rust determine that this code is invalid? It uses a borrow checker</u>**.

### The Borrow Checker

**The Rust compiler has a *borrow checker* that compares scopes to determine whether all borrows are valid**. Listing 10-18 shows the same code as Listing 10-17 but with annotations showing the lifetimes of the variables.

```rust
{
        let r;                // ---------+-- 'a
                              //          |
        {                     //          |
            let x = 5;        // -+-- 'b  |
            r = &x;           //  |       |
        }                     // -+       |
                              //          |
        println!("r: {}", r); //          |
    }                         // ---------+
```

Listing 10-18: Annotations of the lifetimes of `r` and `x`, named `'a` and `'b`, respectively

<u>Here, we’ve annotated the lifetime of `r` with `'a` and the lifetime of `x` with `'b`. As you can see, the inner `'b` block is much smaller than the outer `'a` lifetime block. At compile time, Rust compares the size of the two lifetimes and sees that `r` has a lifetime of `'a` but that it refers to memory with a lifetime of `'b`. The program is rejected because `'b` is shorter than `'a`: the subject of the reference doesn’t live as long as the reference</u>.

Listing 10-19 fixes the code so it doesn’t have a dangling reference and compiles without any errors.

```rust
    {
        let x = 5;            // ----------+-- 'b
                              //           |
        let r = &x;           // --+-- 'a  |
                              //   |       |
        println!("r: {}", r); //   |       |
                              // --+       |
    }                         // ----------+
```

Listing 10-19: **A valid reference because the data has a longer lifetime than the reference**

<u>Here, `x` has the lifetime `'b`, which in this case is larger than `'a`. This means `r` can reference `x` because Rust knows that the reference in `r` will always be valid while `x` is valid</u>.

Now that you know where the lifetimes of references are and how Rust analyzes lifetimes to ensure references will always be valid, let’s explore generic lifetimes of parameters and return values in the context of functions.

### Generic Lifetimes in Functions

Let’s write a function that returns the longer of two string slices. This function will take two string slices and return a string slice. After we’ve implemented the `longest` function, the code in Listing 10-20 should print `The longest string is abcd`.

Filename: src/main.rs

```rust
fn main() {
    let string1 = String::from("abcd");
    let string2 = "xyz";

    let result = longest(string1.as_str(), string2);
    println!("The longest string is {}", result);
}
```

Listing 10-20: A `main` function that calls the `longest` function to find the longer of two string slices

Note that we want the function to take string slices, which are references, because we don’t want the `longest` function to take ownership of its parameters. Refer to the [“String Slices as Parameters”](https://doc.rust-lang.org/book/ch04-03-slices.html#string-slices-as-parameters) section in Chapter 4 for more discussion about why the parameters we use in Listing 10-20 are the ones we want.

If we try to implement the `longest` function as shown in Listing 10-21, it won’t compile.

Filename: src/main.rs

```rust
fn longest(x: &str, y: &str) -> &str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

Listing 10-21: An implementation of the `longest` function that returns the longer of two string slices but does not yet compile

Instead, we get the following error that talks about lifetimes:

```console
$ cargo run
   Compiling chapter10 v0.1.0 (file:///projects/chapter10)
error[E0106]: missing lifetime specifier
 --> src/main.rs:9:33
  |
9 | fn longest(x: &str, y: &str) -> &str {
  |               ----     ----     ^ expected named lifetime parameter
  |
  = help: this function's return type contains a borrowed value, but the signature does not say whether it is borrowed from `x` or `y`
help: consider introducing a named lifetime parameter
  |
9 | fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
  |           ^^^^    ^^^^^^^     ^^^^^^^     ^^^

For more information about this error, try `rustc --explain E0106`.
error: could not compile `chapter10` due to previous error
```

The help text reveals that the return type needs a generic lifetime parameter on it because **Rust can’t tell whether the reference being returned refers to `x` or `y`.** Actually, we don’t know either, because the `if` block in the body of this function returns a reference to `x` and the `else` block returns a reference to `y`!

**上面这个方法定义，第一点是Rust编译期间无法确定返回值到底是x还是y的reference，第二点是Rust无法确定返回值的lifetime是和x一致还是和y一致。（比如x可能比y的lifetime，反之亦然，返回值的lifetime具有不确定性）**

When we’re defining this function, we don’t know the concrete values that will be passed into this function, so we don’t know whether the `if` case or the `else` case will execute. **We also don’t know the concrete lifetimes of the references that will be passed in, so we can’t look at the scopes as we did in Listings 10-18 and 10-19 to determine whether the reference we return will always be valid**. <u>The borrow checker can’t determine this either, because it doesn’t know how the lifetimes of `x` and `y` relate to the lifetime of the return value</u>. To fix this error, we’ll add generic lifetime parameters that define the relationship between the references so the borrow checker can perform its analysis.

### Lifetime Annotation Syntax

**<u>Lifetime annotations don’t change how long any of the references live</u>**. Just as functions can accept any type when the signature specifies a generic type parameter, functions can accept references with any lifetime by specifying a generic lifetime parameter. **Lifetime annotations describe the relationships of the lifetimes of multiple references to each other without affecting the lifetimes**.

**Lifetime annotations have a slightly unusual syntax: the names of lifetime parameters must start with an apostrophe (`'`) and are usually all lowercase and very short, like generic types**. Most people use the name `'a`. 

**We place lifetime parameter annotations after the `&` of a reference, using a space to separate the annotation from the reference’s type**.

Here are some examples: a reference to an `i32` without a lifetime parameter, a reference to an `i32` that has a lifetime parameter named `'a`, and a mutable reference to an `i32` that also has the lifetime `'a`.

```rust
&i32        // a reference
&'a i32     // a reference with an explicit lifetime
&'a mut i32 // a mutable reference with an explicit lifetime
```

<u>One lifetime annotation by itself doesn’t have much meaning, because the annotations are meant to tell Rust how generic lifetime parameters of multiple references relate to each other</u>. 

<u>For example, let’s say we have a function with the parameter `first` that is a reference to an `i32` with lifetime `'a`. The function also has another parameter named `second` that is another reference to an `i32` that also has the lifetime `'a`. The lifetime annotations indicate that the references `first` and `second` must both live as long as that generic lifetime</u>.

### Lifetime Annotations in Function Signatures

Now let’s examine lifetime annotations in the context of the `longest` function. As with generic type parameters, we need to declare generic lifetime parameters inside angle brackets between the function name and the parameter list. <u>The constraint we want to express in this signature is that the lifetimes of both of the parameters and the lifetime of the returned reference are related such that the returned reference will be valid as long as both the parameters are</u>. We’ll name the lifetime `'a` and then add it to each reference, as shown in Listing 10-22.

Filename: src/main.rs

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

Listing 10-22: The `longest` function definition specifying that all the references in the signature must have the same lifetime `'a`

This code should compile and produce the result we want when we use it with the `main` function in Listing 10-20.

The function signature now tells Rust that for some lifetime `'a`, the function takes two parameters, both of which are string slices that live at least as long as lifetime `'a`. The function signature also tells Rust that the string slice returned from the function will live at least as long as lifetime `'a`. In practice, it means that the lifetime of the reference returned by the `longest` function is the same as the smaller of the lifetimes of the references passed in. <u>These relationships are what we want Rust to use when analyzing this code</u>.

**Remember, when we specify the lifetime parameters in this function signature, we’re not changing the lifetimes of any values passed in or returned**. Rather, we’re specifying that the borrow checker should reject any values that don’t adhere to these constraints. Note that the `longest` function doesn’t need to know exactly how long `x` and `y` will live, only that some scope can be substituted for `'a` that will satisfy this signature.

**When annotating lifetimes in functions, the annotations go in the function signature, not in the function body. The lifetime annotations become part of the contract of the function, much like the types in the signature are**. <u>Having function signatures contain the lifetime contract means the analysis the Rust compiler does can be simpler. If there’s a problem with the way a function is annotated or the way it is called, the compiler errors can point to the part of our code and the constraints more precisely. If, instead, the Rust compiler made more inferences about what we intended the relationships of the lifetimes to be, the compiler might only be able to point to a use of our code many steps away from the cause of the problem.</u>

**Rust会从使用同一个generic liftime标志的变量中，选取编译时lifetime最小的一个，作为约束条件**。

**When we pass concrete references to `longest`, the concrete lifetime that is substituted for `'a` is the part of the scope of `x` that overlaps with the scope of `y`**. <u>In other words, the generic lifetime `'a` will get the concrete lifetime that is equal to the smaller of the lifetimes of `x` and `y`.</u> Because we’ve annotated the returned reference with the same lifetime parameter `'a`, the returned reference will also be valid for the length of the smaller of the lifetimes of `x` and `y`.

Let’s look at how the lifetime annotations restrict the `longest` function by passing in references that have different concrete lifetimes. Listing 10-23 is a straightforward example.

Filename: src/main.rs

```rust
fn main() {
    let string1 = String::from("long string is long");

    {
        let string2 = String::from("xyz");
        let result = longest(string1.as_str(), string2.as_str());
        println!("The longest string is {}", result);
    }
}
```

Listing 10-23: Using the `longest` function with references to `String` values that have different concrete lifetimes

In this example, `string1` is valid until the end of the outer scope, `string2` is valid until the end of the inner scope, and `result` references something that is valid until the end of the inner scope. Run this code, and you’ll see that the borrow checker approves of this code; it will compile and print `The longest string is long string is long`.

Next, let’s try an example that shows that the lifetime of the reference in `result` must be the smaller lifetime of the two arguments. We’ll move the declaration of the `result` variable outside the inner scope but leave the assignment of the value to the `result` variable inside the scope with `string2`. Then we’ll move the `println!` that uses `result` outside the inner scope, after the inner scope has ended. The code in Listing 10-24 will not compile.

Filename: src/main.rs

```rust
fn main() {
    let string1 = String::from("long string is long");
    let result;
    {
        let string2 = String::from("xyz");
        result = longest(string1.as_str(), string2.as_str());
    }
    println!("The longest string is {}", result);
}
```

Listing 10-24: Attempting to use `result` after `string2` has gone out of scope

When we try to compile this code, we’ll get this error:

```console
$ cargo run
   Compiling chapter10 v0.1.0 (file:///projects/chapter10)
error[E0597]: `string2` does not live long enough
 --> src/main.rs:6:44
  |
6 |         result = longest(string1.as_str(), string2.as_str());
  |                                            ^^^^^^^ borrowed value does not live long enough
7 |     }
  |     - `string2` dropped here while still borrowed
8 |     println!("The longest string is {}", result);
  |                                          ------ borrow later used here

For more information about this error, try `rustc --explain E0597`.
error: could not compile `chapter10` due to previous error
```

The error shows that for `result` to be valid for the `println!` statement, `string2` would need to be valid until the end of the outer scope. Rust knows this because we annotated the lifetimes of the function parameters and return values using the same lifetime parameter `'a`.

As humans, we can look at this code and see that `string1` is longer than `string2` and therefore `result` will contain a reference to `string1`. Because `string1` has not gone out of scope yet, a reference to `string1` will still be valid for the `println!` statement. However, the compiler can’t see that the reference is valid in this case. We’ve told Rust that the lifetime of the reference returned by the `longest` function is the same as the smaller of the lifetimes of the references passed in. Therefore, the borrow checker disallows the code in Listing 10-24 as possibly having an invalid reference.

Try designing more experiments that vary the values and lifetimes of the references passed in to the `longest` function and how the returned reference is used. Make hypotheses about whether or not your experiments will pass the borrow checker before you compile; then check to see if you’re right!

### Thinking in Terms of Lifetimes

The way in which you need to specify lifetime parameters depends on what your function is doing. For example, if we changed the implementation of the `longest` function to always return the first parameter rather than the longest string slice, we wouldn’t need to specify a lifetime on the `y` parameter. The following code will compile:

Filename: src/main.rs

```rust
fn longest<'a>(x: &'a str, y: &str) -> &'a str {
    x
}
```

In this example, we’ve specified a lifetime parameter `'a` for the parameter `x` and the return type, but not for the parameter `y`, because the lifetime of `y` does not have any relationship with the lifetime of `x` or the return value.

**如果没有声明lifetime parameter，那么reference类型的返回值必须关联一个当前function内的值，这是一个危险操作，因为被引用的值作用域将超出当前function，被带到外部**。

When returning a reference from a function, the lifetime parameter for the return type needs to match the lifetime parameter for one of the parameters. **If the reference returned does *not* refer to one of the parameters, it must refer to a value created within this function, which would be a dangling reference because the value will go out of scope at the end of the function**. Consider this attempted implementation of the `longest` function that won’t compile:

Filename: src/main.rs

```rust
fn longest<'a>(x: &str, y: &str) -> &'a str {
    let result = String::from("really long string");
    result.as_str()
}
```

**Here, even though we’ve specified a lifetime parameter `'a` for the return type, this implementation will fail to compile because <u>the return value lifetime is not related to the lifetime of the parameters at all</u>**. Here is the error message we get:

```console
$ cargo run
   Compiling chapter10 v0.1.0 (file:///projects/chapter10)
error[E0515]: cannot return value referencing local variable `result`
  --> src/main.rs:11:5
   |
11 |     result.as_str()
   |     ------^^^^^^^^^
   |     |
   |     returns a value referencing data owned by the current function
   |     `result` is borrowed here

For more information about this error, try `rustc --explain E0515`.
error: could not compile `chapter10` due to previous error
```

**这里试图将返回值的引用返回给调用方，这相当于扩大了返回值的lifetime（原本只在funciton中生效，但试图让其在function执行完毕后仍生效），Rust不允许这种危险操作。就上述场景，更好的处理方式是直接返回result本身（而不是其reference，由调用方后续清理value，即调用`drop`方法回收内存）**

**The problem is that `result` goes out of scope and gets cleaned up at the end of the `longest` function**. We’re also trying to return a reference to `result` from the function. There is no way we can specify lifetime parameters that would change the dangling reference, and Rust won’t let us create a dangling reference. **In this case, the best fix would be to return an owned data type rather than a reference so the calling function is then responsible for cleaning up the value**.

lifetime syntax能够将各种参数的lifetime限制告知Rust编译器（交由其检查），Rust有足够信息能够保证内存安全的操作，并且在编译期间就禁止违反内存安全的操作（比如试图reference指向已经改被回收的对象等）

**Ultimately, lifetime syntax is about connecting the lifetimes of various parameters and return values of functions. Once they’re connected, Rust has enough information to allow memory-safe operations and disallow operations that would create dangling pointers or otherwise violate memory safety**.

### Lifetime Annotations in Struct Definitions

So far, we’ve only defined structs to hold owned types. It’s possible for structs to hold references, but in that case we would need to <u>add a lifetime annotation on every reference in the struct’s definition</u>. Listing 10-25 has a struct named `ImportantExcerpt` that holds a string slice.

Filename: src/main.rs

```rust
struct ImportantExcerpt<'a> {
    part: &'a str,
}

fn main() {
    let novel = String::from("Call me Ishmael. Some years ago...");
    let first_sentence = novel.split('.').next().expect("Could not find a '.'");
    let i = ImportantExcerpt {
        part: first_sentence,
    };
}
```

Listing 10-25: **A struct that holds a reference, so its definition needs a lifetime annotation**

This struct has one field, `part`, that holds a string slice, which is a reference. As with generic data types, we declare the name of the generic lifetime parameter inside angle brackets after the name of the struct so we can use the lifetime parameter in the body of the struct definition. **This annotation means an instance of `ImportantExcerpt` can’t outlive the reference it holds in its `part` field**.

<u>The `main` function here creates an instance of the `ImportantExcerpt` struct that holds a reference to the first sentence of the `String` owned by the variable `novel`. The data in `novel` exists before the `ImportantExcerpt` instance is created. In addition, `novel` doesn’t go out of scope until after the `ImportantExcerpt` goes out of scope, so the reference in the `ImportantExcerpt` instance is valid</u>.

### Lifetime Elision

You’ve learned that every reference has a lifetime and that you need to specify lifetime parameters for functions or structs that use references. However, <u>in Chapter 4 we had a function in Listing 4-9, which is shown again in Listing 10-26, that compiled without lifetime annotations.</u>

Filename: src/lib.rs

```rust
fn first_word(s: &str) -> &str {
    let bytes = s.as_bytes();

    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return &s[0..i];
        }
    }

    &s[..]
}
```

Listing 10-26: A function we defined in Listing 4-9 that compiled without lifetime annotations, even though the parameter and return type are references

**Rust的早期版本中，上述的代码无法运行，所有的reference参数都需要声明lifetime**。

**The reason this function compiles without lifetime annotations is historical: in early versions (pre-1.0) of Rust, this code wouldn’t have compiled because every reference needed an explicit lifetime**. At that time, the function signature would have been written like this:

```rust
fn first_word<'a>(s: &'a str) -> &'a str {
```

**后续Rust团队发现在一些特定场景下，function的入参和返回值的lifetime是一致的，于是将这些情况编写到编译器中（borrow checker会自动推断这些场景下reference的lifetime），这些情况下就无需声明引用的lifetime了**。

**<u>After writing a lot of Rust code, the Rust team found that Rust programmers were entering the same lifetime annotations over and over in particular situations. These situations were predictable and followed a few deterministic patterns. The developers programmed these patterns into the compiler’s code so the borrow checker could infer the lifetimes in these situations and wouldn’t need explicit annotations</u>**.

未来，Rust会尽量减少需要手动声明lifetime annotaition的场景（将多种lifetime推断直接编写到编译器中）。

<u>This piece of Rust history is relevant because it’s possible that more deterministic patterns will emerge and be added to the compiler. In the future, even fewer lifetime annotations might be required</u>.

**The patterns programmed into Rust’s analysis of references are called the *lifetime elision rules***. These aren’t rules for programmers to follow; they’re a set of particular cases that the compiler will consider, and if your code fits these cases, you don’t need to write the lifetimes explicitly.

The elision rules don’t provide full inference. If Rust deterministically applies the rules but there is still ambiguity as to what lifetimes the references have, the compiler won’t guess what the lifetime of the remaining references should be. In this case, instead of guessing, the compiler will give you an error that you can resolve by adding the lifetime annotations that specify how the references relate to each other.

**Lifetimes on function or method parameters are called *input lifetimes*, and lifetimes on return values are called *output lifetimes*.**

**当没有明确的lifetime annotaions时，编辑器通过三条规则推断lifetime references。第一条与入参有关，第二、第三条与返回值有关。当三条规则都无法推断lifetime时，则编译停止并报错。（这些规则适用于`fn`定义，同时也适用于`impl`代码块）**

+ **每个reference入参都有各自的lifetime parameter**
+ **当仅有一个入参声明了lifetime parameter，返回值的lifetime与其一致（使用相同lifetime parameter）**
+ **当有多个入参带有lifetime parameter，其中一个入参是`&self`或`&mut self`，则返回值的lifetime和`self`保持一致**

**The compiler uses three rules to figure out what lifetimes references have when there aren’t explicit annotations**. <u>The first rule applies to input lifetimes, and the second and third rules apply to output lifetimes</u>. If the compiler gets to the end of the three rules and there are still references for which it can’t figure out lifetimes, the compiler will stop with an error. **These rules apply to `fn` definitions as well as `impl` blocks**.

+ **The first rule is that each parameter that is a reference gets its own lifetime parameter**. In other words, a function with one parameter gets one lifetime parameter: `fn foo<'a>(x: &'a i32)`; a function with two parameters gets two separate lifetime parameters: `fn foo<'a, 'b>(x: &'a i32, y: &'b i32)`; and so on.

+ **The second rule is if there is exactly one input lifetime parameter, that lifetime is assigned to all output lifetime parameters**: `fn foo<'a>(x: &'a i32) -> &'a i32`.

+ **The third rule is if there are multiple input lifetime parameters, but one of them is `&self` or `&mut self` because this is a method, the lifetime of `self` is assigned to all output lifetime parameters**. This third rule makes methods much nicer to read and write because fewer symbols are necessary.

Let’s pretend we’re the compiler. We’ll apply these rules to figure out what the lifetimes of the references in the signature of the `first_word` function in Listing 10-26 are. The signature starts without any lifetimes associated with the references:

```rust
fn first_word(s: &str) -> &str {
```

Then the compiler applies the first rule, which specifies that each parameter gets its own lifetime. We’ll call it `'a` as usual, so now the signature is this:

```rust
fn first_word<'a>(s: &'a str) -> &str {
```

The second rule applies because there is exactly one input lifetime. **The second rule specifies that the lifetime of the one input parameter gets assigned to the output lifetime**, so the signature is now this:

```rust
fn first_word<'a>(s: &'a str) -> &'a str {
```

Now all the references in this function signature have lifetimes, and the compiler can continue its analysis without needing the programmer to annotate the lifetimes in this function signature.

Let’s look at another example, this time using the `longest` function that had no lifetime parameters when we started working with it in Listing 10-21:

```rust
fn longest(x: &str, y: &str) -> &str {
```

Let’s apply the first rule: each parameter gets its own lifetime. This time we have two parameters instead of one, so we have two lifetimes:

```rust
fn longest<'a, 'b>(x: &'a str, y: &'b str) -> &str {
```

**You can see that the second rule doesn’t apply because there is more than one input lifetime. The third rule doesn’t apply either, because `longest` is a function rather than a method, so none of the parameters are `self`**. After working through all three rules, we still haven’t figured out what the return type’s lifetime is. This is why we got an error trying to compile the code in Listing 10-21: the compiler worked through the lifetime elision rules but still couldn’t figure out all the lifetimes of the references in the signature.

Because the third rule really only applies in method signatures, we’ll look at lifetimes in that context next to see why the third rule means we don’t have to annotate lifetimes in method signatures very often.

#### Lifetime Annotations in Method Definitions

When we implement methods on a struct with lifetimes, we use the same syntax as that of generic type parameters shown in Listing 10-11. Where we declare and use the lifetime parameters depends on whether they’re related to the struct fields or the method parameters and return values.

**Lifetime names for struct fields always need to be declared after the `impl` keyword and then used after the struct’s name, because those lifetimes are part of the struct’s type.**

In method signatures inside the `impl` block, references might be tied to the lifetime of references in the struct’s fields, or they might be independent. **<u>In addition, the lifetime elision rules often make it so that lifetime annotations aren’t necessary in method signatures</u>**. Let’s look at some examples using the struct named `ImportantExcerpt` that we defined in Listing 10-25.

First, we’ll use a method named `level` whose only parameter is a reference to `self` and whose return value is an `i32`, which is not a reference to anything:

```rust
impl<'a> ImportantExcerpt<'a> {
    fn level(&self) -> i32 {
        3
    }
}
```

上述代码在`impl`和type name之后声明了lifetime parameter，却没有在入参`&self`前声明，因为第一条elision法则（所有reference入参都有各自的lifetime paramter）

**The lifetime parameter declaration after `impl` and its use after the type name are required, but we’re not required to annotate the lifetime of the reference to `self` because of the first elision rule.**

下面的代码示范了第三条elision法则（多个参数有lifetime paramter时，如果其中一个入参是`&self`或`&mut self`，则返回值的lifetime和`self`保持一致）

Here is an example where the third lifetime elision rule applies:

```rust
impl<'a> ImportantExcerpt<'a> {
    fn announce_and_return_part(&self, announcement: &str) -> &str {
        println!("Attention please: {}", announcement);
        self.part
    }
}
```

首先，根据elision第一条法则，两个reference入参有各自lifetime parameter；再根据第三条，得出

**There are two input lifetimes, so Rust applies the first lifetime elision rule and gives both `&self` and `announcement` their own lifetimes. Then, because one of the parameters is `&self`, the return type gets the lifetime of `&self`, and all lifetimes have been accounted for**.

#### The Static Lifetime

`'static'`表明reference的生命周期和程序保持一致。

**所有的string literals（字符串字面值/常量）都是`'static` lifetime**。

**One special lifetime we need to discuss is `'static`, which means that this reference *can* live for the entire duration of the program**. 

**All string literals have the `'static` lifetime**, which we can annotate as follows:

```rust
let s: &'static str = "I have a static lifetime.";
```

上述文本直接被存储到程序binary中，所以程序运行时永远有效，因此所有字符串字面值的lifetime都是`'static`。

**The text of this string is stored directly in the program’s binary, which is always available**. Therefore, the lifetime of all string literals is `'static`.

大多数场景并不需要使用`'static`，用前三思。

You might see suggestions to use the `'static` lifetime in error messages. But before specifying `'static` as the lifetime for a reference, think about whether the reference you have actually lives the entire lifetime of your program or not. You might consider whether you want it to live that long, even if it could. **Most of the time, the problem results from attempting to create a dangling reference or a mismatch of the available lifetimes. In such cases, the solution is fixing those problems, not specifying the `'static` lifetime.**

### Generic Type Parameters, Trait Bounds, and Lifetimes Together

Let’s briefly look at the syntax of specifying generic type parameters, trait bounds, and lifetimes all in one function!

```rust
use std::fmt::Display;

fn longest_with_an_announcement<'a, T>(
    x: &'a str,
    y: &'a str,
    ann: T,
) -> &'a str
where
    T: Display,
{
    println!("Announcement! {}", ann);
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

This is the `longest` function from Listing 10-22 that returns the longer of two string slices. But now it has an extra parameter named `ann` of the generic type `T`, which can be filled in by any type that implements the `Display` trait as specified by the `where` clause. This extra parameter will be printed using `{}`, which is why the `Display` trait bound is necessary. Because lifetimes are a type of generic, the declarations of the lifetime parameter `'a` and the generic type parameter `T` go in the same list inside the angle brackets after the function name.

### Summary

We covered a lot in this chapter! Now that you know about generic type parameters, traits and trait bounds, and generic lifetime parameters, you’re ready to write code without repetition that works in many different situations. <u>Generic type parameters let you apply the code to different types. Traits and trait bounds ensure that even though the types are generic, they’ll have the behavior the code needs. You learned how to use lifetime annotations to ensure that this flexible code won’t have any dangling references. And all of this analysis happens at compile time, which doesn’t affect runtime performance</u>!

Believe it or not, there is much more to learn on the topics we discussed in this chapter: Chapter 17 discusses **trait objects, which are another way to use traits**. **There are also more complex scenarios involving lifetime annotations that you will only need in very advanced scenarios; for those, you should read the [Rust Reference](https://doc.rust-lang.org/reference/index.html)**. But next, you’ll learn how to write tests in Rust so you can make sure your code is working the way it should.

# 11. Writing Automated Tests

Correctness in our programs is the extent to which our code does what we intend it to do. Rust is designed with a high degree of concern about the correctness of programs, but correctness is complex and not easy to prove. Rust’s type system shoulders a huge part of this burden, but the type system cannot catch every kind of incorrectness. As such, Rust includes support for writing automated software tests within the language.

Testing is a complex skill: although we can’t cover every detail about how to write good tests in one chapter, we’ll discuss the mechanics of Rust’s testing facilities. We’ll talk about the annotations and macros available to you when writing your tests, the default behavior and options provided for running your tests, and how to organize tests into unit tests and integration tests.

## 11.1 How to Write Tests

Tests are Rust functions that verify that the non-test code is functioning in the expected manner. The bodies of test functions typically perform these three actions:

1. Set up any needed data or state.
2. Run the code you want to test.
3. Assert the results are what you expect.

Let’s look at the features Rust provides specifically for writing tests that take these actions, which include the `test` attribute, a few macros, and the `should_panic` attribute.

### The Anatomy of a Test Function

At its simplest, a test in Rust is a function that’s annotated with the `test` attribute. Attributes are metadata about pieces of Rust code; one example is the `derive` attribute we used with structs in Chapter 5. **To change a function into a test function, add `#[test]` on the line before `fn`. When you run your tests with the `cargo test` command, Rust builds a test runner binary that runs the functions annotated with the `test` attribute and reports on whether each test function passes or fails**.

**When we make a new library project with Cargo, a test module with a test function in it is automatically generated for us**. This module helps you start writing your tests so you don’t have to look up the exact structure and syntax of test functions every time you start a new project. You can add as many additional test functions and as many test modules as you want!

We’ll explore some aspects of how tests work by experimenting with the template test generated for us without actually testing any code. Then we’ll write some real-world tests that call some code that we’ve written and assert that its behavior is correct.

Let’s create a new library project called `adder`:

```shell
$ cargo new adder --lib
     Created library `adder` project
$ cd adder
```

The contents of the *src/lib.rs* file in your `adder` library should look like Listing 11-1.

Filename: src/lib.rs

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn it_works() {
        assert_eq!(2 + 2, 4);
    }
}
```

Listing 11-1: The test module and function generated automatically by `cargo new`

For now, let’s ignore the top two lines and focus on the function to see how it works. **Note the `#[test]` annotation before the `fn` line: this attribute indicates this is a test function, so the test runner knows to treat this function as a test**. We could also have non-test functions in the `tests` module to help set up common scenarios or perform common operations, so we need to indicate which functions are tests by using the `#[test]` attribute.

The function body uses the `assert_eq!` macro to assert that 2 + 2 equals 4. This assertion serves as an example of the format for a typical test. Let’s run it to see that this test passes.

The `cargo test` command runs all tests in our project, as shown in Listing 11-2.

```console
$ cargo test
   Compiling adder v0.1.0 (file:///projects/adder)
    Finished test [unoptimized + debuginfo] target(s) in 0.57s
     Running unittests (target/debug/deps/adder-92948b65e88960b4)

running 1 test
test tests::it_works ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests adder

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Listing 11-2: The output from running the automatically generated test

Cargo compiled and ran the test. After the `Compiling`, `Finished`, and `Running` lines is the line `running 1 test`. The next line shows the name of the generated test function, called `it_works`, and the result of running that test, `ok`. The overall summary of running the tests appears next. The text `test result: ok.` means that all the tests passed, and the portion that reads `1 passed; 0 failed` totals the number of tests that passed or failed.

Because we don’t have any tests we’ve marked as ignored, the summary shows `0 ignored`. We also haven’t filtered the tests being run, so the end of the summary shows `0 filtered out`. We’ll talk about ignoring and filtering out tests in the next section, [“Controlling How Tests Are Run.”](https://doc.rust-lang.org/book/ch11-02-running-tests.html#controlling-how-tests-are-run)

**The `0 measured` statistic is for benchmark tests that measure performance**. Benchmark tests are, as of this writing, only available in nightly Rust. See [the documentation about benchmark tests](https://doc.rust-lang.org/unstable-book/library-features/test.html) to learn more.

The next part of the test output, which starts with `Doc-tests adder`, is for the results of any documentation tests. We don’t have any documentation tests yet, but Rust can compile any code examples that appear in our API documentation. This feature helps us keep our docs and our code in sync! We’ll discuss how to write documentation tests in the [“Documentation Comments as Tests”](https://doc.rust-lang.org/book/ch14-02-publishing-to-crates-io.html#documentation-comments-as-tests) section of Chapter 14. For now, we’ll ignore the `Doc-tests` output.

Let’s change the name of our test to see how that changes the test output. Change the `it_works` function to a different name, such as `exploration`, like so:

Filename: src/lib.rs

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn exploration() {
        assert_eq!(2 + 2, 4);
    }
}
```

Then run `cargo test` again. The output now shows `exploration` instead of `it_works`:

```console
$ cargo test
   Compiling adder v0.1.0 (file:///projects/adder)
    Finished test [unoptimized + debuginfo] target(s) in 0.59s
     Running unittests (target/debug/deps/adder-92948b65e88960b4)

running 1 test
test tests::exploration ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests adder

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Let’s add another test, but this time we’ll make a test that fails! Tests fail when something in the test function panics. Each test is run in a new thread, and when the main thread sees that a test thread has died, the test is marked as failed. We talked about the simplest way to cause a panic in Chapter 9, which is to call the `panic!` macro. Enter the new test, `another`, so your *src/lib.rs* file looks like Listing 11-3.

Filename: src/lib.rs

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn exploration() {
        assert_eq!(2 + 2, 4);
    }

    #[test]
    fn another() {
        panic!("Make this test fail");
    }
}
```

Listing 11-3: Adding a second test that will fail because we call the `panic!` macro

Run the tests again using `cargo test`. The output should look like Listing 11-4, which shows that our `exploration` test passed and `another` failed.

```console
$ cargo test
   Compiling adder v0.1.0 (file:///projects/adder)
    Finished test [unoptimized + debuginfo] target(s) in 0.72s
     Running unittests (target/debug/deps/adder-92948b65e88960b4)

running 2 tests
test tests::another ... FAILED
test tests::exploration ... ok

failures:

---- tests::another stdout ----
thread 'main' panicked at 'Make this test fail', src/lib.rs:10:9
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace


failures:
    tests::another

test result: FAILED. 1 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

error: test failed, to rerun pass '--lib'
```

Listing 11-4: Test results when one test passes and one test fails

Instead of `ok`, the line `test tests::another` shows `FAILED`. Two new sections appear between the individual results and the summary: the first section displays the detailed reason for each test failure. In this case, `another` failed because it `panicked at 'Make this test fail'`, which happened on line 10 in the *src/lib.rs* file. The next section lists just the names of all the failing tests, which is useful when there are lots of tests and lots of detailed failing test output. We can use the name of a failing test to run just that test to more easily debug it; we’ll talk more about ways to run tests in the [“Controlling How Tests Are Run”](https://doc.rust-lang.org/book/ch11-02-running-tests.html#controlling-how-tests-are-run) section.

The summary line displays at the end: overall, our test result is `FAILED`. We had one test pass and one test fail.

Now that you’ve seen what the test results look like in different scenarios, let’s look at some macros other than `panic!` that are useful in tests.

### Checking Results with the `assert!` Macro

The `assert!` macro, provided by the standard library, is useful when you want to ensure that some condition in a test evaluates to `true`. We give the `assert!` macro an argument that evaluates to a Boolean. **If the value is `true`, `assert!` does nothing and the test passes. If the value is `false`, the `assert!` macro calls the `panic!` macro, which causes the test to fail**. Using the `assert!` macro helps us check that our code is functioning in the way we intend.

In Chapter 5, Listing 5-15, we used a `Rectangle` struct and a `can_hold` method, which are repeated here in Listing 11-5. Let’s put this code in the *src/lib.rs* file and write some tests for it using the `assert!` macro.

Filename: src/lib.rs

```rust
#[derive(Debug)]
struct Rectangle {
    width: u32,
    height: u32,
}

impl Rectangle {
    fn can_hold(&self, other: &Rectangle) -> bool {
        self.width > other.width && self.height > other.height
    }
}
```

Listing 11-5: Using the `Rectangle` struct and its `can_hold` method from Chapter 5

The `can_hold` method returns a Boolean, which means it’s a perfect use case for the `assert!` macro. In Listing 11-6, we write a test that exercises the `can_hold` method by creating a `Rectangle` instance that has a width of 8 and a height of 7 and asserting that it can hold another `Rectangle` instance that has a width of 5 and a height of 1.

Filename: src/lib.rs

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn larger_can_hold_smaller() {
        let larger = Rectangle {
            width: 8,
            height: 7,
        };
        let smaller = Rectangle {
            width: 5,
            height: 1,
        };

        assert!(larger.can_hold(&smaller));
    }
}
```

Listing 11-6: A test for `can_hold` that checks whether a larger rectangle can indeed hold a smaller rectangle

**Note that we’ve added a new line inside the `tests` module: `use super::*;`**. The `tests` module is a regular module that follows the usual visibility rules we covered in Chapter 7 in the [“Paths for Referring to an Item in the Module Tree”](https://doc.rust-lang.org/book/ch07-03-paths-for-referring-to-an-item-in-the-module-tree.html) section. Because the `tests` module is an inner module, we need to bring the code under test in the outer module into the scope of the inner module. We use a glob here so anything we define in the outer module is available to this `tests` module.

We’ve named our test `larger_can_hold_smaller`, and we’ve created the two `Rectangle` instances that we need. Then we called the `assert!` macro and passed it the result of calling `larger.can_hold(&smaller)`. This expression is supposed to return `true`, so our test should pass. Let’s find out!

```console
$ cargo test
   Compiling rectangle v0.1.0 (file:///projects/rectangle)
    Finished test [unoptimized + debuginfo] target(s) in 0.66s
     Running unittests (target/debug/deps/rectangle-6584c4561e48942e)

running 1 test
test tests::larger_can_hold_smaller ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests rectangle

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

It does pass! Let’s add another test, this time asserting that a smaller rectangle cannot hold a larger rectangle:

Filename: src/lib.rs

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn larger_can_hold_smaller() {
        // --snip--
    }

    #[test]
    fn smaller_cannot_hold_larger() {
        let larger = Rectangle {
            width: 8,
            height: 7,
        };
        let smaller = Rectangle {
            width: 5,
            height: 1,
        };

        assert!(!smaller.can_hold(&larger));
    }
}
```

Because the correct result of the `can_hold` function in this case is `false`, we need to negate that result before we pass it to the `assert!` macro. As a result, our test will pass if `can_hold` returns `false`:

```console
$ cargo test
   Compiling rectangle v0.1.0 (file:///projects/rectangle)
    Finished test [unoptimized + debuginfo] target(s) in 0.66s
     Running unittests (target/debug/deps/rectangle-6584c4561e48942e)

running 2 tests
test tests::larger_can_hold_smaller ... ok
test tests::smaller_cannot_hold_larger ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests rectangle

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Two tests that pass! Now let’s see what happens to our test results when we introduce a bug in our code. Let’s change the implementation of the `can_hold` method by replacing the greater than sign with a less than sign when it compares the widths:

```rust
// --snip--
impl Rectangle {
    fn can_hold(&self, other: &Rectangle) -> bool {
        self.width < other.width && self.height > other.height
    }
}
```

Running the tests now produces the following:

```console
$ cargo test
   Compiling rectangle v0.1.0 (file:///projects/rectangle)
    Finished test [unoptimized + debuginfo] target(s) in 0.66s
     Running unittests (target/debug/deps/rectangle-6584c4561e48942e)

running 2 tests
test tests::larger_can_hold_smaller ... FAILED
test tests::smaller_cannot_hold_larger ... ok

failures:

---- tests::larger_can_hold_smaller stdout ----
thread 'main' panicked at 'assertion failed: larger.can_hold(&smaller)', src/lib.rs:28:9
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace


failures:
    tests::larger_can_hold_smaller

test result: FAILED. 1 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

error: test failed, to rerun pass '--lib'
```

Our tests caught the bug! Because `larger.width` is 8 and `smaller.width` is 5, the comparison of the widths in `can_hold` now returns `false`: 8 is not less than 5.

### Testing Equality with the `assert_eq!` and `assert_ne!` Macros

A common way to test functionality is to compare the result of the code under test to the value you expect the code to return to make sure they’re equal. You could do this using the `assert!` macro and passing it an expression using the `==` operator. However, this is such a common test that the standard library provides a pair of macros—**`assert_eq!` and `assert_ne!`—to perform this test more conveniently. These macros compare two arguments for equality or inequality, respectively. They’ll also print the two values if the assertion fails, which makes it easier to see *why* the test failed**; **conversely, the `assert!` macro only indicates that it got a `false` value for the `==` expression, not the values that led to the `false` value**.

In Listing 11-7, we write a function named `add_two` that adds `2` to its parameter and returns the result. Then we test this function using the `assert_eq!` macro.

Filename: src/lib.rs

```rust
pub fn add_two(a: i32) -> i32 {
    a + 2
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn it_adds_two() {
        assert_eq!(4, add_two(2));
    }
}
```

Listing 11-7: Testing the function `add_two` using the `assert_eq!` macro

Let’s check that it passes!

```console
$ cargo test
   Compiling adder v0.1.0 (file:///projects/adder)
    Finished test [unoptimized + debuginfo] target(s) in 0.58s
     Running unittests (target/debug/deps/adder-92948b65e88960b4)

running 1 test
test tests::it_adds_two ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests adder

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

The first argument we gave to the `assert_eq!` macro, `4`, is equal to the result of calling `add_two(2)`. The line for this test is `test tests::it_adds_two ... ok`, and the `ok` text indicates that our test passed!

Let’s introduce a bug into our code to see what it looks like when a test that uses `assert_eq!` fails. Change the implementation of the `add_two` function to instead add `3`:

```rust
pub fn add_two(a: i32) -> i32 {
    a + 3
}
```

Run the tests again:

```console
$ cargo test
   Compiling adder v0.1.0 (file:///projects/adder)
    Finished test [unoptimized + debuginfo] target(s) in 0.61s
     Running unittests (target/debug/deps/adder-92948b65e88960b4)

running 1 test
test tests::it_adds_two ... FAILED

failures:

---- tests::it_adds_two stdout ----
thread 'main' panicked at 'assertion failed: `(left == right)`
  left: `4`,
 right: `5`', src/lib.rs:11:9
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace


failures:
    tests::it_adds_two

test result: FAILED. 0 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

error: test failed, to rerun pass '--lib'
```

Note that in some languages and test frameworks, the parameters to the functions that assert two values are equal are called `expected` and `actual`, and the order in which we specify the arguments matters. **However, in Rust, they’re called `left` and `right`, and the order in which we specify the value we expect and the value that the code under test produces doesn’t matter**. We could write the assertion in this test as `assert_eq!(add_two(2), 4)`, which would result in a failure message that displays `assertion failed: (left == right) and that left was 5 and right was 4`.

The `assert_ne!` macro will pass if the two values we give it are not equal and fail if they’re equal. This macro is most useful for cases when we’re not sure what a value *will* be, but we know what the value definitely *won’t* be if our code is functioning as we intend. 

**Under the surface, the `assert_eq!` and `assert_ne!` macros use the operators `==` and `!=`, respectively. When the assertions fail, these macros print their arguments using debug formatting, which means the values being compared must implement the `PartialEq` and `Debug` traits. All the primitive types and most of the standard library types implement these traits**. For structs and enums that you define, you’ll need to implement `PartialEq` to assert that values of those types are equal or not equal. You’ll need to implement `Debug` to print the values when the assertion fails. Because both traits are derivable traits, as mentioned in Listing 5-12 in Chapter 5, this is usually as straightforward as adding the `#[derive(PartialEq, Debug)]` annotation to your struct or enum definition. See Appendix C, [“Derivable Traits,”](https://doc.rust-lang.org/book/appendix-03-derivable-traits.html) for more details about these and other derivable traits.

### Adding Custom Failure Messages

**You can also add a custom message to be printed with the failure message as optional arguments to the `assert!`, `assert_eq!`, and `assert_ne!` macros**. Any arguments specified after the one required argument to `assert!` or the two required arguments to `assert_eq!` and `assert_ne!` are passed along to the `format!` macro (discussed in Chapter 8 in the [“Concatenation with the `+` Operator or the `format!` Macro”](https://doc.rust-lang.org/book/ch08-02-strings.html#concatenation-with-the--operator-or-the-format-macro) section), so you can pass a format string that contains `{}` placeholders and values to go in those placeholders. Custom messages are useful to document what an assertion means; when a test fails, you’ll have a better idea of what the problem is with the code.

For example, let’s say we have a function that greets people by name and we want to test that the name we pass into the function appears in the output:

Filename: src/lib.rs

```rust
pub fn greeting(name: &str) -> String {
    format!("Hello {}!", name)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn greeting_contains_name() {
        let result = greeting("Carol");
        assert!(result.contains("Carol"));
    }
}
```

The requirements for this program haven’t been agreed upon yet, and we’re pretty sure the `Hello` text at the beginning of the greeting will change. We decided we don’t want to have to update the test when the requirements change, so instead of checking for exact equality to the value returned from the `greeting` function, we’ll just assert that the output contains the text of the input parameter.

Let’s introduce a bug into this code by changing `greeting` to not include `name` to see what this test failure looks like:

```rust
pub fn greeting(name: &str) -> String {
    String::from("Hello!")
}
```

Running this test produces the following:

```console
$ cargo test
   Compiling greeter v0.1.0 (file:///projects/greeter)
    Finished test [unoptimized + debuginfo] target(s) in 0.91s
     Running unittests (target/debug/deps/greeter-170b942eb5bf5e3a)

running 1 test
test tests::greeting_contains_name ... FAILED

failures:

---- tests::greeting_contains_name stdout ----
thread 'main' panicked at 'assertion failed: result.contains(\"Carol\")', src/lib.rs:12:9
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace


failures:
    tests::greeting_contains_name

test result: FAILED. 0 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

error: test failed, to rerun pass '--lib'
```

This result just indicates that the assertion failed and which line the assertion is on. A more useful failure message in this case would print the value we got from the `greeting` function. Let’s change the test function, giving it a custom failure message made from a format string with a placeholder filled in with the actual value we got from the `greeting` function:

```rust
    #[test]
    fn greeting_contains_name() {
        let result = greeting("Carol");
        assert!(
            result.contains("Carol"),
            "Greeting did not contain name, value was `{}`",
            result
        );
    }
```

Now when we run the test, we’ll get a more informative error message:

```console
$ cargo test
   Compiling greeter v0.1.0 (file:///projects/greeter)
    Finished test [unoptimized + debuginfo] target(s) in 0.93s
     Running unittests (target/debug/deps/greeter-170b942eb5bf5e3a)

running 1 test
test tests::greeting_contains_name ... FAILED

failures:

---- tests::greeting_contains_name stdout ----
thread 'main' panicked at 'Greeting did not contain name, value was `Hello!`', src/lib.rs:12:9
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace


failures:
    tests::greeting_contains_name

test result: FAILED. 0 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

error: test failed, to rerun pass '--lib'
```

We can see the value we actually got in the test output, which would help us debug what happened instead of what we were expecting to happen.

### Checking for Panics with `should_panic`

In addition to checking that our code returns the correct values we expect, it’s also important to check that our code handles error conditions as we expect. For example, consider the `Guess` type that we created in Chapter 9, Listing 9-13. Other code that uses `Guess` depends on the guarantee that `Guess` instances will contain only values between 1 and 100. We can write a test that ensures that attempting to create a `Guess` instance with a value outside that range panics.

**We do this by adding another attribute, `should_panic`, to our test function. This attribute makes a test pass if the code inside the function panics; the test will fail if the code inside the function doesn’t panic**.

Listing 11-8 shows a test that checks that the error conditions of `Guess::new` happen when we expect them to.

Filename: src/lib.rs

```rust
pub struct Guess {
    value: i32,
}

impl Guess {
    pub fn new(value: i32) -> Guess {
        if value < 1 || value > 100 {
            panic!("Guess value must be between 1 and 100, got {}.", value);
        }

        Guess { value }
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    #[should_panic]
    fn greater_than_100() {
        Guess::new(200);
    }
}
```

Listing 11-8: Testing that a condition will cause a `panic!`

We place the `#[should_panic]` attribute after the `#[test]` attribute and before the test function it applies to. Let’s look at the result when this test passes:

```console
$ cargo test
   Compiling guessing_game v0.1.0 (file:///projects/guessing_game)
    Finished test [unoptimized + debuginfo] target(s) in 0.58s
     Running unittests (target/debug/deps/guessing_game-57d70c3acb738f4d)

running 1 test
test tests::greater_than_100 - should panic ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests guessing_game

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Looks good! Now let’s introduce a bug in our code by removing the condition that the `new` function will panic if the value is greater than 100:

```rust
// --snip--
impl Guess {
    pub fn new(value: i32) -> Guess {
        if value < 1 {
            panic!("Guess value must be between 1 and 100, got {}.", value);
        }

        Guess { value }
    }
}
```

When we run the test in Listing 11-8, it will fail:

```console
$ cargo test
   Compiling guessing_game v0.1.0 (file:///projects/guessing_game)
    Finished test [unoptimized + debuginfo] target(s) in 0.62s
     Running unittests (target/debug/deps/guessing_game-57d70c3acb738f4d)

running 1 test
test tests::greater_than_100 - should panic ... FAILED

failures:

---- tests::greater_than_100 stdout ----
note: test did not panic as expected

failures:
    tests::greater_than_100

test result: FAILED. 0 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

error: test failed, to rerun pass '--lib'
```

We don’t get a very helpful message in this case, but when we look at the test function, we see that it’s annotated with `#[should_panic]`. The failure we got means that the code in the test function did not cause a panic.

Tests that use `should_panic` can be imprecise because they only indicate that the code has caused some panic. A `should_panic` test would pass even if the test panics for a different reason from the one we were expecting to happen. **To make `should_panic` tests more precise, we can add an optional `expected` parameter to the `should_panic` attribute. The test harness will make sure that the failure message contains the provided text**. For example, consider the modified code for `Guess` in Listing 11-9 where the `new` function panics with different messages depending on whether the value is too small or too large.

Filename: src/lib.rs

```rust
// --snip--
impl Guess {
    pub fn new(value: i32) -> Guess {
        if value < 1 {
            panic!(
                "Guess value must be greater than or equal to 1, got {}.",
                value
            );
        } else if value > 100 {
            panic!(
                "Guess value must be less than or equal to 100, got {}.",
                value
            );
        }

        Guess { value }
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    #[should_panic(expected = "Guess value must be less than or equal to 100")]
    fn greater_than_100() {
        Guess::new(200);
    }
}
```

Listing 11-9: Testing that a condition will cause a `panic!` with a particular panic message

This test will pass because the value we put in the `should_panic` attribute’s `expected` parameter is a substring of the message that the `Guess::new` function panics with. We could have specified the entire panic message that we expect, which in this case would be `Guess value must be less than or equal to 100, got 200.` What you choose to specify in the expected parameter for `should_panic` depends on how much of the panic message is unique or dynamic and how precise you want your test to be. In this case, a substring of the panic message is enough to ensure that the code in the test function executes the `else if value > 100` case.

To see what happens when a `should_panic` test with an `expected` message fails, let’s again introduce a bug into our code by swapping the bodies of the `if value < 1` and the `else if value > 100` blocks:

```rust
        if value < 1 {
            panic!(
                "Guess value must be less than or equal to 100, got {}.",
                value
            );
        } else if value > 100 {
            panic!(
                "Guess value must be greater than or equal to 1, got {}.",
                value
            );
        }
```

This time when we run the `should_panic` test, it will fail:

```console
$ cargo test
   Compiling guessing_game v0.1.0 (file:///projects/guessing_game)
    Finished test [unoptimized + debuginfo] target(s) in 0.66s
     Running unittests (target/debug/deps/guessing_game-57d70c3acb738f4d)

running 1 test
test tests::greater_than_100 - should panic ... FAILED

failures:

---- tests::greater_than_100 stdout ----
thread 'main' panicked at 'Guess value must be greater than or equal to 1, got 200.', src/lib.rs:13:13
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
note: panic did not contain expected string
      panic message: `"Guess value must be greater than or equal to 1, got 200."`,
 expected substring: `"Guess value must be less than or equal to 100"`

failures:
    tests::greater_than_100

test result: FAILED. 0 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

error: test failed, to rerun pass '--lib'
```

The failure message indicates that this test did indeed panic as we expected, but the panic message did not include the expected string `'Guess value must be less than or equal to 100'`. The panic message that we did get in this case was `Guess value must be greater than or equal to 1, got 200.` Now we can start figuring out where our bug is!

### Using `Result` in Tests

So far, we’ve written tests that panic when they fail. We can also write tests that use `Result<T, E>`! Here’s the test from Listing 11-1, rewritten to use `Result<T, E>` and return an `Err` instead of panicking:

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn it_works() -> Result<(), String> {
        if 2 + 2 == 4 {
            Ok(())
        } else {
            Err(String::from("two plus two does not equal four"))
        }
    }
}
```

The `it_works` function now has a return type, `Result<(), String>`. In the body of the function, rather than calling the `assert_eq!` macro, we return `Ok(())` when the test passes and an `Err` with a `String` inside when the test fails.

Writing tests so they return a `Result<T, E>` enables you to use the question mark operator in the body of tests, which can be a convenient way to write tests that should fail if any operation within them returns an `Err` variant.

**You can’t use the `#[should_panic]` annotation on tests that use `Result<T, E>`. Instead, you should return an `Err` value directly when the test should fail.**

Now that you know several ways to write tests, let’s look at what is happening when we run our tests and explore the different options we can use with `cargo test`.

## 11.2 Controlling How Tests Are Run

Just as `cargo run` compiles your code and then runs the resulting binary, `cargo test` compiles your code in test mode and runs the resulting test binary. You can specify command line options to change the default behavior of `cargo test`. For example, the default behavior of the binary produced by `cargo test` is to **run all the tests in parallel** and capture output generated during test runs, preventing the output from being displayed and making it easier to read the output related to the test results.

Some command line options go to `cargo test`, and some go to the resulting test binary. To separate these two types of arguments, you list the arguments that go to `cargo test` followed by the separator `--` and then the ones that go to the test binary. **Running `cargo test --help` displays the options you can use with `cargo test`, and running `cargo test -- --help` displays the options you can use after the separator `--`.**

### Running Tests in Parallel or Consecutively

存在多个test函数时，默认多线程并行执行。

**When you run multiple tests, by default they run in parallel using threads**. This means the tests will finish running faster so you can get feedback quicker on whether or not your code is working. Because the tests are running at the same time, make sure your tests don’t depend on each other or on any shared state, including a shared environment, such as the current working directory or environment variables.

For example, say each of your tests runs some code that creates a file on disk named *test-output.txt* and writes some data to that file. Then each test reads the data in that file and asserts that the file contains a particular value, which is different in each test. Because the tests run at the same time, one test might overwrite the file between when another test writes and reads the file. The second test will then fail, not because the code is incorrect but because the tests have interfered with each other while running in parallel. One solution is to make sure each test writes to a different file; another solution is to run the tests one at a time.

如果不想并行执行test函数，可以指定执行test的线程数量为1

**If you don’t want to run the tests in parallel or if you want more fine-grained control over the number of threads used, you can send the `--test-threads` flag and the number of threads you want to use to the test binary**. Take a look at the following example:

```console
$ cargo test -- --test-threads=1
```

We set the number of test threads to `1`, telling the program not to use any parallelism. Running the tests using one thread will take longer than running them in parallel, but the tests won’t interfere with each other if they share state.

### Showing Function Output

默认情况下，如果test通过，则Rust test libray拦截其标准输出的内容（即标准输出中不显示原本该输出内容）。

**By default, if a test passes, Rust’s test library captures anything printed to standard output**. For example, if we call `println!` in a test and the test passes, we won’t see the `println!` output in the terminal; we’ll see only the line that indicates the test passed. If a test fails, we’ll see whatever was printed to standard output with the rest of the failure message.

As an example, Listing 11-10 has a silly function that prints the value of its parameter and returns 10, as well as a test that passes and a test that fails.

Filename: src/lib.rs

```rust
fn prints_and_returns_10(a: i32) -> i32 {
    println!("I got the value {}", a);
    10
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn this_test_will_pass() {
        let value = prints_and_returns_10(4);
        assert_eq!(10, value);
    }

    #[test]
    fn this_test_will_fail() {
        let value = prints_and_returns_10(8);
        assert_eq!(5, value);
    }
}
```

Listing 11-10: Tests for a function that calls `println!`

When we run these tests with `cargo test`, we’ll see the following output:

```console
$ cargo test
   Compiling silly-function v0.1.0 (file:///projects/silly-function)
    Finished test [unoptimized + debuginfo] target(s) in 0.58s
     Running unittests (target/debug/deps/silly_function-160869f38cff9166)

running 2 tests
test tests::this_test_will_fail ... FAILED
test tests::this_test_will_pass ... ok

failures:

---- tests::this_test_will_fail stdout ----
I got the value 8
thread 'main' panicked at 'assertion failed: `(left == right)`
  left: `5`,
 right: `10`', src/lib.rs:19:9
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace


failures:
    tests::this_test_will_fail

test result: FAILED. 1 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

error: test failed, to rerun pass '--lib'
```

Note that nowhere in this output do we see `I got the value 4`, which is what is printed when the test that passes runs. That output has been captured. The output from the test that failed, `I got the value 8`, appears in the section of the test summary output, which also shows the cause of the test failure.

如果即使test通过，也要正常显示标准输出内容，则使用指令`cargo test -- --show-output`

**If we want to see printed values for passing tests as well, we can tell Rust to also show the output of successful tests at the end with `--show-output`**.

```console
$ cargo test -- --show-output
```

When we run the tests in Listing 11-10 again with the `--show-output` flag, we see the following output:

```console
$ cargo test -- --show-output
   Compiling silly-function v0.1.0 (file:///projects/silly-function)
    Finished test [unoptimized + debuginfo] target(s) in 0.60s
     Running unittests (target/debug/deps/silly_function-160869f38cff9166)

running 2 tests
test tests::this_test_will_fail ... FAILED
test tests::this_test_will_pass ... ok

successes:

---- tests::this_test_will_pass stdout ----
I got the value 4


successes:
    tests::this_test_will_pass

failures:

---- tests::this_test_will_fail stdout ----
I got the value 8
thread 'main' panicked at 'assertion failed: `(left == right)`
  left: `5`,
 right: `10`', src/lib.rs:19:9
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace


failures:
    tests::this_test_will_fail

test result: FAILED. 1 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

error: test failed, to rerun pass '--lib'
```

### Running a Subset of Tests by Name

Sometimes, running a full test suite can take a long time. If you’re working on code in a particular area, you might want to run only the tests pertaining to that code. You can choose which tests to run by passing `cargo test` the name or names of the test(s) you want to run as an argument.

To demonstrate how to run a subset of tests, we’ll create three tests for our `add_two` function, as shown in Listing 11-11, and choose which ones to run.

Filename: src/lib.rs

```rust
pub fn add_two(a: i32) -> i32 {
    a + 2
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn add_two_and_two() {
        assert_eq!(4, add_two(2));
    }

    #[test]
    fn add_three_and_two() {
        assert_eq!(5, add_two(3));
    }

    #[test]
    fn one_hundred() {
        assert_eq!(102, add_two(100));
    }
}
```

Listing 11-11: Three tests with three different names

If we run the tests without passing any arguments, as we saw earlier, all the tests will run in parallel:

```console
$ cargo test
   Compiling adder v0.1.0 (file:///projects/adder)
    Finished test [unoptimized + debuginfo] target(s) in 0.62s
     Running unittests (target/debug/deps/adder-92948b65e88960b4)

running 3 tests
test tests::add_three_and_two ... ok
test tests::add_two_and_two ... ok
test tests::one_hundred ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests adder

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

#### Running Single Tests

**We can pass the name of any test function to `cargo test` to run only that test**:

```console
$ cargo test one_hundred
   Compiling adder v0.1.0 (file:///projects/adder)
    Finished test [unoptimized + debuginfo] target(s) in 0.69s
     Running unittests (target/debug/deps/adder-92948b65e88960b4)

running 1 test
test tests::one_hundred ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 2 filtered out; finished in 0.00s
```

Only the test with the name `one_hundred` ran; the other two tests didn’t match that name. The test output lets us know we had more tests than what this command ran by displaying `2 filtered out` at the end of the summary line.

We can’t specify the names of multiple tests in this way; only the first value given to `cargo test` will be used. But there is a way to run multiple tests.

#### Filtering to Run Multiple Tests

We can specify part of a test name, and any test whose name matches that value will be run. For example, because two of our tests’ names contain `add`, we can run those two by running `cargo test add`:

```console
$ cargo test add
   Compiling adder v0.1.0 (file:///projects/adder)
    Finished test [unoptimized + debuginfo] target(s) in 0.61s
     Running unittests (target/debug/deps/adder-92948b65e88960b4)

running 2 tests
test tests::add_three_and_two ... ok
test tests::add_two_and_two ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 1 filtered out; finished in 0.00s
```

This command ran all tests with `add` in the name and filtered out the test named `one_hundred`. Also note that the module in which a test appears becomes part of the test’s name, so we can run all the tests in a module by filtering on the module’s name.

### Ignoring Some Tests Unless Specifically Requested

Sometimes a few specific tests can be very time-consuming to execute, so you might want to exclude them during most runs of `cargo test`. Rather than listing as arguments all tests you do want to run, **you can instead annotate the time-consuming tests using the `ignore` attribute to exclude them, as shown here**:

Filename: src/lib.rs

```rust
#[test]
fn it_works() {
    assert_eq!(2 + 2, 4);
}

#[test]
#[ignore]
fn expensive_test() {
    // code that takes an hour to run
}
```

**After `#[test]` we add the `#[ignore]` line to the test we want to exclude**. Now when we run our tests, `it_works` runs, but `expensive_test` doesn’t:

```console
$ cargo test
   Compiling adder v0.1.0 (file:///projects/adder)
    Finished test [unoptimized + debuginfo] target(s) in 0.60s
     Running unittests (target/debug/deps/adder-92948b65e88960b4)

running 2 tests
test expensive_test ... ignored
test it_works ... ok

test result: ok. 1 passed; 0 failed; 1 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests adder

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

The `expensive_test` function is listed as `ignored`. **If we want to run only the ignored tests, we can use `cargo test -- --ignored`**:

```console
$ cargo test -- --ignored
   Compiling adder v0.1.0 (file:///projects/adder)
    Finished test [unoptimized + debuginfo] target(s) in 0.61s
     Running unittests (target/debug/deps/adder-92948b65e88960b4)

running 1 test
test expensive_test ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 1 filtered out; finished in 0.00s

   Doc-tests adder

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

By controlling which tests run, you can make sure your `cargo test` results will be fast. When you’re at a point where it makes sense to check the results of the `ignored` tests and you have time to wait for the results, you can run `cargo test -- --ignored` instead.

## 11.3 Test Organization

As mentioned at the start of the chapter, testing is a complex discipline, and different people use different terminology and organization. The Rust community thinks about tests in terms of two main categories: ***unit tests* and *integration tests***. 

+ Unit tests are small and more focused, testing one module in isolation at a time, and **can test private interfaces**. 
+ Integration tests are entirely external to your library and use your code in the same way any other external code would, using only the public interface and potentially exercising multiple modules per test.

Writing both kinds of tests is important to ensure that the pieces of your library are doing what you expect them to, separately and together.

### Unit Tests

The purpose of unit tests is to test each unit of code in isolation from the rest of the code to quickly pinpoint where code is and isn’t working as expected. <u>You’ll put unit tests in the *src* directory in each file with the code that they’re testing. The convention is to create a module named `tests` in each file to contain the test functions and to annotate the module with `cfg(test)`.</u>

#### The Tests Module and `#[cfg(test)]`

**The `#[cfg(test)]` annotation on the tests module tells Rust to compile and run the test code only when you run `cargo test`, not when you run `cargo build`**. This saves compile time when you only want to build the library and saves space in the resulting compiled artifact because the tests are not included. You’ll see that <u>because integration tests go in a different directory, they don’t need the `#[cfg(test)]` annotation. However, because unit tests go in the same files as the code, you’ll use `#[cfg(test)]` to specify that they shouldn’t be included in the compiled result</u>.

Recall that when we generated the new `adder` project in the first section of this chapter, Cargo generated this code for us:

Filename: src/lib.rs

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn it_works() {
        assert_eq!(2 + 2, 4);
    }
}
```

This code is the automatically generated test module. **The attribute `cfg` stands for *configuration* and tells Rust that the following item should only be included given a certain configuration option**. In this case, the configuration option is `test`, which is provided by Rust for compiling and running tests. By using the `cfg` attribute, Cargo compiles our test code only if we actively run the tests with `cargo test`. This includes any helper functions that might be within this module, in addition to the functions annotated with `#[test]`.

#### Testing Private Functions

There’s debate within the testing community about whether or not private functions should be tested directly, and other languages make it difficult or impossible to test private functions. Regardless of which testing ideology you adhere to, **Rust’s privacy rules do allow you to test private functions**. Consider the code in Listing 11-12 with the private function `internal_adder`.

Filename: src/lib.rs

```rust
pub fn add_two(a: i32) -> i32 {
    internal_adder(a, 2)
}

fn internal_adder(a: i32, b: i32) -> i32 {
    a + b
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn internal() {
        assert_eq!(4, internal_adder(2, 2));
    }
}
```

Listing 11-12: Testing a private function

Note that the `internal_adder` function is not marked as `pub`. Tests are just Rust code, and the `tests` module is just another module. As we discussed in the [“Paths for Referring to an Item in the Module Tree”](https://doc.rust-lang.org/book/ch07-03-paths-for-referring-to-an-item-in-the-module-tree.html) section, items in child modules can use the items in their ancestor modules. In this test, we bring all of the `test` module’s parent’s items into scope with `use super::*`, and then the test can call `internal_adder`. If you don’t think private functions should be tested, there’s nothing in Rust that will compel you to do so.

### Integration Tests

**In Rust, integration tests are entirely external to your library**. <u>They use your library in the same way any other code would, which means they can only call functions that are part of your library’s public API.</u> Their purpose is to test whether many parts of your library work together correctly. Units of code that work correctly on their own could have problems when integrated, so test coverage of the integrated code is important as well. To create integration tests, you first need a *tests* directory.

#### The *tests* Directory

<u>We create a *tests* directory at the top level of our project directory, next to *src*. Cargo knows to look for integration test files in this directory. We can then make as many test files as we want to in this directory, and Cargo will compile each of the files as an individual crate</u>.

Let’s create an integration test. With the code in Listing 11-12 still in the *src/lib.rs* file, make a *tests* directory, create a new file named *tests/integration_test.rs*, and enter the code in Listing 11-13.

Filename: tests/integration_test.rs

```rust
use adder;

#[test]
fn it_adds_two() {
    assert_eq!(4, adder::add_two(2));
}
```

Listing 11-13: An integration test of a function in the `adder` crate

**We’ve added `use adder` at the top of the code, which we didn’t need in the unit tests. The reason is that <u>each file in the `tests` directory is a separate crate, so we need to bring our library into each test crate’s scope</u>**.

**We don’t need to annotate any code in *tests/integration_test.rs* with `#[cfg(test)]`. Cargo treats the `tests` directory specially and compiles files in this directory only when we run `cargo test`**. Run `cargo test` now:

```console
$ cargo test
   Compiling adder v0.1.0 (file:///projects/adder)
    Finished test [unoptimized + debuginfo] target(s) in 1.31s
     Running unittests (target/debug/deps/adder-1082c4b063a8fbe6)

running 1 test
test tests::internal ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running tests/integration_test.rs (target/debug/deps/integration_test-1082c4b063a8fbe6)

running 1 test
test it_adds_two ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests adder

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

The three sections of output include the unit tests, the integration test, and the doc tests. The first section for the unit tests is the same as we’ve been seeing: one line for each unit test (one named `internal` that we added in Listing 11-12) and then a summary line for the unit tests.

The integration tests section starts with the line `Running target/debug/deps/integration_test-1082c4b063a8fbe6` (the hash at the end of your output will be different). Next, there is a line for each test function in that integration test and a summary line for the results of the integration test just before the `Doc-tests adder` section starts.

<u>Similarly to how adding more unit test functions adds more result lines to the unit tests section, adding more test functions to the integration test file adds more result lines to this integration test file’s section. Each integration test file has its own section, so if we add more files in the *tests* directory, there will be more integration test sections</u>.

We can still run a particular integration test function by specifying the test function’s name as an argument to `cargo test`. **To run all the tests in a particular integration test file, use the `--test` argument of `cargo test` followed by the name of the file**:

```console
$ cargo test --test integration_test
   Compiling adder v0.1.0 (file:///projects/adder)
    Finished test [unoptimized + debuginfo] target(s) in 0.64s
     Running tests/integration_test.rs (target/debug/deps/integration_test-82e7799c1bc62298)

running 1 test
test it_adds_two ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

This command runs only the tests in the *tests/integration_test.rs* file.

#### Submodules in Integration Tests

在`tests`目录下的测试文件各自有各自的crate。

As you add more integration tests, you might want to make more than one file in the *tests* directory to help organize them; for example, you can group the test functions by the functionality they’re testing. **As mentioned earlier, each file in the *tests* directory is compiled as its own separate crate**.

Treating each integration test file as its own crate is useful to create separate scopes that are more like the way end users will be using your crate. However, this means files in the *tests* directory don’t share the same behavior as files in *src* do, as you learned in Chapter 7 regarding how to separate code into modules and files.

The different behavior of files in the *tests* directory is most noticeable when you have a set of helper functions that would be useful in multiple integration test files and you try to follow the steps in the [“Separating Modules into Different Files”](https://doc.rust-lang.org/book/ch07-05-separating-modules-into-different-files.html) section of Chapter 7 to extract them into a common module. For example, if we create *tests/common.rs* and place a function named `setup` in it, we can add some code to `setup` that we want to call from multiple test functions in multiple test files:

Filename: tests/common.rs

```rust
pub fn setup() {
    // setup code specific to your library's tests would go here
}
```

When we run the tests again, we’ll see a new section in the test output for the *common.rs* file, even though this file doesn’t contain any test functions nor did we call the `setup` function from anywhere:

```console
$ cargo test
   Compiling adder v0.1.0 (file:///projects/adder)
    Finished test [unoptimized + debuginfo] target(s) in 0.89s
     Running unittests (target/debug/deps/adder-92948b65e88960b4)

running 1 test
test tests::internal ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running tests/common.rs (target/debug/deps/common-92948b65e88960b4)

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running tests/integration_test.rs (target/debug/deps/integration_test-92948b65e88960b4)

running 1 test
test it_adds_two ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests adder

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Having `common` appear in the test results with `running 0 tests` displayed for it is not what we wanted. We just wanted to share some code with the other integration test files.

**To avoid having `common` appear in the test output, instead of creating *tests/common.rs*, we’ll create *tests/common/mod.rs*. This is an alternate naming convention that Rust also understands. Naming the file this way tells Rust not to treat the `common` module as an integration test file**. When we move the `setup` function code into *tests/common/mod.rs* and delete the *tests/common.rs* file, the section in the test output will no longer appear. **Files in subdirectories of the *tests* directory don’t get compiled as separate crates or have sections in the test output**.

After we’ve created *tests/common/mod.rs*, we can use it from any of the integration test files as a module. Here’s an example of calling the `setup` function from the `it_adds_two` test in *tests/integration_test.rs*:

Filename: tests/integration_test.rs

```rust
use adder;

mod common;

#[test]
fn it_adds_two() {
    common::setup();
    assert_eq!(4, adder::add_two(2));
}
```

Note that the `mod common;` declaration is the same as the module declaration we demonstrated in Listing 7-21. Then in the test function, we can call the `common::setup()` function.

#### Integration Tests for Binary Crates

<u>If our project is a binary crate that only contains a *src/main.rs* file and doesn’t have a *src/lib.rs* file, we can’t create integration tests in the *tests* directory and bring functions defined in the *src/main.rs* file into scope with a `use` statement.</u> **<u>Only library crates expose functions that other crates can use; binary crates are meant to be run on their own</u>**.

This is one of the reasons Rust projects that provide a binary have a straightforward *src/main.rs* file that calls logic that lives in the *src/lib.rs* file. Using that structure, integration tests *can* test the library crate with `use` to make the important functionality available. <u>If the important functionality works, the small amount of code in the *src/main.rs* file will work as well, and that small amount of code doesn’t need to be tested.</u>

### Summary

Rust’s testing features provide a way to specify how code should function to ensure it continues to work as you expect, even as you make changes. 

+ Unit tests exercise different parts of a library separately and can test private implementation details. 
+ Integration tests check that many parts of the library work together correctly, and they use the library’s public API to test the code in the same way external code will use it. 

Even though Rust’s type system and ownership rules help prevent some kinds of bugs, tests are still important to reduce logic bugs having to do with how your code is expected to behave.

Let’s combine the knowledge you learned in this chapter and in previous chapters to work on a project!

# 12. An I/O Project: Building a Command Line Program

Rust’s speed, safety, single binary output, and cross-platform support make it an ideal language for creating command line tools, so for our project, we’ll make our own version of the classic command line tool `grep` (**g**lobally search a **r**egular **e**xpression and **p**rint). In the simplest use case, `grep` searches a specified file for a specified string. To do so, `grep` takes as its arguments a filename and a string. Then it reads the file, finds lines in that file that contain the string argument, and prints those lines.

Along the way, we’ll show how to make our command line tool use features of the terminal that many command line tools use. <u>We’ll read the value of an environment variable to allow the user to configure the behavior of our tool</u>. We’ll also print error messages to the standard error console stream (`stderr`) instead of standard output (`stdout`), so, for example, the user can redirect successful output to a file while still seeing error messages onscreen.

One Rust community member, Andrew Gallant, has already created a fully featured, very fast version of `grep`, called `ripgrep`. By comparison, our version of `grep` will be fairly simple, but this chapter will give you some of the background knowledge you need to understand a real-world project such as `ripgrep`.

Our `grep` project will combine a number of concepts you’ve learned so far:

- Organizing code (using what you learned about modules in [Chapter 7](https://doc.rust-lang.org/book/ch07-00-managing-growing-projects-with-packages-crates-and-modules.html))
- Using vectors and strings (collections, [Chapter 8](https://doc.rust-lang.org/book/ch08-00-common-collections.html))
- Handling errors ([Chapter 9](https://doc.rust-lang.org/book/ch09-00-error-handling.html))
- Using traits and lifetimes where appropriate ([Chapter 10](https://doc.rust-lang.org/book/ch10-00-generics.html))
- Writing tests ([Chapter 11](https://doc.rust-lang.org/book/ch11-00-testing.html))

We’ll also briefly introduce closures, iterators, and trait objects, which Chapters [13](https://doc.rust-lang.org/book/ch13-00-functional-features.html) and [17](https://doc.rust-lang.org/book/ch17-00-oop.html) will cover in detail.

## 12.1 Accepting Command Line Arguments

Let’s create a new project with, as always, `cargo new`. We’ll call our project `minigrep` to distinguish it from the `grep` tool that you might already have on your system.

```shell
$ cargo new minigrep
     Created binary (application) `minigrep` project
$ cd minigrep
```

The first task is to make `minigrep` accept its two command line arguments: the filename and a string to search for. That is, we want to be able to run our program with `cargo run`, a string to search for, and a path to a file to search in, like so:

```shell
$ cargo run searchstring example-filename.txt
```

Right now, the program generated by `cargo new` cannot process arguments we give it. Some existing libraries on [crates.io](https://crates.io/) can help with writing a program that accepts command line arguments, but because you’re just learning this concept, let’s implement this capability ourselves.

### Reading the Argument Values

To enable `minigrep` to read the values of command line arguments we pass to it, we’ll need a function provided in Rust’s standard library, which is `std::env::args`. This function returns an iterator of the command line arguments that were given to `minigrep`. We’ll cover iterators fully in [Chapter 13](https://doc.rust-lang.org/book/ch13-00-functional-features.html). For now, you only need to know two details about iterators: iterators produce a series of values, and we can call the `collect` method on an iterator to turn it into a collection, such as a vector, containing all the elements the iterator produces.

Use the code in Listing 12-1 to allow your `minigrep` program to read any command line arguments passed to it and then collect the values into a vector.

Filename: src/main.rs

```rust
use std::env;

fn main() {
    let args: Vec<String> = env::args().collect();
    println!("{:?}", args);
}
```

Listing 12-1: Collecting the command line arguments into a vector and printing them

First, we bring the `std::env` module into scope with a `use` statement so we can use its `args` function. Notice that the `std::env::args` function is nested in two levels of modules. As we discussed in [Chapter 7](https://doc.rust-lang.org/book/ch07-04-bringing-paths-into-scope-with-the-use-keyword.html#creating-idiomatic-use-paths), in cases where the desired function is nested in more than one module, it’s conventional to bring the parent module into scope rather than the function. By doing so, we can easily use other functions from `std::env`. It’s also less ambiguous than adding `use std::env::args` and then calling the function with just `args`, because `args` might easily be mistaken for a function that’s defined in the current module.

> **The `args` Function and Invalid Unicode**
>
> <u>Note that `std::env::args` will panic if any argument contains invalid Unicode</u>. If your program needs to accept arguments containing invalid Unicode, use `std::env::args_os` instead. That function returns an iterator that produces `OsString` values instead of `String` values. We’ve chosen to use `std::env::args` here for simplicity, because `OsString` values differ per platform and are more complex to work with than `String` values.

On the first line of `main`, we call `env::args`, and we immediately use `collect` to turn the iterator into a vector containing all the values produced by the iterator. We can use the `collect` function to create many kinds of collections, so we explicitly annotate the type of `args` to specify that we want a vector of strings. **Although we very rarely need to annotate types in Rust, `collect` is one function you do often need to annotate because Rust isn’t able to infer the kind of collection you want**.

Finally, we print the vector using the debug formatter, `:?`. Let’s try running the code first with no arguments and then with two arguments:

```console
$ cargo run
   Compiling minigrep v0.1.0 (file:///projects/minigrep)
    Finished dev [unoptimized + debuginfo] target(s) in 0.61s
     Running `target/debug/minigrep`
["target/debug/minigrep"]
$ cargo run needle haystack
   Compiling minigrep v0.1.0 (file:///projects/minigrep)
    Finished dev [unoptimized + debuginfo] target(s) in 1.57s
     Running `target/debug/minigrep needle haystack`
["target/debug/minigrep", "needle", "haystack"]
```

**Notice that the first value in the vector is `"target/debug/minigrep"`, which is the name of our binary**. <u>This matches the behavior of the arguments list in C, letting programs use the name by which they were invoked in their execution</u>. It’s often convenient to have access to the program name in case you want to print it in messages or change behavior of the program based on what command line alias was used to invoke the program. But for the purposes of this chapter, we’ll ignore it and save only the two arguments we need.

### Saving the Argument Values in Variables

Printing the value of the vector of arguments illustrated that the program is able to access the values specified as command line arguments. Now we need to save the values of the two arguments in variables so we can use the values throughout the rest of the program. We do that in Listing 12-2.

Filename: src/main.rs

```rust
use std::env;

fn main() {
    let args: Vec<String> = env::args().collect();

    let query = &args[1];
    let filename = &args[2];

    println!("Searching for {}", query);
    println!("In file {}", filename);
}
```

Listing 12-2: Creating variables to hold the query argument and filename argument

As we saw when we printed the vector, the program’s name takes up the first value in the vector at `args[0]`, so we’re starting at index `1`. The first argument `minigrep` takes is the string we’re searching for, so we put a reference to the first argument in the variable `query`. The second argument will be the filename, so we put a reference to the second argument in the variable `filename`.

We temporarily print the values of these variables to prove that the code is working as we intend. Let’s run this program again with the arguments `test` and `sample.txt`:

```console
$ cargo run test sample.txt
   Compiling minigrep v0.1.0 (file:///projects/minigrep)
    Finished dev [unoptimized + debuginfo] target(s) in 0.0s
     Running `target/debug/minigrep test sample.txt`
Searching for test
In file sample.txt
```

Great, the program is working! The values of the arguments we need are being saved into the right variables. Later we’ll add some error handling to deal with certain potential erroneous situations, such as when the user provides no arguments; for now, we’ll ignore that situation and work on adding file-reading capabilities instead.

## 12.2 Reading a File

Now we’ll add functionality to read the file that is specified in the `filename` command line argument. First, we need a sample file to test it with: the best kind of file to use to make sure `minigrep` is working is one with a small amount of text over multiple lines with some repeated words. Listing 12-3 has an Emily Dickinson poem that will work well! Create a file called *poem.txt* at the root level of your project, and enter the poem “I’m Nobody! Who are you?”

Filename: poem.txt

```text
I'm nobody! Who are you?
Are you nobody, too?
Then there's a pair of us - don't tell!
They'd banish us, you know.

How dreary to be somebody!
How public, like a frog
To tell your name the livelong day
To an admiring bog!
```

Listing 12-3: A poem by Emily Dickinson makes a good test case

With the text in place, edit *src/main.rs* and add code to read the file, as shown in Listing 12-4.

Filename: src/main.rs

```rust
use std::env;
use std::fs;

fn main() {
    // --snip--
    println!("In file {}", filename);

    let contents = fs::read_to_string(filename)
        .expect("Something went wrong reading the file");

    println!("With text:\n{}", contents);
}
```

Listing 12-4: Reading the contents of the file specified by the second argument

First, we add another `use` statement to bring in a relevant part of the standard library: we need `std::fs` to handle files.

In `main`, we’ve added a new statement: `fs::read_to_string` takes the `filename`, opens that file, and returns a `Result<String>` of the file’s contents.

After that statement, we’ve again added a temporary `println!` statement that prints the value of `contents` after the file is read, so we can check that the program is working so far.

Let’s run this code with any string as the first command line argument (because we haven’t implemented the searching part yet) and the *poem.txt* file as the second argument:

```console
$ cargo run the poem.txt
   Compiling minigrep v0.1.0 (file:///projects/minigrep)
    Finished dev [unoptimized + debuginfo] target(s) in 0.0s
     Running `target/debug/minigrep the poem.txt`
Searching for the
In file poem.txt
With text:
I'm nobody! Who are you?
Are you nobody, too?
Then there's a pair of us - don't tell!
They'd banish us, you know.

How dreary to be somebody!
How public, like a frog
To tell your name the livelong day
To an admiring bog!
```

Great! The code read and then printed the contents of the file. But the code has a few flaws. The `main` function has multiple responsibilities: generally, functions are clearer and easier to maintain if each function is responsible for only one idea. The other problem is that we’re not handling errors as well as we could. The program is still small, so these flaws aren’t a big problem, but as the program grows, it will be harder to fix them cleanly. It’s good practice to begin refactoring early on when developing a program, because it’s much easier to refactor smaller amounts of code. We’ll do that next.

## 12.3 Refactoring to Improve Modularity and Error Handling

To improve our program, we’ll fix four problems that have to do with the program’s structure and how it’s handling potential errors.

First, our `main` function now performs two tasks: it parses arguments and reads files. For such a small function, this isn’t a major problem. However, if we continue to grow our program inside `main`, the number of separate tasks the `main` function handles will increase. As a function gains responsibilities, it becomes more difficult to reason about, harder to test, and harder to change without breaking one of its parts. It’s best to separate functionality so each function is responsible for one task.

This issue also ties into the second problem: although `query` and `filename` are configuration variables to our program, variables like `contents` are used to perform the program’s logic. The longer `main` becomes, the more variables we’ll need to bring into scope; the more variables we have in scope, the harder it will be to keep track of the purpose of each. <u>It’s best to group the configuration variables into one structure to make their purpose clear.</u>

The third problem is that we’ve used `expect` to print an error message when reading the file fails, but the error message just prints `Something went wrong reading the file`. Reading a file can fail in a number of ways: for example, the file could be missing, or we might not have permission to open it. Right now, regardless of the situation, we’d print the `Something went wrong reading the file` error message, which wouldn’t give the user any information!

Fourth, we use `expect` repeatedly to handle different errors, and if the user runs our program without specifying enough arguments, they’ll get an `index out of bounds` error from Rust that doesn’t clearly explain the problem. It would be best if all the error-handling code were in one place so future maintainers had only one place to consult in the code if the error-handling logic needed to change. Having all the error-handling code in one place will also ensure that we’re printing messages that will be meaningful to our end users.

Let’s address these four problems by refactoring our project.

### Separation of Concerns for Binary Projects

**The organizational problem of allocating responsibility for multiple tasks to the `main` function is common to many binary projects**. As a result, the Rust community has developed a process to use as a guideline for splitting the separate concerns of a binary program when `main` starts getting large. The process has the following steps:

- Split your program into a *main.rs* and a *lib.rs* and move your program’s logic to *lib.rs*.
- As long as your command line parsing logic is small, it can remain in *main.rs*.
- **When the command line parsing logic starts getting complicated, extract it from *main.rs* and move it to *lib.rs*.**

The responsibilities that remain in the `main` function after this process should be limited to the following:

- **Calling the command line parsing logic with the argument values**
- **Setting up any other configuration**
- **Calling a `run` function in *lib.rs***
- **Handling the error if `run` returns an error**

This pattern is about separating concerns: *main.rs* handles running the program, and *lib.rs* handles all the logic of the task at hand. Because you can’t test the `main` function directly, this structure lets you test all of your program’s logic by moving it into functions in *lib.rs*. The only code that remains in *main.rs* will be small enough to verify its correctness by reading it. Let’s rework our program by following this process.

#### Extracting the Argument Parser

We’ll extract the functionality for parsing arguments into a function that `main` will call to prepare for moving the command line parsing logic to *src/lib.rs*. Listing 12-5 shows the new start of `main` that calls a new function `parse_config`, which we’ll define in *src/main.rs* for the moment.

Filename: src/main.rs

```rust
fn main() {
    let args: Vec<String> = env::args().collect();

    let (query, filename) = parse_config(&args);

    // --snip--
}

fn parse_config(args: &[String]) -> (&str, &str) {
    let query = &args[1];
    let filename = &args[2];

    (query, filename)
}
```

Listing 12-5: Extracting a `parse_config` function from `main`

We’re still collecting the command line arguments into a vector, but instead of assigning the argument value at index 1 to the variable `query` and the argument value at index 2 to the variable `filename` within the `main` function, we pass the whole vector to the `parse_config` function. The `parse_config` function then holds the logic that determines which argument goes in which variable and passes the values back to `main`. We still create the `query` and `filename` variables in `main`, but `main` no longer has the responsibility of determining how the command line arguments and variables correspond.

This rework may seem like overkill for our small program, but we’re refactoring in small, incremental steps. After making this change, run the program again to verify that the argument parsing still works. It’s good to check your progress often, to help identify the cause of problems when they occur.

#### Grouping Configuration Values

We can take another small step to improve the `parse_config` function further. At the moment, we’re returning a tuple, but then we immediately break that tuple into individual parts again. This is a sign that perhaps we don’t have the right abstraction yet.

Another indicator that shows there’s room for improvement is the `config` part of `parse_config`, which implies that the two values we return are related and are both part of one configuration value. We’re not currently conveying this meaning in the structure of the data other than by grouping the two values into a tuple; we could put the two values into one struct and give each of the struct fields a meaningful name. Doing so will make it easier for future maintainers of this code to understand how the different values relate to each other and what their purpose is.

> Note: Using primitive values when a complex type would be more appropriate is an anti-pattern known as *primitive obsession*.

Listing 12-6 shows the improvements to the `parse_config` function.

Filename: src/main.rs

```rust
fn main() {
    let args: Vec<String> = env::args().collect();

    let config = parse_config(&args);

    println!("Searching for {}", config.query);
    println!("In file {}", config.filename);

    let contents = fs::read_to_string(config.filename)
        .expect("Something went wrong reading the file");

    // --snip--
}

struct Config {
    query: String,
    filename: String,
}

fn parse_config(args: &[String]) -> Config {
    let query = args[1].clone();
    let filename = args[2].clone();

    Config { query, filename }
}
```

Listing 12-6: Refactoring `parse_config` to return an instance of a `Config` struct

We’ve added a struct named `Config` defined to have fields named `query` and `filename`. The signature of `parse_config` now indicates that it returns a `Config` value. In the body of `parse_config`, where we used to return string slices that reference `String` values in `args`, we now define `Config` to contain owned `String` values. The `args` variable in `main` is the owner of the argument values and is only letting the `parse_config` function borrow them, which means we’d violate Rust’s borrowing rules if `Config` tried to take ownership of the values in `args`.

注意这里用了clone方法，虽然损失了部分性能，但是不需要管理原来的引用的lifetime，换来简洁性，是值得的。

We could manage the `String` data in a number of different ways, but the easiest, though somewhat inefficient, route is to call the `clone` method on the values. This will make a full copy of the data for the `Config` instance to own, which takes more time and memory than storing a reference to the string data. <u>However, cloning the data also makes our code very straightforward because we don’t have to manage the lifetimes of the references</u>; **in this circumstance, giving up a little performance to gain simplicity is a worthwhile trade-off**.

> **The Trade-Offs of Using `clone`**
>
> There’s a tendency among many Rustaceans to avoid using `clone` to fix ownership problems because of its runtime cost. In [Chapter 13](https://doc.rust-lang.org/book/ch13-00-functional-features.html), you’ll learn how to use more efficient methods in this type of situation. But for now, it’s okay to copy a few strings to continue making progress because you’ll make these copies only once and your filename and query string are very small. **<u>It’s better to have a working program that’s a bit inefficient than to try to hyperoptimize code on your first pass</u>**. As you become more experienced with Rust, it’ll be easier to start with the most efficient solution, but for now, it’s perfectly acceptable to call `clone`.

We’ve updated `main` so it places the instance of `Config` returned by `parse_config` into a variable named `config`, and we updated the code that previously used the separate `query` and `filename` variables so it now uses the fields on the `Config` struct instead.

Now our code more clearly conveys that `query` and `filename` are related and that their purpose is to configure how the program will work. Any code that uses these values knows to find them in the `config` instance in the fields named for their purpose.

#### Creating a Constructor for `Config`

So far, we’ve extracted the logic responsible for parsing the command line arguments from `main` and placed it in the `parse_config` function. Doing so helped us to see that the `query` and `filename` values were related and that relationship should be conveyed in our code. We then added a `Config` struct to name the related purpose of `query` and `filename` and to be able to return the values’ names as struct field names from the `parse_config` function.

So now that the purpose of the `parse_config` function is to create a `Config` instance, we can change `parse_config` from a plain function to a function named `new` that is associated with the `Config` struct. Making this change will make the code more idiomatic. We can create instances of types in the standard library, such as `String`, by calling `String::new`. Similarly, by changing `parse_config` into a `new` function associated with `Config`, we’ll be able to create instances of `Config` by calling `Config::new`. Listing 12-7 shows the changes we need to make.

Filename: src/main.rs

```rust
fn main() {
    let args: Vec<String> = env::args().collect();

    let config = Config::new(&args);

    // --snip--
}

// --snip--

impl Config {
    fn new(args: &[String]) -> Config {
        let query = args[1].clone();
        let filename = args[2].clone();

        Config { query, filename }
    }
}
```

Listing 12-7: Changing `parse_config` into `Config::new`

We’ve updated `main` where we were calling `parse_config` to instead call `Config::new`. We’ve changed the name of `parse_config` to `new` and moved it within an `impl` block, which associates the `new` function with `Config`. Try compiling this code again to make sure it works.

### Fixing the Error Handling

Now we’ll work on fixing our error handling. Recall that attempting to access the values in the `args` vector at index 1 or index 2 will cause the program to panic if the vector contains fewer than three items. Try running the program without any arguments; it will look like this:

```console
$ cargo run
   Compiling minigrep v0.1.0 (file:///projects/minigrep)
    Finished dev [unoptimized + debuginfo] target(s) in 0.0s
     Running `target/debug/minigrep`
thread 'main' panicked at 'index out of bounds: the len is 1 but the index is 1', src/main.rs:27:21
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

The line `index out of bounds: the len is 1 but the index is 1` is an error message intended for programmers. It won’t help our end users understand what happened and what they should do instead. Let’s fix that now.

#### Improving the Error Message

In Listing 12-8, we add a check in the `new` function that will verify that the slice is long enough before accessing index 1 and 2. If the slice isn’t long enough, the program panics and displays a better error message than the `index out of bounds` message.

Filename: src/main.rs

```rust
    // --snip--
    fn new(args: &[String]) -> Config {
        if args.len() < 3 {
            panic!("not enough arguments");
        }
        // --snip--
```

Listing 12-8: Adding a check for the number of arguments

This code is similar to [the `Guess::new` function we wrote in Listing 9-10](https://doc.rust-lang.org/book/ch09-03-to-panic-or-not-to-panic.html#creating-custom-types-for-validation), where we called `panic!` when the `value` argument was out of the range of valid values. Instead of checking for a range of values here, we’re checking that the length of `args` is at least 3 and the rest of the function can operate under the assumption that this condition has been met. If `args` has fewer than three items, this condition will be true, and we call the `panic!` macro to end the program immediately.

With these extra few lines of code in `new`, let’s run the program without any arguments again to see what the error looks like now:

```console
$ cargo run
   Compiling minigrep v0.1.0 (file:///projects/minigrep)
    Finished dev [unoptimized + debuginfo] target(s) in 0.0s
     Running `target/debug/minigrep`
thread 'main' panicked at 'not enough arguments', src/main.rs:26:13
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

This output is better: we now have a reasonable error message. However, we also have extraneous information we don’t want to give to our users. Perhaps using the technique we used in Listing 9-13 isn’t the best to use here: a call to `panic!` is more appropriate for a programming problem than a usage problem, [as discussed in Chapter 9](https://doc.rust-lang.org/book/ch09-03-to-panic-or-not-to-panic.html#guidelines-for-error-handling). Instead, we can use the other technique you learned about in Chapter 9—[returning a `Result`](https://doc.rust-lang.org/book/ch09-02-recoverable-errors-with-result.html) that indicates either success or an error.

#### Returning a `Result` from `new` Instead of Calling `panic!`

We can instead return a `Result` value that will contain a `Config` instance in the successful case and will describe the problem in the error case. When `Config::new` is communicating to `main`, we can use the `Result` type to signal there was a problem. Then we can change `main` to convert an `Err` variant into a more practical error for our users without the surrounding text about `thread 'main'` and `RUST_BACKTRACE` that a call to `panic!` causes.

Listing 12-9 shows the changes we need to make to the return value of `Config::new` and the body of the function needed to return a `Result`. Note that this won’t compile until we update `main` as well, which we’ll do in the next listing.

Filename: src/main.rs

```rust
impl Config {
    fn new(args: &[String]) -> Result<Config, &str> {
        if args.len() < 3 {
            return Err("not enough arguments");
        }

        let query = args[1].clone();
        let filename = args[2].clone();

        Ok(Config { query, filename })
    }
}
```

Listing 12-9: Returning a `Result` from `Config::new`

Our `new` function now returns a `Result` with a `Config` instance in the success case and a `&str` in the error case.

We’ve made two changes in the body of the `new` function: instead of calling `panic!` when the user doesn’t pass enough arguments, we now return an `Err` value, and we’ve wrapped the `Config` return value in an `Ok`. These changes make the function conform to its new type signature.

Returning an `Err` value from `Config::new` allows the `main` function to handle the `Result` value returned from the `new` function and exit the process more cleanly in the error case.

#### Calling `Config::new` and Handling Errors

To handle the error case and print a user-friendly message, we need to update `main` to handle the `Result` being returned by `Config::new`, as shown in Listing 12-10. We’ll also take the responsibility of exiting the command line tool with a nonzero error code from `panic!` and implement it by hand. A nonzero exit status is a convention to signal to the process that called our program that the program exited with an error state.

Filename: src/main.rs

```rust
use std::process;

fn main() {
    let args: Vec<String> = env::args().collect();

    let config = Config::new(&args).unwrap_or_else(|err| {
        println!("Problem parsing arguments: {}", err);
        process::exit(1);
    });

    // --snip--
```

Listing 12-10: Exiting with an error code if creating a new `Config` fails

In this listing, we’ve used a method we haven’t covered in detail yet: `unwrap_or_else`, which is defined on `Result<T, E>` by the standard library. Using `unwrap_or_else` allows us to define some custom, non-`panic!` error handling. If the `Result` is an `Ok` value, this method’s behavior is similar to `unwrap`: it returns the inner value `Ok` is wrapping. However, if the value is an `Err` value, this method calls the code in the *closure*, which is an anonymous function we define and pass as an argument to `unwrap_or_else`. We’ll cover closures in more detail in [Chapter 13](https://doc.rust-lang.org/book/ch13-00-functional-features.html). For now, you just need to know that `unwrap_or_else` will pass the inner value of the `Err`, which in this case is the static string `"not enough arguments"` that we added in Listing 12-9, to our closure in the argument `err` that appears between the vertical pipes. The code in the closure can then use the `err` value when it runs.

We’ve added a new `use` line to bring `process` from the standard library into scope. The code in the closure that will be run in the error case is only two lines: we print the `err` value and then call `process::exit`. **The `process::exit` function will stop the program immediately and return the number that was passed as the exit status code**. This is similar to the `panic!`-based handling we used in Listing 12-8, but we no longer get all the extra output. Let’s try it:

```console
$ cargo run
   Compiling minigrep v0.1.0 (file:///projects/minigrep)
    Finished dev [unoptimized + debuginfo] target(s) in 0.48s
     Running `target/debug/minigrep`
Problem parsing arguments: not enough arguments
```

Great! This output is much friendlier for our users.

### Extracting Logic from `main`

Now that we’ve finished refactoring the configuration parsing, let’s turn to the program’s logic. As we stated in [“Separation of Concerns for Binary Projects”](https://doc.rust-lang.org/book/ch12-03-improving-error-handling-and-modularity.html#separation-of-concerns-for-binary-projects), we’ll extract a function named `run` that will hold all the logic currently in the `main` function that isn’t involved with setting up configuration or handling errors. When we’re done, `main` will be concise and easy to verify by inspection, and we’ll be able to write tests for all the other logic.

Listing 12-11 shows the extracted `run` function. For now, we’re just making the small, incremental improvement of extracting the function. We’re still defining the function in *src/main.rs*.

Filename: src/main.rs

```rust
fn main() {
    // --snip--

    println!("Searching for {}", config.query);
    println!("In file {}", config.filename);

    run(config);
}

fn run(config: Config) {
    let contents = fs::read_to_string(config.filename)
        .expect("Something went wrong reading the file");

    println!("With text:\n{}", contents);
}

// --snip--
```

Listing 12-11: Extracting a `run` function containing the rest of the program logic

The `run` function now contains all the remaining logic from `main`, starting from reading the file. The `run` function takes the `Config` instance as an argument.

#### Returning Errors from the `run` Function

With the remaining program logic separated into the `run` function, we can improve the error handling, as we did with `Config::new` in Listing 12-9. Instead of allowing the program to panic by calling `expect`, the `run` function will return a `Result<T, E>` when something goes wrong. This will let us further consolidate into `main` the logic around handling errors in a user-friendly way. Listing 12-12 shows the changes we need to make to the signature and body of `run`.

Filename: src/main.rs

```rust
use std::error::Error;

// --snip--

fn run(config: Config) -> Result<(), Box<dyn Error>> {
    let contents = fs::read_to_string(config.filename)?;

    println!("With text:\n{}", contents);

    Ok(())
}
```

Listing 12-12: Changing the `run` function to return `Result`

We’ve made three significant changes here. First, we changed the return type of the `run` function to `Result<(), Box<dyn Error>>`. This function previously returned the unit type, `()`, and we keep that as the value returned in the `Ok` case.

For the error type, we used the *trait object* `Box<dyn Error>` (and we’ve brought `std::error::Error` into scope with a `use` statement at the top). We’ll cover trait objects in [Chapter 17](https://doc.rust-lang.org/book/ch17-00-oop.html). For now, just know that `Box<dyn Error>` means the function will return a type that implements the `Error` trait, but we don’t have to specify what particular type the return value will be. This gives us flexibility to return error values that may be of different types in different error cases. **The `dyn` keyword is short for “dynamic.**”

Second, we’ve removed the call to `expect` in favor of the `?` operator, as we talked about in [Chapter 9](https://doc.rust-lang.org/book/ch09-02-recoverable-errors-with-result.html#a-shortcut-for-propagating-errors-the--operator). Rather than `panic!` on an error, **`?` will return the error value from the current function for the caller to handle**.

Third, the `run` function now returns an `Ok` value in the success case. We’ve declared the `run` function’s success type as `()` in the signature, which means we need to wrap the unit type value in the `Ok` value. **This `Ok(())` syntax might look a bit strange at first, but using `()` like this is the idiomatic way to indicate that we’re calling `run` for its side effects only; it doesn’t return a value we need**.

When you run this code, it will compile but will display a warning:

```console
$ cargo run the poem.txt
   Compiling minigrep v0.1.0 (file:///projects/minigrep)
warning: unused `Result` that must be used
  --> src/main.rs:19:5
   |
19 |     run(config);
   |     ^^^^^^^^^^^^
   |
   = note: `#[warn(unused_must_use)]` on by default
   = note: this `Result` may be an `Err` variant, which should be handled

warning: `minigrep` (bin "minigrep") generated 1 warning
    Finished dev [unoptimized + debuginfo] target(s) in 0.71s
     Running `target/debug/minigrep the poem.txt`
Searching for the
In file poem.txt
With text:
I'm nobody! Who are you?
Are you nobody, too?
Then there's a pair of us - don't tell!
They'd banish us, you know.

How dreary to be somebody!
How public, like a frog
To tell your name the livelong day
To an admiring bog!
```

Rust tells us that our code ignored the `Result` value and the `Result` value might indicate that an error occurred. But we’re not checking to see whether or not there was an error, and the compiler reminds us that we probably meant to have some error-handling code here! Let’s rectify that problem now.

#### Handling Errors Returned from `run` in `main`

We’ll check for errors and handle them using a technique similar to one we used with `Config::new` in Listing 12-10, but with a slight difference:

Filename: src/main.rs

```rust
fn main() {
    // --snip--

    println!("Searching for {}", config.query);
    println!("In file {}", config.filename);

    if let Err(e) = run(config) {
        println!("Application error: {}", e);

        process::exit(1);
    }
}
```

We use `if let` rather than `unwrap_or_else` to check whether `run` returns an `Err` value and call `process::exit(1)` if it does. The `run` function doesn’t return a value that we want to `unwrap` in the same way that `Config::new` returns the `Config` instance. **Because `run` returns `()` in the success case, we only care about detecting an error, so we don’t need `unwrap_or_else` to return the unwrapped value because it would only be `()`.**

The bodies of the `if let` and the `unwrap_or_else` functions are the same in both cases: we print the error and exit.

### Splitting Code into a Library Crate

Our `minigrep` project is looking good so far! Now we’ll split the *src/main.rs* file and put some code into the *src/lib.rs* file so we can test it and have a *src/main.rs* file with fewer responsibilities.

Let’s move all the code that isn’t the `main` function from *src/main.rs* to *src/lib.rs*:

- The `run` function definition
- The relevant `use` statements
- The definition of `Config`
- The `Config::new` function definition

The contents of *src/lib.rs* should have the signatures shown in Listing 12-13 (we’ve omitted the bodies of the functions for brevity). Note that this won’t compile until we modify *src/main.rs* in Listing 12-14.

Filename: src/lib.rs

```rust
use std::error::Error;
use std::fs;

pub struct Config {
    pub query: String,
    pub filename: String,
}

impl Config {
    pub fn new(args: &[String]) -> Result<Config, &str> {
        // --snip--
    }
}

pub fn run(config: Config) -> Result<(), Box<dyn Error>> {
    // --snip--
}
```

Listing 12-13: Moving `Config` and `run` into *src/lib.rs*

We’ve made liberal use of the `pub` keyword: on `Config`, on its fields and its `new` method, and on the `run` function. We now have a library crate that has a public API that we can test!

Now we need to bring the code we moved to *src/lib.rs* into the scope of the binary crate in *src/main.rs*, as shown in Listing 12-14.

Filename: src/main.rs

```rust
use std::env;
use std::process;

use minigrep::Config;

fn main() {
    // --snip--
    if let Err(e) = minigrep::run(config) {
        // --snip--
    }
}
```

Listing 12-14: Using the `minigrep` library crate in *src/main.rs*

We add a `use minigrep::Config` line to bring the `Config` type from the library crate into the binary crate’s scope, and we prefix the `run` function with our crate name. Now all the functionality should be connected and should work. Run the program with `cargo run` and make sure everything works correctly.

Whew! That was a lot of work, but we’ve set ourselves up for success in the future. Now it’s much easier to handle errors, and we’ve made the code more modular. Almost all of our work will be done in *src/lib.rs* from here on out.

Let’s take advantage of this newfound modularity by doing something that would have been difficult with the old code but is easy with the new code: we’ll write some tests!

## 12.4 Developing the Library’s Functionality with Test-Driven Development

> [TDD（测试驱动开发(Test-Driven Development)）_百度百科 (baidu.com)](https://baike.baidu.com/item/TDD/9064369?fr=aladdin)

Now that we’ve extracted the logic into *src/lib.rs* and left the argument collecting and error handling in *src/main.rs*, it’s much easier to write tests for the core functionality of our code. We can call functions directly with various arguments and check return values without having to call our binary from the command line. Feel free to write some tests for the functionality in the `Config::new` and `run` functions on your own.

**In this section, we’ll add the searching logic to the `minigrep` program by using the Test-driven development (TDD) process**. This software development technique follows these steps:

1. **Write a test that fails and run it to make sure it fails for the reason you expect**.
2. **Write or modify just enough code to make the new test pass**.
3. **Refactor the code you just added or changed and make sure the tests continue to pass.**
4. Repeat from step 1!

This process is just one of many ways to write software, but TDD can help drive code design as well. Writing the test before you write the code that makes the test pass helps to maintain high test coverage throughout the process.

We’ll test drive the implementation of the functionality that will actually do the searching for the query string in the file contents and produce a list of lines that match the query. We’ll add this functionality in a function called `search`.

### Writing a Failing Test

Because we don’t need them anymore, let’s remove the `println!` statements from *src/lib.rs* and *src/main.rs* that we used to check the program’s behavior. Then, in *src/lib.rs*, we’ll add a `tests` module with a test function, as we did in [Chapter 11](https://doc.rust-lang.org/book/ch11-01-writing-tests.html#the-anatomy-of-a-test-function). The test function specifies the behavior we want the `search` function to have: it will take a query and the text to search for the query in, and it will return only the lines from the text that contain the query. Listing 12-15 shows this test, which won’t compile yet.

Filename: src/lib.rs

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn one_result() {
        let query = "duct";
        let contents = "\
Rust:
safe, fast, productive.
Pick three.";

        assert_eq!(vec!["safe, fast, productive."], search(query, contents));
    }
}
```

Listing 12-15: Creating a failing test for the `search` function we wish we had

This test searches for the string `"duct"`. The text we’re searching is three lines, only one of which contains `"duct"` (Note that the backslash after the opening double quote tells Rust not to put a newline character at the beginning of the contents of this string literal). We assert that the value returned from the `search` function contains only the line we expect.

We aren’t able to run this test and watch it fail because the test doesn’t even compile: the `search` function doesn’t exist yet! So now we’ll add just enough code to get the test to compile and run by adding a definition of the `search` function that always returns an empty vector, as shown in Listing 12-16. Then the test should compile and fail because an empty vector doesn’t match a vector containing the line `"safe, fast, productive."`

Filename: src/lib.rs

```rust
pub fn search<'a>(query: &str, contents: &'a str) -> Vec<&'a str> {
    vec![]
}
```

Listing 12-16: Defining just enough of the `search` function so our test will compile

<u>Notice that we need an explicit lifetime `'a` defined in the signature of `search` and used with the `contents` argument and the return value</u>. Recall in [Chapter 10](https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html) that the lifetime parameters specify which argument lifetime is connected to the lifetime of the return value. In this case, we indicate that the returned vector should contain string slices that reference slices of the argument `contents` (rather than the argument `query`).

<u>In other words, we tell Rust that the data returned by the `search` function will live as long as the data passed into the `search` function in the `contents` argument. This is important! The data referenced *by* a slice needs to be valid for the reference to be valid</u>; if the compiler assumes we’re making string slices of `query` rather than `contents`, it will do its safety checking incorrectly.

If we forget the lifetime annotations and try to compile this function, we’ll get this error:

```console
$ cargo build
   Compiling minigrep v0.1.0 (file:///projects/minigrep)
error[E0106]: missing lifetime specifier
  --> src/lib.rs:28:51
   |
28 | pub fn search(query: &str, contents: &str) -> Vec<&str> {
   |                      ----            ----         ^ expected named lifetime parameter
   |
   = help: this function's return type contains a borrowed value, but the signature does not say whether it is borrowed from `query` or `contents`
help: consider introducing a named lifetime parameter
   |
28 | pub fn search<'a>(query: &'a str, contents: &'a str) -> Vec<&'a str> {
   |              ^^^^        ^^^^^^^            ^^^^^^^         ^^^

For more information about this error, try `rustc --explain E0106`.
error: could not compile `minigrep` due to previous error
```

Rust can’t possibly know which of the two arguments we need, so we need to tell it. Because `contents` is the argument that contains all of our text and we want to return the parts of that text that match, we know `contents` is the argument that should be connected to the return value using the lifetime syntax.

Other programming languages don’t require you to connect arguments to return values in the signature. Although this might seem strange, it will get easier over time. You might want to compare this example with the [“Validating References with Lifetimes”](https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html#validating-references-with-lifetimes) section in Chapter 10.

Now let’s run the test:

```console
$ cargo test
   Compiling minigrep v0.1.0 (file:///projects/minigrep)
    Finished test [unoptimized + debuginfo] target(s) in 0.97s
     Running unittests (target/debug/deps/minigrep-9cd200e5fac0fc94)

running 1 test
test tests::one_result ... FAILED

failures:

---- tests::one_result stdout ----
thread 'main' panicked at 'assertion failed: `(left == right)`
  left: `["safe, fast, productive."]`,
 right: `[]`', src/lib.rs:44:9
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace


failures:
    tests::one_result

test result: FAILED. 0 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

error: test failed, to rerun pass '--lib'
```

Great, the test fails, exactly as we expected. Let’s get the test to pass!

### Writing Code to Pass the Test

Currently, our test is failing because we always return an empty vector. To fix that and implement `search`, our program needs to follow these steps:

- Iterate through each line of the contents.
- Check whether the line contains our query string.
- If it does, add it to the list of values we’re returning.
- If it doesn’t, do nothing.
- Return the list of results that match.

Let’s work through each step, starting with iterating through lines.

#### Iterating Through Lines with the `lines` Method

Rust has a helpful method to handle line-by-line iteration of strings, conveniently named `lines`, that works as shown in Listing 12-17. Note this won’t compile yet.

Filename: src/lib.rs

```rust
pub fn search<'a>(query: &str, contents: &'a str) -> Vec<&'a str> {
    for line in contents.lines() {
        // do something with line
    }
}
```

Listing 12-17: Iterating through each line in `contents`

The `lines` method returns an iterator. We’ll talk about iterators in depth in [Chapter 13](https://doc.rust-lang.org/book/ch13-02-iterators.html), but recall that you saw this way of using an iterator in [Listing 3-5](https://doc.rust-lang.org/book/ch03-05-control-flow.html#looping-through-a-collection-with-for), where we used a `for` loop with an iterator to run some code on each item in a collection.

#### Searching Each Line for the Query

Next, we’ll check whether the current line contains our query string. Fortunately, strings have a helpful method named `contains` that does this for us! Add a call to the `contains` method in the `search` function, as shown in Listing 12-18. Note this still won’t compile yet.

Filename: src/lib.rs

```rust
pub fn search<'a>(query: &str, contents: &'a str) -> Vec<&'a str> {
    for line in contents.lines() {
        if line.contains(query) {
            // do something with line
        }
    }
}
```

Listing 12-18: Adding functionality to see whether the line contains the string in `query`

#### Storing Matching Lines

We also need a way to store the lines that contain our query string. For that, we can make a mutable vector before the `for` loop and call the `push` method to store a `line` in the vector. After the `for` loop, we return the vector, as shown in Listing 12-19.

Filename: src/lib.rs

```rust
pub fn search<'a>(query: &str, contents: &'a str) -> Vec<&'a str> {
    let mut results = Vec::new();

    for line in contents.lines() {
        if line.contains(query) {
            results.push(line);
        }
    }

    results
}
```

Listing 12-19: Storing the lines that match so we can return them

Now the `search` function should return only the lines that contain `query`, and our test should pass. Let’s run the test:

```console
$ cargo test
   Compiling minigrep v0.1.0 (file:///projects/minigrep)
    Finished test [unoptimized + debuginfo] target(s) in 1.22s
     Running unittests (target/debug/deps/minigrep-9cd200e5fac0fc94)

running 1 test
test tests::one_result ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running unittests (target/debug/deps/minigrep-9cd200e5fac0fc94)

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests minigrep

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Our test passed, so we know it works!

At this point, we could consider opportunities for refactoring the implementation of the search function while keeping the tests passing to maintain the same functionality. The code in the search function isn’t too bad, but it doesn’t take advantage of some useful features of iterators. We’ll return to this example in [Chapter 13](https://doc.rust-lang.org/book/ch13-02-iterators.html), where we’ll explore iterators in detail, and look at how to improve it.

#### Using the `search` Function in the `run` Function

Now that the `search` function is working and tested, we need to call `search` from our `run` function. We need to pass the `config.query` value and the `contents` that `run` reads from the file to the `search` function. Then `run` will print each line returned from `search`:

Filename: src/lib.rs

```rust
pub fn run(config: Config) -> Result<(), Box<dyn Error>> {
    let contents = fs::read_to_string(config.filename)?;

    for line in search(&config.query, &contents) {
        println!("{}", line);
    }

    Ok(())
}
```

We’re still using a `for` loop to return each line from `search` and print it.

Now the entire program should work! Let’s try it out, first with a word that should return exactly one line from the Emily Dickinson poem, “frog”:

```console
$ cargo run frog poem.txt
   Compiling minigrep v0.1.0 (file:///projects/minigrep)
    Finished dev [unoptimized + debuginfo] target(s) in 0.38s
     Running `target/debug/minigrep frog poem.txt`
How public, like a frog
```

Cool! Now let’s try a word that will match multiple lines, like “body”:

```console
$ cargo run body poem.txt
   Compiling minigrep v0.1.0 (file:///projects/minigrep)
    Finished dev [unoptimized + debuginfo] target(s) in 0.0s
     Running `target/debug/minigrep body poem.txt`
I'm nobody! Who are you?
Are you nobody, too?
How dreary to be somebody!
```

And finally, let’s make sure that we don’t get any lines when we search for a word that isn’t anywhere in the poem, such as “monomorphization”:

```console
$ cargo run monomorphization poem.txt
   Compiling minigrep v0.1.0 (file:///projects/minigrep)
    Finished dev [unoptimized + debuginfo] target(s) in 0.0s
     Running `target/debug/minigrep monomorphization poem.txt`
```

Excellent! We’ve built our own mini version of a classic tool and learned a lot about how to structure applications. We’ve also learned a bit about file input and output, lifetimes, testing, and command line parsing.

To round out this project, we’ll briefly demonstrate how to work with environment variables and how to print to standard error, both of which are useful when you’re writing command line programs.

## 12.5 Working with Environment Variables

We’ll improve `minigrep` by adding an extra feature: an option for case-insensitive searching that the user can turn on via an environment variable. We could make this feature a command line option and require that users enter it each time they want it to apply, but instead we’ll use an environment variable. Doing so allows our users to set the environment variable once and have all their searches be case insensitive in that terminal session.

### Writing a Failing Test for the Case-Insensitive `search` Function

We want to add a new `search_case_insensitive` function that we’ll call when the environment variable is on. We’ll continue to follow the TDD process, so the first step is again to write a failing test. We’ll add a new test for the new `search_case_insensitive` function and rename our old test from `one_result` to `case_sensitive` to clarify the differences between the two tests, as shown in Listing 12-20.

Filename: src/lib.rs

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn case_sensitive() {
        let query = "duct";
        let contents = "\
Rust:
safe, fast, productive.
Pick three.
Duct tape.";

        assert_eq!(vec!["safe, fast, productive."], search(query, contents));
    }

    #[test]
    fn case_insensitive() {
        let query = "rUsT";
        let contents = "\
Rust:
safe, fast, productive.
Pick three.
Trust me.";

        assert_eq!(
            vec!["Rust:", "Trust me."],
            search_case_insensitive(query, contents)
        );
    }
}
```

Listing 12-20: Adding a new failing test for the case-insensitive function we’re about to add

Note that we’ve edited the old test’s `contents` too. We’ve added a new line with the text `"Duct tape."` using a capital D that shouldn’t match the query `"duct"` when we’re searching in a case-sensitive manner. Changing the old test in this way helps ensure that we don’t accidentally break the case-sensitive search functionality that we’ve already implemented. This test should pass now and should continue to pass as we work on the case-insensitive search.

The new test for the case-*insensitive* search uses `"rUsT"` as its query. In the `search_case_insensitive` function we’re about to add, the query `"rUsT"` should match the line containing `"Rust:"` with a capital R and match the line `"Trust me."` even though both have different casing from the query. This is our failing test, and it will fail to compile because we haven’t yet defined the `search_case_insensitive` function. Feel free to add a skeleton implementation that always returns an empty vector, similar to the way we did for the `search` function in Listing 12-16 to see the test compile and fail.

### Implementing the `search_case_insensitive` Function

The `search_case_insensitive` function, shown in Listing 12-21, will be almost the same as the `search` function. The only difference is that we’ll lowercase the `query` and each `line` so whatever the case of the input arguments, they’ll be the same case when we check whether the line contains the query.

Filename: src/lib.rs

```rust
pub fn search_case_insensitive<'a>(
    query: &str,
    contents: &'a str,
) -> Vec<&'a str> {
    let query = query.to_lowercase();
    let mut results = Vec::new();

    for line in contents.lines() {
        if line.to_lowercase().contains(&query) {
            results.push(line);
        }
    }

    results
}
```

Listing 12-21: Defining the `search_case_insensitive` function to lowercase the query and the line before comparing them

First, we lowercase the `query` string and store it in a shadowed variable with the same name. Calling `to_lowercase` on the query is necessary so no matter whether the user’s query is `"rust"`, `"RUST"`, `"Rust"`, or `"rUsT"`, we’ll treat the query as if it were `"rust"` and be insensitive to the case. <u>While `to_lowercase` will handle basic Unicode, it won’t be 100% accurate</u>. If we were writing a real application, we’d want to do a bit more work here, but this section is about environment variables, not Unicode, so we’ll leave it at that here.

Note that `query` is now a `String` rather than a string slice, because calling `to_lowercase` creates new data rather than referencing existing data. Say the query is `"rUsT"`, as an example: that string slice doesn’t contain a lowercase `u` or `t` for us to use, so we have to allocate a new `String` containing `"rust"`. When we pass `query` as an argument to the `contains` method now, we need to add an ampersand because the signature of `contains` is defined to take a string slice.

Next, we add a call to `to_lowercase` on each `line` before we check whether it contains `query` to lowercase all characters. Now that we’ve converted `line` and `query` to lowercase, we’ll find matches no matter what the case of the query is.

Let’s see if this implementation passes the tests:

```console
$ cargo test
   Compiling minigrep v0.1.0 (file:///projects/minigrep)
    Finished test [unoptimized + debuginfo] target(s) in 1.33s
     Running unittests (target/debug/deps/minigrep-9cd200e5fac0fc94)

running 2 tests
test tests::case_insensitive ... ok
test tests::case_sensitive ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running unittests (target/debug/deps/minigrep-9cd200e5fac0fc94)

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests minigrep

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Great! They passed. Now, let’s call the new `search_case_insensitive` function from the `run` function. First, we’ll add a configuration option to the `Config` struct to switch between case-sensitive and case-insensitive search. Adding this field will cause compiler errors because we aren’t initializing this field anywhere yet:

Filename: src/lib.rs

```rust
pub struct Config {
    pub query: String,
    pub filename: String,
    pub case_sensitive: bool,
}
```

Note that we added the `case_sensitive` field that holds a Boolean. Next, we need the `run` function to check the `case_sensitive` field’s value and use that to decide whether to call the `search` function or the `search_case_insensitive` function, as shown in Listing 12-22. Note this still won’t compile yet.

Filename: src/lib.rs

```rust
pub fn run(config: Config) -> Result<(), Box<dyn Error>> {
    let contents = fs::read_to_string(config.filename)?;

    let results = if config.case_sensitive {
        search(&config.query, &contents)
    } else {
        search_case_insensitive(&config.query, &contents)
    };

    for line in results {
        println!("{}", line);
    }

    Ok(())
}
```

Listing 12-22: Calling either `search` or `search_case_insensitive` based on the value in `config.case_sensitive`

Finally, we need to check for the environment variable. <u>The functions for working with environment variables are in the `env` module in the standard library, so we want to bring that module into scope with a `use std::env;` line at the top of *src/lib.rs*</u>. Then we’ll use the `var` function from the `env` module to check for an environment variable named `CASE_INSENSITIVE`, as shown in Listing 12-23.

Filename: src/lib.rs

```rust
use std::env;
// --snip--

impl Config {
    pub fn new(args: &[String]) -> Result<Config, &str> {
        if args.len() < 3 {
            return Err("not enough arguments");
        }

        let query = args[1].clone();
        let filename = args[2].clone();

        let case_sensitive = env::var("CASE_INSENSITIVE").is_err();

        Ok(Config {
            query,
            filename,
            case_sensitive,
        })
    }
}
```

Listing 12-23: Checking for an environment variable named `CASE_INSENSITIVE`

**Here, we create a new variable `case_sensitive`. To set its value, we call the `env::var` function and pass it the name of the `CASE_INSENSITIVE` environment variable**. The `env::var` function returns a `Result` that will be the successful `Ok` variant that contains the value of the environment variable if the environment variable is set. It will return the `Err` variant if the environment variable is not set.

**We’re using the `is_err` method on the `Result` to check whether it’s an error and therefore unset, which means it *should* do a case-sensitive search**. If the `CASE_INSENSITIVE` environment variable is set to anything, `is_err` will return false and the program will perform a case-insensitive search. We don’t care about the *value* of the environment variable, just whether it’s set or unset, so we’re checking `is_err` rather than using `unwrap`, `expect`, or any of the other methods we’ve seen on `Result`.

We pass the value in the `case_sensitive` variable to the `Config` instance so the `run` function can read that value and decide whether to call `search` or `search_case_insensitive`, as we implemented in Listing 12-22.

Let’s give it a try! First, we’ll run our program without the environment variable set and with the query `to`, which should match any line that contains the word “to” in all lowercase:

```console
$ cargo run to poem.txt
   Compiling minigrep v0.1.0 (file:///projects/minigrep)
    Finished dev [unoptimized + debuginfo] target(s) in 0.0s
     Running `target/debug/minigrep to poem.txt`
Are you nobody, too?
How dreary to be somebody!
```

Looks like that still works! Now, let’s run the program with `CASE_INSENSITIVE` set to `1` but with the same query `to`.

If you’re using PowerShell, you will need to set the environment variable and run the program as separate commands:

```console
PS> $Env:CASE_INSENSITIVE=1; cargo run to poem.txt
```

This will make `CASE_INSENSITIVE` persist for the remainder of your shell session. It can be unset with the `Remove-Item` cmdlet:

```console
PS> Remove-Item Env:CASE_INSENSITIVE
```

We should get lines that contain “to” that might have uppercase letters:

```console
$ CASE_INSENSITIVE=1 cargo run to poem.txt
    Finished dev [unoptimized + debuginfo] target(s) in 0.0s
     Running `target/debug/minigrep to poem.txt`
Are you nobody, too?
How dreary to be somebody!
To tell your name the livelong day
To an admiring bog!
```

Excellent, we also got lines containing “To”! Our `minigrep` program can now do case-insensitive searching controlled by an environment variable. Now you know how to manage options set using either command line arguments or environment variables.

Some programs allow arguments *and* environment variables for the same configuration. In those cases, the programs decide that one or the other takes precedence. For another exercise on your own, try controlling case insensitivity through either a command line argument or an environment variable. Decide whether the command line argument or the environment variable should take precedence if the program is run with one set to case sensitive and one set to case insensitive.

The `std::env` module contains many more useful features for dealing with environment variables: check out its documentation to see what is available.

## 12.6 Writing Error Messages to Standard Error Instead of Standard Output

At the moment, we’re writing all of our output to the terminal using the `println!` macro. **Most terminals provide two kinds of output: *standard output* (`stdout`) for general information and *standard error* (`stderr`) for error messages**. This distinction enables users to choose to direct the successful output of a program to a file but still print error messages to the screen.

<u>The `println!` macro is only capable of printing to standard output, so we have to use something else to print to standard error.</u>

### Checking Where Errors Are Written

First, let’s observe how the content printed by `minigrep` is currently being written to standard output, including any error messages we want to write to standard error instead. We’ll do that by redirecting the standard output stream to a file while also intentionally causing an error. We won’t redirect the standard error stream, so any content sent to standard error will continue to display on the screen.

Command line programs are expected to send error messages to the standard error stream so we can still see error messages on the screen even if we redirect the standard output stream to a file. Our program is not currently well-behaved: we’re about to see that it saves the error message output to a file instead!

The way to demonstrate this behavior is by running the program with `>` and the filename, *output.txt*, that we want to redirect the standard output stream to. We won’t pass any arguments, which should cause an error:

```console
$ cargo run > output.txt
```

The `>` syntax tells the shell to write the contents of standard output to *output.txt* instead of the screen. We didn’t see the error message we were expecting printed to the screen, so that means it must have ended up in the file. This is what *output.txt* contains:

```text
Problem parsing arguments: not enough arguments
```

Yup, our error message is being printed to standard output. It’s much more useful for error messages like this to be printed to standard error so only data from a successful run ends up in the file. We’ll change that.

### Printing Errors to Standard Error

We’ll use the code in Listing 12-24 to change how error messages are printed. Because of the refactoring we did earlier in this chapter, all the code that prints error messages is in one function, `main`. **The standard library provides the `eprintln!` macro that prints to the standard error stream**, so let’s change the two places we were calling `println!` to print errors to use `eprintln!` instead.

Filename: src/main.rs

```rust
fn main() {
    let args: Vec<String> = env::args().collect();

    let config = Config::new(&args).unwrap_or_else(|err| {
        eprintln!("Problem parsing arguments: {}", err);
        process::exit(1);
    });

    if let Err(e) = minigrep::run(config) {
        eprintln!("Application error: {}", e);

        process::exit(1);
    }
}
```

Listing 12-24: Writing error messages to standard error instead of standard output using `eprintln!`

After changing `println!` to `eprintln!`, let’s run the program again in the same way, without any arguments and redirecting standard output with `>`:

```console
$ cargo run > output.txt
Problem parsing arguments: not enough arguments
```

Now we see the error onscreen and *output.txt* contains nothing, which is the behavior we expect of command line programs.

Let’s run the program again with arguments that don’t cause an error but still redirect standard output to a file, like so:

```console
$ cargo run to poem.txt > output.txt
```

We won’t see any output to the terminal, and *output.txt* will contain our results:

Filename: output.txt

```text
Are you nobody, too?
How dreary to be somebody!
```

This demonstrates that we’re now using standard output for successful output and standard error for error output as appropriate.

### Summary

This chapter recapped some of the major concepts you’ve learned so far and covered how to perform common I/O operations in Rust. By using <u>command line arguments</u>, files, <u>environment variables</u>, and the `eprintln!` macro for printing errors, you’re now prepared to write command line applications. By using the concepts in previous chapters, your code will be well organized, store data effectively in the appropriate data structures, handle errors nicely, and be well tested.

Next, we’ll explore some Rust features that were influenced by functional languages: closures and iterators.

# 13. Functional Language Features: Iterators and Closures

Rust吸收了许多语言的设计灵感，其中也包含函数式编程。

Rust’s design has taken inspiration from many existing languages and techniques, and one significant influence is *functional programming*. Programming in a functional style often includes using functions as values by passing them in arguments, returning them from other functions, assigning them to variables for later execution, and so forth.

In this chapter, we won’t debate the issue of what functional programming is or isn’t but will instead discuss some features of Rust that are similar to features in many languages often referred to as functional.

More specifically, we’ll cover:

- *Closures*, a function-like construct you can store in a variable（闭包）
- *Iterators*, a way of processing a series of elements（迭代器/遍历器）
- How to use these two features to improve the I/O project in Chapter 12
- The performance of these two features (Spoiler alert: they’re faster than you might think!)

Other Rust features, such as pattern matching and enums, which we’ve covered in other chapters, are influenced by the functional style as well. Mastering closures and iterators is an important part of writing idiomatic, fast Rust code, so we’ll devote this entire chapter to them.

## 13.1 Closures: Anonymous Functions that Can Capture Their Environment

Rust的闭包就是能传递参数的匿名函数。

Rust’s closures are **anonymous functions** you can save in a variable or pass as arguments to other functions. You can create the closure in one place and then call the closure to evaluate it in a different context. **Unlike functions, closures can capture values from the scope in which they’re defined**. We’ll demonstrate how these closure features allow for code reuse and behavior customization.

### Creating an Abstraction of Behavior with Closures

Let’s work on an example of a situation in which it’s useful to store a closure to be executed later. Along the way, we’ll talk about the syntax of closures, type inference, and traits.

Consider this hypothetical situation: we work at a startup that’s making an app to generate custom exercise workout plans. The backend is written in Rust, and the algorithm that generates the workout plan takes into account many factors, such as the app user’s age, body mass index, exercise preferences, recent workouts, and an intensity number they specify. The actual algorithm used isn’t important in this example; what’s important is that this calculation takes a few seconds. **We want to call this algorithm only when we need to and <u>only call it once</u> so we don’t make the user wait more than necessary**.

We’ll simulate calling this hypothetical algorithm with the function `simulated_expensive_calculation` shown in Listing 13-1, which will print `calculating slowly...`, wait for two seconds, and then return whatever number we passed in.

Filename: src/main.rs

```rust
use std::thread;
use std::time::Duration;

fn simulated_expensive_calculation(intensity: u32) -> u32 {
    println!("calculating slowly...");
    thread::sleep(Duration::from_secs(2));
    intensity
}
```

Listing 13-1: A function to stand in for a hypothetical calculation that takes about 2 seconds to run

Next is the `main` function, which contains the parts of the workout app important for this example. This function represents the code that the app will call when a user asks for a workout plan. Because the interaction with the app’s frontend isn’t relevant to the use of closures, we’ll hardcode values representing inputs to our program and print the outputs.

The required inputs are these:

- An intensity number from the user, which is specified when they request a workout to indicate whether they want a low-intensity workout or a high-intensity workout
- A random number that will generate some variety in the workout plans

The output will be the recommended workout plan. Listing 13-2 shows the `main` function we’ll use.

Filename: src/main.rs

```rust
fn main() {
    let simulated_user_specified_value = 10;
    let simulated_random_number = 7;

    generate_workout(simulated_user_specified_value, simulated_random_number);
}
```

Listing 13-2: A `main` function with hardcoded values to simulate user input and random number generation

We’ve hardcoded the variable `simulated_user_specified_value` as 10 and the variable `simulated_random_number` as 7 for simplicity’s sake; in an actual program, we’d get the intensity number from the app frontend, and we’d use the `rand` crate to generate a random number, as we did in the Guessing Game example in Chapter 2. The `main` function calls a `generate_workout` function with the simulated input values.

Now that we have the context, let’s get to the algorithm. The function `generate_workout` in Listing 13-3 contains the business logic of the app that we’re most concerned with in this example. The rest of the code changes in this example will be made to this function.

Filename: src/main.rs

```rust
fn generate_workout(intensity: u32, random_number: u32) {
    if intensity < 25 {
        println!(
            "Today, do {} pushups!",
            simulated_expensive_calculation(intensity)
        );
        println!(
            "Next, do {} situps!",
            simulated_expensive_calculation(intensity)
        );
    } else {
        if random_number == 3 {
            println!("Take a break today! Remember to stay hydrated!");
        } else {
            println!(
                "Today, run for {} minutes!",
                simulated_expensive_calculation(intensity)
            );
        }
    }
}
```

Listing 13-3: The business logic that prints the workout plans based on the inputs and calls to the `simulated_expensive_calculation` function

The code in Listing 13-3 has multiple calls to the slow calculation function. The first `if` block calls `simulated_expensive_calculation` twice, the `if` inside the outer `else` doesn’t call it at all, and the code inside the second `else` case calls it once.

The desired behavior of the `generate_workout` function is to first check whether the user wants a low-intensity workout (indicated by a number less than 25) or a high-intensity workout (a number of 25 or greater).

Low-intensity workout plans will recommend a number of push-ups and sit-ups based on the complex algorithm we’re simulating.

If the user wants a high-intensity workout, there’s some additional logic: if the value of the random number generated by the app happens to be 3, the app will recommend a break and hydration. If not, the user will get a number of minutes of running based on the complex algorithm.

This code works the way the business wants it to now, but let’s say the data science team decides that we need to make some changes to the way we call the `simulated_expensive_calculation` function in the future. <u>To simplify the update when those changes happen, we want to refactor this code so it calls the `simulated_expensive_calculation` function only once. We also want to cut the place where we’re currently unnecessarily calling the function twice without adding any other calls to that function in the process. That is, we don’t want to call it if the result isn’t needed, and we still want to call it only once</u>.

#### Refactoring Using Functions

We could restructure the workout program in many ways. First, we’ll try extracting the duplicated call to the `simulated_expensive_calculation` function into a variable, as shown in Listing 13-4.

Filename: src/main.rs

```rust
fn generate_workout(intensity: u32, random_number: u32) {
    let expensive_result = simulated_expensive_calculation(intensity);

    if intensity < 25 {
        println!("Today, do {} pushups!", expensive_result);
        println!("Next, do {} situps!", expensive_result);
    } else {
        if random_number == 3 {
            println!("Take a break today! Remember to stay hydrated!");
        } else {
            println!("Today, run for {} minutes!", expensive_result);
        }
    }
}
```

Listing 13-4: Extracting the calls to `simulated_expensive_calculation` to one place and storing the result in the `expensive_result` variable

This change unifies all the calls to `simulated_expensive_calculation` and solves the problem of the first `if` block unnecessarily calling the function twice. Unfortunately, we’re now calling this function and waiting for the result in all cases, which includes the inner `if` block that doesn’t use the result value at all.

<u>We want to refer to `simulated_expensive_calculation` only once in `generate_workout`, but defer the expensive calculation to only where we actually need the result. This is a use case for closures</u>!

#### Refactoring with Closures to Store Code

使用闭包存储函数调用，赋值到一个变量上。

**Instead of always calling the `simulated_expensive_calculation` function before the `if` blocks, we can define a closure and store the *closure* in a variable rather than storing the result of the function call**, as shown in Listing 13-5. We can actually move the whole body of `simulated_expensive_calculation` within the closure we’re introducing here.

Filename: src/main.rs

```rust
    let expensive_closure = |num| {
        println!("calculating slowly...");
        thread::sleep(Duration::from_secs(2));
        num
    };
```

Listing 13-5: Defining a closure and storing it in the `expensive_closure` variable

需要注意，这里闭包的使用，并不需要像funciton一样声明参数的类型，也没有声明返回值类型（就返回了num）。

The closure definition comes after the `=` to assign it to the variable `expensive_closure`. **To define a closure, we start with a pair of vertical pipes (`|`), inside which we specify the parameters to the closure; this syntax was chosen because of its similarity to closure definitions in Smalltalk and Ruby**. This closure has one parameter named `num`: if we had more than one parameter, we would separate them with commas, like `|param1, param2|`.

After the parameters, we place curly brackets that hold the body of the closure—these are optional if the closure body is a single expression. **The end of the closure, after the curly brackets, needs a semicolon to complete the `let` statement. The value returned from the last line in the closure body (`num`) will be the value returned from the closure when it’s called, because that line doesn’t end in a semicolon; just as in function bodies**.

这里的let赋值，指的是把匿名函数赋值给变量，而不是把函数返回值赋值给变量。

**Note that this `let` statement means `expensive_closure` contains the *definition* of an anonymous function, not the *resulting value* of calling the anonymous function**. Recall that we’re using a closure because we want to define the code to call at one point, store that code, and call it at a later point; the code we want to call is now stored in `expensive_closure`.

With the closure defined, we can change the code in the `if` blocks to call the closure to execute the code and get the resulting value. We call a closure like we do a function: we specify the variable name that holds the closure definition and follow it with parentheses containing the argument values we want to use, as shown in Listing 13-6.

Filename: src/main.rs

```rust
fn generate_workout(intensity: u32, random_number: u32) {
    let expensive_closure = |num| {
        println!("calculating slowly...");
        thread::sleep(Duration::from_secs(2));
        num
    };

    if intensity < 25 {
        println!("Today, do {} pushups!", expensive_closure(intensity));
        println!("Next, do {} situps!", expensive_closure(intensity));
    } else {
        if random_number == 3 {
            println!("Take a break today! Remember to stay hydrated!");
        } else {
            println!(
                "Today, run for {} minutes!",
                expensive_closure(intensity)
            );
        }
    }
}
```

Listing 13-6: Calling the `expensive_closure` we’ve defined

**上面代码中，`num`被编译器自动识别成了`u32`类型**。

Now how to perform the expensive calculation is defined in only one place, and we’re only executing that code where we need the results.

However, we’ve reintroduced one of the problems from Listing 13-3: we’re still calling the closure twice in the first `if` block, which will call the expensive code twice and make the user wait twice as long as they need to. We could fix this problem by creating a variable local to that `if` block to hold the result of calling the closure, but closures provide us with another solution. We’ll talk about that solution in a bit. **But first let’s talk about why there aren’t type annotations in the closure definition and the traits involved with closures**.

### Closure Type Inference and Annotation

使用闭包时，不需要像使用函数function一样声明参数类型和返回值类型。

+ function需要命名、参数类型声明、返回值声明，因为需要对外暴露和提供使用
+ 闭包作为匿名函数，只用于当前作用域的某个变量，无需对外达成一致的参数类型、返回值声明

**<u>Closures don’t require you to annotate the types of the parameters or the return value like `fn` functions do</u>**. <u>Type annotations are required on functions because they’re part of an explicit interface exposed to your users. Defining this interface rigidly is important for ensuring that everyone agrees on what types of values a function uses and returns</u>. But closures aren’t used in an exposed interface like this: they’re stored in variables and used without naming them and exposing them to users of our library.

闭包通常只使用在较小范围的作用域下，编译器有能力根据上下文推断参数类型和返回值类型，类似平时变量的类型推断。

**Closures are usually short and relevant only within a narrow context rather than in any arbitrary scenario. Within these limited contexts, the compiler is reliably able to infer the types of the parameters and the return type, similar to how it’s able to infer the types of most variables**.

Making programmers annotate the types in these small, anonymous functions would be annoying and largely redundant with the information the compiler already has available.

当然，我们也可以主动声明闭包的参数类型、返回值类型，以提高代码的可读性（当然这不是严格要求的）

**As with variables, we can add type annotations if we want to increase explicitness and clarity at the cost of being more verbose than is strictly necessary**. Annotating the types for the closure we defined in Listing 13-5 would look like the definition shown in Listing 13-7.

Filename: src/main.rs

```rust
    let expensive_closure = |num: u32| -> u32 {
        println!("calculating slowly...");
        thread::sleep(Duration::from_secs(2));
        num
    };
```

Listing 13-7: Adding optional type annotations of the parameter and return value types in the closure

With type annotations added, the syntax of closures looks more similar to the syntax of functions. The following is a vertical comparison of the syntax for the definition of a function that adds 1 to its parameter and a closure that has the same behavior. We’ve added some spaces to line up the relevant parts. This illustrates how closure syntax is similar to function syntax except for the use of pipes and the amount of syntax that is optional:

```rust
fn  add_one_v1   (x: u32) -> u32 { x + 1 }
let add_one_v2 = |x: u32| -> u32 { x + 1 };
let add_one_v3 = |x|             { x + 1 };
let add_one_v4 = |x|               x + 1  ;
```

<u>The first line shows a function definition, and the second line shows a fully annotated closure definition. The third line removes the type annotations from the closure definition, and **the fourth line removes the brackets, which are optional because the closure body has only one expression**. These are all valid definitions that will produce the same behavior when they’re called</u>. Calling the closures is required for `add_one_v3` and `add_one_v4` to be able to compile because the types will be inferred from their usage.

闭包的定义在编译时，每个参数和返回值都会有确定的类型，所以不能在两次调用时传入不同类型的参数。

**Closure definitions will have one concrete type inferred for each of their parameters and for their return value**. For instance, Listing 13-8 shows the definition of a short closure that just returns the value it receives as a parameter. This closure isn’t very useful except for the purposes of this example. Note that we haven’t added any type annotations to the definition: if we then try to call the closure twice, using a `String` as an argument the first time and a `u32` the second time, we’ll get an error.

Filename: src/main.rs

```rust
    let example_closure = |x| x;

    let s = example_closure(String::from("hello"));
    let n = example_closure(5);
```

Listing 13-8: Attempting to call a closure whose types are inferred with two different types

The compiler gives us this error:

```console
$ cargo run
   Compiling closure-example v0.1.0 (file:///projects/closure-example)
error[E0308]: mismatched types
 --> src/main.rs:5:29
  |
5 |     let n = example_closure(5);
  |                             ^
  |                             |
  |                             expected struct `String`, found integer
  |                             help: try using a conversion method: `5.to_string()`

For more information about this error, try `rustc --explain E0308`.
error: could not compile `closure-example` due to previous error
```

由于我们第一次调用这个闭包时传入的参数是`String`，编译器推断闭包的参数类型和返回值类型都是`String`。所以第二次调用时，传入i32类心的参数则报错。

**The first time we call `example_closure` with the `String` value, the compiler infers the type of `x` and the return type of the closure to be `String`. Those types are then locked into the closure in `example_closure`, and we get a type error if we try to use a different type with the same closure**.

### Storing Closures Using Generic Parameters and the `Fn` Traits

> [备忘录模式 | 菜鸟教程 (runoob.com)](https://www.runoob.com/design-pattern/memento-pattern.html)
>
> [Lazy evaluation_王山山-CSDN博客](https://blog.csdn.net/Watkins_OS/article/details/100042846)

Let’s return to our workout generation app. In Listing 13-6, our code was still calling the expensive calculation closure more times than it needed to. One option to solve this issue is to save the result of the expensive closure in a variable for reuse and use the variable in each place we need the result, instead of calling the closure again. However, this method could result in a lot of repeated code.

我们可以创建一个struct来存放闭包和调用闭包的结果。struct只会在我们需要闭包的返回值时才执行闭包（并将返回值缓存）。这是通过**“备忘录”模式，或者“延时计算/惰性计算”**实现的。

Fortunately, another solution is available to us. **We can create a struct that will hold the closure and the resulting value of calling the closure. The struct will execute the closure only if we need the resulting value, and it will cache the resulting value so the rest of our code doesn’t have to be responsible for saving and reusing the result**. <u>You may know this pattern as *memoization* or *lazy evaluation*</u>.

+ 在struct中使用闭包时，需要声明闭包中参数、返回值的类型。（struct必须知道每个字段的类型）
+ **每个闭包实例都拥有不同的类型（即使签名（参数、返回值）一致）**
+ 在struct、enum或function参数中使用闭包，我们也一样可以使用泛型和trait。

**To make a struct that holds a closure, we need to specify the type of the closure, because a struct definition needs to know the types of each of its fields**. <u>**Each closure instance has its own unique anonymous type: that is, even if two closures have the same signature, their types are still considered different**</u>. <u>To define structs, enums, or function parameters that use closures, we use generics and trait bounds, as we discussed in Chapter 10</u>.

`Fn`是标准库中的trait，所有的闭包，至少实现了其中一个trait（`Fn`、`FnMut`、`FnOnce`）。

**The `Fn` traits are provided by the standard library. All closures implement at least one of the traits: `Fn`, `FnMut`, or `FnOnce`.** We’ll discuss the difference between these traits in the [“Capturing the Environment with Closures”](https://doc.rust-lang.org/book/ch13-01-closures.html#capturing-the-environment-with-closures) section; in this example, we can use the `Fn` trait.

**We add types to the `Fn` trait bound to represent the types of the parameters and return values the closures must have to match this trait bound. In this case, our closure has a parameter of type `u32` and returns a `u32`, so the trait bound we specify is `Fn(u32) -> u32`**.

Listing 13-9 shows the definition of the `Cacher` struct that holds a closure and an optional result value.

Filename: src/main.rs

```rust
struct Cacher<T>
where
    T: Fn(u32) -> u32,
{
    calculation: T,
    value: Option<u32>,
}
```

Listing 13-9: **Defining a `Cacher` struct that holds a closure in `calculation` and an optional result in `value`**

<u>The `Cacher` struct has a `calculation` field of the generic type `T`. The trait bounds on `T` specify that it’s a closure by using the `Fn` trait</u>. Any closure we want to store in the `calculation` field must have one `u32` parameter (specified within the parentheses after `Fn`) and must return a `u32` (specified after the `->`).

> 注意：Function一样可以实现上述的三种 `Fn` traits。如果我们不需要从当前环境获取变量，我们可以直接使用funciton而不是闭包。
>
> **Note: Functions can implement all three of the `Fn` traits too. If what we want to do doesn’t require capturing a value from the environment, we can use a function rather than a closure where we need something that implements an `Fn` trait**.

上述代码中，当我们执行闭包之前，`value`字段是`None`；让我们请求获取闭包的返回值时，则实现闭包，并缓存结果到`value`中，此时它时一个`Some`。下次请求闭包的结果时，`Cacher`会直接`Some`中的值，而不会重复再执行一次闭包。

**The `value` field is of type `Option<u32>`. Before we execute the closure, `value` will be `None`. When code using a `Cacher` asks for the *result* of the closure, the `Cacher` will execute the closure at that time and store the result within a `Some` variant in the `value` field. Then if the code asks for the result of the closure again, instead of executing the closure again, the `Cacher` will return the result held in the `Some` variant**.

The logic around the `value` field we’ve just described is defined in Listing 13-10.

Filename: src/main.rs

```rust
impl<T> Cacher<T>
where
    T: Fn(u32) -> u32,
{
    fn new(calculation: T) -> Cacher<T> {
        Cacher {
            calculation,
            value: None,
        }
    }

    fn value(&mut self, arg: u32) -> u32 {
        match self.value {
            Some(v) => v,
            None => {
                let v = (self.calculation)(arg);
                self.value = Some(v);
                v
            }
        }
    }
}
```

Listing 13-10: The caching logic of `Cacher`

We want `Cacher` to manage the struct fields’ values rather than letting the calling code potentially change the values in these fields directly, so these fields are private.

The `Cacher::new` function takes a generic parameter `T`, which we’ve defined as having the same trait bound as the `Cacher` struct. Then `Cacher::new` returns a `Cacher` instance that holds the closure specified in the `calculation` field and a `None` value in the `value` field, because we haven’t executed the closure yet.

**When the calling code needs the result of evaluating the closure, instead of calling the closure directly, it will call the `value` method. This method checks whether we already have a resulting value in `self.value` in a `Some`; if we do, it returns the value within the `Some` without executing the closure again**.

If `self.value` is `None`, the code calls the closure stored in `self.calculation`, saves the result in `self.value` for future use, and returns the value as well.

Listing 13-11 shows how we can use this `Cacher` struct in the function `generate_workout` from Listing 13-6.

Filename: src/main.rs

```rust
fn generate_workout(intensity: u32, random_number: u32) {
    let mut expensive_result = Cacher::new(|num| {
        println!("calculating slowly...");
        thread::sleep(Duration::from_secs(2));
        num
    });

    if intensity < 25 {
        println!("Today, do {} pushups!", expensive_result.value(intensity));
        println!("Next, do {} situps!", expensive_result.value(intensity));
    } else {
        if random_number == 3 {
            println!("Take a break today! Remember to stay hydrated!");
        } else {
            println!(
                "Today, run for {} minutes!",
                expensive_result.value(intensity)
            );
        }
    }
}
```

Listing 13-11: Using `Cacher` in the `generate_workout` function to abstract away the caching logic

Instead of saving the closure in a variable directly, we save a new instance of `Cacher` that holds the closure. Then, in each place we want the result, we call the `value` method on the `Cacher` instance. We can call the `value` method as many times as we want, or not call it at all, and the expensive calculation will be run a maximum of once.

Try running this program with the `main` function from Listing 13-2. Change the values in the `simulated_user_specified_value` and `simulated_random_number` variables to verify that in all the cases in the various `if` and `else` blocks, `calculating slowly...` appears only once and only when needed. The `Cacher` takes care of the logic necessary to ensure we aren’t calling the expensive calculation more than we need to so `generate_workout` can focus on the business logic.

### Limitations of the `Cacher` Implementation

Caching values is a generally useful behavior that we might want to use in other parts of our code with different closures. However, there are two problems with the current implementation of `Cacher` that would make reusing it in different contexts difficult.

The first problem is that a `Cacher` instance assumes it will always get the same value for the parameter `arg` to the `value` method. That is, this test of `Cacher` will fail:

```rust
    #[test]
    fn call_with_different_values() {
        let mut c = Cacher::new(|a| a);

        let v1 = c.value(1);
        let v2 = c.value(2);

        assert_eq!(v2, 2);
    }
```

This test creates a new `Cacher` instance with a closure that returns the value passed into it. We call the `value` method on this `Cacher` instance with an `arg` value of 1 and then an `arg` value of 2, and we expect the call to `value` with the `arg` value of 2 to return 2.

Run this test with the `Cacher` implementation in Listing 13-9 and Listing 13-10, and the test will fail on the `assert_eq!` with this message:

```console
$ cargo test
   Compiling cacher v0.1.0 (file:///projects/cacher)
    Finished test [unoptimized + debuginfo] target(s) in 0.72s
     Running unittests (target/debug/deps/cacher-074d7c200c000afa)

running 1 test
test tests::call_with_different_values ... FAILED

failures:

---- tests::call_with_different_values stdout ----
thread 'main' panicked at 'assertion failed: `(left == right)`
  left: `1`,
 right: `2`', src/lib.rs:43:9
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace


failures:
    tests::call_with_different_values

test result: FAILED. 0 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

error: test failed, to rerun pass '--lib'
```

The problem is that the first time we called `c.value` with 1, the `Cacher` instance saved `Some(1)` in `self.value`. Thereafter, no matter what we pass into the `value` method, it will always return 1.

可以改使map来缓存值，避免只缓存第一次输入的值。

Try modifying `Cacher` to hold a hash map rather than a single value. The keys of the hash map will be the `arg` values that are passed in, and the values of the hash map will be the result of calling the closure on that key. Instead of looking at whether `self.value` directly has a `Some` or a `None` value, the `value` function will look up the `arg` in the hash map and return the value if it’s present. If it’s not present, the `Cacher` will call the closure and save the resulting value in the hash map associated with its `arg` value.

目前的代码实现中，闭包只能接受`u32`类型的参数，和返回一个`u32`类型的值。我们后续可能需要string slice作为参数，并且返回`usize`类型的值。（我们可以使用更灵活的泛型参数来解决问题）

The second problem with the current `Cacher` implementation is that it only accepts closures that take one parameter of type `u32` and return a `u32`. We might want to cache the results of closures that take a string slice and return `usize` values, for example. To fix this issue, try introducing more generic parameters to increase the flexibility of the `Cacher` functionality.

### Capturing the Environment with Closures

和Function不同的是，闭包能够读取当前作用域下的变量。

In the workout generator example, we only used closures as inline anonymous functions. **However, closures have an additional capability that functions don’t have: they can capture their environment and access variables from the scope in which they’re defined**.

Listing 13-12 has an example of a closure stored in the `equal_to_x` variable that uses the `x` variable from the closure’s surrounding environment.

Filename: src/main.rs

```rust
fn main() {
    let x = 4;

    let equal_to_x = |z| z == x;

    let y = 4;

    assert!(equal_to_x(y));
}
```

Listing 13-12: Example of a closure that refers to a variable in its enclosing scope

这里x虽然不是闭包的入参，但是由于和闭包处于相同的作用域内，所以闭包内也能使用变量x。

**Here, even though `x` is not one of the parameters of `equal_to_x`, the `equal_to_x` closure is allowed to use the `x` variable that’s defined in the same scope that `equal_to_x` is defined in**.

We can’t do the same with functions; if we try with the following example, our code won’t compile:

Filename: src/main.rs

```rust
fn main() {
    let x = 4;

    fn equal_to_x(z: i32) -> bool {
        z == x
    }

    let y = 4;

    assert!(equal_to_x(y));
}
```

We get an error:

```console
$ cargo run
   Compiling equal-to-x v0.1.0 (file:///projects/equal-to-x)
error[E0434]: can't capture dynamic environment in a fn item
 --> src/main.rs:5:14
  |
5 |         z == x
  |              ^
  |
  = help: use the `|| { ... }` closure form instead

For more information about this error, try `rustc --explain E0434`.
error: could not compile `equal-to-x` due to previous error
```

The compiler even reminds us that this only works with closures!

但闭包使用当前环境的一个值时，会用内存存储这个被用到闭包内部的值。使用内存是额外的开销，通常情况下我们不希望有额外的开销。

**When a closure captures a value from its environment, it uses memory to store the values for use in the closure body**. This use of memory is overhead that we don’t want to pay in more common cases where we want to execute code that doesn’t capture its environment. Because functions are never allowed to capture their environment, defining and using functions will never incur this overhead.

闭包有三种途径从当前环境获取值（和function一样，获取值的方式有3种）：taking ownership, borrowing mutably, and borrowing immutably。`Fn` traits已经实现了这三种：

**Closures can capture values from their environment in three ways, which directly map to the three ways a function can take a parameter: <u>taking ownership, borrowing mutably, and borrowing immutably</u>**. These are encoded in the three `Fn` traits as follows:

+ `FnOnce`会获取变量的所有权
+ `FnMut`和`&mut`类似，borrow所有权，并且可以修改值
+ `Fn` 和`&`类型，borrow所有权，但是不可修改

- **`FnOnce` consumes the variables it captures from its enclosing scope, known as the closure’s *environment***. To consume the captured variables, the closure must take ownership of these variables and move them into the closure when it is defined. **The `Once` part of the name represents the fact that the closure can’t take ownership of the same variables more than once, so <u>it can be called only once</u>**.
- **`FnMut` can change the environment because it mutably borrows values**.
- **`Fn` borrows values from the environment immutably**.

创建闭包时，Rust自动根据上下文推断使用哪种`Fn` trait 实现。

**When you create a closure, Rust infers which trait to use based on how the closure uses the values from the environment.** 

+ 所有的闭包都实现了`FnOnce`，因为他们至少能被调用一次
+ 所有不需要move所有权的闭包，都实现了`FnMut`
+ 所有不需要修改捕获的变量的闭包，都实现了`Fn`

**<u>All closures implement `FnOnce` because they can all be called at least once. Closures that don’t move the captured variables also implement `FnMut`, and closures that don’t need mutable access to the captured variables also implement `Fn`</u>.** In Listing 13-12, the `equal_to_x` closure borrows `x` immutably (so `equal_to_x` has the `Fn` trait) because the body of the closure only needs to read the value in `x`.

**如果你需要强制获取值的ownership，可以在参数列表前使用`move`关键字。在将闭包传入一个新线程时这是一个常用的操作，这使得新线程拥有这个值的所有权**。

**If you want to force the closure to take ownership of the values it uses in the environment, you can use the `move` keyword before the parameter list**. <u>This technique is mostly useful when passing a closure to a new thread to move the data so it’s owned by the new thread</u>.

> **注意：`move`闭包仍然实现了`Fn`和`FnMut`，即使是通过move来获取变量。因为这些trait实现，只是为了表明闭包是如何对待获取的变量的，而不是表明如何获取变量的。`move`仅仅是表明了当前闭包通过`move`获取变量。**
>
> **Note: `move` closures may still implement `Fn` or `FnMut`, even though they capture variables by move. This is because the traits implemented by a closure type are determined by what the closure does with captured values, not how it captures them. The `move` keyword only specifies the latter.**

<u>We’ll have more examples of `move` closures in Chapter 16 when we talk about concurrency</u>. For now, here’s the code from Listing 13-12 with the `move` keyword added to the closure definition and using vectors instead of integers, <u>because integers can be copied rather than moved</u>; note that this code will not yet compile.

Filename: src/main.rs

```rust
fn main() {
    let x = vec![1, 2, 3];

    let equal_to_x = move |z| z == x;

    println!("can't use x here: {:?}", x);

    let y = vec![1, 2, 3];

    assert!(equal_to_x(y));
}
```

We receive the following error:

```console
$ cargo run
   Compiling equal-to-x v0.1.0 (file:///projects/equal-to-x)
error[E0382]: borrow of moved value: `x`
 --> src/main.rs:6:40
  |
2 |     let x = vec![1, 2, 3];
  |         - move occurs because `x` has type `Vec<i32>`, which does not implement the `Copy` trait
3 | 
4 |     let equal_to_x = move |z| z == x;
  |                      --------      - variable moved due to use in closure
  |                      |
  |                      value moved into closure here
5 | 
6 |     println!("can't use x here: {:?}", x);
  |                                        ^ value borrowed here after move

For more information about this error, try `rustc --explain E0382`.
error: could not compile `equal-to-x` due to previous error
```

**The `x` value is moved into the closure when the closure is defined, because we added the `move` keyword. The closure then has ownership of `x`, and `main` isn’t allowed to use `x` anymore in the `println!` statement. Removing `println!` will fix this example**.

**<u>Most of the time when specifying one of the `Fn` trait bounds, you can start with `Fn` and the compiler will tell you if you need `FnMut` or `FnOnce` based on what happens in the closure body</u>**.

To illustrate situations where closures that can capture their environment are useful as function parameters, let’s move on to our next topic: iterators.

## 13.2 Processing a Series of Items with Iterators

The iterator pattern allows you to perform some task on a sequence of items in turn. An iterator is responsible for the logic of iterating over each item and determining when the sequence has finished. When you use iterators, you don’t have to reimplement that logic yourself.

在Rust中，迭代器是懒加载的，只有调用消费iterator的方法时，才真正生效。

**In Rust, iterators are *lazy*, meaning they have no effect until you call methods that consume the iterator to use it up**. For example, the code in Listing 13-13 creates an iterator over the items in the vector `v1` by calling the `iter` method defined on `Vec<T>`. This code by itself doesn’t do anything useful.

```rust
    let v1 = vec![1, 2, 3];

    let v1_iter = v1.iter();
```

Listing 13-13: Creating an iterator

Once we’ve created an iterator, we can use it in a variety of ways. In Listing 3-5 in Chapter 3, we used iterators with `for` loops to execute some code on each item, although we glossed over what the call to `iter` did until now.

The example in Listing 13-14 separates the creation of the iterator from the use of the iterator in the `for` loop. <u>The iterator is stored in the `v1_iter` variable, and no iteration takes place at that time. When the `for` loop is called using the iterator in `v1_iter`, each element in the iterator is used in one iteration of the loop</u>, which prints out each value.

```rust
    let v1 = vec![1, 2, 3];

    let v1_iter = v1.iter();

    for val in v1_iter {
        println!("Got: {}", val);
    }
```

Listing 13-14: Using an iterator in a `for` loop

In languages that don’t have iterators provided by their standard libraries, you would likely write this same functionality by starting a variable at index 0, using that variable to index into the vector to get a value, and incrementing the variable value in a loop until it reached the total number of items in the vector.

Iterators handle all that logic for you, cutting down on repetitive code you could potentially mess up. Iterators give you more flexibility to use the same logic with many different kinds of sequences, not just data structures you can index into, like vectors. Let’s examine how iterators do that.

### The `Iterator` Trait and the `next` Method

All iterators implement a trait named `Iterator` that is defined in the standard library. The definition of the trait looks like this:

```rust
pub trait Iterator {
    type Item;

    fn next(&mut self) -> Option<Self::Item>;

    // methods with default implementations elided
}
```

**Notice this definition uses some new syntax: `type Item` and `Self::Item`, which are defining an *associated type* with this trait**. We’ll talk about associated types in depth in Chapter 19. For now, all you need to know is that <u>this code says implementing the `Iterator` trait requires that you also define an `Item` type, and this `Item` type is used in the return type of the `next` method</u>. In other words, the `Item` type will be the type returned from the iterator.

**The `Iterator` trait only requires implementors to define one method: the `next` method, which returns one item of the iterator at a time wrapped in `Some` and, when iteration is over, returns `None`**.

We can call the `next` method on iterators directly; Listing 13-15 demonstrates what values are returned from repeated calls to `next` on the iterator created from the vector.

Filename: src/lib.rs

```rust
    #[test]
    fn iterator_demonstration() {
        let v1 = vec![1, 2, 3];

        let mut v1_iter = v1.iter();

        assert_eq!(v1_iter.next(), Some(&1));
        assert_eq!(v1_iter.next(), Some(&2));
        assert_eq!(v1_iter.next(), Some(&3));
        assert_eq!(v1_iter.next(), None);
    }
```

Listing 13-15: Calling the `next` method on an iterator

**调用迭代器的`next`方法会修改迭代器的内部state状态，所以需要声明`mut`。**

**使用for循环时不需要声明迭代器`mut`，因为for循环已经获取了迭代器的所有权，并且使它是`mut`的**。

**<u>Note that we needed to make `v1_iter` mutable: calling the `next` method on an iterator changes internal state that the iterator uses to keep track of where it is in the sequence</u>**. In other words, this code *consumes*, or uses up, the iterator. Each call to `next` eats up an item from the iterator. **<u>We didn’t need to make `v1_iter` mutable when we used a `for` loop because the loop took ownership of `v1_iter` and made it mutable behind the scenes.</u>**

**另外需要注意的是，如果使用`iter`获取迭代器，那么我们从`next`方法获取到的值的引用是immutable的**。如果需要获取返回的值的所有权，可以使用`into_iter`来获取迭代器；类似的，如果需要mutable references，可以使用`iter_mut`。

**<u>Also note that the values we get from the calls to `next` are immutable references to the values in the vector</u>**. **The `iter` method produces an iterator over immutable references. If we want to create an iterator that takes ownership of `v1` and returns owned values, we can call `into_iter` instead of `iter`. Similarly, if we want to iterate over mutable references, we can call `iter_mut` instead of `iter`.**

### Methods that Consume the Iterator

<u>The `Iterator` trait has a number of different methods with default implementations provided by the standard library; you can find out about these methods by looking in the standard library API documentation for the `Iterator` trait.</u> Some of these methods call the `next` method in their definition, which is why you’re required to implement the `next` method when implementing the `Iterator` trait.

**Methods that call `next` are called *consuming adaptors*, because calling them uses up the iterator**. <u>One example is the `sum` method, which takes ownership of the iterator and iterates through the items by repeatedly calling `next`, thus consuming the iterator</u>. As it iterates through, it adds each item to a running total and returns the total when iteration is complete. Listing 13-16 has a test illustrating a use of the `sum` method:

Filename: src/lib.rs

```rust
    #[test]
    fn iterator_sum() {
        let v1 = vec![1, 2, 3];

        let v1_iter = v1.iter();

        let total: i32 = v1_iter.sum();

        assert_eq!(total, 6);
    }
```

Listing 13-16: Calling the `sum` method to get the total of all items in the iterator

<u>We aren’t allowed to use `v1_iter` after the call to `sum` because `sum` takes ownership of the iterator we call it on</u>.

### Methods that Produce Other Iterators

**Other methods defined on the `Iterator` trait, known as *iterator adaptors*, allow you to change iterators into different kinds of iterators**. You can chain multiple calls to iterator adaptors to perform complex actions in a readable way. <u>**But because all iterators are lazy, you have to call one of the consuming adaptor methods to get results from calls to iterator adaptors**.</u>

Listing 13-17 shows an example of **calling the iterator adaptor method `map`, which takes a closure to call on each item to produce a new iterator**. The closure here creates a new iterator in which each item from the vector has been incremented by 1. However, this code produces a warning:

Filename: src/main.rs

```rust
    let v1: Vec<i32> = vec![1, 2, 3];

    v1.iter().map(|x| x + 1);
```

Listing 13-17: Calling the iterator adaptor `map` to create a new iterator

The warning we get is this:

```console
$ cargo run
   Compiling iterators v0.1.0 (file:///projects/iterators)
warning: unused `Map` that must be used
 --> src/main.rs:4:5
  |
4 |     v1.iter().map(|x| x + 1);
  |     ^^^^^^^^^^^^^^^^^^^^^^^^^
  |
  = note: `#[warn(unused_must_use)]` on by default
  = note: iterators are lazy and do nothing unless consumed

warning: `iterators` (bin "iterators") generated 1 warning
    Finished dev [unoptimized + debuginfo] target(s) in 0.47s
     Running `target/debug/iterators`
```

The code in Listing 13-17 doesn’t do anything; <u>the closure we’ve specified never gets called</u>. The warning reminds us why: <u>iterator adaptors are lazy, and we need to consume the iterator here</u>.

To fix this and consume the iterator, we’ll use the `collect` method, which we used in Chapter 12 with `env::args` in Listing 12-1. This method consumes the iterator and collects the resulting values into a collection data type.

In Listing 13-18, we collect the results of iterating over the iterator that’s returned from the call to `map` into a vector. This vector will end up containing each item from the original vector incremented by 1.

Filename: src/main.rs

```rust
    let v1: Vec<i32> = vec![1, 2, 3];

    let v2: Vec<_> = v1.iter().map(|x| x + 1).collect();

    assert_eq!(v2, vec![2, 3, 4]);
```

Listing 13-18: Calling the `map` method to create a new iterator and then calling the `collect` method to consume the new iterator and create a vector

Because `map` takes a closure, we can specify any operation we want to perform on each item. This is a great example of how closures let you customize some behavior while reusing the iteration behavior that the `Iterator` trait provides.

### Using Closures that Capture Their Environment

Now that we’ve introduced iterators, we can demonstrate a common use of closures that capture their environment by using the `filter` iterator adaptor. **The `filter` method on an iterator takes a closure that takes each item from the iterator and returns a Boolean. If the closure returns `true`, the value will be included in the iterator produced by `filter`. If the closure returns `false`, the value won’t be included in the resulting iterator**.

In Listing 13-19, we use `filter` with a closure that captures the `shoe_size` variable from its environment to iterate over a collection of `Shoe` struct instances. It will return only shoes that are the specified size.

Filename: src/lib.rs

```rust
#[derive(PartialEq, Debug)]
struct Shoe {
    size: u32,
    style: String,
}

fn shoes_in_size(shoes: Vec<Shoe>, shoe_size: u32) -> Vec<Shoe> {
    shoes.into_iter().filter(|s| s.size == shoe_size).collect()
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn filters_by_size() {
        let shoes = vec![
            Shoe {
                size: 10,
                style: String::from("sneaker"),
            },
            Shoe {
                size: 13,
                style: String::from("sandal"),
            },
            Shoe {
                size: 10,
                style: String::from("boot"),
            },
        ];

        let in_my_size = shoes_in_size(shoes, 10);

        assert_eq!(
            in_my_size,
            vec![
                Shoe {
                    size: 10,
                    style: String::from("sneaker")
                },
                Shoe {
                    size: 10,
                    style: String::from("boot")
                },
            ]
        );
    }
}
```

Listing 13-19: Using the `filter` method with a closure that captures `shoe_size`

The `shoes_in_size` function **takes ownership** of a vector of shoes and a shoe size as parameters. It returns a vector containing only shoes of the specified size.

In the body of `shoes_in_size`, we call `into_iter` to create an iterator that takes ownership of the vector. Then we call `filter` to adapt that iterator into a new iterator that only contains elements for which the closure returns `true`.

The closure captures the `shoe_size` parameter from the environment and compares the value with each shoe’s size, keeping only shoes of the size specified. Finally, calling `collect` gathers the values returned by the adapted iterator into a vector that’s returned by the function.

The test shows that when we call `shoes_in_size`, we get back only shoes that have the same size as the value we specified.

### Creating Our Own Iterators with the `Iterator` Trait

<u>We’ve shown that you can create an iterator by calling `iter`, `into_iter`, or `iter_mut` on a vector. You can create iterators from the other collection types in the standard library, such as hash map. You can also create iterators that do anything you want by implementing the `Iterator` trait on your own types</u>. **As previously mentioned, the only method you’re required to provide a definition for is the `next` method**. Once you’ve done that, you can use all other methods that have default implementations provided by the `Iterator` trait!

To demonstrate, let’s create an iterator that will only ever count from 1 to 5. First, we’ll create a struct to hold some values. Then we’ll make this struct into an iterator by implementing the `Iterator` trait and using the values in that implementation.

Listing 13-20 has the definition of the `Counter` struct and an associated `new` function to create instances of `Counter`:

Filename: src/lib.rs

```rust
struct Counter {
    count: u32,
}

impl Counter {
    fn new() -> Counter {
        Counter { count: 0 }
    }
}
```

Listing 13-20: Defining the `Counter` struct and a `new` function that creates instances of `Counter` with an initial value of 0 for `count`

The `Counter` struct has one field named `count`. This field holds a `u32` value that will keep track of where we are in the process of iterating from 1 to 5. The `count` field is private because we want the implementation of `Counter` to manage its value. The `new` function enforces the behavior of always starting new instances with a value of 0 in the `count` field.

Next, we’ll implement the `Iterator` trait for our `Counter` type by defining the body of the `next` method to specify what we want to happen when this iterator is used, as shown in Listing 13-21:

Filename: src/lib.rs

```rust
impl Iterator for Counter {
    type Item = u32;

    fn next(&mut self) -> Option<Self::Item> {
        if self.count < 5 {
            self.count += 1;
            Some(self.count)
        } else {
            None
        }
    }
}
```

Listing 13-21: Implementing the `Iterator` trait on our `Counter` struct

We set the associated `Item` type for our iterator to `u32`, meaning the iterator will return `u32` values. Again, don’t worry about associated types yet, we’ll cover them in Chapter 19.

We want our iterator to add 1 to the current state, so we initialized `count` to 0 so it would return 1 first. If the value of `count` is less than 5, `next` will increment `count` and return the current value wrapped in `Some`. Once `count` is 5, our iterator will stop incrementing `count` and always return `None`.

#### Using Our `Counter` Iterator’s `next` Method

Once we’ve implemented the `Iterator` trait, we have an iterator! Listing 13-22 shows a test demonstrating that we can use the iterator functionality of our `Counter` struct by calling the `next` method on it directly, just as we did with the iterator created from a vector in Listing 13-15.

Filename: src/lib.rs

```rust
    #[test]
    fn calling_next_directly() {
        let mut counter = Counter::new();

        assert_eq!(counter.next(), Some(1));
        assert_eq!(counter.next(), Some(2));
        assert_eq!(counter.next(), Some(3));
        assert_eq!(counter.next(), Some(4));
        assert_eq!(counter.next(), Some(5));
        assert_eq!(counter.next(), None);
    }
```

Listing 13-22: Testing the functionality of the `next` method implementation

This test creates a new `Counter` instance in the `counter` variable and then calls `next` repeatedly, verifying that we have implemented the behavior we want this iterator to have: returning the values from 1 to 5.

#### Using Other `Iterator` Trait Methods

We implemented the `Iterator` trait by defining the `next` method, so we can now use any `Iterator` trait method’s default implementations as defined in the standard library, because they all use the `next` method’s functionality.

For example, if for some reason we wanted to take the values produced by an instance of `Counter`, pair them with values produced by another `Counter` instance after skipping the first value, multiply each pair together, keep only those results that are divisible by 3, and add all the resulting values together, we could do so, as shown in the test in Listing 13-23:

Filename: src/lib.rs

```rust
    #[test]
    fn using_other_iterator_trait_methods() {
        let sum: u32 = Counter::new()
            .zip(Counter::new().skip(1))
            .map(|(a, b)| a * b)
            .filter(|x| x % 3 == 0)
            .sum();
        assert_eq!(18, sum);
    }
```

Listing 13-23: Using a variety of `Iterator` trait methods on our `Counter` iterator

**Note that `zip` produces only four pairs; the theoretical fifth pair `(5, None)` is never produced because `zip` returns `None` when either of its input iterators return `None`.**

All of these method calls are possible because we specified how the `next` method works, and the standard library provides default implementations for other methods that call `next`.

## 13.3 Improving Our I/O Project

With this new knowledge about iterators, we can improve the I/O project in Chapter 12 by using iterators to make places in the code clearer and more concise. Let’s look at how iterators can improve our implementation of the `Config::new` function and the `search` function.

### Removing a `clone` Using an Iterator

In Listing 12-6, we added code that took a slice of `String` values and created an instance of the `Config` struct by indexing into the slice and cloning the values, allowing the `Config` struct to own those values. In Listing 13-24, we’ve reproduced the implementation of the `Config::new` function as it was in Listing 12-23:

Filename: src/lib.rs

```rust
impl Config {
    pub fn new(args: &[String]) -> Result<Config, &str> {
        if args.len() < 3 {
            return Err("not enough arguments");
        }

        let query = args[1].clone();
        let filename = args[2].clone();

        let case_sensitive = env::var("CASE_INSENSITIVE").is_err();

        Ok(Config {
            query,
            filename,
            case_sensitive,
        })
    }
}
```

Listing 13-24: Reproduction of the `Config::new` function from Listing 12-23

At the time, we said not to worry about the inefficient `clone` calls because we would remove them in the future. Well, that time is now!

We needed `clone` here because we have a slice with `String` elements in the parameter `args`, but the `new` function doesn’t own `args`. To return ownership of a `Config` instance, we had to clone the values from the `query` and `filename` fields of `Config` so the `Config` instance can own its values.

With our new knowledge about iterators, we can change the `new` function to take ownership of an iterator as its argument instead of borrowing a slice. We’ll use the iterator functionality instead of the code that checks the length of the slice and indexes into specific locations. This will clarify what the `Config::new` function is doing because the iterator will access the values.

<u>Once `Config::new` takes ownership of the iterator and stops using indexing operations that borrow, we can move the `String` values from the iterator into `Config` rather than calling `clone` and making a new allocation.</u>

#### Using the Returned Iterator Directly

Open your I/O project’s *src/main.rs* file, which should look like this:

Filename: src/main.rs

```rust
fn main() {
    let args: Vec<String> = env::args().collect();

    let config = Config::new(&args).unwrap_or_else(|err| {
        eprintln!("Problem parsing arguments: {}", err);
        process::exit(1);
    });

    // --snip--
}
```

We’ll change the start of the `main` function that we had in Listing 12-24 to the code in Listing 13-25. This won’t compile until we update `Config::new` as well.

Filename: src/main.rs

```rust
fn main() {
    let config = Config::new(env::args()).unwrap_or_else(|err| {
        eprintln!("Problem parsing arguments: {}", err);
        process::exit(1);
    });

    // --snip--
}
```

Listing 13-25: Passing the return value of `env::args` to `Config::new`

<u>The `env::args` function returns an iterator! Rather than collecting the iterator values into a vector and then passing a slice to `Config::new`, now we’re passing ownership of the iterator returned from `env::args` to `Config::new` directly</u>.

Next, we need to update the definition of `Config::new`. In your I/O project’s *src/lib.rs* file, let’s change the signature of `Config::new` to look like Listing 13-26. This still won’t compile because we need to update the function body.

Filename: src/lib.rs

```rust
impl Config {
    pub fn new(mut args: env::Args) -> Result<Config, &'static str> {
        // --snip--
```

Listing 13-26: Updating the signature of `Config::new` to expect an iterator

The standard library documentation for the `env::args` function shows that the type of the iterator it returns is `std::env::Args`. We’ve updated the signature of the `Config::new` function so the parameter `args` has the type `std::env::Args` instead of `&[String]`. Because we’re taking ownership of `args` and we’ll be mutating `args` by iterating over it, we can add the `mut` keyword into the specification of the `args` parameter to make it mutable.

<u>We also needed to specify that the string slice error type can now only have the `'static` lifetime</u>. **Because we’re only ever returning string literals**, this was true before. However, <u>when we had a reference in the parameters, there was the possibility that the reference in the return type could have had the same lifetime as the reference in the parameters. The rules that we discussed in the [“Lifetime Elision”](https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html#lifetime-elision) section of Chapter 10 applied, and we weren’t required to annotate the lifetime of `&str`. With the change to `args`, the lifetime elision rules no longer apply, and we must specify the `'static` lifetime.</u>

#### Using `Iterator` Trait Methods Instead of Indexing

Next, we’ll fix the body of `Config::new`. The standard library documentation also mentions that `std::env::Args` implements the `Iterator` trait, so we know we can call the `next` method on it! Listing 13-27 updates the code from Listing 12-23 to use the `next` method:

Filename: src/lib.rs

```rust
impl Config {
    pub fn new(mut args: env::Args) -> Result<Config, &'static str> {
        args.next();

        let query = match args.next() {
            Some(arg) => arg,
            None => return Err("Didn't get a query string"),
        };

        let filename = match args.next() {
            Some(arg) => arg,
            None => return Err("Didn't get a file name"),
        };

        let case_sensitive = env::var("CASE_INSENSITIVE").is_err();

        Ok(Config {
            query,
            filename,
            case_sensitive,
        })
    }
}
```

Listing 13-27: Changing the body of `Config::new` to use iterator methods

**Remember that the first value in the return value of `env::args` is the name of the program**. We want to ignore that and get to the next value, so first we call `next` and do nothing with the return value. Second, we call `next` to get the value we want to put in the `query` field of `Config`. If `next` returns a `Some`, we use a `match` to extract the value. If it returns `None`, it means not enough arguments were given and we return early with an `Err` value. We do the same thing for the `filename` value.

### Making Code Clearer with Iterator Adaptors

We can also take advantage of iterators in the `search` function in our I/O project, which is reproduced here in Listing 13-28 as it was in Listing 12-19:

Filename: src/lib.rs

```rust
pub fn search<'a>(query: &str, contents: &'a str) -> Vec<&'a str> {
    let mut results = Vec::new();

    for line in contents.lines() {
        if line.contains(query) {
            results.push(line);
        }
    }

    results
}
```

Listing 13-28: The implementation of the `search` function from Listing 12-19

We can write this code in a more concise way using iterator adaptor methods. Doing so also lets us avoid having a mutable intermediate `results` vector. The functional programming style prefers to minimize the amount of mutable state to make code clearer. <u>Removing the mutable state might enable a future enhancement to make searching happen in parallel, because we wouldn’t have to manage concurrent access to the `results` vector</u>. Listing 13-29 shows this change:

Filename: src/lib.rs

```rust
pub fn search<'a>(query: &str, contents: &'a str) -> Vec<&'a str> {
    contents
        .lines()
        .filter(|line| line.contains(query))
        .collect()
}
```

Listing 13-29: Using iterator adaptor methods in the implementation of the `search` function

Recall that the purpose of the `search` function is to return all lines in `contents` that contain the `query`. Similar to the `filter` example in Listing 13-19, this code uses the `filter` adaptor to keep only the lines that `line.contains(query)` returns `true` for. We then collect the matching lines into another vector with `collect`. Much simpler! Feel free to make the same change to use iterator methods in the `search_case_insensitive` function as well.

The next logical question is which style you should choose in your own code and why: the original implementation in Listing 13-28 or the version using iterators in Listing 13-29. **Most Rust programmers prefer to use the iterator style**. It’s a bit tougher to get the hang of at first, but once you get a feel for the various iterator adaptors and what they do, <u>iterators can be easier to understand</u>. Instead of fiddling with the various bits of looping and building new vectors, the code focuses on the high-level objective of the loop. This abstracts away some of the commonplace code so it’s easier to see the concepts that are unique to this code, such as the filtering condition each element in the iterator must pass.

**But are the two implementations truly equivalent? The intuitive assumption might be that the more low-level loop will be faster. Let’s talk about performance**.

## 13.4 Comparing Performance: Loops vs. Iterators

To determine whether to use loops or iterators, you need to know which implementation is faster: the version of the `search` function with an explicit `for` loop or the version with iterators.

We ran a benchmark by loading the entire contents of *The Adventures of Sherlock Holmes* by Sir Arthur Conan Doyle into a `String` and looking for the word *the* in the contents. Here are the results of the benchmark on the version of `search` using the `for` loop and the version using iterators:

```text
test bench_search_for  ... bench:  19,620,300 ns/iter (+/- 915,700)
test bench_search_iter ... bench:  19,234,900 ns/iter (+/- 657,200)
```

The iterator version was slightly faster! We won’t explain the benchmark code here, because the point is not to prove that the two versions are equivalent but to get a general sense of how these two implementations compare performance-wise.

在不同场景下，有时候迭代器遍历快，有时候for循环快，iterator迭代器编译后的产物和我们自己for循环代码类似，需要我们自己酌情比较运行效率。

Iterator迭代器是Rust的zero-cost抽象中的一个，意味着运行时不会有任何额外开销。

For a more comprehensive benchmark, you should check using various texts of various sizes as the `contents`, different words and words of different lengths as the `query`, and all kinds of other variations. **<u>The point is this: iterators, although a high-level abstraction, get compiled down to roughly the same code as if you’d written the lower-level code yourself</u>**. **Iterators are one of Rust’s *zero-cost abstractions*, by which we mean using the abstraction imposes no additional runtime overhead**. This is analogous to how Bjarne Stroustrup, the original designer and implementor of C++, defines *zero-overhead* in “Foundations of C++” (2012):

> **In general, C++ implementations obey the zero-overhead principle: What you don’t use, you don’t pay for. And further: <u>What you do use, you couldn’t hand code any better</u>**.

As another example, the following code is taken from an audio decoder. The decoding algorithm uses the linear prediction mathematical operation to estimate future values based on a linear function of the previous samples. This code uses an iterator chain to do some math on three variables in scope: a `buffer` slice of data, an array of 12 `coefficients`, and an amount by which to shift data in `qlp_shift`. We’ve declared the variables within this example but not given them any values; although this code doesn’t have much meaning outside of its context, it’s still a concise, real-world example of how Rust translates high-level ideas to low-level code.

```rust
let buffer: &mut [i32];
let coefficients: [i64; 12];
let qlp_shift: i16;

for i in 12..buffer.len() {
    let prediction = coefficients.iter()
                                 .zip(&buffer[i - 12..i])
                                 .map(|(&c, &s)| c * s as i64)
                                 .sum::<i64>() >> qlp_shift;
    let delta = buffer[i];
    buffer[i] = prediction as i32 + delta;
}
```

To calculate the value of `prediction`, this code iterates through each of the 12 values in `coefficients` and uses the `zip` method to pair the coefficient values with the previous 12 values in `buffer`. Then, for each pair, we multiply the values together, sum all the results, and shift the bits in the sum `qlp_shift` bits to the right.

Calculations in applications like audio decoders often prioritize performance most highly. Here, we’re creating an iterator, using two adaptors, and then consuming the value. What assembly code would this Rust code compile to? Well, as of this writing, it compiles down to the same assembly you’d write by hand. There’s no loop at all corresponding to the iteration over the values in `coefficients`: Rust knows that there are 12 iterations, so it “unrolls” the loop. *Unrolling* is an optimization that removes the overhead of the loop controlling code and instead generates repetitive code for each iteration of the loop.

使用迭代器，编译时Rust就能知道边界，并且进行一些优化，使得运行时能够更高效。

**All of the coefficients get stored in registers, which means accessing the values is very fast. There are no bounds checks on the array access at runtime**. All these optimizations that Rust is able to apply make the resulting code extremely efficient. Now that you know this, you can use iterators and closures without fear! They make code seem like it’s higher level but don’t impose a runtime performance penalty for doing so.

### Summary

Closures and iterators are Rust features inspired by functional programming language ideas. They contribute to Rust’s capability to clearly express high-level ideas at low-level performance. **<u>The implementations of closures and iterators are such that runtime performance is not affected</u>**. <u>This is part of Rust’s goal to strive to provide zero-cost abstractions</u>.

Now that we’ve improved the expressiveness of our I/O project, let’s look at some more features of `cargo` that will help us share the project with the world.

# 14. More About Cargo and Crates.io

> [More About Cargo and Crates.io](https://doc.rust-lang.org/book/ch14-00-more-about-cargo.html#more-about-cargo-and-cratesio)

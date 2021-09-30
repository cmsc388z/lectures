# Enums, pattern matching, Collections, Error handling

Source: [Rust Book](https://doc.rust-lang.org/book/ch06-00-enums.html)

## Enums

With enums you can define a type that has multiple possible variants. We'll first look at how enums can be defined, then we'll look at a special enum, called Option, which which expresses that a value can be either something or nothing. Then we’ll look at how pattern matching in the match expression makes it easy to run different code for different values of an enum. Finally, we’ll cover how the if let construct is another convenient and concise idiom available to you to handle enums in your code.

Enums are a feature in many languages, but their capabilities differ in each language. Rust’s enums are most similar to algebraic data types in functional languages, such as F#, OCaml, and Haskell.

### Defining and enum

```rust
enum IpAddrKind {
    V4,
    V6,
}
```

We can create instances of each of the two variants of IpAddrKind like this:

```rust
let four = IpAddrKind::V4;
let six = IpAddrKind::V6;
```

We can also give our enums a field. This new definition of the IpAddr enum says that both V4 and V6 variants will have associated String values.

```rust
enum IpAddr {
    V4(String),
    V6(String),
}

let home = IpAddr::V4(String::from("127.0.0.1"));

let loopback = IpAddr::V6(String::from("::1"));
```

Or multiple fields.

```rust
enum IpAddr {
    V4(u8, u8, u8, u8),
    V6(String),
}

let home = IpAddr::V4(127, 0, 0, 1);

let loopback = IpAddr::V6(String::from("::1"));
```

### The Option enum

The Option type is used in many places because it encodes the very common scenario in which a value could be something or it could be nothing. Expressing this concept in terms of the type system means the compiler can check whether you’ve handled all the cases you should be handling; this functionality can prevent bugs that are extremely common in other programming languages.

Programming language design is often thought of in terms of which features you include, but the features you exclude are important too. Rust doesn’t have the null feature that many other languages have. Null is a value that means there is no value there. In languages with null, variables can always be in one of two states: null or not-null.

In his 2009 presentation “Null References: The Billion Dollar Mistake,” Tony Hoare, the inventor of null, has this to say:

> I call it my billion-dollar mistake. At that time, I was designing the first comprehensive type system for references in an object-oriented language. My goal was to ensure that all use of references should be absolutely safe, with checking performed automatically by the compiler. But I couldn’t resist the temptation to put in a null reference, simply because it was so easy to implement. This has led to innumerable errors, vulnerabilities, and system crashes, which have probably caused a billion dollars of pain and damage in the last forty years.

The problem with null values is that if you try to use a null value as a not-null value, you’ll get an error of some kind. Because this null or not-null property is pervasive, it’s extremely easy to make this kind of error.

As such, Rust does not have nulls, but it does have an enum that can encode the concept of a value being present or absent. This enum is `Option<T>`, and it is defined by the standard library as follows:

```rust
enum Option<T> {
    None,
    Some(T),
}
```

The `<T>` syntax is a feature of Rust we haven’t talked about yet. It’s a generic type parameter. 

Here's some basic usage of the option type. 

```rust
let mut some_number = Some(5);
let mut some_string = Some("a string");

some_number = None;
some_string = None;

let mut absent_number: Option<i32> = None
absent_number = Some(5);
```

You cannot simply use an `Option<T>` type the same way you would type T. The following example, where an `Option<i8>` is added to an i8 won't compile. 

```rust
let x: i8 = 5;
let y: Option<i8> = Some(5);

let sum = x + y;
```

With the error:

```rust
$ cargo run
   Compiling enums v0.1.0 (file:///projects/enums)
error[E0277]: cannot add `Option<i8>` to `i8`
 --> src/main.rs:5:17
  |
5 |     let sum = x + y;
  |                 ^ no implementation for `i8 + Option<i8>`
  |
  = help: the trait `Add<Option<i8>>` is not implemented for `i8`

error: aborting due to previous error

For more information about this error, try `rustc --explain E0277`.
error: could not compile `enums`

To learn more, run the command again with --verbose.
```

In order to add these values you have to convert an `Option<T>` to a T. Generally, this helps catch one of the most common issues with null: assuming that something isn’t null when it actually is.

## Pattern matching

One of Rust's most important control flow operators is called match, which allows you to compare some value against a series of pattens and then execute code based on which pattern matches. Patterns can be made up of literal values, variable names, wildcards, and many other things; Chapter 18 covers all the different kinds of patterns and what they do. The power of match comes from the expressiveness of the patterns and the fact that the compiler confirms that all possible cases are handled.

```rust
fn main() {
        enum IpAddr {
            V4(u8, u8, u8, u8),
            V6(String),
        }

        let home = IpAddr::V4(127, 0, 0, 1);
        let loopback = IpAddr::V6(String::from("::1"));

        match home {
                IpAddr::V4(a, b, c, d) => println!("Is V4"),
                IpAddr::V6(a) => println!("Is V6")
        };
}
```

This example shows a basic match with the ipaddr enum we created previously. We can also match with specific values in the pattern match.

```rust
fn main() {
        enum IpAddr {
            V4(u8, u8, u8, u8),
            V6(String),
        }

        let home = IpAddr::V4(127, 0, 0, 1);
        let loopback = IpAddr::V6(String::from("::1"));

        match home {
                IpAddr::V4(a, b, c, d) => println!("Is V4"),
                IpAddr::V4(127, b, c, d) => println!("Is V4 loopback"),
                IpAddr::V6(a) => println!("Is V6")
        };
}
```

Rust will require that our matches be *exaustive*. 

```rust
fn main() {
        enum IpAddr {
            V4(u8, u8, u8, u8),
            V6(String),
        }

        let home = IpAddr::V4(127, 0, 0, 1);
        let loopback = IpAddr::V6(String::from("::1"));

        match home {
                IpAddr::V4(a, b, c, d) => println!("Is V4"),
                IpAddr::V4(127, b, c, d) => println!("Is V4 loopback"),
        };
}
```

For example, if we don't attempt to match with the V6 variant, the compiler will throw an error. 

```rust
error[E0004]: non-exhaustive patterns: `V6(_)` not covered
  --> temp.rs:10:8
   |
2  | /     enum IpAddr {
3  | |         V4(u8, u8, u8, u8),
4  | |         V6(String),
   | |         -- not covered
5  | |     }
   | |_____- `main::IpAddr` defined here
...
10 |       match home {
   |             ^^^^ pattern `V6(_)` not covered
   |
   = help: ensure that all possible cases are being handled, possibly by adding wildcards or more match arms
   = note: the matched value is of type `main::IpAddr`
```

We don't have to match explicitly however. We can use _ to catch all remaining cases. 

```rust
fn main() {
        enum IpAddr {
            V4(u8, u8, u8, u8),
            V6(String),
        }

        let home = IpAddr::V4(127, 0, 0, 1);
        let loopback = IpAddr::V6(String::from("::1"));

        match home {
                IpAddr::V4(a, b, c, d) => println!("Is V4"),
                IpAddr::V4(127, b, c, d) => println!("Is V4 loopback"),
                _ => println!("Is V6")
        };
}
```

Pattern matches can also be used to set variables as shown below. 

```rust
fn main() {
        enum IpAddr {
            V4(u8, u8, u8, u8),
            V6(String),
        }

        let home = IpAddr::V4(127, 0, 0, 1);
        let loopback = IpAddr::V6(String::from("::1"));

        let loopback = match home {
                IpAddr::V4(127, b, c, d) => Some(home),
                _ => None
        };
}
```

### Concise control flow with `if` `let`

The if let syntax lets you combine if and let into a less verbose way to handle values that match one pattern while ignoring the rest. Without it we might do something like this.

```rust
    let some_u8_value = Some(0u8);
    match some_u8_value {
        Some(3) => println!("three"),
        _ => (),
    }
```

We want to do something with the `Some(3)` match but do nothing with any other `Some<u8>` value or the None value. To satisfy the match expression, we have to add _ => () after processing just one variant, which is a lot of boilerplate code to add.

Instead, we could write this in a shorter way using if let.

```rust
    let some_u8_value = Some(0u8);
    if let Some(3) = some_u8_value {
        println!("three");
    }
```

### Error handling

Rust groups errors into two major categories: recoverable and unrecoverable errors. For a recoverable error, such as a file not found error, it’s reasonable to report the problem to the user and retry the operation. Unrecoverable errors are always symptoms of bugs, like trying to access a location beyond the end of an array.

Rust doesn’t have exceptions. Instead, it has the type `Result<T, E>` for recoverable errors and the panic! macro that stops execution when the program encounters an unrecoverable error.

Result enum is defined as having two variants, Ok and Err, as follows:

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

Let’s call a function that returns a Result value because the function could fail. In Listing 9-3 we try to open a file.

```rust
use std::fs::File;

fn main() {
    let f = File::open("hello.txt");
}
```

How do we know what type File::open returns? We can find out by asking the compiler, or with online documentation.

```rust
    let f: u32 = File::open("hello.txt");
```

This will produce an error since the method returns a Result type. Since it's an enum, we can match with it as shown below.

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

We can also nest our matching expressions to match with a specific kind of error.

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

The type of the value that File::open returns inside the Err variant is io::Error, which is a struct provided by the standard library. This struct has a method kind that we can call to get an io::ErrorKind value. The enum io::ErrorKind is provided by the standard library and has variants representing the different kinds of errors that might result from an io operation. The variant we want to use is ErrorKind::NotFound, which indicates the file we’re trying to open doesn’t exist yet. So we match on f, but we also have an inner match on error.kind().

### Shortcuts for panic on error

Using match can sometimes be a bit verbose, and doesn't always express the intent of a code block well. In some cases we may prefer to use the `unwrap()` or `expect()` methods. 

The Result<T, E> type has many helper methods defined on it to do various tasks. One of those methods, called unwrap, is a shortcut method. 

```rust
use std::fs::File;

fn main() {
    let f = File::open("hello.txt").unwrap();
}
```

If we run this code and the file doesn't exist, it will panic with the following error. 

```rust
thread 'main' panicked at 'called `Result::unwrap()` on an `Err` value: Error {
repr: Os { code: 2, message: "No such file or directory" } }',
src/libcore/result.rs:906:4
```

We can also use `expect()`, to print a specific message when our code runs. 

```rust
use std::fs::File;

fn main() {
    let f = File::open("hello.txt").expect("Failed to open hello.txt");
}
```

If we run this example we get

```
thread 'main' panicked at 'Failed to open hello.txt: Error { repr: Os { code:
2, message: "No such file or directory" } }', src/libcore/result.rs:906:4
```

### The panic macro

We briefly above saw the `panic!` macro. This macro is used to generate an unrecoverable error. Here is an example. 

```rust
fn main() {
    panic!("crash and burn");
}
```

Your programs may also panic for reasons like accessing an out-of-bounds vector element.

```rust
fn main() {
    let v = vec![1, 2, 3];

    v[99];
}
```

### Error propogation

When writing a function whose implementation may fail, it is generally advised to return teh error to the calling code as opposed to handling it in the function, so the calling code has more control. This is known as propogating the error. 

A naive way to accompolish this is shown below.

```rust
use std::fs::File;
use std::io;
use std::io::Read;

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

Using the match expressions here is quite verbose. To make this more concise, we can use the `?` operator, which makes the above example much more concise.

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

This operator should only be used on functions that return a result. If we attempt to use this in the main function, for example, the compiler will throw an error. 

### Panic vs Result

How do you know when to call `panic!` and when to return a `Result`? When you choose to return a Result value, you give the calling code options rather than making the decision for it. The calling code could choose to attempt to recover in a way that’s appropriate for its situation, or it could decide that an Err value in this case is unrecoverable, so it can call panic! and turn your recoverable error into an unrecoverable one. Therefore, returning Result is a good default choice when you’re defining a function that might fail.

## Errors in collections

Rust’s standard library includes a number of very useful data structures called collections. Most other data types represent one specific value, but collections can contain multiple values. Unlike the built-in array and tuple types, the data these collections point to is stored on the heap, which means the amount of data does not need to be known at compile time and can grow or shrink as the program runs. 

We've already spent a lot of time using the `Vec` type in the class examples and projects. As a reminder, vectors can be initialized in multiple ways:

```rust
let mut v1 = Vec::new();
let mut v2 = vec![1, 2, 3];
```

Elements can be acccessed using `[]`, or with the `.get()` method. 

```rust
fn main() {

    let num = vec![10, 20];
    
    for x in num.iter() {
        println!("{}", x);
    }
    
    println!("num[0]: {}", num[0]);
    println!("num[1]: {}", num[1]);
    println!("num[2]: {}", num.get(0).unwrap());
    println!("num[3]: {}", num.get(1).unwrap());
}
```

So what's the difference between using `[]` and using `.get()`? The difference is that `.get()` returns an `Option`, as introduced earlier. 

```rust
fn main() {

    let num = vec![10, 20];

    println!("num[0]: {}", num[0]);
    println!("num[1]: {}", num[1]);
    println!("num[2]: {}", num.get(0).unwrap());
    println!("num[3]: {}", num.get(1).unwrap());
    
    if let Some(num) = num.get(2) {
        println!("Found a value at index");
    } else {
        println!("Found no value at index");
    }
}
```

In this example, `num.get(2)` will return `None`. 

If we access the element using `[]`, however, our program will panic.

```rust
fn main() {

    let num = vec![10, 20];

    println!("num[0]: {}", num[0]);
    println!("num[1]: {}", num[1]);
    println!("num[2]: {}", num.get(0).unwrap());
    println!("num[3]: {}", num.get(1).unwrap());
    
    let temp = num[2];
}
```


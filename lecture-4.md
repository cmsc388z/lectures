# Lecture 4: Rust build system. Modules, crates. Testing.

## Lifetimes review

```rust
fn main() {
  let ret = get_value();
  println!(“The return value is ”, ret);
}

fn get_value() -> &String {
  let s1 = String::from(“hello”);
  &s1
}
```
Will this code compile? Why?


```rust
fn main() {
  let val = String::from("Hello world");
  
  {
    let val2 = &val;
    println!(“Second value is: {}”, val2);
  }
}
```
Will this code compile? Why?


```rust
fn main() {
  let val;
  {
    let val2 = String::from("Hello world");
    val = &x;
  }
  println!(“Value is {}”, val);
}
```
Will this code compile? Why?


```rust
pub fn main() {
{ 
   let x = String::from(“Hello world”);
   let z;
   {
      let y = String::from(“there”);
      z = longer(&x,&y);
    }
    println!(“z is {}”,z);
}


fn longer(x:&str, y:&str) -> &str {
  if x.len() > y.len() { x } else { y }
}
```
Will this code compile? Why?

## Build system
### Some review

Important cargo commands:
 - check: Check package and dependencies for errors, without doing code generation
 - build: Check package and dependencies for errors and build the package
 - run: Build and run
 - fmt: Format rust code
 - clippy: Collection of checks for common mistakes in Rust code
 - clean: Clean directories
 - test: Run integration or unit tests

### Build scripts

Some packages need to compile third-party non-Rust code, for example C libraries. Other packages need to link to C libraries which can either be located on the system or possibly need to be built from source. Others still need facilities for functionality such as code generation before building (think parser generators).

Cargo does not aim to replace other tools that are well-optimized for these tasks, but it does integrate with them with custom build scripts. Placing a file named build.rs in the root of a package will cause Cargo to compile that script and execute it just before building the package.

Here is an example build script.

```rust
fn main() {
    println!("cargo:rerun-if-changed=src/hello.c");
    
    cc::Build::new()
        .file("src/hello.c")
        .compile("hello");
}
```

These may be used in cases where you need to
 - Bundle a C library
 - Gnerate a rust module from a specification
 - Perform platform-specific configuration for the crate
 - Generate bindings to some other langauge

Just before a package is built, Cargo will compile a build script into an executable. It will then run the script, which may perform any number of tasks. The script may communicate with Cargo by printing specially formatted commands prefixed with cargo: to stdout.

Here are the most important instructions recognized by cargo. 

 - cargo:rerun-if-changed=PATH — Tells Cargo when to re-run the script.
 - cargo:rerun-if-env-changed=VAR — Tells Cargo when to re-run the script.
 - cargo:rustc-link-lib=[KIND=]NAME — Adds a library to link.
 - cargo:rustc-link-search=[KIND=]PATH — Adds to the library search path.
 - cargo:rustc-flags=FLAGS — Passes certain flags to the compiler.
 - cargo:rustc-cfg=KEY[="VALUE"] — Enables compile-time cfg settings.
 - cargo:rustc-env=VAR=VALUE — Sets an environment variable.
 - cargo:rustc-cdylib-link-arg=FLAG — Passes custom flags to a linker for cdylib crates.
 - cargo:warning=MESSAGE — Displays a warning on the terminal.
 - cargo:KEY=VALUE — Metadata, used by links scripts.

These are the build scripts outputs. Inputs to the build scripts are environment variables. Here is an example that uses them.

```rust
// build.rs

use std::env;
use std::fs;
use std::path::Path;

fn main() {
    let out_dir = env::var_os("OUT_DIR").unwrap();
    let dest_path = Path::new(&out_dir).join("hello.rs");
    fs::write(
        &dest_path,
        "pub fn message() -> &'static str {
            \"Hello, World!\"
        }
        "
    ).unwrap();
    println!("cargo:rerun-if-changed=build.rs");
}
```

## Modules and Crates
As you write large programs, organizing your code will be important because keeping track of your entire program in your head will become impossible. By grouping related functionality and separating code with distinct features, you’ll clarify where to find code that implements a particular feature and where to go to change how a feature works.

The programs we’ve written so far have been in one module in one file. As a project grows, you can organize code by splitting it into multiple modules and then multiple files. A package can contain multiple binary crates and optionally one library crate.

Rust has a number of features that allow you to manage your code’s organization, including which details are exposed, which details are private, and what names are in each scope in your programs. These features, sometimes collectively referred to as the module system, include:

 - Packages: A Cargo feature that lets you build, test, and share crates
 - Crates: A tree of modules that produces a library or executable
 - Modules and use: Let you control the organization, scope, and privacy of paths
 - Paths: A way of naming an item, such as a struct, function, or module

### Some review
```
$ cargo new my-project
     Created binary (application) `my-project` package
$ ls my-project
Cargo.toml
src
$ ls my-project/src
main.rs
```

When we entered the command, Cargo created a Cargo.toml file, giving us a package. Looking at the contents of Cargo.toml, there’s no mention of src/main.rs because Cargo follows a convention that src/main.rs is the crate root of a binary crate with the same name as the package. Likewise, Cargo knows that if the package directory contains src/lib.rs, the package contains a library crate with the same name as the package, and src/lib.rs is its crate root. Cargo passes the crate root files to rustc to build the library or binary.

Here, we have a package that only contains src/main.rs, meaning it only contains a binary crate named my-project. If a package contains src/main.rs and src/lib.rs, it has two crates: a library and a binary, both with the same name as the package. A package can have multiple binary crates by placing files in the src/bin directory: each file will be a separate binary crate.

### Modules

Modules let us organize code within a crate into groups for readability and easy reuse. Modules also control the privacy of items, which is whether an item can be used by outside code (public) or is an internal implementation detail and not available for outside use (private).

```
cargo new --lib shapes
```

We can define a module using the `mod` keyword followed by the name of the module. 

in `src/lib.rs`

```rust
mod shapes {
        mod rectangles {
                struct Rect {
                        width: u32,
                        height: u32
                }

                impl Rect {
                        fn get_area(&self) -> u32 {}
                        fn get_perimeter(&self) -> u32 {}
                }

        }
}
```

Modules allow us to group related definitions together. 

### Module tree

Using `mod` we can build a tree of definitions, which is often called the module tree. Paths to definitions in the module tree can take two forms:

 - An absolute path starts from a crate root by using a crate name or a literal crate.
 - A relative path starts from the current module and uses self, super, or an identifier in the current module.

For example, from the previous `lib.rs` we could reference `Rect` with

```rust
pub mod shapes {
        pub mod rectangles {
                pub struct Rect {
                        pub width: u32,
                        pub height: u32
                }

                impl Rect {
                        pub fn get_area(&self) -> u32 {self.width * self.height }
                        pub fn get_perimeter(&self) -> u32 {self.width * 2 + self.height * 2 }
                }

        }
}

pub fn create_rectangle() {
        crate::shapes::rectangles::Rect {
                width: 5,
                height: 5
        };
}
```

To make this example compile, each nested module and function must be marked as `pub`. 

We can also construct relative paths using the `super` or `self` keywords. Here's an example of the former.

```rust
pub mod shapes {
        pub fn new_rect(width: u32, height: u32) -> rectangles::Rect {
                rectangles::Rect {
                        width,
                        height
                }
        }

        pub mod rectangles {
                pub struct Rect {
                        pub width: u32,
                        pub height: u32
                }

                impl Rect {
                        pub fn get_area(&self) -> u32 {self.width * self.height }
                        pub fn get_perimeter(&self) -> u32 {self.width * 2 + self.height * 2 }
                }

        }
}

pub fn create_rectangle() {
        crate::shapes::rectangles::Rect {
                width: 5,
                height: 5
        };

        shapes::new_rect(5, 5);
}
```

#### Using pub with structs and enums

We can also use pub to designate structs and enums as public, but there are a few extra details. If we use pub before a struct definition, we make the struct public, but the struct’s fields will still be private. We can make each field public or not on a case-by-case basis.

Enums, on the other hand, have all their variants public if the enum itself is prefaced with `pub`.

### The use keyword

Using a semicolan after the `mod temp` rather than a block tells rust to load the contents of the module from another file with the same name as the module. We can then bring specific definitions into scope with the `use` keyword. 

In `temp.rs`
```rust
pub mod shapes {
        pub mod ovals {
                pub struct Circle { 
                        pub radius: u32
                }
        }
}
```

In `lib.rs`
```rust
mod temp;

pub mod shapes {
        pub mod rectangles {
                pub struct Rect {
                        pub width: u32,
                        pub height: u32
                }

                impl Rect {
                        pub fn get_area(&self) -> u32 {self.width * self.height }
                        pub fn get_perimeter(&self) -> u32 {self.width * 2 + self.height * 2 }
                }

        }
}

use temp::shapes::*;

pub fn create() {
        crate::shapes::rectangles::Rect {
                width: 5,
                height: 5
        };

        ovals::Circle {
                radius: 5
        };
}
```

### Using crates

Standard crates can be brought into scope with the use keyword.

```rust
use std::collections::HashMap;

fn main() {
        let mut map = HashMap::new();
        map.insert(1, 1);
}
```

Sometimes you might want to use libraries not contained within `std`. To do this, we'll need to use `Cargo.toml`.

The Cargo.toml file for each package is called its manifest. It is written in the TOML format. Every manifest file consists of many sections, some of which are listed below. 

 - cargo-features — Unstable, nightly-only features.
 - [package] — Defines a package.
   - name — The name of the package.
   - version — The version of the package.
   - authors — The authors of the package.
   - edition — The Rust edition.
   - description — A description of the package.
   - documentation — URL of the package documentation.
   - readme — Path to the package's README file.

We can look at the `Cargo.toml` from the snake game project. This example uses dependencies from the [crates.io](https://crates.io/) registry. 

```
[package]
name = "cmsc388z-snake-game"
version = "0.1.0"

[dependencies]
piston_window = "0.120.0"
rand = "0.3"
```

We can also depend on a library located in a `git` repository. Your package can depend on any specified version or commit. 

```
[dependencies]
piston_window = { version = "0.120.0", git = "https://github.com/PistonDevelopers/piston" }
rand = { git = "https://github.com/example/rand", rev = "9f35b8e" }
```

Or a local directory. 

```
[dependencies]
piston_window = { version = "0.120.0", path = "~/github/piston" }
```


To use these crates we must first load them like in `main.rs`.

```
extern crate piston_window;
extern crate rand;
```

Then they can be brought into scope like in `game.rs`.

```
use piston_window::*;
use rand::{thread_rng, Rng};
```

### Cargo.lock

You may see `Cargo.lock` file appear in your package directory. `Cargo.toml` and `Cargo.lock` serve two different purposes

 - Cargo.toml is about describing your dependencies in a broad sense, and is written by you.
 - Cargo.lock contains exact information about your dependencies. It is maintained by Cargo and should not be manually edited.

## Testing

A test in Rust is a function that’s annotated with the test attribute. Attributes are metadata about pieces of Rust code; one example is the derive attribute we used with structs in Chapter 5. To change a function into a test function, add #[test] on the line before fn. When you run your tests with the cargo test command, Rust builds a test runner binary that runs the functions annotated with the test attribute and reports on whether each test function passes or fails.

When we make a new library project with Cargo, a test module with a test function in it is automatically generated for us. This module helps you start writing your tests so you don’t have to look up the exact structure and syntax of test functions every time you start a new project.

We should now have the knowledge to understand the auto-generated code.

```
$ cargo new adder --lib
     Created library `adder` project
$ cd adder
```

Then in `src/lib.rs`. 

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn it_works() {
        assert_eq!(2 + 2, 4);
    }
}
```

Previously we've had you all write unit tests. Unit tests are testing one module in isolation at a time: they're small and can test private code. This is in contrast to integration tests, which are external to your crate and use only its public interface in the same way any other code would. Their purpose is to test that many parts of your library work correctly together.

In `src/lib.rs`
```rust
// Define this in a crate called `adder`.
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

In `tests/integration_test.rs`
```rust
#[test]
fn test_add() {
    assert_eq!(adder::add(3, 2), 5);
}
```

Then we can run the integration tests.

```
$ cargo test
running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out

     Running target/debug/deps/integration_test-bcd60824f5fbfe19

running 1 test
test test_add ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out

   Doc-tests adder

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```


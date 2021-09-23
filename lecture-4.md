# Lecture 4: Rust build system. Modules, crates. Testing.

## Rust build system


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

```
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

```
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
```
pub mod shapes {
        pub mod ovals {
                pub struct Circle { 
                        pub radius: u32
                }
        }
}
```

In `lib.rs`
```
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

```
use std::collections::HashMap;

fn main() {
        let mut map = HashMap::new();
        map.insert(1, 1);
}
```

Sometimes you might want to use libraries not contained within `std`. To do this, we'll need to use `Cargo.toml`.

The Cargo.toml file for each package is called its manifest. It is written in the TOML format. Every manifest file consists of the following sections:

 - cargo-features — Unstable, nightly-only features.
 - [package] — Defines a package.
   - name — The name of the package.
   - version — The version of the package.
   - authors — The authors of the package.
   - edition — The Rust edition.
   - description — A description of the package.
   - documentation — URL of the package documentation.
   - readme — Path to the package's README file.

## Testing


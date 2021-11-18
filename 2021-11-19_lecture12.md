# week 12

Refence: 
- [The Rust Reference](https://doc.rust-lang.org/reference/macros-by-example.html)
- [TRPL](https://doc.rust-lang.org/book/ch19-06-macros.html)
- [RBE](https://doc.rust-lang.org/rust-by-example/macros.html)
- [The Little Book of Rust Macros](https://veykril.github.io/tlborm/proc-macros/methodical.html)

This week we will talk about macro.

Fundamentally, macros are a way of *writing code that writes other code*, which is known as metaprogramming.

Some frequently used macros are `vec!` and `println!`. All of these macros expand at compile time to produce more code than the code you’ve written manually. Because of this, macro definitions are usually more complicated than function definitions.

# The difference between macros and functions

- common point:
    Metaprogramming and function are both useful for reducing the amount of code you have to write and maintain.

- difference:
    1. A function signature *must* declare the number and type of parameters the function has. Macros, on the other hand, can take a *variable* number of parameters.
    2. macros are *expanded* before the compiler interprets the meaning of the code, so a macro can, for example, implement a trait on a given type. A function can’t, because it gets called at *runtime* and a trait needs to be implemented at compile time.
    3. Macro definitions are usually more complex and you have to bring them into scope before you can call them in a file.

There are three kinds of macros
* Custom `#[derive]` macros that specify code added with the derive attribute used on structs and enums
* Attribute-like macros that define custom attributes usable on any item
* Function-like macros that look like function calls but operate on the tokens specified as their argument

# Declarative Macros for General Metaprogramming

The most widely used form of macros in Rust is `declarative macros`. For detailed usage, please check its [document](https://doc.rust-lang.org/reference/macros-by-example.html).

At their core, declarative macros allow you to write something similar to a Rust `match` expression. In this situation, the value is the literal Rust source code passed to the macro; the patterns are compared with the structure of that source code; and the code associated with each pattern, when matched, replaces the code passed to the macro. This all happens during compilation.

To define a macro, you use the `macro_rules!` construct. Let’s explore how to use `macro_rules!` by looking at how the `vec!` macro is defined. 

```rust
#![allow(unused)]
fn main() {
let v1: Vec<u32> = vec![1, 2];
let v2: Vec<u32> = vec![1, 2, 3];
let v3: Vec<u32> = vec![1, 2, 3, 4];
}
```
Have you ever think about why you can pass any number of parameters you want to `vec!`? This is the magic of macro! A slightly simplified definition of the `vec!` macro is defined as:

```rust
#[macro_export] // bring this macro into scope
macro_rules! vec { // similar with fn keyword. {} wraps the macro body.
    ( $( $x:expr ),* ) => { // pattern to match
        {
            let mut temp_vec = Vec::new();
            $(
                temp_vec.push($x);
            )*
            temp_vec
        }
    };
}
```

The valid pattern syntax in macro is different than `match` because macro patterns are matched against Rust code structure rather than values. In the above example,
- First, a set of parentheses encompasses the whole pattern. 
- A dollar sign (`$`) is next, followed by a set of parentheses that captures values that match the pattern within the parentheses for use in the replacement code.
-  Within `$()` is `$x:expr`, which matches any Rust expression and gives the expression the name `$x`. Here the `$x` pattern matches three times with the three expressions 1, 2, and 3.  `expr`, which represents expression, is one of the designators. Other designators include `ident` (var/fn name), `path`, `tt`(token tree), etc.  
- `temp_vec.push()` within `$(),*` is generated for each part that matches `$()` in the pattern zero or more times depending on how many times the pattern matches. `,` is a separator. The `$x` is replaced with each expression matched.  Other repeat symbols include `+` (at least one) and `?` (zero or one)


Here the macro expands the code as:
    ```rust
    {
        let mut temp_vec = Vec::new();
        temp_vec.push(1);
        temp_vec.push(2);
        temp_vec.push(3);
        temp_vec
    }
    ```

We’ve defined a macro that can take *any* number of arguments of *any* type and can generate code to create a vector containing the specified elements.

# Procedural macros

Procedural macros accept some *code* as an input, operate on that code, and produce some *code* as an output rather than matching against patterns and replacing the code with other code as declarative macros do.

There are three kinds of procedural macros
- custom derive 
- attribute-like
- function-like

But overall, all those three types follow the definition:

```rust
use proc_macro;

#[some_attribute] // place holder
pub fn some_name (input: TokenStream) -> TokenStream {
}
```

The name "some_attribute" and "some_name" are place holders, and the type `TokenStream` is a type defined in the `proc_macro` crate and represents a sequence of tokens.

This is the core of the macro: the source code that the macro is operating on makes up the input `TokenStream`, and the code the macro produces is the output `TokenStream`. 

When creating procedural macros, the definitions *must* reside in their own crate with a special crate type. 

## Custom derive Macro

Let’s create a crate named `hello_macro` that defines a trait named `HelloMacro` with one associated function named `hello_macro`.

Rather than making our crate users implement the `HelloMacro` trait for each of their types, we’ll provide a procedural macro so users can annotate their type with `#[derive(HelloMacro)]` to get a default implementation of the `hello_macro` function. The default implementation will print `Hello, Macro! My name is TypeName!` where `TypeName` is the name of the type on which this trait has been defined. In other words, we’ll write a crate that enables another programmer to write code using our crate.

Without procedural macro, we can define a `HelloMacro` trait and then implement this trait for every type:

```rust
pub trait HelloMacro {
    fn hello_macro();
}

struct Pancakes;

impl HelloMacro for Pancakes {
    fn hello_macro() {
        println!("Hello, Macro! My name is Pancakes!");
    }
}

fn main() {
    Pancakes::hello_macro();
}
```

As a smart coder, we don't want to do this for every type. Instead, we want something like


Filename: src/main.rs
```rust=
use hello_macro::HelloMacro;
use hello_macro_derive::HelloMacro;

#[derive(HelloMacro)]
struct Pancakes;

fn main() {
    Pancakes::hello_macro();
}
```

Currently, procedural macros need to be in their own crate. Let’s start a new crate called `hello_macro_derive` inside our `hello_macro` project

```bash
$ cargo new hello_macro_derive --lib
```

We need to declare the `hello_macro_derive` crate as a procedural macro crate. We’ll also need functionality from the `syn` and `quote` crates, so we need to add them as dependencies. Add the following to the *Cargo.toml* file for `hello_macro_derive`:

Filename: hello_macro_derive/Cargo.toml

```rust
proc-macro = true

[dependencies]
syn = "1.0"
quote = "1.0"
```

As for the actual definition, almost all procedural macro has the same parsing step, which is the following:

Filename: hello_macro_derive/src/lib.rs

```rust
extern crate proc_macro;

use proc_macro::TokenStream;
use quote::quote;
use syn;

#[proc_macro_derive(HelloMacro)]
pub fn hello_macro_derive(input: TokenStream) -> TokenStream {
    // Construct a representation of Rust code as a syntax tree
    // that we can manipulate
    let ast = syn::parse(input).unwrap();

    // Build the trait implementation
    impl_hello_macro(&ast)
}
```

The code you specify in the body of the inner function (`impl_hello_macro` in this case) will be different depending on your procedural macro’s purpose.

We’ve introduced three new crates: 
- `proc_macro`
    This crate is the compiler’s API that allows us to read and manipulate Rust code from our code.
* `syn`
    This crate parses Rust code from a string into a data structure that we can perform operations on.
* `quote`
    turns syn data structures back into Rust code. 

The `hello_macro_derive` function will be called when a user of our library specifies `#[derive(HelloMacro)]` on a type. This is possible because we’ve annotated the `hello_macro_derive` function here with `proc_macro_derive` and specified the name, `HelloMacro`, which matches our trait name; this is the convention most procedural macros follow.

The `hello_macro_derive` function first converts the input from a `TokenStream` to a data structure that we can then interpret and perform operations on. This is where `syn` comes into play. The parse function in `syn` takes a `TokenStream` and returns a `DeriveInput` struct representing the parsed Rust code. The relevant parts of the `DeriveInput` struct we get from parsing the `struct Pancakes;` string is:

```rust
DeriveInput {
    // --snip--

    ident: Ident {
        ident: "Pancakes",
        span: #0 bytes(95..103)
    },
    data: Struct(
        DataStruct {
            struct_token: Struct,
            fields: Unit,
            semi_token: Some(
                Semi
            )
        }
    )
}

```

The `DeriveInput` struct is able to describe all sorts of Rust code, but the part we will use is the `ident`, which contains the name of the struct.

Note that the output for our derive macro is also a `TokenStream` and must be a `TokenStream`. The returned `TokenStream` is added to the code that our crate users write, so when they compile their crate, they’ll get the extra functionality that we provide in the modified `TokenStream`.

As for the actual defination of `impl_hello_macro`, it takes the parsed `DeriveInput` as the input and returns a `TokenStream`.

Filename: hello_macro_derive/src/lib.rs
```rust
fn impl_hello_macro(ast: &syn::DeriveInput) -> TokenStream {
    let name = &ast.ident;
    let gen = quote! {
        impl HelloMacro for #name {
            fn hello_macro() {
                println!("Hello, Macro! My name is {}!", stringify!(#name));
            }
        }
    };
    gen.into()
}
```

The variable `name` contains the name of the struct as we described above, which can be printed as "pancakes".


The `quote!` macro lets us define the Rust code that we want to return. The compiler expects something different to the direct result of the `quote!` macro’s execution, so we need to convert it to a `TokenStream`. We do this by calling the `into` method, which consumes this intermediate representation and returns a value of the required `TokenStream` type. The `quote!` macro also provides some very cool templating mechanics: we can enter `#name`, and `quote!` will replace it with the value in the variable name. You can even do some repetition similar to the way regular macros work. Check out [the `quote` crate’s docs](https://docs.rs/quote) for a thorough introduction.

The `stringify!` macro used here is built into Rust. It takes a Rust expression, such as `1 + 2`, and at compile time turns the expression into a string literal, such as `"1 + 2"`. This is different than `format!` or `println!`, macros which *evaluate* the expression and then turn the result into a `String`, so if you give them `1 + 2` they will allocate a string `3`.

Now, let's see how to use our procedural macro.

We create a new binary project named `pancakes` and add the two crates as the dependencises in the `pancakes` crate's *Cargo.toml*

```rust
hello_macro = { path = "../hello_macro" }
hello_macro_derive = { path = "../hello_macro/hello_macro_derive" }
```

### `#[derive()]`

Rust also provides many derivable traits for us.

I will list some:
- The `Debug` trait enables debug formatting in format strings, which you indicate by adding :? within {} placeholders.
- `PartialEq` and `Eq` for Equality Comparisons
- `PartialOrd` and `Ord` for Ordering Comparisons
- `Clone` and `Copy` for Duplicating Values
- `Hash` for Mapping a Value to a Value of Fixed Size
- `Default` for Default Values

## Attribute-like macros

Attribute-like macros are similar to custom derive macros, but instead of generating code for the `derive` attribute, they allow you to create new attributes. Attributes can be applied to structs, enums and functions, etc.

Say we have an attribute `route` that annotates functions when using a we application framework

```rust
#[route(GET, "/")]
fn index() {
```

This `#[route]` attribute would be defined by the framework as a procedural macro. The signature of the macro definition function would look like this:

```rust
#[proc_macro_attribute]
pub fn route(attr: TokenStream,  //  GET,"/" part
             item: TokenStream    // fn index() {}
) -> TokenStream {

```

You create a crate with the `proc-macro` crate type and implement a function that generates the code you want!


##  Function-like macros
Function-like macros are similarly to `macro_rules!` macros, but are more flexible.

Function-like macros take a `TokenStream` parameter and their definition manipulates that `TokenStream` using Rust code as the other two types of procedural macros do. An example of a function-like macro is an `sql!` macro that might be called like so:

```rust
let sql = sql!(SELECT * FROM posts WHERE id=1);
```

This macro would parse the SQL statement inside it and check that it’s syntactically correct, which is much more complex processing than a `macro_rules!` macro can do. The `sql!` macro would be defined like this:

```rust=
#[proc_macro]
pub fn sql(input: TokenStream) -> TokenStream {
```

This definition is similar to the custom derive macro’s signature: we receive the tokens that are inside the parentheses and return the code we wanted to generate.





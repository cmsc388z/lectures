
This week we will talk about **trait** and **polymorphism**.

# Trait

## Generic types

Every programming language has tools for effectively handling the duplication of concepts. In Rust, one such tool is generics. Generics are abstract stand-ins for concrete types or other properties. We have seen many examples of generics, such as `Option<T>`, `Vec<T>`, `HashMap<K, V>`,  `Result<T, E>`.

The purpose of using Generic Data Types is reducing duplication of code. To understand how important generics is, imagine that you need to define a `Option` enum for every single data type you have... 

### Generic in struct/enum

Without generic, you need to define `Option` for every single data type you want to use

```rust
enum Option<u32> {
    Some(u32),
    None,
}
enum Option<i32> {
    Some(u32),
    None,
}
// ... No!
```

As smart coders, we cannot let it happen! So, we can tell rust that we have a generic type `T`, and ask rust to take care of the rest.

```rust
enum Option<T> {
    Some(T),
    None,
}
```

The `Result` enum is generic over two types, `T` and `E`, and has two variants: `Ok`, which holds a value of type `T`, and `Err`, which holds a value of type `E`.

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

If we want the fields in our custom struct to be generic:

```rust
struct Point<T, U> {
    x: T,
    y: U,
}

impl<T,U> Point<T,U> {
    fn x(&self) -> &T {
        &self.x
    }
}


fn main() {
    let both_integer = Point { x: 5, y: 10 };
    let both_float = Point { x: 1.0, y: 4.0 };
    let integer_and_float = Point { x: 5, y: 4.0 };
}
```

Note that we have to declare `<T,U>` just after `impl` so we can use it to specify that we’re implementing methods on the type `Point<T,U>`. By doing so, Rust can identify that the type in the angle brackets in `Point` is a generic type rather than a concrete type.

Of course, we could implement methods only on some concrete type rather than the generic type

```rust
impl Point<f32,f32> {
    fn distance_from_origin(&self) -> f32 {
        (self.x.powi(2) + self.y.powi(2)).sqrt()
    }
}
```

This code means the type `Point<f32,f32>` will have a method named `distance_from_origin` and other concrete types of `Point<T>` where `T` is not of type `f32` will not have this method defined.


### Generic in function definitions

In this example, we have a unit struct `A`, a concrete struct `S` and a generic type struct `SGen`. 
- Function `reg_fn(_s: S) {}` is not a generic function as there is no `<>`. 
- Function `gen_spec_t(_s: SGen<A>) {}` is also not a generic function, because `SGen<A>` is a concrete type.
- Function `gen_spec_i32(_s: SGen<i32>) {}` is not generic.
- Function `generic<T>(_s: SGen<T>) {}` is a generic function over `T`. 


Let's see a concrate example:

```rust
struct A;          // Concrete type `A`.
struct S(A);       // Concrete type `S`.
struct SGen<T>(T); // Generic type `SGen`.

// The following functions all take ownership of the variable passed into
// them and immediately go out of scope, freeing the variable.

// Define a function `reg_fn` that takes an argument `_s` of type `S`.
// This has no `<T>` so this is not a generic function.
fn reg_fn(_s: S) {}

// Define a function `gen_spec_t` that takes an argument `_s` of type `SGen<T>`.
// It has been explicitly given the type parameter `A`, but because `A` has not 
// been specified as a generic type parameter for `gen_spec_t`, it is not generic.
fn gen_spec_t(_s: SGen<A>) {}

// Define a function `gen_spec_i32` that takes an argument `_s` of type `SGen<i32>`.
// It has been explicitly given the type parameter `i32`, which is a specific type.
// Because `i32` is not a generic type, this function is also not generic.
fn gen_spec_i32(_s: SGen<i32>) {}

// Define a function `generic` that takes an argument `_s` of type `SGen<T>`.
// Because `SGen<T>` is preceded by `<T>`, this function is generic over `T`.
fn generic<T>(_s: SGen<T>) {}

fn main() {
    // Using the non-generic functions
    reg_fn(S(A));          // Concrete type.
    gen_spec_t(SGen(A));   // Implicitly specified type parameter `A`.
    gen_spec_i32(SGen(6)); // Implicitly specified type parameter `i32`.

    // Explicitly specified type parameter `char` to `generic()`.
    generic::<char>(SGen('a'));

    // Implicitly specified type parameter `char` to `generic()`.
    generic(SGen('c'));
}

```

### Performance of Code Using Generics

Fortunately, Rust implements generics in such a way that your code doesn’t run any slower using generic types than it would with concrete types. Rust accomplishes this by performing monomorphization of the code that is using generics at compile time. Monomorphization is the process of turning generic code into specific code by filling in the concrete types that are used when compiled.

Let’s look at how this works with an example that uses the standard library’s `Option<T>` enum:

```rust
fn main() {
    let integer = Some(5);
    let float = Some(5.0);
}
```

When you run this code, *our magic Rust compiler will help you write the monomorphized version of the code:*

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


### Trait can help to define the type of generic type

Imagine we are working on many many different type of numbers, and we want to implement a `the_large_one()` function that can return the largest value of two input values.

Without generic, we need to define this function for every single data type:

```rust
fn the_large_one(x: i8, y:i8) {if x > y {x} else {y}};
fn the_large_one(x: i16, y:i16) {if x > y {x} else {y}};
// ...
```

As a smart coder, we cannot let it happen. So, we define a generic type `T`, and tell rust to do the rest for us.

```rust
fn the_large_one<T>(x: T, y: T) -> T {if x > y {x} else {y}};

```

However, rust will complain because not every data type can be compared. 

```
error[E0369]: binary operation `>` cannot be applied to type `T`
 --> src/main.rs:1:44
  |
1 | fn the_large_one<T>(x: T, y: T) -> T {if x > y {x} else {y}}
  |                                          - ^ - T
  |                                          |
  |                                          T
  |
help: consider restricting type parameter `T`
  |
1 | fn the_large_one<T: std::cmp::PartialOrd>(x: T, y: T) -> T {if x > y {x} else {y}}
  |                   ^^^^^^^^^^^^^^^^^^^^^^

For more information about this error, try `rustc --explain E0369`.
```

How to fix this error? We need trait to tell rust what is our generic type `T`!

# Trait
A *trait* tells the Rust compiler about functionality a particular **type** has and can share with other types. 
- We can use traits to define shared behavior in an abstract way. 
- We can use trait bounds to specify that a generic type can be any type that has certain behavior.

## Defining a trait

A type’s behavior consists of the methods we can call on that type. Different types share the same behavior if we can call the same methods on all of those types. *Trait definitions are a way to group method signatures together to define a set of behaviors necessary to accomplish some purpose.*

For example, let’s say we have multiple structs that hold various kinds and amounts of text: a `NewsArticle` struct that holds a news story filed in a particular location and a `Tweet` that can have at most 280 characters along with metadata that indicates whether it was a new tweet, a retweet, or a reply to another tweet.

We want to make a media aggregator library that can display summaries of data that might be stored in a `NewsArticle` or `Tweet` instance. To do this, we need a summary from each type, and we need to request that summary by calling a `summarize` method on an instance. 

```rust
pub trait Summary {
    fn summarize(&self) -> String;
    fn say_hello() {
        println!("Hello")
    };
}
```

Here, we declare a `trait` using the `trait` keyword and then the trait’s name, which is `Summary` in this case. Inside the curly brackets, we declare the method signatures that describe the behaviors of the types that implement this trait, which in this case is `fn summarize(&self) -> String`.

We don't give the function body to `summarize`, because we want the `struct`s that implement this trait to implement their own `summarize` function. However, we give a default implementation to function `say_hello()`, so that the concrete type can choose to use this default implementation, or implement their own.

## Implementing a Trait on a Type

If we want to define the desired behavior of a struct or any type of a trait, we need to implement the trait for the type.

```rust
pub trait Summary {
    fn summarize(&self) -> String;
}

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

If you want to call the `Summary` trait from other modules, you need to use `use` keyword as you do for your struct or module. You will also need to specify that `Summary` is a public trait before calling from other modules by saying `pub trait Summary {}`.

You have the choice to provide a default implementation for the desired behavior, like what we did for the `say_hello()` function. Suppose we also give `summarize()` a default implementation, then you just need to say `impl Summary for NewsArticle {}` if you want `NewsArticle` to use the default implementation. Of course you can replace the default implementation using your custom implementation.


## Traits as Parameters

We can use traits in function definition as the parameter type to tell rust this function can take multiple type. For example, if we want to write a function that takes a struct that implement the `Summary` trait as the input, we can say

```rust
pub fn notify(item: &impl Summary) {
    println!("Breaking news! {}", item.summarize());
}
```

### Trait Bound Syntax

the `impl Trait` syntax is actually syntax sugar for a long form, which is called a trait bound. Trait bound is a way to set a bound for the types that can be used in functions or methods. 

```rust
pub fn notify<T: Summary>(item: &T) {
    println!("Breaking news! {}", item.summarize());
}
```

The `impl Trait` syntax is convenient and makes for more concise code in simple cases. The trait bound syntax can express more complexity in other cases. For example, we can have two parameters that implement `Summary`. Using the `impl Trait` syntax looks like this:

```rust
pub fn notify(item1: &impl Summary + Display, item2: &impl Summary + Display) {}

pub fn notify<T: Summary + Display>(item1: &T, item2: &T) {}

```

We can also specify more than one trait bound using the `+` syntax.



### Clearer Trait Bounds with where Clauses

If you have really fancy trait bounds for your types, you function signature will be very very long. To make our life easier, rust defines a **`where` clause** in which you can put all your trait bounds inside.


```rust
fn some_function<T: Display + Clone, U: Clone + Debug>(t: &T, u: &U) -> i32 {}

fn some_function<T, U>(t: &T, u: &U) -> i32
    where T: Display + Clone,
          U: Clone + Debug
{}
```

If you want to use `impl Trait` syntax in the return position to return a value of some type that implements a trait, the function must have a fixed return type. For example, you cannot do `if true {NewsArticle} else {Tweet}`, even though both of them implemented `Summary`. However, we can do some tricks to achieve this. We will talk about how to achieve this with **trait object**.


## Returning Traits with `dyn`

The Rust compiler *needs to know how much space every function's return type requires*. This means all your functions have to return a concrete type. So if we want to use a custom trait as the return type of your function, you need to use some trick. 

Here we use `Box<dyn Trait>`, a trait object as the return type to solve this problem. 

```rust
struct Sheep {}
struct Cow {}

trait Animal {
    // Instance method signature
    fn noise(&self) -> &'static str;
}

// Implement the `Animal` trait for `Sheep`.
impl Animal for Sheep {
    fn noise(&self) -> &'static str {
        "baaaaah!"
    }
}

// Implement the `Animal` trait for `Cow`.
impl Animal for Cow {
    fn noise(&self) -> &'static str {
        "moooooo!"
    }
}

// Returns some struct that implements Animal, but we don't know which one at compile time.
fn random_animal(random_number: f64) -> Box<dyn Animal> {
    if random_number < 0.5 {
        Box::new(Sheep {})
    } else {
        Box::new(Cow {})
    }
}

fn main() {
    let random_number = 0.234;
    let animal = random_animal(random_number);
    println!("You've randomly chosen an animal, and it says {}", animal.noise());
}
```

### Fixing the largest Function with Trait Bounds

```rust
fn the_large_one<T: PartialOrd>(x: T, y: T) -> T {if x > y {x} else {y}};

```

## Finer controls!

### Using Trait Bounds to Conditionally Implement Methods

We can implement methods conditionally for types that implement a specific trait. 

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

We can also conditionally implement a trait *for any type that implements another trait*. Implementations of a trait on any type that satisfies the trait bounds are called *blanket implementations* and are extensively used in the Rust standard library. 

For example, the standard library implements the `ToString` trait on **any** type that implements the `Display` trait. It means we can call the `to_string` method defined by the `ToString` trait on any type that implements the `Display` trait. The `impl` block in the standard library looks similar to this code:

```rust
impl<T: Display> ToString for T {
    // --snip--
}
```

See more example in (RBE)[https://doc.rust-lang.org/rust-by-example/trait.html].


### Derivable traits

The smart compiler provides basic implementations for some traits via the `#[derive]` [attribute](https://doc.rust-lang.org/rust-by-example/attribute.html). We can plug and use those traits without thinking about the implementation. *These traits can still be manually implemented if a more complex behavior is required.* 

The following is a list of derivable traits:

* Comparison traits: Eq, PartialEq, Ord, PartialOrd.
* Clone, to create `T` from `&T` via a copy.
* Copy, to give a type 'copy semantics' instead of 'move semantics'.
* Hash, to compute a hash from `&T`.
* Default, to create an empty instance of a data type.
* Debug, to format a value using the `{:?}` formatter.

```rust
// `Centimeters`, a tuple struct that can be compared
#[derive(PartialEq, PartialOrd)]
struct Centimeters(f64);

// `Inches`, a tuple struct that can be printed
#[derive(Debug)]
struct Inches(i32);

impl Inches {
    fn to_centimeters(&self) -> Centimeters {
        let &Inches(inches) = self;

        Centimeters(inches as f64 * 2.54)
    }
}

// `Seconds`, a tuple struct with no additional attributes
struct Seconds(i32);

fn main() {
    let _one_second = Seconds(1);

    // Error: `Seconds` can't be printed; it doesn't implement the `Debug` trait
    //println!("One second looks like: {:?}", _one_second);
    // TODO ^ Try uncommenting this line

    // Error: `Seconds` can't be compared; it doesn't implement the `PartialEq` trait
    //let _this_is_true = (_one_second == _one_second);
    // TODO ^ Try uncommenting this line

    let foot = Inches(12);

    println!("One foot equals {:?}", foot);

    let meter = Centimeters(100.0);

    let cmp =
        if foot.to_centimeters() < meter {
            "smaller"
        } else {
            "bigger"
        };

    println!("One foot is {} than one meter.", cmp);
}
```


## Operator Overloading

Some operators can be overloaded by trait. For example, the `+` operator in `a + b` calls the `add` method (as in `a.add(b)`). This `add` method is part of the `Add` trait. Hence, the `+` operator can be used by any implementor of the `Add` trait.

```rust
use std::ops;

struct Foo;
struct Bar;

#[derive(Debug)]
struct FooBar;

#[derive(Debug)]
struct BarFoo;

impl ops::Add<Bar> for Foo {
    type Output = FooBar;

    fn add(self, _rhs: Bar) -> FooBar {
        println!("> Foo.add(Bar) was called");

        FooBar
    }
}

impl ops::Add<Foo> for Bar {
    type Output = BarFoo;

    fn add(self, _rhs: Foo) -> BarFoo {
        println!("> Bar.add(Foo) was called");

        BarFoo
    }
}

fn main() {
    println!("Foo + Bar = {:?}", Foo + Bar);
    println!("Bar + Foo = {:?}", Bar + Foo);
}
```

### Implement iterator for your own trait

The `Iterator` trait is used to implement iterators over collections such as arrays.

The trait requires only a method to be defined for the `next` element, which may be manually defined in an `impl` block or automatically defined (as in arrays and ranges).

As a point of convenience `for` common situations, the for construct turns some collections into iterators using the `.into_iter()` method.

```rust
struct Fibonacci {
    curr: u32,
    next: u32,
}

impl Iterator for Fibonacci {
    // We can refer to this type using Self::Item
    type Item = u32;
    
    // Here, we define the sequence using `.curr` and `.next`.
    // The return type is `Option<T>`:
    //     * When the `Iterator` is finished, `None` is returned.
    //     * Otherwise, the next value is wrapped in `Some` and returned.
    // We use Self::Item in the return type, so we can change
    // the type without having to update the function signatures.
    fn next(&mut self) -> Option<Self::Item> {
        let new_next = self.curr + self.next;

        self.curr = self.next;
        self.next = new_next;

        // Since there's no endpoint to a Fibonacci sequence, the `Iterator` 
        // will never return `None`, and `Some` is always returned.
        Some(self.curr)
    }
}

// Returns a Fibonacci sequence generator
fn fibonacci() -> Fibonacci {
    Fibonacci { curr: 0, next: 1 }
}

fn main() {
    // `0..3` is an `Iterator` that generates: 0, 1, and 2.
    let mut sequence = 0..3;

    println!("Four consecutive `next` calls on 0..3");
    println!("> {:?}", sequence.next());
    println!("> {:?}", sequence.next());
    println!("> {:?}", sequence.next());
    println!("> {:?}", sequence.next());

    // `for` works through an `Iterator` until it returns `None`.
    // Each `Some` value is unwrapped and bound to a variable (here, `i`).
    println!("Iterate through 0..3 using `for`");
    for i in 0..3 {
        println!("> {}", i);
    }

    // The `take(n)` method reduces an `Iterator` to its first `n` terms.
    println!("The first four terms of the Fibonacci sequence are: ");
    for i in fibonacci().take(4) {
        println!("> {}", i);
    }

    // The `skip(n)` method shortens an `Iterator` by dropping its first `n` terms.
    println!("The next four terms of the Fibonacci sequence are: ");
    for i in fibonacci().skip(4).take(4) {
        println!("> {}", i);
    }

    let array = [1u32, 3, 3, 7];

    // The `iter` method produces an `Iterator` over an array/slice.
    println!("Iterate the following array {:?}", &array);
    for i in array.iter() {
        println!("> {}", i);
    }
}

```

### Supertraits

Rust doesn't have "inheritance", but you can define a trait as being a superset of another trait. For example:

```rust
trait Person {
    fn name(&self) -> String;
}

// Person is a supertrait of Student.
// Implementing Student requires you to also impl Person.
trait Student: Person {
    fn university(&self) -> String;
}

trait Programmer {
    fn fav_language(&self) -> String;
}

// CompSciStudent (computer science student) is a subtrait of both Programmer 
// and Student. Implementing CompSciStudent requires you to impl both supertraits.
trait CompSciStudent: Programmer + Student {
    fn git_username(&self) -> String;
}

fn comp_sci_student_greeting(student: &dyn CompSciStudent) -> String {
    format!(
        "My name is {} and I attend {}. My favorite language is {}. My Git username is {}",
        student.name(),
        student.university(),
        student.fav_language(),
        student.git_username()
    )
}

fn main() {}

```

### Disambiguating overlapping traits

A type can implement many different traits. What if two traits both require the same name? For example, many traits might have a method named `get()`. They might even have different return types!

Good news: because each trait implementation gets its own `impl` block, it's clear which trait's `get` method you're implementing.

What about when it comes time to call those methods? To disambiguate between them, we have to use Fully Qualified Syntax. For example, `<MyStruct as MyTrait1>::get(&form)` and ``<MyStruct as MyTrait2>::get(&form)``

# Polymorphism

To many people, polymorphism is synonymous with inheritance. But it’s actually a more general concept that refers to code that can work with data of multiple types. For inheritance, those types are generally subclasses.

Rust instead uses generics to abstract over different possible types and trait bounds to impose constraints on what those types must provide. This is sometimes called *bounded parametric polymorphism*.

## Rust is sort of a Object-Oriented Language
Arguably, OOP languages share certain common characteristics, namely objects, encapsulation, and inheritance. Let’s look at what each of those characteristics means and whether Rust supports it.

### 1. Objects Contain Data and Behavior

> Object-oriented programs are made up of objects. An object packages both data and the procedures that operate on that data. The procedures are typically called methods or operations.[name=The Gang of Four book]

Using this definition, Rust is object oriented: structs and enums have data, and `impl` blocks provide methods on structs and enums. 


### 2. Encapsulation that Hides Implementation Details

The option to use `pub` or not for different parts of code enables encapsulation of implementation details.


### 3. Inheritance as a Type System and as Code Sharing

Inheritance is a mechanism whereby an object can inherit from another object’s definition, thus gaining the parent object’s data and behavior without you having to define them again.

In Rust, there is no way to define a struct that inherits the parent struct’s fields and method implementations. However, we can use **trait** to achieve the same thing.

The other reason to use inheritance *relates to the type system: to enable a child type to be used in the same places as the parent type*. This is also called **polymorphism**, which means that you can substitute multiple objects for each other at **runtime** if they share certain characteristics. the trait object `Box<dyn Trait>` is the secret. When talking about generics, we said the compiler will figure out what type a generics is at compile time.

## Defining a Trait for common behavior
**A trait object `Box<dyn Trait>` points to both an instance of a type implementing our specified trait as well as a table used to look up trait methods on that type at runtime.** 

The Rust compiler restricts that all the values in a vector must have the same type. Even if we define a generic type, the generic type can be substituted with one contrete type at a time

```rust
// generic
pub struct Screen<T: Draw> {
    pub components: Vec<T>,
}

impl<T> Screen<T>
where
    T: Draw,
{
    pub fn run(&self) {
        for component in self.components.iter() {
            component.draw();
        }
    }
}
```

We can use trait objects in place of a generic or concrete type. Wherever we use a trait object, Rust’s type system will ensure at compile time that any value used in that context will implement the trait object’s trait. Consequently, **we don’t need to know all the possible types at compile time.**

```rust
pub trait Draw {
    fn draw(&self);
}

pub struct Screen {
    pub components: Vec<Box<dyn Draw>>,
}

impl Screen {
    pub fn run(&self) {
        for component in self.components.iter() {
            component.draw();
        }
    }
}
```

Now, what the magic thing `Screen` can do is take different concrete type in its components field. When we iterate over the vector, the Rust compiler will figure out what contrete type a value is, and call the methods of that contrete type using the pointer to the methods inside the trait object.

```rust
pub struct Button {
    pub width: u32,
    pub height: u32,
    pub label: String,
}

impl Draw for Button {
    fn draw(&self) {
        // code to actually draw a button
    }
}

struct SelectBox {
    width: u32,
    height: u32,
    options: Vec<String>,
}

impl Draw for SelectBox {
    fn draw(&self) {
        // code to actually draw a select box
    }
}

use gui::{Button, Screen};

fn main() {
    let screen = Screen {
        components: vec![
            Box::new(SelectBox {
                width: 75,
                height: 10,
                options: vec![
                    String::from("Yes"),
                    String::from("Maybe"),
                    String::from("No"),
                ],
            }),
            Box::new(Button {
                width: 50,
                height: 10,
                label: String::from("OK"),
            }),
        ],
    };

    screen.run();
}
```

## Trait Objects Perform Dynamic Dispatch

We have talked abou that when using generics, the Rust compiler will figure out what the concrete type a generic type is for each usage, and then implement everything for that concrete type.  The code that results from monomorphization is doing *static dispatch*, which is when the compiler knows what method you’re calling at compile time.

However when we use trait object, the compiler cannot tell at compile time which method you are calling. Instead, at runtime, Rust uses the pointers inside the trait object to know which method to call. So, the compiler emits code that at runtime will figure out which method to call. This is called *dynamic dispatch*.

There is a runtime cost when this lookup happens that doesn’t occur with static dispatch. Dynamic dispatch also prevents the compiler from choosing to inline a method’s code, which in turn prevents some optimizations. However, we did get extra flexibility in the code

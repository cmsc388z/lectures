# Lecture 10: The builder pattern, writing iterators and closures

## Introduction

Rust’s design has taken inspiration from many existing languages and techniques, and one significant influence is functional programming. Programming in a functional style often includes using functions as values by passing them in arguments, returning them from other functions, assigning them to variables for later execution, and so forth.

In this section we'll discuss some features of Rust that are similar to features in many languages often referred to as functional. Two important features include iterators and closures. 

## Closures

Rust’s closures are anonymous functions you can save in a variable or pass as arguments to other functions. You can create the closure in one place and then call the closure to evaluate it in a different context. Unlike functions, closures can capture values from the scope in which they’re defined. We’ll demonstrate how these closure features allow for code reuse and behavior customization.

Consider this hypothetical situation: we work at a startup that’s making an app to generate custom exercise workout plans. The backend is written in Rust, and the algorithm that generates the workout plan takes into account many factors, such as the app user’s age, body mass index, exercise preferences, recent workouts, and an intensity number they specify. The actual algorithm used isn’t important in this example; what’s important is that this calculation takes a few seconds. We want to call this algorithm only when we need to and only call it once so we don’t make the user wait more than necessary.

We’ll simulate calling this hypothetical algorithm with the function simulated_expensive_calculation shown below, which will print calculating slowly..., wait for two seconds, and then return whatever number we passed in.

```rust
use std::thread;
use std::time::Duration;

fn simulated_expensive_calculation(intensity: u32) -> u32 {
    println!("calculating slowly...");
    thread::sleep(Duration::from_secs(2));
    intensity
}
```

Next is the main function, which contains the parts of the workout app important for this example. This function represents the code that the app will call when a user asks for a workout plan. Because the interaction with the app’s frontend isn’t relevant to the use of closures, we’ll hardcode values representing inputs to our program and print the outputs.

The program takes inputs for intensity and a random number.

```rust
fn main() {
    let simulated_user_specified_value = 10;
    let simulated_random_number = 7;

    generate_workout(simulated_user_specified_value, simulated_random_number);
}
```

Given this context, we can move on to the main algorithm. The function generate_workout below contains the business logic of the app that we’re most concerned with in this example. The rest of the code changes in this example will be made to this function.

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

We could restructure the workout program in many ways. First, we’ll try extracting the duplicated call to the simulated_expensive_calculation function into a variable, as shown below.

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

These changes unify all the calls to simulated_expensive_calculation and solves the problem of the first if block unnecessarily calling the function twice. Unfortunately, we’re now calling this function and waiting for the result in all cases, which includes the inner if block that doesn’t use the result value at all. 

Instead of always calling the simulated_expensive_calculation function before the if blocks, we can define a closure and store the closure in a variable rather than storing the result of the function call, as shown below. We can actually move the whole body of simulated_expensive_calculation within the closure we’re introducing here.

```rust
    let expensive_closure = |num| {
        println!("calculating slowly...");
        thread::sleep(Duration::from_secs(2));
        num
    };
```

The closure definition comes after the = to assign it to the variable expensive_closure. To define a closure, we start with a pair of vertical pipes (|), inside which we specify the parameters to the closure; this syntax was chosen because of its similarity to closure definitions in Smalltalk and Ruby. This closure has one parameter named num: if we had more than one parameter, we would separate them with commas, like |param1, param2|.

After the parameters, we place curly brackets that hold the body of the closure—these are optional if the closure body is a single expression. The end of the closure, after the curly brackets, needs a semicolon to complete the let statement. The value returned from the last line in the closure body (num) will be the value returned from the closure when it’s called, because that line doesn’t end in a semicolon; just as in function bodies.

Note that this let statement means expensive_closure contains the definition of an anonymous function, not the resulting value of calling the anonymous function. Recall that we’re using a closure because we want to define the code to call at one point, store that code, and call it at a later point; the code we want to call is now stored in expensive_closure.

With the closure defined, we can change the code in the if blocks to call the closure to execute the code and get the resulting value. We call a closure like we do a function.

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

You are not required to annotate the types of the parameters or the return value like fn functions do. Type annotations are required on functions because they’re part of an explicit interface exposed to your users. Defining this interface rigidly is important for ensuring that everyone agrees on what types of values a function uses and returns. But closures aren’t used in an exposed interface like this: they’re stored in variables and used without naming them and exposing them to users of our library.

Closures they are usually short and relevant only within a narrow context rather than in any arbitrary scenario. Within these limited contexts, the compiler is reliably able to infer the types of the parameters and the return type, similar to how it’s able to infer the types of most variables.

We can also add type annotations to our closure if we wish.

```rust
    let expensive_closure = |num: u32| -> u32 {
        println!("calculating slowly...");
        thread::sleep(Duration::from_secs(2));
        num
    };
```

At which point the syntax begins to look more like a function definition. The following is a vertical comparison of the syntax for the definition of a function that adds 1 to its parameter and a closure that has the same behavior. We’ve added some spaces to line up the relevant parts. This illustrates how closure syntax is similar to function syntax except for the use of pipes and the amount of syntax that is optional:

```rust
fn  add_one_v1   (x: u32) -> u32 { x + 1 }
let add_one_v2 = |x: u32| -> u32 { x + 1 };
let add_one_v3 = |x|             { x + 1 };
let add_one_v4 = |x|               x + 1  ;
```

The first line shows a function definition, and the second line shows a fully annotated closure definition. The third line removes the type annotations from the closure definition, and the fourth line removes the brackets, which are optional because the closure body has only one expression. These are all valid definitions that will produce the same behavior when they’re called. Calling the closures is required for add_one_v3 and add_one_v4 to be able to compile because the types will be inferred from their usage.

Closure definitions will have one concrete type inferred for each of their parameters and for their return value. For instance, the example below shows the definition of a short closure that just returns the value it receives as a parameter. This closure isn’t very useful except for the purposes of this example. Note that we haven’t added any type annotations to the definition: if we then try to call the closure twice, using a String as an argument the first time and a u32 the second time, we’ll get an error.

```rust
    let example_closure = |x| x;

    let s = example_closure(String::from("hello"));
    let n = example_closure(5);
```

When attempting to compile this we get the following error.

```
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

error: aborting due to previous error

For more information about this error, try `rustc --explain E0308`.
error: could not compile `closure-example`

To learn more, run the command again with --verbose.
```

The first time we call example_closure with the String value, the compiler infers the type of x and the return type of the closure to be String. Those types are then locked into the closure in example_closure, and we get a type error if we try to use a different type with the same closure.

## Iterators

The iterator pattern allows you to perform some task on a sequence of items in turn. An iterator is responsible for the logic of iterating over each item and determining when the sequence has finished. When you use iterators, you don’t have to reimplement that logic yourself.

In Rust, iterators are lazy, meaning they have no effect until you call methods that consume the iterator to use it up. For example, the code in the example below creates an iterator over the items in the vector v1 by calling the iter method defined on Vec\<T\>. This code by itself doesn’t do anything useful.

```rust
    let v1 = vec![1, 2, 3];
    let v1_iter = v1.iter();
```

These iterators can be used in a variety of ways. For example, we can separate the creation of the iterator from the use of the iterator in the for loop, as shown below.

```rust
    let v1 = vec![1, 2, 3];
    let v1_iter = v1.iter();

    for val in v1_iter {
        println!("Got: {}", val);
    }
```

Other languages that don’t have iterators provided by their standard libraries, you would likely write this same functionality by starting a variable at index 0, using that variable to index into the vector to get a value, and incrementing the variable value in a loop until it reached the total number of items in the vector.

Iterators handle all that logic for you, cutting down on repetitive code you could potentially mess up. Iterators give you more flexibility to use the same logic with many different kinds of sequences, not just data structures you can index into, like vectors. Let’s examine how iterators do that.

### Iterator trait

All iterators implement a trait named Iterator that is defined in the standard library. The definition of the trait looks like this:

```rust
pub trait Iterator {
    type Item;

    fn next(&mut self) -> Option<Self::Item>;

    // methods with default implementations elided
}
```

Notice this definition uses some new syntax: type Item and Self::Item, which are defining an associated type with this trait. We’ll talk about associated types in depth in Chapter 19. For now, all you need to know is that this code says implementing the Iterator trait requires that you also define an Item type, and this Item type is used in the return type of the next method. In other words, the Item type will be the type returned from the iterator.


## The Builder Pattern



# Example

https://subscription.packtpub.com/book/application_development/9781788623926/1/ch01lvl1sec17/using-the-builder-pattern

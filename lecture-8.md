# Interior mutability, type markers and phantom types

Source: [rust book](https://doc.rust-lang.org/book/ch15-05-interior-mutability.html)

## Project introduction

Project details.

## `RefCell<T>`

Interior mutability is a design pattern in Rust that allows you to mutate data even when there are immutable references to that data; normally, this action is disallowed by the borrowing rules. To mutate data, the pattern uses unsafe code inside a data structure to bend Rust’s usual rules that govern mutation and borrowing.

We will explore this concept using the `RefCell<T>` type, which follows the interior mutability pattern.

Unlike `Rc<T>`, the `RefCell<T>` type represents single ownership over the data it holds. So, what makes `RefCell<T>` different from a type like `Box<T>`?

Recall the borrowing rules we learned in previous lectures:
 - At any given time, you can have either one mutable reference or any number of immutable references.
 - References must always be valid.

When using `Box<T>`, the borrowing rules’ invariants are enforced at compile time. With `RefCell<T>`, these invariants are enforced at runtime. With references, if you break these rules, you’ll get a compiler error. With `RefCell<T>`, if you break these rules, your program will panic and exit.

Checking borrowing rules at compile time has the advantage of catching errors sooner in the development process, and there is no impact on runtime performance because all the analysis is completed beforehand. For those reasons, checking the borrowing rules at compile time is the best choice in the majority of cases.

However, there are other scenarios where one might want to take advantage of the additional flexibility afforded by checking the borrowing rules at runtime. Because some analyses are impossible, if the Rust compiler can’t be sure the code complies with the ownership rules, it might reject a correct program; in this way, it is conservative. The `RefCell<T>` type is useful when you’re sure your code follows the borrowing rules but the compiler is unable to understand and guarantee that.

### When to use each type?
 
 - `Rc<T>` enables multiple owners of the same data; `Box<T>` and `RefCell<T>` have single owners.
 - `Box<T>` allows immutable or mutable borrows checked at compile time; `Rc<T>` allows only immutable borrows checked at compile time; `RefCell<T>` allows immutable or mutable borrows checked at runtime.
 - Because `RefCell<T>` allows mutable borrows checked at runtime, you can mutate the value inside the `RefCell<T>` even when the `RefCell<T>` is immutable.


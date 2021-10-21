# Interior mutability, type markers and phantom types

Source: [Rust book](https://doc.rust-lang.org/book/ch15-05-interior-mutability.html), [Additional notes](https://ricardomartins.cc/2016/06/08/interior-mutability)

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

### Borrowing review

Will this code compile? Why?

```rust
fn main() {
    let x = 5;
    let y = &mut x;
}
```

However, there are situations in which it would be useful for a value to mutate itself in its methods but appear immutable to other code. Code outside the value’s methods would not be able to mutate the value. Using `RefCell<T>` is one way to get the ability to have interior mutability. But `RefCell<T>` doesn’t get around the borrowing rules completely: the borrow checker in the compiler allows this interior mutability, and the borrowing rules are checked at runtime instead. If you violate the rules, you’ll get a panic! instead of a compiler error.

### Interior mutability example

In this example, we’ll create a library that tracks a value against a maximum value and sends messages based on how close to the maximum value the current value is. This library could be used to keep track of a user’s quota for the number of API calls they’re allowed to make, for example. Our library will only provide the functionality of tracking how close to the maximum a value is and what the messages should be at what times. Applications that use our library will be expected to provide the mechanism for sending the messages: the application could put a message in the application, send an email, send a text message, or something else. The library doesn’t need to know that detail. It only uses a trait we'll provide called Messenger. 

```rust
pub trait Messenger {
    fn send(&self, msg: &str);
}

pub struct LimitTracker<'a, T: Messenger> {
    messenger: &'a T,
    value: usize,
    max: usize,
}

impl<'a, T> LimitTracker<'a, T>
where
    T: Messenger,
{
    pub fn new(messenger: &T, max: usize) -> LimitTracker<T> {
        LimitTracker {
            messenger,
            value: 0,
            max,
        }
    }

    pub fn set_value(&mut self, value: usize) {
        self.value = value;

        let percentage_of_max = self.value as f64 / self.max as f64;

        if percentage_of_max >= 1.0 {
            self.messenger.send("Error: You are over your quota!");
        } else if percentage_of_max >= 0.9 {
            self.messenger
                .send("Urgent warning: You've used up over 90% of your quota!");
        } else if percentage_of_max >= 0.75 {
            self.messenger
                .send("Warning: You've used up over 75% of your quota!");
        }
    }
}
```

The Messenger trait is the interface our mock object needs to implement so that the mock can be used in the same way a real object is. The other important part is that we want to test the behavior of the set_value method on the LimitTracker. We can change what we pass in for the value parameter, but set_value doesn’t return anything for us to make assertions on. We want to be able to say that if we create a LimitTracker with something that implements the Messenger trait and a particular value for max, when we pass different numbers for value, the messenger is told to send the appropriate messages.

We need a mock object that, instead of sending an email or text message when we call send, will only keep track of the messages it’s told to send. We can create a new instance of the mock object, create a LimitTracker that uses the mock object, call the set_value method on LimitTracker, and then check that the mock object has the messages we expect. The following example shows an attempt to do this, but the borrow checker won't allow it. 

```rust
#[cfg(test)]
mod tests {
    use super::*;

    struct MockMessenger {
        sent_messages: Vec<String>,
    }

    impl MockMessenger {
        fn new() -> MockMessenger {
            MockMessenger {
                sent_messages: vec![],
            }
        }
    }

    impl Messenger for MockMessenger {
        fn send(&self, message: &str) {
            self.sent_messages.push(String::from(message));
        }
    }

    #[test]
    fn it_sends_an_over_75_percent_warning_message() {
        let mock_messenger = MockMessenger::new();
        let mut limit_tracker = LimitTracker::new(&mock_messenger, 100);

        limit_tracker.set_value(80);

        assert_eq!(mock_messenger.sent_messages.len(), 1);
    }
}
```

What's the problem with this code? Why won't the borrow checker allow this. (Hint: Look at `send()`). When compiling this example, we get the following error:

```
$ cargo test
   Compiling limit-tracker v0.1.0 (file:///projects/limit-tracker)
error[E0596]: cannot borrow `self.sent_messages` as mutable, as it is behind a `&` reference
  --> src/lib.rs:58:13
   |
2  |     fn send(&self, msg: &str);
   |             ----- help: consider changing that to be a mutable reference: `&mut self`
...
58 |             self.sent_messages.push(String::from(message));
   |             ^^^^^^^^^^^^^^^^^^ `self` is a `&` reference, so the data it refers to cannot be borrowed as mutable

error: aborting due to previous error

For more information about this error, try `rustc --explain E0596`.
error: could not compile `limit-tracker`

To learn more, run the command again with --verbose.
warning: build failed, waiting for other jobs to finish...
error: build failed
```

Modifying the MockMessenger to keep track of the messages, because the send method takes an immutable reference to self. We also can’t take the suggestion from the error text to use &mut self instead, because then the signature of send wouldn’t match the signature in the Messenger trait definition. However, we can fix this example by using interior mutability. We will store the sent_messsages within a `RefCell<T>`, and then the send method will be able to modify hte sent_messages to store the messages we've seen. Here is an example implementation.

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use std::cell::RefCell;

    struct MockMessenger {
        sent_messages: RefCell<Vec<String>>,
    }

    impl MockMessenger {
        fn new() -> MockMessenger {
            MockMessenger {
                sent_messages: RefCell::new(vec![]),
            }
        }
    }

    impl Messenger for MockMessenger {
        fn send(&self, message: &str) {
            self.sent_messages.borrow_mut().push(String::from(message));
        }
    }

    #[test]
    fn it_sends_an_over_75_percent_warning_message() {
        // --snip--

        assert_eq!(mock_messenger.sent_messages.borrow().len(), 1);
    }
}
```

The sent_messages field is now of type `RefCell<Vec<String>>` instead of `Vec<String>`. In the new function, we create a new `RefCell<Vec<String>>` instance around the empty vector. 

For the implementation of the send method, the first parameter is still an immutable borrow of self, which matches the trait definition. We call borrow_mut on the `RefCell<Vec<String>>` in self.sent_messages to get a mutable reference to the value inside the `RefCell<Vec<String>>`, which is the vector. Then we can call push on the mutable reference to the vector to keep track of the messages sent during the test.

The last change we have to make is in the assertion: to see how many items are in the inner vector, we call borrow on the `RefCell<Vec<String>>` to get an immutable reference to the vector.

---

### How `Refcell<T>` works

When creating immutable and mutable references, we use the & and &mut syntax, respectively. With `RefCell<T>`, we use the borrow and borrow_mut methods, which are part of the safe API that belongs to `RefCell<T>`. The borrow method returns the smart pointer type `Ref<T>`, and borrow_mut returns the smart pointer type `RefMut<T>`. Both types implement Deref, so we can treat them like regular references.
 
The `RefCell<T>` keeps track of how many `Ref<T>` and `RefMut<T>` smart pointers are currently active. Every time we call borrow, the `RefCell<T>` increases its count of how many immutable borrows are active. When a `Ref<T>` value goes out of scope, the count of immutable borrows goes down by one. Just like the compile-time borrowing rules, `RefCell<T>` lets us have many immutable borrows or one mutable borrow at any point in time.
 
Now if we violate the borrowing rules, we'll get an error at compile time rather than at runtime. Here is an example.

```rust
    impl Messenger for MockMessenger {
        fn send(&self, message: &str) {
            let mut one_borrow = self.sent_messages.borrow_mut();
            let mut two_borrow = self.sent_messages.borrow_mut();

            one_borrow.push(String::from(message));
            two_borrow.push(String::from(message));
        }
    }
```

We can see that with refcell this example will compile, but when we run it the program will panic.

```
thread 'main' panicked at 'already borrowed: BorrowMutError', src/lib.rs:60:53
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

Notice that the code panicked with the message already borrowed: BorrowMutError. This is how `RefCell<T>` handles violations of the borrowing rules at runtime.

### Combining Rc and RefCell

It is common to use `RefCell<T>` together with `Rc<T>`. Recall that `Rc<T>` lets you have multiple owners of some data, but it only gives immutable access to that data. If you have an `Rc<T>` that holds a `RefCell<T>`, you can get a value that can have multiple owners and that you can mutate! Because `Rc<T>` holds only immutable values, we can’t change any of the values in the list once we’ve created them. Now we will add in `RefCell<T>` to gain the ability to change the values.

```rust
#[derive(Debug)]
enum List {
    Cons(Rc<RefCell<i32>>, Rc<List>),
    Nil,
}

use crate::List::{Cons, Nil};
use std::cell::RefCell;
use std::rc::Rc;

fn main() {
    let value = Rc::new(RefCell::new(5));

    let a = Rc::new(Cons(Rc::clone(&value), Rc::new(Nil)));

    let b = Cons(Rc::new(RefCell::new(3)), Rc::clone(&a));
    let c = Cons(Rc::new(RefCell::new(4)), Rc::clone(&a));

    *value.borrow_mut() += 10;

    println!("a after = {:?}", a);
    println!("b after = {:?}", b);
    println!("c after = {:?}", c);
}
```

Here we create a value that is an instance of `Rc<RefCell<i32>>` and store it in a variable named value so we can access it directly later. Then we create a List in a with a Cons variant that holds value. We need to clone value so both a and value have ownership of the inner 5 value rather than transferring ownership from value to a or having a borrow from value.

We wrap the list a in an `Rc<T>` so when we create lists b and c, they can both refer to a. After we’ve created the lists in a, b, and c, we add 10 to the value in value. We do this by calling borrow_mut on value, which uses automatic dereferencing to dereference the `Rc<T>` to the inner `RefCell<T>` value. The borrow_mut method returns a `RefMut<T>` smart pointer, and we use the dereference operator on it to change the inner value. When we print a, b, and c, we can see that they all have the modified value of 15 rather than 5.

```
$ cargo run
   Compiling cons-list v0.1.0 (file:///projects/cons-list)
    Finished dev [unoptimized + debuginfo] target(s) in 0.63s
     Running `target/debug/cons-list`
a after = Cons(RefCell { value: 15 }, Nil)
b after = Cons(RefCell { value: 3 }, Cons(RefCell { value: 15 }, Nil))
c after = Cons(RefCell { value: 4 }, Cons(RefCell { value: 15 }, Nil))
```

Here we have an outwardly immutable List value. But we can use the methods on `RefCell<T>` that provide access to its interior mutability so we can modify our data when we need to. The runtime checks of the borrowing rules protect us from data races, and it’s sometimes worth trading a bit of speed for this flexibility in our data structures.
 
The standard library has other types that provide interior mutability, such as `Cell<T>`, which is similar except that instead of giving references to the inner value, the value is copied in and out of the `Cell<T>`. There’s also `Mutex<T>`, which offers interior mutability that’s safe to use across threads.


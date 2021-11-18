# Lecture 12: Macros

We’ve used macros like println! so far in the class, but we haven’t fully explored what a macro is and how it works. The term macro refers to a family of features in Rust: declarative macros with macro_rules! and three kinds of procedural macros:

 - Custom #[derive] macros that specify code added with the derive attribute used on structs and enums
 - Attribute-like macros that define custom attributes usable on any item
 - Function-like macros that look like function calls but operate on the tokens specified as their argument
 
 First, let’s look at why we even need macros when we already have functions.
 
## Macros vs Functions


## References and Borrowing

The issue with the tuple code in the previous task is that we have to return the `String` to the calling function so we can still use the `String` after the call to `calculate_length` because the `String` was moved into `calculate_length`.

Here is how you would define and use a `calculate_length` function that has a reference to an object as a parameter instead of taking ownership of the value:

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

First, notice that all the tuple code in the variable declaration and the function return value is gone. Second, note that we pass `&s1` into `calculate_length` and, in its definition, we take `&String` rather than `String`.

These ampersands are _references_, and they allow you to refer to some value without taking ownership of it. Figure 5 shows a diagram.

<img alt="&amp;String s pointing at String s1" src="https://doc.rust-lang.org/stable/book/img/trpl04-05.svg" class="center">

##### Figure 5: A diagram of &String s pointing at String s1

> Note: The opposite of referencing by using `&` is _dereferencing_, which is accomplished with the dereference operator, `*`. We’ll see some uses of the dereference operator in Chapter 8 and discuss details of dereferencing in Chapter 15.

Let’s take a closer look at the function call here:

```rust
    let s1 = String::from("hello");

    let len = calculate_length(&s1);
```

The `&s1` syntax lets us create a reference that _refers_ to the value of `s1` but does not own it. Because it does not own it, the value it points to will not be dropped when the reference goes out of scope.

Likewise, the signature of the function uses `&` to indicate that the type of the parameter `s` is a reference. Let’s add some explanatory annotations:

```rust
    fn calculate_length(s: &String) -> usize { // s is a reference to a String
        s.len()
    } // Here, s goes out of scope. But because it does not have ownership of what
      // it refers to, nothing happens.
```

The scope in which the variable `s` is valid is the same as any function parameter’s scope, but we don’t drop what the reference points to when it goes out of scope because we don’t have ownership. When functions have references as parameters instead of the actual values, we won’t need to return the values in order to give back ownership because we never had ownership.

We call having references as function parameters _borrowing_. As in real life, if a person owns something, you can borrow it from them. When you’re done, you have to give it back.

So what happens if we try to modify something we’re borrowing? Try the code in the code snippet below. Spoiler alert: it doesn’t work!

```rust
    fn main() {
        let s = String::from("hello");

        change(&s);
    }

    fn change(some_string: &String) {
        some_string.push_str(", world");
    }
```

##### Attempting to modify a borrowed value

Here’s the error:

```text
    error[E0596]: cannot borrow `*some_string` as mutable, as it is behind a `&` reference
 --> src/main.rs:8:5
  |
7 | fn change(some_string: &String) {
  |                        ------- help: consider changing this to be a mutable reference: `&mut String`
8 |     some_string.push_str(", world");
  |     ^^^^^^^^^^^ `some_string` is a `&` reference, so the data it refers to cannot be borrowed as mutable
```

Just as variables are immutable by default, so are references. We’re not allowed to modify something we have a reference to.

### Mutable References

We can fix the error in the code from the snippet above with just a small tweak:

```rust
    fn main() {
        let mut s = String::from("hello");

        change(&mut s);
    }

    fn change(some_string: &mut String) {
        some_string.push_str(", world");
    }
```

First, we had to change `s` to be `mut`. Then we had to create a mutable reference with `&mut s` and accept a mutable reference with `some_string: &mut String`.

But mutable references have one big restriction: you can have only one mutable reference to a particular piece of data in a particular scope. This code will fail:

```rust
    let mut s = String::from("hello");

    let r1 = &mut s;
    let r2 = &mut s;

    println!("{}, {}", r1, r2);
```

Here’s the error:

```text
    error[E0499]: cannot borrow `s` as mutable more than once at a time
     --> src/main.rs:5:14
      |
    4 |     let r1 = &mut s;
      |              ------ first mutable borrow occurs here
    5 |     let r2 = &mut s;
      |              ^^^^^^ second mutable borrow occurs here
    6 |
    7 |     println!("{}, {}", r1, r2);
      |                        -- first borrow later used here
```

This restriction allows for mutation but in a very controlled fashion. It’s something that new Rustaceans struggle with because most languages let you mutate whenever you’d like.

The benefit of having this restriction is that Rust can prevent data races at compile time. A _data race_ is similar to a race condition and happens when these three behaviors occur:

*   Two or more pointers access the same data at the same time.
*   At least one of the pointers is being used to write to the data.
*   There’s no mechanism being used to synchronize access to the data.

Data races cause undefined behavior and can be difficult to diagnose and fix when you’re trying to track them down at runtime; Rust prevents this problem from happening because it won’t even compile code with data races!

As always, we can use curly brackets to create a new scope, allowing for multiple mutable references, just not _simultaneous_ ones:

```rust
    let mut s = String::from("hello");

    {
        let r1 = &mut s;

    } // r1 goes out of scope here, so we can make a new reference with no problems.

    let r2 = &mut s;
```

A similar rule exists for combining mutable and immutable references. This code results in an error:

```rust
    let mut s = String::from("hello");

    let r1 = &s; // no problem
    let r2 = &s; // no problem
    let r3 = &mut s; // BIG PROBLEM

    println!("{}, {}, and {}", r1, r2, r3);
```

Here’s the error:

```text
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
```

Whew! We _also_ cannot have a mutable reference while we have an immutable one. Users of an immutable reference don’t expect the values to suddenly change out from under them! However, multiple immutable references are okay because no one who is just reading the data has the ability to affect anyone else’s reading of the data.

Note that a reference's scope starts from where it is introduced and continues through the last time that reference is used. For instance, this code will compile because the last usage of the immutable references occurs before the mutable reference is introduced:

```rust
    let mut s = String::from("hello");

    let r1 = &s; // no problem
    let r2 = &s; // no problem
    println!("{} and {}", r1, r2);
    // r1 and r2 are no longer used after this point

    let r3 = &mut s; // no problem
    println!("{}", r3);
```

The scopes of the immutable references `r1` and `r2` end after the `println!` where they are last used, which is before the mutable reference `r3` is created. These scopes don't overlap, so this code is allowed.

Even though borrowing errors may be frustrating at times, remember that it’s the Rust compiler pointing out a potential bug early (at compile time rather than at runtime) and showing you exactly where the problem is. Then you don’t have to track down why your data isn’t what you thought it was.

### Dangling References

In languages with pointers, it’s easy to erroneously create a _dangling pointer_, a pointer that references a location in memory that may have been given to someone else, by freeing some memory while preserving a pointer to that memory. In Rust, by contrast, the compiler guarantees that references will never be dangling references: if you have a reference to some data, the compiler will ensure that the data will not go out of scope before the reference to the data does.

Let’s try to create a dangling reference, which Rust will prevent with a compile-time error:

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

```text
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

```

This error message refers to a feature we haven’t covered yet: lifetimes. We’ll discuss lifetimes in detail in Chapter 10\. But, if you disregard the parts about lifetimes, the message does contain the key to why this code is a problem:

```text
    this function's return type contains a borrowed value, but there is no value
    for it to be borrowed from.
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

*   At any given time, you can have _either_ one mutable reference _or_ any number of immutable references.
*   References must always be valid.

Next, we’ll look at a different kind of reference: slices.

_You can refer to the following chapter in the Rust Programming Language Book: [References and Borrowing](https://doc.rust-lang.org/stable/book/ch04-02-references-and-borrowing.html#references-and-borrowing)_

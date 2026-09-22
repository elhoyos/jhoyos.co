---
draft: true
title: Rust Ownership Notes
date: 2026-07-23T20:00:00+01:00
---
Ownership: set of rules that govern how memory is managed in a Rust program.

Stack (LIFO)
- Push & Pop
- All data stored on the stack must have a known, fixed size.
- Faster to store
- Faster to get (access, data is close to each other)

Heap
- Allocate
- Memory allocator finds a space big enough and mark it as used, returning a pointer.
- Slower to store (find space + bookkepping to cleanup)
- Slower to get (follow a pointer, data is far from each other). Processors are faster if data is close to other data.

Problems addressed by ownership:
- Track which code uses which data on the heap
- Minimize duplicate data on the heap
- Cleanup unused data on the heap

Rules
- Each value has an owner
- There can only be one owner at a time. Q: what's "there"? is it a scope?
- When the owner gets out of scope the value is dropped. Q: dropped means cleaned up from the heap, i.e. get zero'ed?

Variable scope

A scope is a range (Q: lines/code range?) for which an item is valid. In the following, a scope within brackets:

```rust
{                     // s is not yet defined, not in scope
  let s = "Hi Juan!"  // s is in scope

                      // s is still in scope
}                     // s is dropped, not in scope anymore
```


Move

```rust
let s1 = String::from("hello");
let s2 = s1; // s1 goes ot of scope here, a move occurs
```

In a move the pointer, length and capacity to the data in the heap is moved from s1 to s2. s1 becomes out of scope and dropped.
Thus, not two allocations can have two owners.

Drop occurs when a heap-capable type implements the `Drop` trait, which implemnets the `drop` method. `drop` is called automatically and cannot be called explicitly. If an explicit drop is wanted, one should call `mem::drop`:

```rust
struct Foo {
    x: i32,
}
impl Drop for Foo {
    fn drop(&mut self) {
        println!("kaboom");
    }
}

fn main() {
    let mut x = Foo { x: -7 };
    x.drop() // E0040, manually calling destructors is not allowed.
    drop(x); // ok!
}
```

Rust will never automatically create deep copies of values in the heap. Use `clone` to create an explicit copy.

In contrast, Rust will deep copy values in the stack since there's no actual difference between a shallow and a deep copy.

```rust
let x = 5;
let y = x; // stack-value, deep copied
```

Types annotated with the `Copy` trait are assumed to be stack-capable and automatically removed from it.

Rust won't allow these types to also implement the `Drop` trait.

Scope and assignment

```rust
let mut s = String::from("hello");
s = String::from("ahoy"); // "hello" data is no longer in scope due to re-assignment
```

Function parameters follow the same previous ownership mechanics.

Borrowing: to avoid the transfer of ownership between functions, use `&` to create a reference. Unlike a pointer, a reference is guaranteed to point to an existing value.

The analogy: one can borrow the value from its owner but one has to return it after using it because we don't own it. Read "reference" when you see "borrowing".

A reference will point to the pointer, length, capacity tuple.

```rust
{
  let s1 = String::from("hello");

  let len = calculate_length(&s1); // creates a reference to s1 without owning its value

  println!("{s1} is {len} bytes long");

  fn calculate_length(s: &String) -> usize { // takes a reference to a String type
    s.len()
  } // the reference s goes out of scope here, but it does not own s1
}
```

Q: do references always do not hold ownership of values they point to?

The opposite to `&`, dereferrencing is done with `*`.

An analogous in other languages to borrowing is passing by reference.

Use `&mut` to create a mutable reference. However, to prevent a data race (race condition) Rust enforces that only one mutable reference at a time.

```rust
{
  let s1 = String::from("hello");
  
  let r1 = &s1; 
  let r2 = &s1; // this is fine
  let r3 = &mut s1; // BOOM

  println!("{r1}, {r2} and {r3}")
}
```

```rust
{
  let s1 = String::from("hello");
  
  let r1 = &s1; 
  let r2 = &s1;
  println!("{r1}, {r2}");

  let r3 = &mut s1; // this is fine since r1 and r2 are no longer accessed
  println!("and {r3}");
}
```

Also, to have non-simultaneuous mutable references one can create a new scope:

```rust
{
  let s1 = String::from("hello");

  {
    let r1 = &mut s1; // r1 exists until the end of the inner scope
  }

  let r2 = &mut s1; // no problem, r1 reference is gone at this point
}
```



# 00 — Rust intro

Core ideas of the language, in plain Rust (no embedded stuff yet). Try each
snippet with `cargo run` in a scratch project (`cargo new scratch`), or later
tie it back to code in `src/bin/`.

## Variables are immutable by default

```rust
let x = 5;
x = 6; // error: cannot assign twice to immutable variable
```

Add `mut` to allow change:

```rust
let mut x = 5;
x = 6; // fine
```

Why: the compiler catches accidental mutation. If a value isn't marked
`mut`, you know nothing later in the function changes it.

## Ownership: one owner at a time

Every value has exactly one owner. When the owner goes out of scope, the
value is dropped (freed).

```rust
let s1 = String::from("hello");
let s2 = s1;        // ownership MOVES from s1 to s2
println!("{s1}");   // error: s1 no longer owns the value
```

This is Rust's alternative to garbage collection: no runtime needed to
track memory, the compiler tracks it at compile time instead.

Types that are cheap to copy (numbers, bools, chars) don't move, they copy:

```rust
let a = 5;
let b = a; // a is copied, both are valid
println!("{a} {b}"); // fine
```

## Borrowing: use a value without taking it

Instead of moving, pass a reference (`&`):

```rust
fn len(s: &String) -> usize {
    s.len()
}

let s = String::from("hello");
let n = len(&s);       // borrow s
println!("{s} {n}");   // s is still valid here
```

Rules the compiler enforces:
- Any number of `&T` (read-only) references at once, **or**
- Exactly one `&mut T` (read-write) reference, never both at the same time.

This is how Rust prevents data races and use-after-free without a garbage
collector: it's checked once, at compile time.

## Structs: group related data

```rust
struct Point {
    x: f32,
    y: f32,
}

let p = Point { x: 1.0, y: 2.0 };
println!("{}", p.x);
```

Add behaviour with `impl`:

```rust
impl Point {
    fn distance_from_origin(&self) -> f32 {
        (self.x * self.x + self.y * self.y).sqrt()
    }
}

let d = p.distance_from_origin();
```

## Enums: one of several variants

```rust
enum Direction {
    North,
    South,
    East,
    West,
}
```

Variants can carry data, and each variant its own shape:

```rust
enum Shape {
    Circle(f32),           // radius
    Rectangle(f32, f32),   // width, height
}
```

## Pattern matching with `match`

```rust
fn area(shape: Shape) -> f32 {
    match shape {
        Shape::Circle(r) => 3.14159 * r * r,
        Shape::Rectangle(w, h) => w * h,
    }
}
```

`match` must cover every variant, the compiler checks it. Miss one, and it
won't compile. This is what makes enums so useful in Rust: you can't forget
a case.

## No null: `Option<T>`

Rust has no `null`. A value that might be absent is `Option<T>`:

```rust
enum Option<T> {
    Some(T),
    None,
}

fn find(items: &[i32], target: i32) -> Option<usize> {
    for (i, &item) in items.iter().enumerate() {
        if item == target {
            return Some(i);
        }
    }
    None
}
```

You're forced to handle the `None` case before you can use the value:

```rust
match find(&[1, 2, 3], 2) {
    Some(i) => println!("found at {i}"),
    None => println!("not found"),
}
```

Shortcut for "give up if there's nothing there":

```rust
let i = find(&[1, 2, 3], 2)?;   // returns early with None if find() gave None
```

(`?` only works in a function that itself returns `Option` or `Result`.)

## No exceptions: `Result<T, E>`

Operations that can fail return `Result` instead of throwing:

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}

fn parse_number(s: &str) -> Result<i32, std::num::ParseIntError> {
    s.parse()
}

match parse_number("42") {
    Ok(n) => println!("got {n}"),
    Err(e) => println!("failed: {e}"),
}
```

Same `?` shortcut works here too: propagate the error upward instead of
handling it on the spot.

## Traits: shared behaviour across types

A trait is like an interface: a set of methods a type promises to have.

```rust
trait Greet {
    fn greet(&self) -> String;
}

struct Person { name: String }

impl Greet for Person {
    fn greet(&self) -> String {
        format!("Hello, {}", self.name)
    }
}

fn say_hello(g: &impl Greet) {
    println!("{}", g.greet());
}
```

Any type that implements `Greet` can be passed to `say_hello`. This is how
Rust does polymorphism without inheritance.

## Summary

| Concept | What it solves |
| --- | --- |
| `mut` | accidental mutation |
| ownership / move | memory safety without a garbage collector |
| borrowing (`&`, `&mut`) | safe sharing, checked at compile time |
| `struct` | group data |
| `enum` + `match` | one of several known shapes, exhaustively handled |
| `Option<T>` | "maybe no value", no null |
| `Result<T, E>` | "maybe an error", no exceptions |
| `trait` | shared behaviour across types |

Next: how these ideas look in **embedded, `no_std` Rust** — the kind this
repo is written in (no heap, no operating system, no `std`).

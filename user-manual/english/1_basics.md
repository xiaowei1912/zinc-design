# Basics

## hello world

A basic Zinc program looks like this:

```rust
fn main() {
    println("hello world");
}
```

Source files end with the `.zn` extension, and the file contents must be UTF-8 encoded. The compile command is:

```
zinc main.zn
```

After it finishes, you will see a newly generated executable in the current folder. Running that program prints the string `hello world`.

## Comments

Zinc supports two kinds of comments: line comments and block comments.

1. Line comments

    A line comment starts with `//` and runs to the end of the line.

2. Block comments

    A block comment starts with `/*` and ends at the first `*/`.

> **TODO**: Documentation comments are not implemented yet.

## Functions

An ordinary Zinc function definition starts with the `fn` keyword, followed by the function name and a pair of parentheses. Inside the parentheses is the parameter list. After the parentheses you can write the function's return type. After that comes the function body, enclosed in braces.

Example:

```rust
fn my_func(arg1: Int32, arg2: String) -> Bool {
    return true;
}
```

If a Zinc project needs to produce an executable, it must define a unique `main` function in the root module as the program entry point.

## Variables and types

Identifiers: start with `_` or a letter; afterward `_`, digits, or letters are allowed.

A lone `_` is a special identifier meaning "ignore."

> **TODO**: Unicode identifiers and raw identifiers are not supported yet; to be implemented.

### Local variables

Local variables do not have to be explicitly annotated with a type; type inference is allowed.

If a variable name is not marked with `mut`, it is immutable by default.

```rust
fn main() {
    let x = 5_i;
    x = 6; // error: x is immutable
}
```

For readability, local variables in the same block are **not** allowed to have the same name. Local variables in different blocks can of course have the same name.

### Static variables

Static variables are defined with the `static` keyword.

A static variable's type must satisfy the `Send` constraint (note that which types satisfy `Send` differs from Rust); see the "Parallelism and Concurrency" chapter. To use a non-`Send` type as a static variable, it must be marked `unsafe`.

A static variable must be initialized at definition time, and the initializer must be a "constant expression."

A static variable may be marked `mut`. But a `mut` static variable must be marked `unsafe`. Reads and writes of it must both be in an unsafe context.

```rust
unsafe static mut G: Int32 = 1; // a static variable marked mut, or a static variable whose type does not satisfy Send, must be marked unsafe

fn main() {
    println(G.to_string()); // error: G can only be used in an unsafe context
}
```

> **Planned feature**: Type inference for static variables may be supported later.

### Constants

Constants are defined with the `const` keyword.

Similar to static variables, a constant's type must satisfy the `Send` constraint. A constant must be initialized at definition time, and the initializer must be a "constant expression."

Constants cannot be marked `mut`.

```
const PI: F32 = 3.14;
```
> **Planned feature**: Type inference for `const` constants may be supported later.

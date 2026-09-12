
# Pattern Matching

Pattern matching is used mainly with `is` expressions and `match` statements.

The purpose of pattern matching is to check whether an expression's structure matches expectations, and also to allow extracting inner values and binding them to new variables.

Examples:

```
// x is a tuple with 3 elements; bind the second element to middle, then check whether middle > 5
if (x is (_, middle, _)) && middle > 5 {

}

// y is a pointer to a trait; we can use this to check what concrete type y actually points to. If the check succeeds, the pointer is bound to p.
if y is p: *S {
    // p can be used inside block
}

// z has type Option<S>; the following handles the Some and None cases separately
match z {
    Some(s) => {

    }
    _ => {}
}
```

Like expressions, patterns can be nested.

## WildcardPattern

Placeholder patterns.

The `_` underscore can occupy one position.

The `...` ellipsis can occupy multiple positions.

## LiteralPattern

Literals of types such as integers, floating-point numbers, Bool, and Char may be used directly as patterns.

`Str` literal patterns are not implemented yet.

## TypePattern

Type patterns are used mainly to check whether an expression is a concrete subtype. A type pattern starts with the `type` keyword.

Example:
```
// Suppose TR is a trait name and S is a struct name.
// The following checks whether a pointer to a trait can be successfully downcast to a pointer to S.
fn test(p: &TR) {
    let b = p is type &S;  // start with the type keyword to avoid a syntax conflict with the modifier+identifier pattern below
}
```

## StructPattern

A struct pattern matches a struct value and can destructure its fields.

Example:
```rust
struct Point { x: Int, y: Int }

fn test(p: Point) {
    // match the struct and bind fields to new variables
    if p is { .x = a, .y = b }: Point {
        println(f"x = $(a), y = $(b)");
    }
    
    // use an ellipsis to ignore other fields
    if p is { .x = 10, ... }: Point {
        println("x is 10");
    }
    
    // nested pattern matching
    match p {
        { .x = 0, .y = 0 } => println("origin");
        { .x = 0, .y = y } => println(f"on y-axis at $(y)");
        { .x = x, .y = 0 } => println(f"on x-axis at $(x)");
        { .x = x, .y = y } => println(f"point ($(x), $(y))");
    }
}
```

## TuplePattern

A tuple pattern matches a tuple value and can destructure its elements.

Example:
```rust
fn test(t: (Int, String, Bool)) {
    // match the tuple and bind elements to new variables
    if t is (num, str, flag) {
        println(f"num = $(num), str = $(str), flag = $(flag)");
    }
    
    // use underscores to ignore some elements
    if t is (1, _, true) {
        println("first element is 1 and third is true");
    }
    
    // nested tuple pattern
    let nested = ((1, 2), (3, 4));
    if nested is ((a, b), (c, d)) {
        println(f"($(a), $(b)), ($(c), $(d))");
    }
}
```

## EnumVariantPattern

An enum-variant pattern matches a specific variant of an enum type and can destructure its associated values.

Example:
```rust
enum Message {
    Quit,
    Move(Int, Int),
    Write(String),
}

fn test(msg: Message) {
    match msg {
        Message::Quit => println("quit");
        Message::Move(x, y) => println(f"move to ($(x), $(y))");
        Message::Write(text) => println(f"write: $(text)");
    }
}
```

## ModifierPattern

Modifier patterns control how variables are bound in pattern matching.

- `mut`: mark the bound variable as mutable
- `&`: match a reference
- `&mut`: match a mutable reference
- `ref`: bind by reference rather than moving
- `ref mut`: bind by mutable reference

Example:
```rust
fn test(s: String) {
    // mut modifier: make the bound variable mutable
    let mut x = s;
    x.push_str(" modified");
    
    // ref modifier: bind by reference, avoiding a move

    // ref mut modifier: bind by mutable reference

}
```

## IdentifierPattern

An identifier pattern binds the matched value to a variable name.

Example:
```rust
fn test(x: Int) {
    // simple identifier binding
    if x is y {
        println(f"y = $(y)");
    }
    
    // used in match
    match x {
        0 => println("zero");
        n => println(f"other: $(n)");
    }
    
    // combined with other patterns
    let tuple = (1, 2, 3);
    if tuple is (first, ...) {
        println(f"first = $(first)");
    }
}
```



## Not yet supported
StrPattern
GroupedPattern
MacroInvocationPattern
RangePattern
SlicePattern
Pattern Guard
Or pattern

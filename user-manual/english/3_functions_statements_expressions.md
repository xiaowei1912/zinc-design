## Expressions, statements, and functions

## Expressions

Zinc is not an expression language.
1. Keywords such as `break` `continue` `return` `goto` `panic` can only be used in statements, not directly in expressions.
2. Only expressions can produce a value. Statements cannot.
3. Statements may contain expressions, but expressions cannot directly contain statements. Lambda expressions are an exception: they may contain statements, but `return`/`break`/`continue` inside those statements only affect the lambda itself. Outwardly the lambda still behaves as an expression and cannot transfer control flow.

If the user needs to execute relatively complex statements in sequence inside an expression and finally produce a value, use a lambda and call it immediately.
Control-flow statements inside the lambda only take effect in the lambda's function body and have no effect on the surrounding expression.

### Operator expressions

#### The `Result` type

Zinc uses `Result<R, E>` as the primary error-handling mechanism. It is an enum defined in the standard library:
```
enum Result<R, E> {
    Ok(R), Err(E)
}
```

The `E` in `Result<R, E>` represents error information. Users may use any type to describe the error.

* If there are only a very limited number of possible errors, designing an enum as the error information is quite reasonable.

* If we only need the error information for logging, simply using a string as the error type is also fine.

* If we need to classify errors across multiple levels, designing a set of `trait MyBussinessError` and its inheritance hierarchy, and using `*MyBussinessError` as the returned error type, is also a good design.

* If the function author really does not need the caller to care about the concrete error type, just return `Result<R, *Any>` uniformly. Any error type, once boxed, can be converted to `*Any`.
The caller can downcast as needed and handle only the error types it cares about.

* If we want the caller to obtain the call stack at the time of the error, wrap the standard library's `BackTrace` type in a custom type and use that as the error type.
The `BackTrace` type can describe the current call stack; pass it outward as part of the error information.

#### The `?` operator

`?` is a postfix operator, used as `expr?`.

The `?` operator is mainly meant to be used with the `Result` type. `?` requires the expression before it to have type `Result`.

> **Planned feature**: Overloading of the `?` operator will be supported later so it can work with custom types.

The execution semantics of the statement `let x = expr?;` are:

```
match(expr) {
    Ok(r) => {
        let x = r;
    }
    Err(e) => {
        return Err(e);
    }
}
```

Because it can cause the whole function to return early, it places requirements on the function's return type.
If `expr` has type `Result<R1, E1>` and the function's return type is `Result<R2, E2>`, then `E2` must be able to convert from `E1` by natural implicit conversion, or `E2` must implement the `From<E1>` trait.
Only then can the compiler convert `E1` to `E2` and return it.

### Parenthesized expressions

`unsafe(<expr>)` is allowed; then the `<expr>` expression is in an unsafe context. The value of this expression equals the value of `<expr>`.

`const(<expr>)` is allowed; then the `<expr>` expression is in a const context. This expression must be evaluated at compile time.

### Index expressions

### Function-call expressions

Function calls

Method calls

### Member-access expressions

struct members

tuple members

### Closure expressions

Zinc's closure syntax differs from Rust. When designing closure syntax, Zinc mainly considered the following points:
1. Closure syntax needs to work well with "trailing-call" syntactic sugar. Trailing-call syntax can produce very clean, attractive DSLs in many scenarios. Functions such as lock / spawn / lazy are all a good fit for trailing-closure syntax.
2. Closures need both a full form and a short form
3. The full-form closure syntax must be complete: it should support an explicit capture list, generic declarations, a full parameter list, and an explicit return type
4. The short form should be short enough to omit every unnecessary syntactic element, while still resembling the full form

An example of the full-form closure syntax:
```
.{
    async [move x, weak y, ref z]<T>(arg: T) -> ReturnTy where T: Constrait
    =>
    statements(x, arg);
    return y;
}
```

The full closure syntax starts with `.{` and ends with `}`, which works well with "trailing-call" syntax. The contents inside are:
1. A qualifier list, including `async` `const` and so on
2. An explicit capture list in brackets; capture modes support `move` `ref` `ref mut` `weak`
3. A generic parameter list in angle brackets
4. A function parameter list in parentheses
5. An explicit return type after `->`
6. An optional where clause
7. The closure body after `=>`

An example of the most abbreviated form:
```
(x) => x + x
```
The parentheses contain the parameter list; parameters may omit type declarations. After `=>` is the closure body; without braces, this body can only be a single expression and may not contain statements.




### Branch expressions

if cond then expr else expr

### yield/await expressions



### is expressions

The `is` keyword replaces `if-let` `while-let` `matches!`

### as expressions


### Constant expressions


## Statements

Statements are generally terminated by a semicolon. Empty statements are allowed.

### Block statements

Braces contain multiple statements. A statement block introduces a new scope; local variables in an inner scope may have the same name as variables in an outer scope.

Empty blocks are allowed.

### Unsafe statement blocks

A block statement marked `unsafe`.

### let statements

```
let <pattern> = <expr> ;
```

> **TODO**: Currently a variable declaration must be initialized, to simplify the implementation and avoid using CFG analysis to check that a variable is initialized before use. This requirement can be relaxed later, checking only that it is definitely initialized before being read.

### Expression statements

```
<expr> ;
```

### if statements

```rust
fn test(cond: Bool) {
    if cond {
        println("then branch");
    } else {
        println("else branch");
    }
}
```

### match statements


### Loop statements

#### `while`

1. The condition after while must be a Bool expression
2. No semicolon is needed after the while's braces

#### `do-while`

1. The condition after while must be a Bool expression
2. A do-while must end with a semicolon

#### `for`

`for  <pattern> in <expr>  { <statements> }`

1. for supports pattern matching
2. expr must be a type that satisfies the `Iterable` or `IterableMut` trait, or a pointer to such a type.

There are three desugarings of a `for-in` statement, depending on how `<pattern>` is written:

1. The pattern is an identifier: `for i in v { <body> }`, in which case `i` has type `v`'s `Iterable::Item`. Desugars to:

    ```
    {
        let mut iter = v.iter();
        while (iter.next() is Some(p)) { // p is a read-only borrow of Item
            let i = *p;                // i is a new Item copied from the read-only borrow
            <body>
        }
    }
    ```

2. The pattern is a ref identifier: `for ref i in v { <body> }`, in which case `i` has type `v`'s `&Iterable::Item`. Desugars to:

    ```
    {
        let mut iter = v.iter();
        while (iter.next() is Some(i)) { // i is a read-only borrow of Item
            <body>
        }
    }
    ```

3. The pattern is a ref mut identifier: `for ref mut i in v { <body> }`, in which case `i` has type `v`'s `&mut Iterable::Item`. Desugars to:

    ```
    {
        let mut iter = v.iter_mut();
        while (iter.next() is Some(i)) { // i is a read-write borrow of Item
            <body>
        }
    }
    ```

### Jump statements

#### `return`

1. return may or may not be followed by an expression.
2. The type of the expression after return must be compatible with the function's return type.

#### `break`

1. break can only be used in loop blocks, including `while` `do-while` `for`.

#### `continue`

1. continue can only be used in loop blocks, including `while` `do-while` `for`.

#### `panic`

The panic keyword can be compared to the throw keyword in languages such as C++/Java. But Zinc has no try-catch syntax and does not provide a `catch_unwind` function.

panic should only be used when an unrecoverable error occurs. The panic keyword may be followed by a string expression. Example:

```rust
fn unwrap<T>(o: Option<T>) -> T {
    if o is Some(v) {
        return v;
    } else {
        panic "unwrap Option failed";
    }
}
```

#### `goto`

`goto` is a reserved keyword and is not implemented yet. It is reserved for several purposes.

1. Together with label statements, it would provide jump functionality, replacing Rust's break-label statement. The main issue is that break-label use cases are limited; it can only be used in loop blocks. But goto-label in C has another more common use: writing error handling uniformly at the end of a function, and goto-ing there from other places in the function when an error is encountered. That style is extremely convenient and readable. Using goto for this scenario is very reasonable. Of course, to prevent programmers from abusing it, we should place some restrictions on goto statements:
    1. goto is a statement and cannot be used inside an expression.
    2. It cannot jump across functions or lambdas.
    3. It can only jump out of a block, not into a block. Loop bodies and different match arms are also different blocks.
    4. It can only jump forward, not backward.
    5. It must be guaranteed that on the goto path, all variables are initialized before use.

    In short, I want to provide a safe and reliable goto statement. It should be able to implement C's common pattern of unified error handling at the end of a function, without introducing other risks.

2. Prepared for the "guaranteed tail recursion" feature, replacing Rust's `become` keyword.
    1. Allow `goto call_fn();` to implement guaranteed tail recursion.


## Functions

A function definition starts with the `fn` keyword, followed by the function name and parameter list. The parameter list is enclosed in parentheses, then an optional return type, and finally the function body enclosed in braces.

```rust
// function definition
fn function_name(arg1: Int, arg2: Char) -> Bool {
    println("function body");
    return true;
}

fn main() {
    function_name(0, 'X'); // function call
}
```

Omitting the return type means the function returns `()`.

Parameters support named parameters; named parameters start with a dot. When calling, if you use named arguments, they also start with a dot. The dot syntax is used so it is consistent with struct initialization expressions.
Named parameters must come after non-named parameters.

```
fn run(.from: &Str, .to: &Str) {
    println("run from {} to {}", from, to);
}

fn main() {
    run("home", "bar");  // may be called in parameter order
    run(.to = "bar", .from = "home"); // may also be called by name; then order does not matter.
}
```

Because functions support named parameters, supporting pattern matching on parameters would be a bit complicated. Besides, most patterns are not very useful in parameter position, so function parameters do not support pattern matching.
In Rust, the most useful pattern supported in parameter lists is the `mut` pattern, which allows the parameter itself to be modified. In Zinc, the suggested workaround is:
```
fn f(v: Vec<Int>) {
    let mut v = move v; // the parameter v is not marked mut; you can define a mut variable of the same name inside the function and move the parameter into it.
    // ...
    v.push(2); // this needs a &mut borrow of v, which requires the variable v to be marked mut
    // ...
}
```

Parameters support default values; a default value must be a constant expression. Parameters with default values must come after parameters without default values.

```
fn increase(value: Int = 1) {}

fn main() {
    increase();  // the argument value is 1
    increase(2); // the argument value is 2
}
```

Trailing closures are supported. The meaning of trailing-closure call syntax is: if, when the function is defined, the last parameter's type is an fn/Fn/FnMut type, or a pointer to one of those types, then the caller may write the lambda outside the argument list.
If there are no other parameters besides this function-type parameter, the caller may omit the parentheses required by the argument list.
Example:

```
// the last parameter of lock is a function type
fn lock(f: &Fn()) {
    println("lock");
    f();
    println("unlock");
}

fn main() {
    lock( ()=>println("smth") ); // OK, a short-syntax lambda as an argument
    lock( .{ () => println("smth"); } ); // OK, a full-syntax lambda as an argument

    // trailing-closure call syntax: write the closure outside the argument list
    lock().{
        () => println("smth");
    };
    // trailing-closure call syntax: when there are no other arguments, omit the argument-list parentheses
    lock.{
        () => println("smth");
    };
    // later the lambda syntax can be simplified further, allowing the => symbol to be omitted
    lock.{
        println("smth");
    };
}
```

Function overloading is supported. But the number of parameters must differ.

```
fn f1(arg1: Int, arg2: Int = 1) {} // may accept 1 argument or 2 arguments.
fn f1(arg: String) {} // error: cannot form an overload with the f1 above. This version may accept 1 argument, and the version above may as well. Conflict.
```

todo: Variadic parameters are not supported for now; to be improved later. In ordinary cases, use function overloading instead; when there are too many parameters, use arrays and slices instead.

You can add member functions to a type through an `impl` block.
When the first parameter's name is the `self` keyword, the function may be called with dot syntax.

```rust
impl User {
    fn send_email(&self) {}
}
fn main() {
    let u: User = { ... };
    u.send_email();
}
```

The `self` parameter is special and has several shorthand forms:
* `self` means `self: Self`
* `&self` means `self: &Self`
* `&mut self` means `self: &mut Self`
* `*self` means `self: *Self`
* `*weak self` means `self: *weak Self`

Functions can be used as first-class values. Function-pointer types are supported.

todo: Introduce new syntax in function signatures to indicate that the compiler may automatically insert auto-ref/auto-deref conversions from actual arguments to formal parameters. This replaces Rust's `AsRef trait / AsMut trait`

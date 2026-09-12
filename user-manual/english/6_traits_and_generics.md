# Generics and Traits

## Generics

Both type definitions and function definitions can declare generic parameters.

### Generic types

User-defined types can carry generic parameters.
There are two kinds of generic parameters: lifetime generic parameters and type generic parameters. In the generic parameter list, the order is: lifetime parameters before type parameters.

todo: Later we need to consider whether const generics should be added, and should pay attention to the reasons Swift does not support that feature

```rust
struct S<'a, T> {
    p: &'a T
}

enum E<T> {
    A(T), B(Int)
}
```

Type definitions support an optional `where` clause. A `where` clause can express: lifetime outlives relationships, and whether a type satisfies a trait bound.

Using a generic type requires providing generic arguments.

```rust
fn main() {
    let v = 1;
    let x: S<Int> = S { .p = &v };
}
```

### Generic functions

Functions also support generics. As with generic types, generic parameters support lifetime generic parameters and type generic parameters.
Generic functions also support an optional `where` clause.

```
fn swap<'a, T>(lhs: &'a mut T, rhs: &'a mut T) {}
```

Note: Zinc's generic-function call syntax has changed, avoiding Rust's turbofish syntax.

<details>

<summary>What is turbofish syntax</summary>

<div style="border: 1px solid black; padding: 10px;">

In Rust's syntax design, if a generic function call used the same syntax as a generic function definition, there would be a syntax ambiguity, as in the following example:
```rust
fn main() {
    let (the, guardian, stands, resolute) = ("the", "Turbofish", "remains", "undefeated");
    let _: (Bool, Bool) = (the<guardian, stands>(resolute)); // what does this line mean?
}
```

The core of this ambiguity is that the less-than operator and generic angle brackets reuse the same symbols, so the line above has two interpretations:
1. This is a generic function call inside parentheses; the function name is `the`, the generic arguments are `guardian` `stands`, and the function argument is `resolute`
2. This is a tuple with two elements. The first element is the comparison `the<guardian`; the second element is the comparison `stands>(resolute)`.

To resolve this syntax conflict, Rust requires that when calling a generic function you put `::` after the function name, so the correct call syntax is:
```
the::<guardian, stands>(resolute)
```

That is turbofish syntax. For example `Vec::<i32>::with_capacity(16)`.

</div>
</details>

<br/>

Rust's turbofish syntax eliminates the parse ambiguity, but it introduces syntactic inconsistency and is not very elegant.
Zinc uses a different design that both avoids the ambiguity and keeps the syntax consistent.

1. If a generic type appears in fully-qualified-call-syntax, it must always start with the `type` keyword. Examples:
    * `(type Vec<Int>)::with_capacity(4)`  
      Because `Vec` explicitly carries generic arguments, `Vec<Int>::with_capacity(4)` is a syntax error.
    * `Vec::with_capacity(4)`  
      You may omit specifying `Vec`'s generic parameters; they can be inferred from context, and in that case the type prefix can be omitted.
    * `(type Vec<String> as Default)::default()`  
      You may explicitly specify both the type name and the corresponding trait name, becoming a fully qualified call of a specific function. This means member functions in different traits may have the same name, and may both be implemented for the same type, without conflict.

2. Struct initialization and pattern matching always use postfix type syntax. Examples:
    * Struct initialization: `let v = { .x = 1 } : S<Int>;`
    * Struct pattern matching: `expr is { .x = 1 } : S<Int>`

3. For generic function calls, the generic argument list is written inside the parentheses. Examples:
    * `std::mem::size_of(<Int>)`

The core logic of these changes is that every type name carrying generics always appears in a context prefixed by a specific identifier.
That way, when the compiler is parsing, once it sees the corresponding symbol or keyword it knows that what follows must be a type rather than an expression, and that `<` `>` appearing there should be understood as angle brackets rather than less-than and greater-than.

The following focuses on generic-function call syntax:
```
// the function-definition syntax is unchanged
fn f<T>(arg: T) {}

// the function-call syntax has changed
fn main() {
    f(<Int>, 1); // explicitly specify the generic argument type
    f(1); // inferred form, omitting the generic argument
}
```

The main reasons for this syntax design are, first, to avoid the ugly turbofish syntax, and second, to match how Zinc implements generic functions.

Ignoring special-case compiler optimizations, Zinc generic functions are by default implemented via dictionary passing. The compiler would translate the function f above into:
```
// pseudo-C corresponding to the definition of f:
void f(ZnTypeMeta * typeof_T, void * arg) { }
```

When executing a call such as `f(<Int>, 1)`, the caller really does take the ZnTypeMeta corresponding to `Int` and pass it into f as a function argument.
So writing the function's generic argument list inside the parentheses better matches the execution semantics of Zinc generic functions.

A Zinc function-pointer type may include generics: `let pf: fn<T>(T)->() = f;`.
Zinc does not allow function currying; writing `let pf: fn(Int)->() = f(<Int>);` will fail to compile.
For that kind of function-instantiation scenario, please use a lambda; this is allowed: `let pf: fn(Int)->() = .{ (arg: Int)->() => f(<Int>, arg); };`.

The main benefit of implementing generic functions this way is that generic functions can also be distributed via dynamic libraries. That is a design advantage of Swift.

This is useful for the following scenario:
Suppose we designed a dynamic library `a.so` that exposes generic function interfaces, used by an application `b.exe`.
Then we can update `a.so`'s internal implementation (without making breaking API changes) without recompiling `b.exe`, and the program will still work automatically.
If generics are implemented via monomorphization, this scenario is hard to support. Updating an upstream library requires downstream applications to be recompiled and redeployed.

In addition, this design comes with some other benefits:
1. Support for `virtual` functions that carry generics. See the following sections on `virtual` functions.
2. Reduced code size and reduced compile time.

The cost, of course, is lower execution efficiency. This prevents many compiler optimizations, is cache-unfriendly, and is unfavorable for execution time.
But we can still, at the implementation level, in certain specific scenarios, use extra compiler options to allow monomorphization as a performance optimization, striking a better balance between code size and execution efficiency.
1. When compiling an executable rather than a library, generics can be implemented entirely via monomorphization
2. Even when compiling a library, as long as there is no need to distribute and deploy the dynamic library separately, generics can be implemented via monomorphization

> **TODO**: UnSized types are currently not allowed as generic arguments, and traits cannot be used as generic arguments either.

> **Planned feature**: Later we can support "or" conditions as where-clause conditions, as syntactic sugar for overloading on parameter types:
```
fn f<T>(arg: T) where T: Int32 | UInt32 {}
// equivalent to:
trait AnonymousTR {}
impl AnonymousTR for Int32 {}
impl AnonymousTR for UInt32 {}
fn f<T>(arg: T) where T: AnonymousTR {}
```

## Traits

An example of trait-definition syntax:

```
trait TrName<T> : SuperTrait1 + SuperTrait2 where T: Condition {
    type AssocType;

    virtual fn method(&self);

    fn static_method() -> Self;
}
```

A trait definition supports:
1. Optional generic parameters
2. Optional parent traits; multiple parent traits are supported, separated by `+`
3. An optional where clause

A trait definition body may include:
1. Associated types
2. Associated constants
3. Associated functions

### impl trait

Traits can be used to abstract uniformly over different types. The syntax for specifying that a concrete type `SomeType` implements a trait `TrName` is:

```
impl TrName for SomeType {

}
```

An impl block
1. Functions inside it cannot be marked `pub` individually
2. Nor can they be marked `virtual`

`impl Trait for Trait` is not allowed.

**Orphan rule**

The basic principle is:

### Pointers to traits

The two use cases for traits are:
1. As an upper bound on a generic constraint
2. As a pointer-to-trait type

Global variables, local variables, and fields are not allowed to use a trait directly as a type. But they may use a pointer to a trait as a type.

If type `S` implements `trait R`, then a pointer to `S` can be upcast to a pointer to `R`. Example:

```rust
struct S {}
trait R {
    virtual fn f(&self);
}
impl R for S {
    fn f(&self) {}
}

fn test(p: *S) {
    let p1: *R = p; // ok, upcast
    p1.f();
}
```

### virtual functions

A pointer to a trait can call the trait's member functions. But there is a restriction: only member functions marked `virtual` can be called through a pointer to a trait.

> Note:
> Rust allows using `where Self: Sized` syntax to mark a trait member function as "non-virtual." That syntax is very odd and hard to understand. Zinc introduces a keyword to do the marking, which is more readable.
>

Example:

```rust
trait TR {
    virtual fn f1(&self);
    fn f2(&self);
}

fn test(p: &TR) {
    p.f1(); // OK
    p.f2(); // compile error: f2 is not a virtual function and cannot be called through a pointer to a trait
}

fn test2<T>(arg: T) where T: TR {
    arg.f2(); // OK. Non-virtual functions can still be called in generic form.
}
```

Also note that not every function written in a trait can be marked `virtual`. A function that can be marked `virtual` must satisfy the following requirements:
1. The first parameter's name must be `self`, and its type must be a pointer type pointing at `Self`. `&self` `&mut self` `*self` `*weak self` are all allowed.
2. Other than the first parameter, other parameters may not use the `Self` type or associated types
3. The function's return type may not use the `Self` type or associated types

Example:
```
trait TR {
    virtual fn clone()->Self; // compile error: cannot be marked virtual. No self parameter, and the return type uses Self.
    virtual fn g(arg: Int); // compile error: cannot be marked virtual. No self parameter.
    virtual fn h(self);  // compile error: cannot be marked virtual. The self parameter is not a pointer type.
}
```

**Note**: virtual functions may introduce new generic parameters. Example:
```
trait TR {
    virtual fn f<T>(&self, arg: T) where T: Hash + Ord + Default; // OK
}
```

> **Planned feature**: Later we can support the `* Trait1 + Trait2 + 'a` spelling:
1. When using `+`, it cannot be a concrete type; only a trait plus a trait
2. There can be only one non-marker trait type. Adding a trait to itself is also not allowed.
3. `* Trait1 + Trait2` can be implicitly converted to `*Trait1` or `*Trait2`

This feature may be necessary in some scenarios; for example, we want `* MyTrait + Sync` to express that the pointed-to type satisfies not only the MyTrait bound but also the Sync bound.
There is no other way to express that.

### Associated types

An associated type is a type placeholder defined in a trait; a concrete type must be specified when implementing the trait.

Example:
```rust
trait Container {
    type Item;
    
    fn add(&mut self, item: Self::Item);
    fn get(&self) -> Option<Self::Item>;
}

struct IntContainer {
    items: Vec<Int>,
}

impl Container for IntContainer {
    type Item = Int;
    
    fn add(&mut self, item: Int) {
        self.items.push(item);
    }
    
    fn get(&self) -> Option<Int> {
        self.items.last().copied()
    }
}
```

The `Self` type represents the concrete type that implements the trait.

> **TODO**: Higher-kinded types are not currently supported.

### Trait inheritance

Traits in Zinc support inheritance. When defining a trait you may specify zero or more parent traits.

For traits with an inheritance relationship, a pointer to a child trait can be implicitly converted to a pointer to a parent trait (both ARC and BORROW pointers). Example:
```
trait TR: TR1 + TR2 + TR3 {}

fn test(p: *TR) {
    let p1: *TR1 = p; // OK
    let p2: *TR2 = p; // OK
    let p3: *TR3 = p; // OK
}
```

A pointer to a parent trait can also be downcast to a pointer to a child trait via pattern matching. For example, both `is` and `match` can do this.

```
trait TR: TR1 + TR2 + TR3 {}

fn test(p: *TR1) {
    if (p is p1: *TR) { // try to downcast; it may fail
        // use the p1 variable
    }

    match (p) {
        p2: *TR => {}
        _ => {}
    }
}
```

Other features include:
1. A child trait may specify values for associated types / consts
2. A child trait may override a parent trait's function body
3. Multiple inheritance is supported; name conflicts in multiple-inheritance scenarios need to be considered
    1. In multiple-inheritance scenarios, name conflicts are forbidden
    2. In multiple-inheritance scenarios, adding a parent type or changing the inheritance order both affect the ABI
    3. `trait TR1 : TR2 {}`  and `trait TR1 where Self: TR2 {}` have different meanings.

### reuse

The purpose of the reuse mechanism is to reuse fields and member functions. reuse is syntactic sugar that lets the user proxy an implementation onto another type. Example:

```
struct S1 { }
impl TR1 for S1 {
    fn f1(&self, arg: Int) {}
}

struct S2 {
    base: S1  // the field name is unrestricted; it can be anything
}
impl TR1 for S2 {
    reuse self.base;  // meaning: the member functions in this impl block all reuse the member functions of the same trait implemented by self.base
}

// the above is equivalent to:
impl TR1 for S2 {
    // implement every member function, and each function body calls the corresponding member function of `self.base`.
    fn f1(&self, arg: Int) {
        self.base.f1(arg);
    }
}

```

### Common built-in traits

`Any` trait

`Fn` trait

`Send` trait

`Sync` trait

### The object-oriented programming paradigm

Through the combination of the language features above, Zinc can also support the object-oriented programming paradigm, but in a style different from the common industry style (such as C++/Java/C#/Swift).
Zinc's main characteristic is: it does not support `class` or class inheritance. In Zinc only traits can inherit, and only pointers to traits can do dynamic dispatch.

The following explains from several angles why it is designed this way.

1. A reference-semantic `class` type is not flexible enough in memory layout

    From a memory-layout perspective, a reference-semantic class in other languages can be seen as Zinc's `ARC pointer + struct` combined. You are only allowed to use the reference-semantic form, not the corresponding value-semantic struct.
    That means that when a class-type variable is used as a local variable or a field, there is always a dynamic allocation and an extra layer of pointer indirection. That is actually a regression in expressiveness, not an enhancement.
    If we provide the `ARC pointer` and the `struct` to the user separately, for example `Vec<S>` and `Vec<*S>` are different types that can be chosen as needed in different scenarios, that is more flexible and the user has more freedom.

    A reference-semantic class, in every scenario, carries an "object header" pointer inside each object. Zinc's design does not affect the object's own memory layout.

2. Adding a reference-semantic `class` type would make Zinc's type system inconsistent, requiring special rules in many scenarios and a more complex implementation

    A reference-semantic class would bring a series of extra rules into Zinc's type system, which is not worthwhile. Especially when various pointers are used together with class.
    Zinc already has several pointer types, and a "reference-semantic class" itself can be understood as a combination of an "implicit pointer + struct."
    This feature is not orthogonal to other language features, because the user lacks the choice of whether this "implicit pointer" should be ARC, weak, or a borrow.

    Consider this scenario: when `T` is a class, what does `&T` mean? Does this borrow point at the pointer variable itself, or at the object's start address?
    Because class forcibly binds an `ARC pointer` and a `struct` together, either design choice brings trouble.
    Separating the pointer and the struct makes the semantics clearer. Type `&S` is a borrow of `S`; type `&*S` is a borrow of `*S`. The semantics are clear and the implementation is simple.

    Consider the following generic example; you will find that unifying a value-semantic struct and a reference-semantic class in generics is troublesome:
    ```
    fn test<T>(arg: T) {
        // If we introduce a reference-semantic class, then address-of and dereference expressions behave differently for value-semantic types and reference-semantic types.
        // In a generic scenario, we would need extra runtime checks to implement a uniform abstraction over these two cases with generics.
        let p: &T = &arg;
        let c: T = *p;
    }
    ```

3. What if, like C++, we introduce class and inheritance, but class is not reference-semantic?

    In C++, class and struct have little semantic difference. They support value types and are not bound to pointers.
    But class inheritance means a subtype must accept all interfaces implemented by the parent type, which is problematic.

    In traditional inheritance design, a child class necessarily implements all interfaces/protocols of the parent class.

    Consider a trait like `Send` in Zinc, which requires that all fields satisfy `Send`. If we introduced inheritance, we would get this situation: the parent type implements `Send`,
    the child type inherits the parent type but adds a non-`Send` field; we want the child type to reuse all of the parent type's fields and methods but not implement `Send`, and traditional inheritance cannot do that.

    In a design that uses inheritance, a child type cannot pick only some of the parent type's interfaces to implement. To achieve the above, you must also drop inheritance and use composition. So inheritance can only conveniently reuse code in some scenarios; it cannot adapt to all scenarios.

4. Class inheritance requires special rules for special scenarios

    1. A typical scenario is multiple inheritance.
    Under a traditional inheritance design, if we support a class inheriting from multiple classes, we must consider the "diamond inheritance" scenario.
    C++ designed special syntax and semantic rules for this, and many coding guidelines also place many constraints on this feature.
    Languages such as Java/C#/Swift simply do not support multiple inheritance, limiting code-reuse ability in order to simplify the semantic rules.

    2. Another typical scenario is constructors. Different languages have different rules. In C++, calling a virtual function from a constructor does not have virtual-function effect.
    Java/C# constructors never access uninitialized variables, relying on "zero-value initialization" of fields.
    Swift cannot support "zero-value initialization" and also wants constructors never to access uninitialized members, so its rules are the most complex.
    Constructors restrict the function name, which means supporting constructors requires supporting type-based function overloading; constructors restrict the return type, which means you cannot use the return type for error handling.
    These design ideas do not match Zinc.

    In short, traditional languages support class inheritance, but that feature comes with extra special semantic rules; different languages have different rules, and it is not very elegant.

Zinc follows Rust's design approach and adopts a simpler design: only traits may inherit.

Zinc's design is easier for users to understand and use:
* All field reuse uses the "composition" pattern;
* All method reuse uses the "reuse mechanism";
* All scenarios that need a unified public interface over multiple types use "trait+impl";
* All scenarios that need dynamic dispatch use a "pointer to a trait."

The above rules apply to all types (built-in types, enums, structs, tuples, strings, arrays, and so on), with no special cases. The above rules are complete enough to support the OOP paradigm. There is no expressiveness problem; features supported by traditional languages can all be expressed, just with slightly more verbose syntax in some cases.

The semantic rules are lean, consistent, and orthogonal.

### Memory layout

A pointer to a trait is a fat pointer. The pointer types mentioned in this section include these six: `*T`, `*weak T`, `&T`, `&mut T`, `*raw T`, `*raw mut T`.

```rust
trait TRBase { virtual fn f1(&self); }
trait TRSub : TRBase { virtual fn f2(&self); }

struct S { i: Int }
impl TRBase for S { fn f1(&self) {} }
impl TRSub for S { fn f2(&self) {} }

fn main() {
    let p1: *S = box S{.i = 1}; // p1 is a thin pointer
    let p2: *TRSub = p1; // p2 is a fat pointer
    let p3: *TRBase = p2; // upcast

    if p3 is psub: *TRSub {
        // downcast
    }
    if p2 is ps: *S {
        // downcast
    }
}
```

Zinc's fat-pointer layout differs from Rust. A pointer to a trait contains 3 fields:
```
      p2
┌───────────────────┐
│ object_ptr        │→ points at the object's head. The object may be a built-in type, a user-defined struct, an enum, etc.
│───────────────────│
│ typemeta_ptr      │→ points at the concrete type's TypeMeta, which includes the type's name, size, and so on, and also destructor and copy function pointers
│───────────────────│
│ vtable_ptr        │→ points at the vtable of this concrete type for this trait, which is an array of function pointers
└───────────────────┘
```

* A pointer to a trait can call the trait's virtual member functions. When calling a member function, the corresponding function pointer is taken from the vtable and then called; the offset is determined at compile time.

* A pointer to a trait can be converted to a pointer of another type via pattern matching, because the fat pointer stores the object's runtime type information. At runtime we can determine which concrete type this trait pointer points to; if the match succeeds, take out the object_ptr it contains and use it as a pointer to the concrete type. Downcasting is also supported: `*TRBase` can be checked at runtime to see whether it is a `*TRSub`.

* A pointer to a trait supports upcasting. A `*TRSub` can be converted to a `*TRBase`.

> Note:
> In Rust, a pointer to a trait is called a trait object; it is also a fat pointer, but it does not support downcasting. Only the special `Any` trait supports downcasting; other traits do not.
> Because it is only two pointers in size, it is somewhat fat, but not fat enough to support all of the object-oriented programming paradigm Zinc wants to support.
>
> Referring to Swift's Protocol memory layout, you can see that only one more pointer is needed. The two metadata pointers designed for Zinc's fat pointer can be compared to Swift's value witness table and protocol witness table.
>

## Open issues:

1. Will forbidding impl trait for pointer types cause expressiveness problems? (Planning later to introduce special syntax on function parameters, equivalent to a built-in AsRef/AsMut trait)

2. Will forbidding impl trait for arbitrary types cause expressiveness problems?
    ```
    impl<T> ToString for T where T: Display {} // this spelling is not currently supported. The suggestion is to use trait inheritance instead.
    ```

3. How to allow UnSized types as generic arguments? Currently `Mutex<File>` cannot be implemented; a hardcoded `MutexFile` is too weakly composable.

4. Should we support impl TR as a return type?

5. Should we support const generics? Note that Swift does not; this is likely a significant challenge for the runtime and the ABI.

## Covariance & contravariance

|           |   'a   |   T   |   U   |
|-----------|--------|-------|-------|
| &'a T     |  covariant  |  covariant  |       |
| &'a mut T |  covariant  |  invariant  |       |
| *T        |        |  covariant  |       |
| Mutable<T>|        |  invariant  |       |
| Vec<T>    |        |  covariant  |       |
| fn(T)->U  |        |  contravariant  |  covariant  |
| *raw T    |        |  covariant  |       |
| *raw mut T|        |  invariant  |       |

Type covariance and contravariance can continue to be supported later.
The `in` `out` keywords are already reserved.

Example use cases:
1. Convert `Option<*Sub>` to `Option<*Base>`
2. Convert `Result<*SubR, *SubE>` to `Result<*BaseR, *BaseE>`
3. Convert `(*Sub, *Sub)` to `(*Base, *Base)`
4. Convert `fn(*Base)->*Sub` to `fn(*Sub)->*Base`

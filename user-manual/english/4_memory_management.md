# Memory Management

## ARC pointers

Zinc uses reference counting as its core memory-management strategy. ARC stands for automatic reference counting. Here "automatic" means that when incrementing or decrementing a reference count, the compiler automatically chooses atomic or non-atomic instructions.

Every block of memory allocated on the heap carries two reference-count values: a strong count and a weak count.
When the strong reference count drops to 0:
* If the weak count is not 0, this block's destructor is called, but the memory is not freed. The memory is freed only when the weak count also drops to 0.
* If the weak count is 0, the destructor is called and the memory is freed immediately.

Because reference-counted pointers are the most commonly used pointer type, Zinc maps the most concise pointer syntax onto ARC-pointer semantics. The `*T` type is an ARC reference-counted pointer to `T`.

A weak-reference pointer type is written `*weak T`. A weak-reference pointer to `T` cannot access `T`'s fields or methods and cannot be dereferenced.
A weak-reference pointer must be converted to a strong-reference pointer `*T` via `std::ptr::upgrade` before it can access `T`'s fields or methods.

To allocate an object on the heap, use the `box` keyword. For a `box <expr>` expression, if `<expr>` has type `T`, then `box <expr>` has type `*T`.
If a `box` expression fails to allocate memory, it panics immediately.
If the user really needs to handle allocation failure manually, they can call `std::alloc::try_box` themselves and inspect the return value to determine whether allocation succeeded.
Scenarios that truly need to handle allocation failure manually are not a great fit for Zinc anyway: the entire standard library panics directly on OOM. Judging allocation success only at the application layer is not enough.


Example:
```rust
let x: MyObj = MyObj::new();  // if MyObj::new() returns MyObj
let y: *MyObj = box MyObj::new(); // then box MyObj::new() returns *MyObj

let z = y; // z and y point to the same MyObj object
```

The memory layout of an ARC pointer to a statically sized type (a sized type; we set unsized types aside for now) looks like the following. Each dynamic allocation first does `malloc(sizeof(UInt) + sizeof(MyObj))` and then offsets the pointer to the object's start address.
Accessing a field through an ARC pointer uses that field's offset relative to the object's start address.
```
                ┌──────────────────────┐
                │ strong_count: UShort │
                │ weak_count: UShort   │
ptr: *MyObj  →  │──────────────────────│
                │ field1               │
                │ ......               │
                └──────────────────────┘
```

In the presence of a reference cycle, as long as one pointer on the cycle is a weak-reference pointer, the cycle can be collected normally without leaking. When to use a weak-reference pointer must be specified by the programmer.

Zinc has no ownership concept as in Rust, no type with semantics similar to `Box<T>`, and no move-only "ownership type."

**Note**: There is no `*mut T` type. Whether or not the bound variable itself is `mut`, a `*T` type always has permission to modify its fields.

Example:

```rust
struct S { m: i32, n: i32 }
fn main() {
    let p1: *S = box { .m = 1, .n = 2 }:S ;
    p1.m += 10; // OK: even though the p1 binding is not marked mut, it has permission to modify fields

    p1 = box { .m = 10, .n = 20 }:S; // error: the p1 binding is not marked mut, so it cannot be assigned to directly
}
```

## Value semantics and reference semantics

[Value semantics and reference semantics](https://isocpp.org/wiki/faq/value-vs-ref-semantics#val-vs-ref-semantics) explained:
1. Value semantics means that a copy duplicates the value; the new variable and the old variable have no relationship. A modification of either is not reflected in the other.
2. Reference semantics means that a copy copies the pointer; the new variable and the old variable share data, and modifications to that shared data affect both variables.

The criterion is the logic of the following pseudocode:
```rust
let mut y: T = x;
modify(&mut y); // modify y
print(x);
print(y);
// If a modification of y can never affect x, type T can be said to be value-semantic. Otherwise, type T is reference-semantic.
```

Zinc encourages users, when defining a type, to design it as a "value-semantic" type.

Because if a type `S` is defined as value-semantic, the corresponding `*S` type is reference-semantic, and the user can easily choose `S` or `*S` depending on the scenario.
If it is designed as a "reference-semantic" type at definition time, it is hard to obtain the corresponding "value-semantic" type, which is inconvenient for users.

## Borrowing

Why do we need borrow pointers?

Because ARC/WEAK pointers alone are not expressive enough.
A `*T` type can only ever point at the head of a "dynamically allocated object." A dynamically allocated object always carries a reference count, because the pointer must operate on that count and must guarantee that the count sits at a fixed offset from the address it points to.
If we need a pointer into the middle of an object, we cannot use a `*T` pointer.

If every safe pointer in the language can only point at an object's head, object memory layout cannot be compact. That design is unfriendly to value types.

If we allow taking the address of the middle of an object and obtaining a new pointer, as shown below:
```
                ┌───────────────────┐
                │ strong_count: u32 │
                │ weak_count: u32   │
ptr: *MyObj  →  │───────────────────│
                │ field1: i32       │
                │ field2: MyStruct  │ ← borrow_ptr: &MyStruct
                └───────────────────┘
```

Then this borrow_ptr necessarily has the following properties:
1. At runtime it does not have enough information to find the reference count carried by the object it points into
2. At runtime it has no ability to actively control the lifetime of the object it points into

Therefore a borrow pointer's lifetime must be checked statically at compile time, ensuring the borrow pointer lives shorter than the borrowed object; otherwise dangling-pointer problems appear.

So Zinc also keeps Rust's borrow pointers and lifetime parameters. A borrow pointer can only be obtained by an "address-of" operation, and the compiler tracks the borrow pointer's live range at compile time.
To distinguish different read/write permissions, borrow pointers are split into two kinds: the read-write `&mut T` type and the read-only `&T` type. The corresponding operator expressions are `&mut <expr>` and `&<expr>`.

Borrow pointers are a good complement to the expressiveness of ARC pointers:
1. An ARC pointer can only point at an object dynamically allocated on the heap; a borrow pointer can point at an object on the heap or on the stack.
2. An ARC pointer can only point at the head of a heap-allocated object and does not support pointer arithmetic; a borrow pointer can point at an object's head or middle, and can achieve pointer offsets by borrowing fields.
3. Copying or passing a `*T` type always increments or decrements the reference count, with extra performance cost. Assigning, passing, and returning a borrow pointer does not operate on the reference count. Using borrow pointers reasonably helps reduce redundant increment/decrement operations.

What Zinc and Rust borrow pointers have in common:
* Both have lifetime checking. That is, the compiler must check at compile time that the borrow pointer itself lives shorter than the borrowed object.
* Both have read/write permission checking. An `&T` borrow has read permission but not write permission; an `&mut T` borrow has read and write permission.

How Zinc and Rust borrow pointers differ:
* Mutability rules differ. In Zinc, for an ARC pointer type `*T`, whether or not the variable binding itself is mutable, you can always obtain both `&T` and `&mut T` borrows from that variable.
* Zinc has no "ownership types" and no "borrow-checking rules." A read-write borrow is not exclusive. The semantic rules are much more relaxed.


### Example 1: A borrow pointing at a stack address
```rust
struct S { m: Int, n: Int }
fn main() {
  // Allowed to point directly at temporaries and literals. The compiler automatically generates an anonymous local variable and lets the borrow pointer point at it.
  let mut p2: &Int = &1;

  {
    let x1 = { .m = 1, .n = 2 }: S;
    let p1 = &mut x1; // compile error: cannot obtain a read-write borrow of the read-only variable x1
    p2 = &x1.m; 
  }
  print(p2); // compile error: x1 lives shorter than p2

  let mut x2 = { .m = 3, .n = 3 }: S;
  let p3 = &mut x2; // correct
  let p4 = &x2.n; // correct
  p3.m += 1;    // correct: p3 has write permission
  println(p4); // correct: p3 and p4 existing at the same time is fine; p3 is not exclusive
}
```

### Example 2: A borrow pointing at a heap address

A Zinc code example:

```rust
fn main() {
  let mut b: *Int = box 1_i;

  let p1: &Int = &*b; // allowed, pointing at a heap address
  let p2: &mut Int = &mut *b; // allowed, pointing at a heap address

  // p1/p2 may exist at the same time; a mut borrow is not exclusive
  println(f"$(p1)");
  *p2 += 1;
  println(f"$(p2)"); 

  b = box 10_i; // reassign b

  // reading and writing p1/p2/b is all fine; p1/p2 have not been invalidated
  println(f"$(p1)");
  *p2 += 1;
  println(f"$(p2)"); 
  println(f"$(b)");
}
```
If the example above were written in Rust, changing `*Int` to `Box<i32>`, it would fail to compile. Because Rust's mutable borrow is exclusive, it cannot coexist with other borrows. But Zinc compiles it, and there is no memory-safety problem.
The reason is that, in Zinc, when you borrow a field through an ARC pointer, the compiler always generates an extra temporary that copies `b`, ensuring the reference count is incremented by 1, and then borrows the field. That temporary is released at the end of the current block, at which point the reference count is automatically decremented by 1.
Therefore a later assignment to `b` does not cause the originally pointed-to memory to be freed immediately; `p1`/`p2` remain valid until the function ends, at which point the original memory is freed.

Here is a multi-level ARC pointer scenario:
```rust
struct Obj { p: *Int } // Zinc has no move-semantic pointer type, so only * pointers can be used.

fn main() {
  let mut o: *Obj = box { .p = box 1_i }:Obj;
  let q: &mut Int = &mut *o.p; // this temporarily copies the p pointer, not o
  *q = 2; // q has write permission

  // create a borrow pointer through an ARC pointer
  let r: &mut Obj = &mut *o;
  r.p = box 10_i; // allowed: no compile error and no dangling pointer.

  o = box { .p = box 100_i }:Obj; // allowed: no compile error and no dangling pointer.

  println(*q); // q still points at the original value; prints 2
  println(*r.p); // r.p points at the new value 100
}
```

In short, when creating a new borrow pointer through an Arc pointer, Zinc's way of preventing the borrow pointer from becoming dangling is to protectively increment the reference count and delay freeing the memory.
The compiler only needs to do local static analysis inside a single function to guarantee memory safety.

From an execution-performance perspective: this design will certainly sacrifice some performance, but it should not have too large an impact.

1. What was described above explains expected behavior from a semantic point of view, not the concrete instructions from an optimization point of view.

   From an optimization point of view, we do not need to increment the reference count every time a new borrow pointer is created. Only when the compiler cannot guarantee at compile time that the object the borrow points to will definitely stay alive in that region will it, to be safe, automatically increment the reference count to avoid freeing memory too early.
   In many scenarios the compiler has enough information to optimize away these extra increment/decrement operations.

2. Compared with Rust, the release of some heap-allocated objects is delayed.

   From the perspective of guaranteeing memory safety, mark-and-sweep GC uses a similar idea: as long as a pointer still points at this memory, the runtime guarantees it will not be freed.
   The difference is only that mark-and-sweep GC can handle cycles, while ARC cannot.
   Note that a borrow pointer created inside a function cannot live longer than that function, so if a block of memory is delayed from being freed because a borrow pointer exists, then after the code block containing that borrow pointer ends, the memory can be freed as well; it will not be delayed for very long.

3. As long as we use borrow pointers reasonably, compared with a design that uses ARC everywhere, we can often reduce how frequently reference counting happens.
   1. When a borrow pointer is assigned, passed as an argument, or returned, the reference count does not need to be modified
   2. Obtaining a borrow of a field through a borrow pointer does not require modifying the reference count, as long as the field is not an ARC pointer type
   3. Even when borrowing a field that is an ARC pointer type, some redundant reference-count modifications can be optimized away. That optimization depends on whether the compiler has enough information to know that the object an ARC pointer points to has no risk of being freed during a certain lifetime.


From an ease-of-use perspective: this design is a major simplification of Rust. **The borrow checker is removed, and the restriction that an &mut borrow is exclusive is gone.** At the type level, move-only types are gone.


From a memory-layout perspective: the combination of ARC and borrow pointers can preserve the programmer's control over memory layout.
1. Every type's memory layout is deterministic; the compiler does not implicitly insert hidden fields.
2. Only when we explicitly use the `box` keyword to allocate on the heap is a reference count recorded at the head of the dynamically allocated memory, and only then is there extra reference-count memory overhead. Local variables used on the stack have no extra reference-count memory overhead.
3. When any type is used as a field, it is laid out inline, with no extra reference-count memory overhead. Unless the programmer specifies that a field is an ARC type, the compiler will not implicitly insert a box operation or a reference-count value.

## Slightly more complex borrowing

Zinc container types such as `Vec<T>` also support borrowing their elements. Example:

```rust
fn main() {
    let mut vec = (type Vec<Int>)::from(&[1i,2i,3i,4i] as Slice<Int>);
    let p: &Int = vec.index(1); // later, with operator overloading, this can be written &vec[1]
    vec.push(5); // this may cause vec to reallocate
    println(*p); // how do we guarantee p is valid?
}
```

Here our strategy is still: when creating the borrow `p: &Int`, increment the underlying array's reference count, and decrement it when `p` dies.

The question is: if `p` is a borrow, how does the compiler know that this borrow variable should perform a reference-count operation when it dies, and at which address?

Answer: the code above can compile only through cooperation between the compiler and the standard library. The borrow operation `vec.index(1)` does not actually return a raw borrow type `&Int`, but a struct that carries extra data: a "smart pointer type similar to a borrow type."
Then, through that temporary, a `deref` operation is performed to obtain the final `p: &Int` borrow pointer.

```rust
// This type implements the Deref trait, so the compiler can insert auto-deref to obtain a &T, then access the element through &T.
// This type implements the Drop trait, so it can decrement the corresponding array's reference count in its destructor
struct ItemRef<'a, T> {
    ptr: *Shared<T>, // points at the array header
    item: &'a T, // the actual borrow
}
```

After the compiler desugars it, the code above actually means this:

```rust
fn main() {
    let mut vec = (type Vec<Int>)::from(&[1i,2i,3i,4i] as Slice<Int>);
    let _temp: ItemRef<Int> = vec.index(1); // a local variable implicitly added by the compiler; this is a "fat borrow pointer"
    let p: &Int = _temp.deref(); // an "auto-deref" operation implicitly added by the compiler
    // Note: if we do not bind the return value of index to a local variable, this _temp should be destroyed immediately after the index statement, not at the end of the block.

    vec.push(5); // on reallocation, the copy-on-write condition is triggered; new space is allocated, but the old space's reference count is greater than 0 so it is not freed, and p remains valid
    println(*p);
    // the compiler implicitly destroys the _temp variable, at which point the old space's reference count becomes 0 and the original array contents are freed
}
```

Note: the "auto-deref" rule is that, in pattern matching, argument passing, and return scenarios, if `T: Deref<U>` then `T` can be converted to `U` by automatically calling the `deref` member function.

To achieve the above, the Index trait in the Zinc standard library is defined as follows:
```
pub trait Index<Idx>
{
    type Output<'a>;

    fn index<'s>(&'s self, index: Idx) -> Self::Output<'s>;
}
```

Note that the return type of index is no longer hardcoded as the raw borrow type `&Item`, but a struct type `struct ItemRef<'a, Item>` that wraps `&Item` together with other data.
We can treat it as a custom smart-pointer type: a type that "lets a raw borrow type carry extra metadata," a kind of "fat borrow pointer."
Therefore this `Output` associated type must also be defined as a type constructor, i.e. a generic associated type in Rust.
This pointer contains not only a borrow of the specific element, but also a pointer to the internal array header, and this fat pointer has a destructor, so it has enough information at runtime to maintain the reference count correctly.

Looking at it from a semantic-analysis perspective alone, this seems to have a large performance impact. But if we consider inlining the key functions used here — `index`, `deref`, and `drop` — the extra reference-count operations can be completely eliminated in the performance-optimization stage:
1. If in main no modifying functions such as push are called, and no mutable borrow of vec is taken, the compiler has ample information that this memory has no risk of being freed, and can delete all extra increment/decrement operations and surplus local variables;
2. If in main a mutable borrow of vec is taken and passed into other functions, the compiler is unsure whether vec may be modified; then the protective reference-count operations are indispensable, and that is not a waste of performance.

## How Zinc solves the classic iterator-invalidation problem

When explaining Rust's memory-safety features, the following is a classic example:

```rust
///// Rust code
fn main() {
    let mut vec: Vec<i32> = Vec::from(&[1i,2i,3i]);
    for ref _i in &vec {
        vec.push(4);
    }
}
```

Rust will reject this example at compile time, because `vec.iter()` creates a read-only borrow of `vec` stored inside the iterator, while `vec.push` needs to create a read-write borrow of `vec`.
By Rust's borrow-checking rules, the two conflict, so the compiler reports an error, effectively preventing iterator invalidation and avoiding a kind of undefined behavior common in C++. This is an advantage of Rust worth praising.

Zinc guarantees safety in a different way. The corresponding Zinc code looks very similar at the source level, but the actual execution effect is quite different from Rust:

```rust
fn main() {
    let mut vec: Vec<Int> = (type Vec<Int>)::from(&[1i,2i,3i,4i] as Slice<Int>);
    for ref _i in vec {
        vec.push(4);
    }
}
```

1. Because Zinc has no "ownership types," `Vec<T>` is not move-semantic but copy-semantic. We can freely `let vec2 = vec;` and still continue to use the `vec` variable.
2. Although `Vec<T>` is copy-semantic, that does not mean that when it is copied, all elements are copied immediately; instead it implements a copy-on-write optimization.
When copying a `Vec<T>`, only a shallow copy is performed: the reference count of the internally heap-allocated array is incremented by 1, and the element contents are not actually copied.
3. A deep copy happens when a `Vec<T>` variable is modified. If, at modification time, the internal array's reference count is greater than 1, all elements are copied and then the modification is applied.
4. In the example above, creating the iterator increments the internal array's reference count by 1. If there is no write to `vec` during iteration, a deep copy never happens;
if `vec.push` occurs during iteration, a deep copy is triggered: `vec` creates a new internal array and appends a 4.
5. But the array the iterator points to has not changed; it has not been freed or modified, and iteration continues. The iterator and `vec` now point at two completely different arrays, so there is no undefined behavior.
6. After the for loop finishes, the iterator's lifetime ends; in its destructor it decrements the original array's reference count by 1, and only then is the original array destroyed and freed.

So Zinc likewise avoids "undefined behavior" and does not introduce "memory unsafety," but pays a certain performance cost: when push is executed, the original `Vec<T>` implicitly performs a deep copy.
That cost is acceptable. We neither want undefined behavior as in C++, nor overly strict compile-time checks as in Rust. Under the premise of guaranteeing "safety" and "ease of use," this performance loss is already the smallest price that must be paid.
If the user wants to save this deep copy, they need to take care not to modify the original vec during the loop.

For Rust, memory safety and thread safety are **design goals**. The "alias XOR mutation principle" is the **theoretical basis** for reaching those goals.
Ownership and the borrow checker are the **implementation means**. Zinc chooses copy-on-write to achieve the same goals; different paths, same destination.
Or, from another angle, Zinc's design turns the compile-time static check of "alias XOR mutation" into a runtime guarantee.
Copy-on-write essentially guarantees that when memory is modified, that memory has only one alias, which is the same idea as ownership + borrow checker. A detailed comparison:

* Rust's compile-time ownership + borrow-checker check guarantees at compile time that an object has only a unique mutable alias. Multiple immutable aliases may exist at the same time.

  If the alias XOR mutation principle is violated, compilation fails.

* Rust's `RefCell<T>` is used through the `borrow` and `borrow_mut` member functions. It guarantees at runtime that an object has only a unique mutable alias, but multiple immutable aliases may exist at the same time.

  If the alias XOR mutation principle is violated, it panics at runtime and refuses to continue.

* Rust's `RwLock<T>` is used through the `read` and `write` member functions. It guarantees at runtime that an object has only a unique mutable alias, but multiple immutable aliases may exist at the same time.

  If the alias XOR mutation principle is violated, the current thread blocks at runtime until the condition is satisfied and then continues. `Mutex<T>` is similar; it guarantees that a piece of data can have only one accessor at a time, whether reading or writing.

* Zinc's copy-on-write is implemented by not checking the ref-count in immutable methods, and checking the ref-count in mutable methods, guaranteeing that the object has only a unique alias at modification time.

  If the alias XOR mutation principle is violated, the heap-allocated object is copied. This guarantees that there is only one alias when memory is modified or freed.

All of the implementation techniques above satisfy the same "alias XOR mutation" principle, and they can all achieve the goal of "memory safety." Rust's design is only one of the options for implementing "memory safety"; it is absolutely not the only feasible design.

## Lifetimes

A lifetime is the interval during program execution in which a variable is alive. A variable's lifetime begins when it is created and ends when it is destroyed.

### Lifetimes of global variables

A global variable lives throughout the program's execution, so its lifetime exists from program start to program end.

> **TODO**: In the current implementation, no global variable has its destructor called. Whether this design is reasonable, and whether we need to destroy global variables before exiting the process, still needs discussion.

### Lifetimes of temporaries

A temporary is a variable produced by an expression but not bound to a concrete name.

There are two times when a temporary's lifetime ends: at the end of the current statement, or at the end of the current statement block. Which one applies depends on whether it has been borrowed by a variable with a longer lifetime.

Example:
```rust
fn f() -> S { ... }
// assume S has a member method method
impl S {
  fn method(&self) -> &S { return self; }
}

fn test() {
  f(); // the variable returned by f() is a temporary; it is not bound to a variable name. This temporary's lifetime ends when the statement ends.

  let p1: &S = &f(); // the temporary returned by f() is borrowed, and the borrow's lifetime exceeds this statement. Then this temporary's lifetime is extended to the end of the block; the temporary is destroyed when the function ends.

  let p2: &S = f().method(); // the temporary returned by f() is borrowed, and the borrow's lifetime exceeds this statement. Then this temporary's lifetime is extended to the end of the block; the temporary is destroyed when the function ends.
  // use p1 p2
}
```

### Lifetimes of local variables

In the general case, the lifetime of a local variable, including a function parameter, starts when it is created and ends when the current code block ends.

There is no such thing as a non-lexical-lifetime, and there is no need for one. Rust introduced that because of the exclusivity of mut borrows; if analysis is not precise enough, it causes many unnecessary compile errors and limits the user's expressiveness. If we do not have a compile-time rule checking exclusivity of mut borrows, then we also do not need to find ways to relax the compile-time checks.

* For a custom type, we can implement the `Drop trait`; when such a variable's lifetime ends, the corresponding destructor is called automatically.
* For an ARC type, when its lifetime ends, the reference count of the pointed-to heap memory is automatically decremented by 1.
* For a borrow type, its destructor does nothing.

Example:
```rust
struct S { m: i32 }
impl Drop for S {
  fn drop(&mut self) {
    println("drop S");
  }
}

fn test() {
  let s1 = { .m = 1 }:S;
  let s2 = s1; // a copy happens here

  // s1 and s2 are destroyed when test ends
}
```

The reason we said earlier, "in the general case, a local variable's lifetime ends when the code block ends," is that there are also "special cases." The special case is a `move` expression.

In Zinc, the move keyword may be followed by an expression, called a move expression. Then move semantics occur, not copy semantics.

```rust
fn test() {
  let x = { .m = 1 }:S;
  let y = move x;
  println(x.m); // compile error: x's lifetime has already ended; x cannot be used afterward
}
```

If what is moved is an ARC pointer, we can use this feature to reduce increment/decrement operations in some scenarios:
```rust
fn f(arg: *S) { }

fn test() {
  let s: *S = box { .m = 1 }:S;
  f(s); // if called this way, passing the argument copies it, the reference count is incremented by 1, and at the end of f's body it is decremented by 1. When leaving test, the reference count is decremented by 1 again.
  f(move s); // if called this way, we can ensure that the reference count is not automatically incremented when passing the argument, and is not decremented at the end of test's body. s's lifetime is transferred to f's function body.
  // after s is moved, using s again triggers a compile error

  // ...
}
```

If we want variable `x` to end its lifetime early, we can use a `move x;` statement, so that the result of the move expression is not bound to any variable; then this x is destroyed on the spot, rather than at the end of the block.

In short, in Zinc, the choice of move semantics is not on the "type definition"; every type is naturally copyable or movable.
Whether to use move semantics is a choice on the "expression." Zinc provides move-semantic "expressions," not move-semantic "types."

A `return` statement is move-semantic by default; `return expr;` is equivalent to `return move expr;`, and you do not need to write the `move` keyword explicitly.

## Lifetime annotations

A lifetime annotation can be seen as a kind of generic parameter. Borrow types need this kind of generic parameter.
Explicit lifetime annotations are generally used in function signatures, to express lifetime relationships between parameters and the return type.

```rust
fn find<'a, 'b>(strings: Slice<'a, Str<'b>>) -> Str<'b> {

}
```

> **TODO**: Rules for eliding lifetime annotations in function signatures are not yet fully implemented.

## Destructors and `Drop`

A destructor is an object's member function. When the object's lifetime ends, the compiler automatically inserts instructions to call the destructor.

A destructor does the following:
1. Calls the type's `Drop::drop` function, if it has one
2. Calls the fields' `Drop::drop` functions
3. Decrements the corresponding reference counts of all Arc-pointer and Weak-pointer fields contained in the type. If an object pointed to by an ARC pointer has its reference count drop to 0, that object's destructor is called automatically.

A destructor is always generated automatically by the compiler; the user can only control the behavior of the `Drop::drop` function, which is only part of the destruction process.
If a type and all of its members have no `Drop::drop` function and no reference-counted pointers, the type's destructor is empty.

In general, users should not actively call an object's destructor. But when writing unsafe code, the user may force-call an object's destructor via `unsafe fn destruct_in_place`.

`Drop::drop` can never be called explicitly by the user.

An example of a user-defined type implementing the `impl std::mem::Drop` trait:

```rust
struct S {
  p: *i32,
  m: String,
}

impl Drop for S {
  fn drop(&mut self) {
    println("S is dropped.")
  }
}
```

Note: a custom `Drop::drop` function does not need to worry about destroying fields or freeing their memory; field destructors are called automatically.

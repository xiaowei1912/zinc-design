
# Parallelism and Concurrency

## Thread safety

Rust has an unmatched advantage: it naturally guarantees a program's thread safety. Among commonly used programming languages, it is unique. Zinc also wants to keep this thread-safety characteristic.

### Send/Sync traits

We give a uniform marker to every type that can be passed across threads safely, called the `Send trait`.

First, recall the meaning of [value semantics and reference semantics](https://isocpp.org/wiki/faq/value-vs-ref-semantics#val-vs-ref-semantics).

As you can see, value-semantic types naturally satisfy the `Send trait`. Examples:
1. Basic types such as Bool, Char, and Int follow value semantics, and therefore definitely satisfy the `Send trait`.
2. Composite types such as tuple/struct/enum, if all of their members satisfy `Send`, are also necessarily value-semantic, so those types also satisfy the `Send trait`.
3. Pointer types do not follow value semantics; in general `*T` does not satisfy the `Send` trait. But it also depends on the specific situation. Pointer types, or types that contain pointer members, also satisfy `Send` in the following two cases:
    1. They implement value semantics through a copy-on-write mechanism; these also satisfy the `Send trait`. For example the `String` type: although it contains a pointer member internally, its implementation uses copy-on-write to guarantee "value semantics," so this type also satisfies the `Send trait`. For a type such as `Vec<T>`, although it also implements copy-on-write, because it is generic, whether it satisfies `Send` has a precondition. `Vec<T> : Send` if and only if `T: Send`.
    2. They guarantee that the pointed-to contents are thread-safe through a thread-synchronization mechanism; these also satisfy the `Send trait`. For example the `*AtomicInt32` type: although two different pointers in two threads may point at the same `AtomicInt32`, because `AtomicInt32` is thread-synchronized, the `*AtomicInt32` type satisfies the `Send trait`.

Therefore we introduce another `Sync trait`, meaning that a type itself has thread-synchronization capability. Then we can say that for a `*T` type, `*T` satisfies `Send` if and only if `T` satisfies `Sync`.
The same applies to borrow pointers. `&T : Send` and `&mut T : Send` if and only if `T: Sync`.

Examples of types that satisfy the `Sync trait`:
1. The `Atomic` family of types
2. Types such as `Mutex<T>` and `RwLock<T>`. Note: because these types themselves are generic, whether they satisfy `Sync` has a precondition. `Mutex<T> : Sync` if and only if `T: Send`.

Once we have these two traits, we still need to do two things to guarantee thread safety:
1. In the standard library, mark all basic types, and types defined by the standard library, with `Send` and `Sync`.
2. On APIs that need to pass data between different threads, add a constraint that the type passed across threads must satisfy the `Send` condition.

User-defined types generally do not need to be marked `Send` `Sync` manually; the result inferred by the compiler is enough.
If the type's implementation uses unsafe, the programmer then needs to analyze whether the type satisfies `Send` `Sync` and specify it explicitly via unsafe impl.

### No data races

Once the compiler and standard library have prepared the `Send` `Sync` types above, we can guarantee "no data races" in business code through compile-time checks.
If there is a possibility of a data race, the compiler will automatically report an error.

Example:

```
// suppose send's job is to send data to another thread for use; then it needs to constrain data's type to satisfy Send
fn send<T>(data: T) where T: Send { }

fn main() {
    send(1_i);
    send(String::new());
    send((type Vec<Int>)::new());
    send(box 1_u); // Error. compile error: *UInt does not satisfy the Send constraint
}
```

### Global variables

Global variables can naturally be accessed by different threads, so they also need restrictions to guarantee thread safety.
1. A global variable's definition may be marked with the `static` or `unsafe static` keywords. A global variable marked `unsafe` can only be read and written in an unsafe region. Only a read-only global variable defined with `static` is safe.
2. The type of a global variable not marked `unsafe` must satisfy the `Send trait`.

**Note**: unlike Rust's rules, Zinc requires global variables to satisfy the `Send` constraint, not the `Sync` constraint.

Because Zinc's `Sync trait` and Rust's `Sync trait` have different semantics. Many types that satisfy `Sync` in Rust do not satisfy `Sync` in Zinc.
Take the basic type `Int` as an example: in Rust it satisfies `Sync`; but in Zinc, `Int` cannot satisfy `Sync`.
By contradiction: if we stipulated that `Int` satisfies `Sync`, that would mean a pointer such as `*Int` satisfies `Send`, so it could be passed across threads, and then we would see two threads both obtain a pointer to the same `Int` without thread synchronization, which is wrong.

* Zinc's `Sync` represents a type that has an extra internal thread-synchronization mechanism. Far fewer types satisfy the `Sync` constraint than in Rust. 
* Zinc's `Send` represents a type that can be passed across threads safely. It mainly includes four cases:

    1. Classic value-semantic types that contain no pointers internally; when passed across threads they copy a replica, so they are definitely safe
    2. The type contains a pointer, and the data the pointer points to is read-only. Shared data is read-only, so it is safe.
       For example the `Str` type: its public API provides no modification capability, which guarantees that `Str` satisfies `Send`.
    3. The type contains a pointer, and the data the pointer points to satisfies copy-on-write. If a thread tries to modify the shared data, it copies the shared data and then modifies the copy; as long as the data is in a shared state it will definitely not be modified; what is modified is always a replica copied for itself, so it is safe.
       For example the `Vec<Int>` type: every member function with modification capability checks the reference count, and if it is not exclusive, first copies the data and then modifies it
    4. The type contains a pointer, and the data the pointer points to can be modified under the premise of thread synchronization. After it is passed across threads, multiple threads hold pointers to the same data, but all reads and writes of the shared data have thread synchronization, so it is safe
       For example the `*AtomicInt` and `*Mutex<String>` types


Although some details of Zinc's and Rust's `Send` and `Sync` designs differ, both succeed in guaranteeing thread safety.


<div style="border: 1px solid black; padding: 10px; background: #F0F0F0">

From the design above you can see that types such as `Vec<Int>` satisfy the `Send` constraint. We can safely pass such a type across threads.

1. If we pass a `Vec<Int>` to different threads, and each thread only reads it, then only a shallow copy occurs, and execution efficiency is high.
2. If we pass a `Vec<Int>` to different threads, and some thread modifies it, then a deep copy is triggered when that thread modifies it; there is a larger performance cost at that point, but there is no thread-safety problem.
3. If we want to pass a `Vec<Int>` across threads, allow it to be modified, and not incur a deep copy, then the programmer must guarantee that multiple shallow-copy instances do not exist at the same time. This can be achieved with a move expression.
Transferring ownership of a `Vec<Int>` across different threads can achieve efficient cross-thread modification. But the compiler does not track ownership at the type level, only at the control-flow level, reducing the impact on the user.
4. If we want a reference-semantic `Vec<Int>`, we only need to box it; the resulting `*Vec<Int>` is a reference-semantic type. Different pointers can modify the same `Vec<Int>` and share the same data.
Because `Vec<Int>` does not satisfy `Sync`, `*Vec<Int>` does not satisfy `Send`. The compiler can then help us check that all of these pointers are inside the same thread; a non-`Send` type cannot cross a thread boundary.
So a reference-semantic `*Vec<Int>` type also has no thread-safety problem. And this Arc pointer's reference-count operations can be optimized to non-atomic increment/decrement.

From the above you can see that the copy-on-write optimization is very important for a type such as `Vec<T>`.

Suppose Zinc had not chosen ARC as its foundational memory-management mechanism, but had chosen GC instead, and let a `box` expression return a GC-traceable pointer.
Then a container type such as `Vec<T>` could only use GC pointers in its internal implementation, and could not be a "value-semantic" type. It could only be non-Send. And move expressions would also lose their meaning.
In that case, in multithreaded scenarios, very few types would satisfy the `Send` constraint; to guarantee thread safety, more scenarios would need a deep copy, bringing a huge extra performance cost instead.
Therefore ARC memory management is the best fit for Zinc's thread-safety goal.

And precisely because Zinc also achieves thread safety, we can guarantee at compile time that certain ARC pointers definitely cannot cross a thread boundary; they themselves and their copies can only be used inside the same thread.
Using that information, the compiler can optimize their reference-count increment/decrement operations to non-atomic operations, further improving performance. This optimization depends only on the type, not on control flow.

Therefore thread safety and ARC memory management complement each other perfectly.

</div>

## Threads

The thread-related APIs provided by the Zinc standard library are a simple wrapper around the operating system's thread functionality. Creating a thread uses the following function:

```
// mod std::thread
pub fn spawn<F, T>(f: F) -> JoinHandle<T>
where
    F: Send + 'static + Fn()->T,
    T: Send + 'static,
```

`JoinHandle` can call `join()` to wait for the thread to finish. It can also call `thread()` to obtain a variable of type `&Thread`.

Example:

```Rust
use std::thread::spawn;
fn main() {
    let h = spawn.{
        println("thread 2");
    };
    println("thread 1");
    h.join().get_or_panic();
}
```

### ThreadLocal

The `ThreadLocal` type expresses a "thread-local" variable. Each thread independently maintains its own copy.

Usage:

```
use std::thread::spawn;

fn main() {
    static G: ThreadLocal<Int64> = ThreadLocal::new(0);

    let h = spawn.{
        *G += 1;
    };
    h.join().get_or_panic();
    println(*G);
}
```

## Locks

Locks include `Mutex<T>` and `RwLock<T>`.

Rust's lock API design has a nice advantage: it merges the lock and the protected data into one type, thereby avoiding a mismatch between the lock and the protected data, and making it impossible to access the data while forgetting to lock.

But it has a few small problems:
1. The poison design
2. It is possible in safe code to leak a MutexGuard, resulting in a situation where the lock cannot be unlocked
3. In async code, if an await happens while holding the lock, it is easy to trigger a deadlock

Zinc makes a small modification to this API:
1. Poison-state management is removed, because Zinc has no catch-unwind mechanism
2. Locking and unlocking are changed to a callback style

```
fn synchronized(p: &Mutex<String>) {
    // later we can learn from Swift and add more syntactic sugar to omit this lambda's parameter list as well
    p.lock.{ (data: &mut String) =>  
        data.push_str("tail");
    };
}
```

The benefits of this are:
1. The locked region is clear, enclosed in braces, fairly conspicuous, and suitable for human reading
2. There is no MutexGuard, so there is no leak risk; a lock action necessarily corresponds to an unlock action, avoiding the risk of a lock that cannot be released
3. When locking in multiple layers, lock and unlock are guaranteed to satisfy last-in-first-out. Because Rust exposes MutexGuard and allows the user to manually drop a MutexGuard, the actual unlock order may be unstructured
4. An await expression cannot be used in the callback, because the callback function's API type does not match, which also avoids awaiting while holding the lock

Of course it brings a drawback: you can no longer naturally use control-flow statements such as `break` `continue` `return` while holding the lock and have the compiler unlock automatically. Because control-flow statements inside the callback only affect the lambda; if you need to handle outer control flow, you need to use different return values from this lambda and do extra checks outside the lambda. That said, at least from a readability perspective this is very clear. Not being able to use control-flow statements directly is not necessarily a bad thing.

### Anti-copy design

Rust's lock types are designed with move semantics, but Zinc has no move-semantic types. That introduces a new risk: the user may inadvertently copy a `Mutex<T>` variable, which can easily produce a bug.

Example:
```
// pseudocode example; this code cannot actually compile
fn test(s: &Mutex<String>) {
    let str_copy = *s; // in some cases this copy may be hidden, which can cause a bug
    str_copy.lock.{ (s: &mut String) => println(s) };
}
```

In Zinc, to avoid this situation, the `UnSized` trait is introduced. This trait is a marker trait with no member functions. At the same time it is stipulated that:
1. When defining a type, you may `impl UnSized for MyType {}` on a custom type
2. All `UnSized` types cannot be used directly as global variables, constants, local variables, or fields. Such a type can only be accessed indirectly through a pointer.
3. Dereferencing a pointer to `UnSized` is not allowed.

If we stipulate in the standard library that a type such as `Mutex` is `UnSized`, we can avoid the bug above.

```
struct S {
    value: Mutex<MyType>, // Error: Mutex cannot be used directly as a field; use *Mutex<MyType> instead
}

fn f() { 
    let local = Mutex::new(MyType::new()); // Error: Mutex cannot be used directly as a local variable; use *Mutex<MyType> instead
}

static GLOBAL_DATA: Mutex<MyType> = Mutex::new(MyType::new()); // Error: Mutex cannot be used directly as a global variable; use &'static Mutex<MyType> instead

static GLOBAL_DATA: &'static Mutex<MyType> = &Mutex::new(MyType::new()); // OK
```

Standard-library functions such as `swap` also cannot operate on `UnSized` types. Therefore this design can avoid the above error of "accidentally copying an entire Mutex."

Types such as `File` in the standard library have a similar design. Users can only ever use a "pointer type pointing at File," and cannot use the `File` type directly as a value type.

> **TODO**: The UnSized type design still needs to be completed. UnSized types need to be allowed as function parameters and function return types. Some generic functions should also allow UnSized generic arguments.
> But all UnSized types must be restricted to use as temporaries; before being bound to a sized-type variable, they can only be moved, not copied.

```
impl<T> UnSized for Mutex<T> {}
impl<T> Mutex<T> where T: ?Sized { // need to allow T to possibly be unsized
    fn new(v: T) -> Mutex<T> { // need to allow an unsized type as a function parameter and return
        return {
            .val = move v, // an unsized parameter can only be moved
            ...
        };
    }
}
```

## Asynchronous I/O

> **TODO**: Asynchronous I/O is not implemented yet.

Basic idea: ease of use over performance. Firmly reject Rust's Move/Pin design, and align ease of use with C#. In practice, the complexity introduced by Move/Pin far exceeds the benefit; it is not worth it.
Aside from Arc cyclic-reference issues, Zinc's ease of use should be comparable to C#.

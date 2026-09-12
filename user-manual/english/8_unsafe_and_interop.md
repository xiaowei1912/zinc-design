# Unsafe and Interop

## The unsafe keyword

Zinc uses the `unsafe` keyword to mark code that may produce undefined behavior. Using `unsafe` tells the compiler: "I know this code may be unsafe, but I guarantee it is correct."

### When unsafe is required

The following operations must be performed in an unsafe context:

1. **Function calls and method calls**
   - Functions marked `extern`
   - Functions marked `unsafe`

2. **Raw-pointer dereference**
   - Dereferencing a pointer of type `*raw T` or `*raw mut T`

3. **Type casts**
   - Converting a raw pointer to any type
   - Conversions between integers and pointers

4. **Union field access**
   - Reading and writing fields of a union type

5. **Accessing mut static**
   - Reading and writing global variables defined with `static mut`

### Unsafe blocks

An `unsafe` block creates an unsafe context:

```rust
fn main() {
    let x: Int = 42;
    let p: *raw Int = &x as *raw Int;
    
    unsafe {
        let value = *p; // dereferencing a raw pointer requires unsafe
        println(f"value = $(value)");
    }
}
```

### Unsafe functions

A function defined with `unsafe fn` must be called in an unsafe context:

```rust
unsafe fn dangerous_operation() {
    // code that may produce undefined behavior
}

fn main() {
    unsafe {
        dangerous_operation(); // call an unsafe function
    }
}
```

### unsafe impl

`unsafe impl` can be used to manually implement certain marker traits, such as `Send` and `Sync`:

```rust
struct MyType {
    ptr: *raw mut Int,
}

// manually declare that MyType is Send
// the programmer must guarantee that this declaration is correct
unsafe impl Send for MyType {}
```

## FFI (foreign function interface)

### extern functions

The `extern` keyword can declare an external function, typically used to call a C library:

```rust
// declare the printf function from the C standard library
extern fn printf(fmt: *const UInt8, ...) -> Int32;

fn main() {
    unsafe {
        printf("Hello from C!\n" as *const UInt8);
    }
}
```

### extern blocks

An `extern` block can declare external functions in bulk:

```rust
extern "C" {
    fn malloc(size: UInt) -> *raw mut UInt8;
    fn free(ptr: *raw mut UInt8);
    fn strlen(s: *const UInt8) -> UInt;
}

fn main() {
    unsafe {
        let ptr = malloc(100);
        // use ptr...
        free(ptr);
    }
}
```

### Interop with C

Zinc provides several ways to interoperate with C:

1. **Calling C functions**: declare C functions with `extern`, then call them in an unsafe block
2. **Passing data to C**: pass data using raw pointer types
3. **Receiving data from C**: receive data returned by C using raw pointers

Example:

```rust
// declare C functions
extern fn fopen(filename: *const UInt8, mode: *const UInt8) -> *raw mut UInt8;
extern fn fclose(file: *raw mut UInt8) -> Int32;

fn main() {
    unsafe {
        let file = fopen("test.txt\0" as *const UInt8, "r\0" as *const UInt8);
        if file != null {
            // use the file...
            fclose(file);
        }
    }
}
```

## The boundary between safe and unsafe

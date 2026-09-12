# unsafe 和互操作

## unsafe 关键字

Zinc 语言通过 `unsafe` 关键字来标记可能产生未定义行为的代码。使用 `unsafe` 可以告诉编译器："我知道这段代码可能不安全，但我保证它是正确的。"

### unsafe 的使用场景

以下操作必须在 unsafe 上下文中进行：

1. **函数调用、方法调用**
   - 被 `extern` 修饰的函数
   - 被 `unsafe` 修饰的函数

2. **裸指针解引用**
   - 解引用 `*raw T` 或 `*raw mut T` 类型的指针

3. **类型转型**
   - 裸指针转换为任何类型
   - 整数与指针之间的转换

4. **union 成员访问**
   - 读写 union 类型的成员变量

5. **访问 mut static**
   - 读写用 `static mut` 定义的全局变量

### unsafe 块

使用 `unsafe` 块可以创建一个 unsafe 上下文：

```rust
fn main() {
    let x: Int = 42;
    let p: *raw Int = &x as *raw Int;
    
    unsafe {
        let value = *p; // 解引用裸指针，需要 unsafe
        println(f"value = $(value)");
    }
}
```

### unsafe 函数

使用 `unsafe fn` 定义的函数，调用时必须在 unsafe 上下文中：

```rust
unsafe fn dangerous_operation() {
    // 可能产生未定义行为的代码
}

fn main() {
    unsafe {
        dangerous_operation(); // 调用 unsafe 函数
    }
}
```

### unsafe impl

使用 `unsafe impl` 可以手动实现某些 marker trait，如 `Send` 和 `Sync`：

```rust
struct MyType {
    ptr: *raw mut Int,
}

// 手动声明 MyType 是 Send 的
// 程序员需要保证这个声明是正确的
unsafe impl Send for MyType {}
```

## FFI（外部函数接口）

### extern 函数

使用 `extern` 关键字可以声明外部函数，通常用于调用 C 语言库：

```rust
// 声明 C 标准库中的 printf 函数
extern fn printf(fmt: *const UInt8, ...) -> Int32;

fn main() {
    unsafe {
        printf("Hello from C!\n" as *const UInt8);
    }
}
```

### extern 块

使用 `extern` 块可以批量声明外部函数：

```rust
extern "C" {
    fn malloc(size: UInt) -> *raw mut UInt8;
    fn free(ptr: *raw mut UInt8);
    fn strlen(s: *const UInt8) -> UInt;
}

fn main() {
    unsafe {
        let ptr = malloc(100);
        // 使用 ptr...
        free(ptr);
    }
}
```

### 与 C 语言互操作

Zinc 提供了多种方式与 C 语言互操作：

1. **调用 C 函数**：使用 `extern` 声明 C 函数，然后在 unsafe 块中调用
2. **传递数据给 C**：使用裸指针类型传递数据
3. **从 C 接收数据**：使用裸指针接收 C 返回的数据

示例：

```rust
// 声明 C 函数
extern fn fopen(filename: *const UInt8, mode: *const UInt8) -> *raw mut UInt8;
extern fn fclose(file: *raw mut UInt8) -> Int32;

fn main() {
    unsafe {
        let file = fopen("test.txt\0" as *const UInt8, "r\0" as *const UInt8);
        if file != null {
            // 使用文件...
            fclose(file);
        }
    }
}
```

## 安全与不安全的边界

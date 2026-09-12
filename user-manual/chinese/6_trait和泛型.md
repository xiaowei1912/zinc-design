# 泛型 和 trait

## 泛型

类型定义 和 函数定义，都可以声明泛型参数。

### 泛型类型

用户自定义类型，可以携带泛型参数。
泛型参数包含两种：生命周期泛型参数、类型泛型参数。在泛型参数列表中，排列顺序为：生命周期参数在类型参数前面。

todo: 后续需要考虑是否应该加入常量泛型特性，需要注意 swift 不支持该特性的理由

```rust
struct S<'a, T> {
    p: &'a T
}

enum E<T> {
    A(T), B(Int)
}
```

类型定义支持可选的 `where` 子句。 `where` 子句可以表达：生命周期之间的存活关系，以及类型是否满足 trait 约束的关系。

泛型类型的使用，需要提供泛型实参。

```rust
fn main() {
    let v = 1;
    let x: S<Int> = S { .p = &v };
}
```

### 泛型函数

函数也支持泛型。与泛型类型一样，泛型参数也支持生命周期泛型参数、类型泛型参数。
泛型函数也支持可选的 `where` 子句。

```
fn swap<'a, T>(lhs: &'a mut T, rhs: &'a mut T) {}
```

注意：zinc 中泛型函数调用语法有变化，避免 rust 里面的 turbo fish 语法。

<details>

<summary>什么是 turbo fish 语法</summary>

<div style="border: 1px solid black; padding: 10px;">

Rust 的语法设计中，如果泛型函数调用，直接使用跟泛型函数定义一样的语法，就会有语法歧义，示例如下：
```rust
fn main() {
    let (the, guardian, stands, resolute) = ("the", "Turbofish", "remains", "undefeated");
    let _: (Bool, Bool) = (the<guardian, stands>(resolute)); // 这一行是什么意思？
}
```

这个歧义的核心是小于号运算符和泛型尖括号复用了同样的符号，导致上面的这一行有两种理解方式：
1. 这是小括号里面的一个泛型函数调用，函数名是 `the`，泛型参数是 `guardian` `stands`，函数参数是 `resolute`
2. 这是一个 tuple，有两个元素。第一个元素是 `the<guardian` 组成的比较运算；第二个元素是 `stands>(resolute)` 组成的比较运算。

为了解决这个语法冲突问题，Rust 要求调用泛型函数的时候，要在函数名后面跟上 `::`，因此正确的调用语法是：
```
the::<guardian, stands>(resolute)
```

这就是 turbo fish 语法。比如 `Vec::<i32>::with_capacity(16)`。

</div>
</details>

<br/>

Rust 的 turbo fish 语法消除了语法解析的歧义问题，但它带来了语法的不一致性，不够美观。
Zinc 采用了另外一种设计，既避免了语法歧义问题，也保持了语法的一致性。

1. fully-qualified-call-syntax 中如果出现泛型类型，一律需要改用 `type` 关键字开头。示例如下：
    * `(type Vec<Int>)::with_capacity(4)`  
      因为 `Vec` 显式携带了泛型实参，所以 `Vec<Int>::with_capacity(4)` 是语法错误。
    * `Vec::with_capacity(4)`  
      允许不指定 `Vec` 的泛型参数，泛型参数可以由上下文推导，此时可以不写 type 前缀。
    * `(type Vec<String> as Default)::default()`  
      允许明确指定类型名以及对应的 trait 名，成为完全限定语法调用特定函数。这意味着不同 trait 中的成员函数可以重名，且可以针对同一个类型 impl，不会发生冲突。

2. 结构体初始化以及模式匹配，一律采用类型后置语法。示例如下：
    * 结构体初始化: `let v = { .x = 1 } : S<Int>;`
    * 结构体模式匹配: `expr is { .x = 1 } : S<Int>`

3. 泛型函数调用，泛型参数列表需要写到小括号内部。示例如下：
    * `std::mem::size_of(<Int>)`

上述修改的核心逻辑是，让所有带泛型的类型名字，一律出现在以特定标识作为前缀的上下文中。
这样编译器在做语法解析的时候，碰到对应的符号或者关键字，就知道后面出现的一定是类型而不是表达式，在这些地方出现的 `<` `>` 符号应该理解为尖括号而不是小于号和大于号。

下面着重讲一下泛型函数调用语法：
```
// 函数定义的语法没变
fn f<T>(arg: T) {}

// 函数调用的语法有变化
fn main() {
    f(<Int>, 1); // 显式指定泛型实参类型
    f(1); // 类型推导方式，省略泛型实参的写法
}
```

这么设计语法的主要原因，一是为了避免 turbo fish 这种丑陋的语法，二是为了跟 zinc 的泛型函数的实现方式相匹配。

不考虑特殊场景的编译器优化，zinc 的泛型函数从实现角度来说，默认是通过表传递的方式实现的。编译器会把上面这个函数 f 翻译为：
```
// 函数定义 f 对应的伪 C 代码:
void f(ZnTypeMeta * typeof_T, void * arg) { }
```

在执行 `f(<Int>, 1)` 这样的函数调用的时候，调用方是真的会拿到 `Int` 对应的 ZnTypeMeta 将它作为一个函数实参，传递到这个 f 里面去。
所以把函数的泛型实参列表，写到小括号里面，更符合 zinc 的泛型函数的执行语义。

zinc 的函数指针类型，是可以包含泛型的 `let pf: fn<T>(T)->() = f;`。
zinc 不允许函数科里化(currying)，这么写 `let pf: fn(Int)->() = f(<Int>);` 会编译失败。
这种函数实例化场景请使用 lambda，这么写是可以的： `let pf: fn(Int)->() = .{ (arg: Int)->() => f(<Int>, arg); };` 。

泛型函数这么实现的主要好处是：泛型函数也可以通过动态链接库分发。这是 Swift 语言的一个设计优点。

此事对于这种场景很有用：
假设我们设计了一个动态链接库 `a.so`，它对外暴露了泛型函数接口，被应用程序 `b.exe` 使用。
那么我们可以做到，在更新 `a.so` 的内部实现的时候（当然不能做API破坏性修改），不重新编译 `b.exe`，程序也能自动正常工作。
如果泛型是用实例化的方式实现的，那么这个场景就很难支持。更新上游库，下游应用程序必须重新编译重新部署。

除此之外，这个设计还附带了一些其它的好处：
1. 支持 `virtual` 函数带泛型。关于 `virtual` 函数可参考下述章节。
2. 减少 code size，减少编译时间。

付出的代价，当然是执行效率降低。这么做会阻止很多编译器的优化，对缓存也不友好，对执行时间不利。
但我们依然可以在实现层面，在某些特定场景下，通过额外编译选项，允许使用泛型实例化作为性能优化手段，在 code size 和执行效率之间取得更好的平衡。
1. 在编译可执行程序，而不是库的时候，完全可以使用实例化的方式实现泛型
2. 即使在编译库的时候，只要没有把动态链接库单独分发部署的需要，就可以使用实例化的方式实现泛型

> **TODO**：目前不允许 UnSized 类型作为泛型实参使用，trait 也不能作为泛型实参使用。

> **计划中的功能**：后续可以支持"或"条件作为 where 子句的条件，作为函数参数类型重载的语法糖：
```
fn f<T>(arg: T) where T: Int32 | UInt32 {}
// 等价于：
trait AnonymousTR {}
impl AnonymousTR for Int32 {}
impl AnonymousTR for UInt32 {}
fn f<T>(arg: T) where T: AnonymousTR {}
```

## trait

trait 定义的语法示例如下：

```
trait TrName<T> : SuperTrait1 + SuperTrait2 where T: Condition {
    type AssocType;

    virtual fn method(&self);

    fn static_method() -> Self;
}
```

trait 定义支持：
1. 可选的泛型参数
2. 可选的父类型 trait，支持多个父 trait，用 `+` 分隔
3. 可选的 where 条件子句

trait 定义体内可以包括：
1. 关联类型
2. 关联常量
3. 关联函数

### impl trait

trait 可以用于对不同类型做统一抽象。指定某个具体类型 `SomeType` 实现了某个 trait `TrName` 的语法示例如下：

```
impl TrName for SomeType {

}
```

impl 块
1. 内部的函数不能单独指定 pub
2. 也不能指定 virtual

不允许 impl Trait for Trait。

**孤儿规则**

基本原则为：

### 指向 trait 的指针

trait 的两种使用场景是：
1. 作为泛型约束的上限使用
2. 作为指向 trait 的指针类型使用

全局变量、局部变量、成员变量，都不允许直接使用 trait 作为类型。但可以使用指向 trait 的指针作为类型。

如果类型 `S` 实现了 `trait R`，那么指向 `S` 的指针，可以向上转型为指向 `R` 的指针。示例如下：

```rust
struct S {}
trait R {
    virtual fn f(&self);
}
impl R for S {
    fn f(&self) {}
}

fn test(p: *S) {
    let p1: *R = p; // ok, 向上转型
    p1.f();
}
```

### virtual 函数

指向 trait 的指针，可以调用 trait 的成员函数。但是有个限制，只有用 virtual 标记的成员函数，才可以通过指向 trait 的指针调用。

> 注：
> Rust 允许使用 `where Self: Sized` 语法对 trait 中的成员函数做标记，表明它是 "non-virtual" 的。这个语法非常怪异，很难理解。zinc 直接引入关键字来做标记更易读。
>

示例如下：

```rust
trait TR {
    virtual fn f1(&self);
    fn f2(&self);
}

fn test(p: &TR) {
    p.f1(); // OK
    p.f2(); // 编译错误，f2 不是 virtual 函数，不允许通过指向 trait 的指针调用
}

fn test2<T>(arg: T) where T: TR {
    arg.f2(); // OK. 非 virtual 函数依然可以通过泛型的形式调用。
}
```

同时，注意一下，不是所有写在 trait 中的函数，都能用 virtual 修饰的。能用 virtual 修饰的函数，需要满足以下要求：
1. 第一个参数名，必须是 `self`，且类型是指向 `Self` 的指针类型。包括 `&self` `&mut self` `*self` `*weak self` 都可以。
2. 除了第一个参数之外，其它参数不允许使用 `Self` 类型，以及关联类型
3. 函数的返回类型，不允许使用 `Self` 类型，以及关联类型

示例如下：
```
trait TR {
    virtual fn clone()->Self; // 编译错误，不允许用 virtual 修饰。没有 self 参数，返回类型用了 Self。
    virtual fn g(arg: Int); // 编译错误，不允许用 virtual 修饰。没有 self 参数。
    virtual fn h(self);  // 编译错误，不允许用 virtual 修饰。self 参数不是指针类型。
}
```

**注意**： virtual 函数支持引入新的泛型参数。示例如下：
```
trait TR {
    virtual fn f<T>(&self, arg: T) where T: Hash + Ord + Default; // OK
}
```

> **计划中的功能**：后续可以支持 `* Trait1 + Trait2 + 'a` 写法：
1. 用 `+` 的时候，不能是具体类型，只能是 trait 和 trait 相加
2. 只能有一个 non-marker trait 类型。自己加自己也不行。
3. `* Trait1 + Trait2` 可以隐式转换为 `*Trait1` 或者 `*Trait2`

该特性可能在某些场景下是一个必须的功能，比如，我们想要 `* MyTrait + Sync` 来表达指向的类型不仅满足 MyTrait 约束，也满足 Sync 约束。
用其它办法就表达不出来。

### 关联类型

关联类型是 trait 中定义的类型占位符，在实现 trait 时需要指定具体类型。

示例：
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

`Self` 类型代表实现 trait 的具体类型。

> **TODO**：目前不支持高阶关联类型（Higher-Kinded Types）。

### trait继承

zinc 中的 trait 支持继承。允许在定义 trait 的时候指定零个或者多个父 trait。

具有继承关系的 trait，指向子类型 trait 的指针，可以隐式转换为指向父类型 trait 的指针（ARC / BORROW 指针都可以）。示例如下：
```
trait TR: TR1 + TR2 + TR3 {}

fn test(p: *TR) {
    let p1: *TR1 = p; // OK
    let p2: *TR2 = p; // OK
    let p3: *TR3 = p; // OK
}
```

指向父类型 trait 的指针，也可以通过模式匹配，向下转型为子类型 trait 的指针。比如 `is` `match` 都可以做。

```
trait TR: TR1 + TR2 + TR3 {}

fn test(p: *TR1) {
    if (p is p1: *TR) { // 尝试向下转型，可能失败
        // 使用 p1 变量
    }

    match (p) {
        p2: *TR => {}
        _ => {}
    }
}
```

其它功能包括：
1. 在子 trait 中允许指定 associated type / const 的值
2. 在子 trait 中允许 override 父trait的函数体
3. 支持多继承，需要考虑多继承场景的函数名冲突问题
    1. 多继承场景，禁止名字冲突
    2. 多继承场景，新增父类型、调整继承顺序，都会影响 ABI
    3. `trait TR1 : TR2 {}`  和 `trait TR1 where Self: TR2 {}` 含义不同。

### reuse

reuse 机制的目的是复用成员变量、成员函数。reuse 是一种语法糖，方便用户可以把一个实现，代理到另外一个类型上去。示例如下：

```
struct S1 { }
impl TR1 for S1 {
    fn f1(&self, arg: Int) {}
}

struct S2 {
    base: S1  // 成员变量的名字没有限制，随便是什么都行
}
impl TR1 for S2 {
    reuse self.base;  // 含义是这个 impl 块里的成员函数都复用 self.base 实现的同一个 trait 的成员函数
}

// 以上代码等价于：
impl TR1 for S2 {
    // 把所有成员函数都实现一遍，且函数体都是调用对应的 `self.base` 的成员函数。
    fn f1(&self, arg: Int) {
        self.base.f1(arg);
    }
}

```

### 常用内置 trait

`Any` trait

`Fn` trait

`Send` trait

`Sync` trait

### 面向对象编程范式

通过上述语言特性的组合，zinc也可以支持面向对象编程范式，但与业界常见的风格(如 C++/Java/C#/Swift)有所不同。
Zinc 的主要特点是：不支持 `class` 和 class继承。Zinc 只有 trait 可以继承，只有指向 trait 的指针，可以做动态派遣。

下面从几个方面讲解一下，为什么要这样设计。

1. 引用语义的 `class` 类型，内存布局不够灵活

    从内存布局的角度，可以把其它语言的引用语义的 class 视为 Zinc 里面的 `ARC指针+结构体` 合起来。只允许使用引用语义，不允许使用它对应的这个值语义的结构体。
    这意味着，当 class 类型的变量作为局部变量、成员变量使用的时候，都有动态内存分配，都有一层额外的指针间接引用。这实际上是表达能力的退化，而不是增强。
    而如果我们把 `ARC指针` 和 `结构体` 分别提供给用户使用，比如 `Vec<S>` 和 `Vec<*S>` 是不同的类型，能在不同场景按需选用，这样更灵活，用户有更大的自由度。

    引用语义的class，在任何场景下，每个对象内部都自带一个“对象头”指针。而zinc的这种设计，不会影响对象本身的内存布局。

2. 增加 `class` 这种引用语义的类型，将会让 Zinc 的类型系统出现不一致，很多场景需要特殊规则，实现复杂

    引用语义的 class 会给 Zinc 的类型系统带来一系列的额外规则，不划算。特别是各种指针和 class 配合使用的时候。
    在 zinc 中，已经有多种指针类型存在，而 “引用语义的class” 本身可以理解为 “隐式的指针+结构体” 的组合。
    此功能与其它语言特性不是正交的，因为这个 “隐式的指针” 究竟应该是 ARC 还是 weak 或者是借用，用户缺少选择权。

    我们考虑这个场景：当 `T` 是 class 的时候，`&T` 代表的是什么含义。这个借用是指向的这个指针变量本身，还是指向的是这个对象的首地址？
    因为 class 是把 `ARC指针` 和 `结构体` 强制绑定在一起，所以选择哪种设计方案都会带来一些麻烦。
    把指针和结构体分开，语义表达更清晰。类型 `&S` 就是对 `S` 的借用；类型 `&*S` 就是对 `*S` 的借用。语义清晰，实现也简单。

    考虑下面这个泛型场景的例子，大家会发现想要在泛型中把值语义的 struct 和引用语义的 class 统一起来很麻烦：
    ```
    fn test<T>(arg: T) {
        // 如果我们引入引用语义的 class，那么取借用表达式和解引用表达式，针对值语义类型和引用语义类型的行为不同。
        // 考虑在泛型场景下，我们就需要在运行时做一些额外的判断才能实现，用泛型对这两种情况统一抽象。
        let p: &T = &arg;
        let c: T = *p;
    }
    ```

3. 如果我们像C++一样，引入class以及继承，但是 class 不是引用语义，行不行？

    C++ 里面的 class 与 struct 语义差别不大。支持值类型，没有与指针绑定。
    但是class的继承，意味着子类型必须接受父类型实现的所有接口，这是有问题的。

    传统的继承的设计，子类型class必然实现了父类型class的所有接口(interface/protocol)。

    考虑 Zinc 中的 `Send` 这样的 trait，它要求所有成员变量都满足 `Send`。假设我们引入继承，就会出现这样的情况：父类型实现了 `Send`，
    子类型继承了父类型，但增加了一个 non-`Send` 的成员变量，我们希望子类型复用父类型的所有成员变量、成员方法，但不实现 `Send`，在传统继承的设计下这就做不到。

    在使用继承的设计中，子类型无法从父类型中只挑选一部分接口实现。要达到上述目的，也必须放弃继承，改用组合实现。所以，继承只能在部分场景方便代码复用，不能适应所有场景。

4. class的继承，需要针对特殊场景设计特殊规则

    1. 一个典型场景是，多继承场景。
    按传统的支持继承的设计方案，如果我们支持一个class继承多个class，那么必须考虑“菱形继承”的场景。
    C++针对这个场景设计了特殊的语法语义规则，同时很多编程规范也对此特性做了很多约束。
    Java/C#/Swift等语言，直接不支持多继承，限制了代码复用能力，简化了语义规则。

    2. 另外一个典型场景是，构造函数场景。不同语言的规则都不一样。C++里面构造函数中调用虚函数，是没有虚函数效果的。
    Java/C# 的构造函数不会访问到未初始化变量，依赖于字段的 “0值初始化”。
    Swift 无法支持 “0值初始化”，又希望构造函数中永远不会访问到未初始化成员，所以它的规则最复杂。
    构造函数限制了函数名，这意味着支持构造函数就必须支持基于类型的函数重载；构造函数限制了返回类型，这意味着不能通过返回类型来做错误处理。
    这些设计理念都跟 zinc 不符。

    总之，传统编程语言支持了class的继承，但class继承这个特性附带了一些额外的特殊语义规则，不同的语言规则都不一样，不是很美观。

Zinc 跟随了 Rust 的设计思路，采用的是更简洁的一种设计：只允许 trait 继承。

Zinc 的设计对用户更容易理解和使用：
* 所有的成员变量的复用，一律采用 “组合” 的模式；
* 所有的成员方法的复用，一律采用 “reuse机制” ；
* 所有需要对多个类型统一公共接口的场景，一律采用 “trait+impl” 完成；
* 所有需要动态派遣的场景，一律采用 “指向trait的指针” 完成。

以上规则对所有类型均适用（内置类型、enum、struct、tuple、字符串、数组等），没有特例。以上规则足够完整支持OOP编程范式。表达能力没有问题，传统语言支持的功能都可以表达，只是某些情况语法稍微繁琐一点。

语义规则是精简的、一致的、正交的。

### 内存布局

指向 trait 的指针，是胖指针。本节中提到的指针类型，包含 `*T`, `*weak T`, `&T`, `&mut T`, `*raw T`, `*raw mut T` 这六种。

```rust
trait TRBase { virtual fn f1(&self); }
trait TRSub : TRBase { virtual fn f2(&self); }

struct S { i: Int }
impl TRBase for S { fn f1(&self) {} }
impl TRSub for S { fn f2(&self) {} }

fn main() {
    let p1: *S = box S{.i = 1}; // p1 是瘦指针
    let p2: *TRSub = p1; // p2 是胖指针
    let p3: *TRBase = p2; // 向上转型

    if p3 is psub: *TRSub {
        // 向下转型
    }
    if p2 is ps: *S {
        // 向下转型
    }
}
```

Zinc 的胖指针布局与 Rust 不同。指向 trait 的指针包含3个成员：
```
      p2
┌───────────────────┐
│ object_ptr        │→ 指向对象的头部。对象可以是内置类型、用户自定义的struct、enum等，都可以。
│───────────────────│
│ typemeta_ptr      │→ 指向对象的具体类型的 TypeMeta，它包含了类型的 name, size 等信息，也包含析构函数指针、拷贝函数指针等
│───────────────────│
│ vtable_ptr        │→ 指向这个具体类型对应这个trait的虚函数表，就是一个函数指针组成的数组
└───────────────────┘
```

* 指向trait的指针，支持调用 trait 的 virtual 成员函数。调用成员函数时，就是从 vtable 中把对应位置的函数指针取出来再调用，偏移量是编译时确定的。

* 指向trait的指针，支持通过模式匹配向其它类型的指针转型，因为胖指针中保存了对象的运行时类型信息。我们可以在运行时判断，这个trait指针指向的是哪个具体类型，如果匹配成功，则将其包含的 object_ptr 取出来作为具体类型的指针使用即可。也可以支持向下转型：`*TRBase` 可以运行时判断是否是 `*TRSub` 类型。

* 指向trait的指针，支持向上转型。`*TRSub` 类型可以转型为 `*TRBase` 类型。

> 注：
> Rust 里面指向 trait 的指针叫做 trait object，它也是胖指针，但它不支持向下转型。只有 `Any` 这个特殊 trait 可以支持向下转型，其它的 trait 都不支持。
> 因为它只有两个指针大小，有点胖，但又不够胖，不足以支持 Zinc 期望支持的所有面向对象编程范式。
>
> 参考 Swift 的 Protocol 的内存布局，可以想到，只需要再增加一个指针即可。Zinc 为胖指针设计的两个指向元数据的指针可以类比为 swift 的 value witness table 和 protocol witness table。


## 遗留问题：

1. 不允许给指针类型 impl trait 会不会有什么表达能力问题？(准备后续在函数参数中引入特别的语法，相当于是内置到语言内部的 AsRef/AsMut trait)

2. 不允许给任意类型 impl trait 会不会有什么表达能力问题？
    ```
    impl<T> ToString for T where T: Display {} // 这种写法目前不支持。建议用 trait 的继承来代替。
    ```

3. 如何允许 UnSized type 作为泛型实参？目前没法实现 `Mutex<File>` 这种类型，写死的 `MutexFile` 的可组合性太弱了。

4. 要不要支持 impl TR 作为返回类型？

5. 要不要支持 const generics？注意 swift 不支持，此事很可能对 运行时 以及 ABI 有较大挑战。

## 协变 & 逆变

|           |   'a   |   T   |   U   |
|-----------|--------|-------|-------|
| &'a T     |  协变  |  协变  |       |
| &'a mut T |  协变  |  不变  |       |
| *T        |        |  协变  |       |
| Mutable<T>|        |  不变  |       |
| Vec<T>    |        |  协变  |       |
| fn(T)->U  |        |  逆变  |  协变  |
| *raw T    |        |  协变  |       |
| *raw mut T|        |  不变  |       |

后续可以继续支持类型的协变、逆变功能。
`in` `out` 关键字已经预留。

使用场景举例：
1. `Option<*Sub>` 转换为 `Option<*Base>`
2. `Result<*SubR, *SubE>` 转换为 `Result<*BaseR, *BaseE>`
3. `(*Sub, *Sub)` 转换为 `(*Base, *Base)`
4. `fn(*Base)->*Sub` 转换为 `fn(*Sub)->*Base`

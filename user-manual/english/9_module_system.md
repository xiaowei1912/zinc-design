# Module System

## Components

Zinc's compilation unit is a component. It corresponds to a Rust crate.

> In C and C++ programming language terminology, a translation unit (or more casually a compilation unit) is the ultimate input to a C or C++ compiler from which an object file is generated.

A compilation unit is the source files needed for one compiler-process execution. These source files are the smallest input unit for one compilation; they cannot be split further. After further splitting, a compilation cannot be performed.

For a C/C++ compiler, a compilation unit is one .c or .cpp source file, plus all files it `#include`s directly or indirectly.

For the Rust compiler, a compilation unit is a crate. The user cannot compile the several mods inside a crate separately.

For the Zinc compiler, a compilation unit is a component. Each compiler execution takes as input the source of a complete component. The user cannot compile a component's several source files separately and then assemble them into a component; the compiler does not have that capability. A component must be compiled as a whole.

Components may not depend on each other cyclically.

A component contains modules (mods). Modules inside a component may depend on each other cyclically, because all modules in the same component are compiled together in one compiler process.

After compilation, a component produces a `.o` file containing machine code, and a `.zno` file of the component's publicly exported interface. The name zno comes from the substance "zinc oxide."

A `.zno` file can be understood as a C "header file," except that a header file is written by hand, while a `.zno` file is generated automatically by the compiler.

The contents of a `.zno` file include all public function signatures, type definitions, global-variable declarations, impl declarations, and so on. They also include the bodies of all functions that can be inlined across components.

## `export`

If a component is a library, the user must declare the component's name with `export component_name;`.
If a component is an executable, the user must define a `fn main()` function in the root mod as the program entry point.
If a component has neither, the compiler reports an error.

In Rust, a crate's name is specified by the compile option `--crate-name NAME`. I think that is unreasonable; a component's name should be specified by its source.
After all, the component name affects the mangled names of symbols in the binary; if the component name is controlled by a compile option, it is hard to guarantee that compilation results are consistent across different compilation environments.

## `import`

When we want to describe that component a depends on another component b, we write `import b;` in a's source. It is basically equivalent to Rust's `extern crate b;`.

If we use C/C++ as an analogy, it can be understood simply as `#include "b.h"`.

## Introduction to mods

A module (mod) is the basic unit of code organization in Zinc. A module may contain functions, types, constants, and other modules.

### Module definitions

Define a module with the `mod` keyword:

```rust
// define a module in a file
mod my_module {
    fn helper() -> Int {
        return 42;
    }
    
    pub fn public_function() {
        println(f"helper returned $(helper())");
    }
}
```

### Filesystem modules

Modules can also be organized through the filesystem. If a module has no body, the compiler looks for a file or directory of the same name:

```rust
// in main.zn
mod network;

// the compiler will look for:
// 1. a network.zn file
// 2. a network/mod.zn file
```

### Visibility

Items in a module are private by default and can only be accessed inside the module that defines them. The `pub` keyword makes an item visible outside:

```rust
mod my_module {
    pub fn public_function() {
        // can be accessed from outside
    }
    
    fn private_function() {
        // can only be accessed inside my_module
    }
    
    pub struct PublicStruct {
        pub field: Int,      // a public field
        private_field: Int,  // a private field
    }
}
```

### Nested modules

Modules can be nested, forming a tree:

```rust
mod outer {
    pub fn outer_function() {}
    
    mod inner {
        pub fn inner_function() {}
        
        mod deeply_nested {
            pub fn deep_function() {}
        }
    }
}
```

### Module paths

Use the `::` operator to access items in a module:

```rust
fn main() {
    outer::outer_function();
    outer::inner::inner_function();
    outer::inner::deeply_nested::deep_function();
}
```

### Visibility modifiers

Zinc supports the following visibility modifiers:

1. **Default (private)**: can only be accessed inside the module that defines it
2. **`pub`**: visible to all modules
3. **`pub(in path)`**: visible only inside the module at the specified path (> **Planned feature**: to be supported in the future)

```rust
mod my_module {
    pub fn public_api() {}
    
    fn internal_helper() {}
    
    // future support: visible only in the parent module
    // pub(super) fn parent_visible() {}
}
```

### Re-exports

`pub use` can re-export items from other modules:

```rust
mod inner {
    pub fn helper() {}
}

mod outer {
    // re-export inner::helper
    pub use inner::helper;
}

fn main() {
    // can be accessed via outer::helper
    outer::helper();
}
```

### The relationship between modules and components

> **Planned feature**: Consider supporting export path and import path. That is, not only allow `export ident;` `import ident;`, but also allow `export ident1::ident2::ident3;` and `import ident1::ident2::ident3;`.

The main consideration is that if components only have a sibling relationship and no containment relationship, that is not flexible enough.
The logical structure inside a component is a tree. If components have no containment relationship and only a sibling relationship, then when a component becomes too large and compile time too long and needs to be refactored, the refactoring itself necessarily changes the component's logical structure. That is inappropriate.

We can consider allowing a certain subtree inside a component to also be a component, not merely a module, which helps split a large component into different compilation units without affecting the logical structure inside the component.

We need to keep two principles unchanged:
1. A component's logical structure is a tree, not a forest
2. Components may not depend on each other cyclically

For example, in the figure below, component c contains a series of submodules and is a very large component.

<img src="./mod_tree.png" width="50%" align=center />

To speed up compilation, we can split some highly cohesive submodules into independent components, shown in different colors in the figure.
Module e and all of its descendant modules are one component; module g and all of its descendant modules are also one component; module i and its descendants are also one component. Modules of the same color in the figure are compiled together; we only need to guarantee that there are no cyclic dependencies between different components. At the same time, the logical structure of the whole component is kept unchanged: the root module of component g has a name such as `c::d::g`, and the mangled names of all items inside it also keep that naming.


## use statements

A `use` statement brings items from other modules into the current scope, avoiding writing the full path every time.

### Basic usage

```rust
use std::collections::HashMap;

fn main() {
    let map = HashMap::new();
    // HashMap can be used directly; no need to write the full path
}
```

### Path keywords

- `self`: can access the current mod
- `super`: can access the current mod's parent mod
- `::ident`: can access a name in the global scope, i.e. a component name. Legal names include the name defined in export for this component itself, the names of dependents brought in by import, and the std standard library.
- `::self`: can access the current component's top-level mod. Equivalent to accessing this component's top-level names starting with `::ident`. Note: this differs from Rust. In Rust, the top-level mod is accessed via the `crate::` keyword.

### Examples

```rust
mod outer {
    pub fn outer_fn() {}
    
    mod inner {
        pub fn inner_fn() {}
        
        fn example() {
            // use self to access the current module
            self::inner_fn();
            
            // use super to access the parent module
            super::outer_fn();
        }
    }
}

// import with use
use outer::inner::inner_fn;

fn main() {
    inner_fn();
}
```

### Renaming imports

The `as` keyword can rename an imported item:

```rust
use std::collections::HashMap as Map;

fn main() {
    let map = Map::new();
}
```

### Importing multiple items

```rust
use std::collections::{HashMap, HashSet};

fn main() {
    let map = HashMap::new();
    let set = HashSet::new();
}
```

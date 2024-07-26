Declarative macros in Rust, also known as "macro_rules!" macros, allow you to write code that writes other code, which can be very useful for reducing repetition and abstracting over patterns in your codebase. Here's an introduction to declarative macros in Rust:

### Basic Structure

A declarative macro is defined using the `macro_rules!` keyword followed by the macro name and a set of rules. Each rule consists of a pattern and a corresponding expansion.

```rust
macro_rules! my_macro {
    (pattern) => {
        expansion
    };
}
```

### Simple Example

Let's start with a simple example of a macro that prints "Hello, world!".

```rust
macro_rules! hello {
    () => {
        println!("Hello, world!");
    };
}

fn main() {
    hello!();
}
```

### Matching Patterns

Macros can match various patterns and generate code accordingly. Here’s an example that takes an identifier and creates a function that prints that identifier.

```rust
macro_rules! create_function {
    ($func_name:ident) => {
        fn $func_name() {
            println!("You called {:?}", stringify!($func_name));
        }
    };
}

create_function!(foo);
create_function!(bar);

fn main() {
    foo();
    bar();
}
```

```rust
macro_rules! calculate {
    (add $a:expr, $b:expr) => {
        println!("{} + {} = {}", $a, $b, $a + $b);
    };
    (sub $a:expr, $b:expr) => {
        println!("{} - {} = {}", $a, $b, $a - $b);
    };
}

fn main() {
    calculate!(add 3, 5);
    calculate!(sub 10, 4);
}
```

### Repetition

Macros can also handle repetition, which is useful for generating code for multiple items. Here’s an example that generates multiple functions with a similar pattern.

```rust
macro_rules! create_functions {
    ($($func_name:ident),*) => {
        $(
            fn $func_name() {
                println!("You called {:?}", stringify!($func_name));
            }
        )*
    };
}

create_functions!(foo, bar, baz);

fn main() {
    foo();
    bar();
    baz();
}
```

### Using Macros with Expressions

Macros can accept expressions and manipulate them. Here’s an example that defines a macro to create a vector and initialize it with values.

```rust
macro_rules! create_vector {
    ($($x:expr),*) => {
        {
            let mut temp_vec = Vec::new();
            $(
                temp_vec.push($x);
            )*
            temp_vec
        }
    };
}

fn main() {
    let v = create_vector![1, 2, 3, 4, 5];
    println!("{:?}", v);
}
```

### Nested Macros

You can also nest macros. This is useful for complex code generation. Here's a basic example:

```rust
macro_rules! outer_macro {
    ($name:ident) => {
        macro_rules! $name {
            ($val:expr) => {
                println!("The value is: {}", $val);
            };
        }
    };
}

outer_macro!(inner_macro);

fn main() {
    inner_macro!(42);
}
```

### Conditional Compilation

Macros can include conditional compilation based on whether certain features are enabled.

```rust
macro_rules! conditional_macro {
    ($name:ident) => {
        #[cfg(feature = "my_feature")]
        fn $name() {
            println!("Feature enabled!");
        }
        #[cfg(not(feature = "my_feature"))]
        fn $name() {
            println!("Feature not enabled.");
        }
    };
}

conditional_macro!(my_func);

fn main() {
    my_func();
}
```

### Practical Example

A more practical example might be implementing a macro that defines getters and setters for struct fields.

```rust
macro_rules! define_struct {
    ($name:ident, $($field:ident: $type:ty),*) => {
        struct $name {
            $(
                $field: $type,
            )*
        }

        impl $name {
            $(
                fn $field(&self) -> &$type {
                    &self.$field
                }

                fn paste::paste!([<set_ $field>])(&mut self, value: $type) {
                    self.$field = value;
                }
            )*
        }
    };
}

define_struct!(Person, name: String, age: u32);

fn main() {
    let mut p = Person {
        name: "Alice".to_string(),
        age: 30,
    };

    println!("Name: {}", p.name());
    println!("Age: {}", p.age());

    p.set_name("Bob".to_string());
    p.set_age(25);

    println!("Updated Name: {}", p.name());
    println!("Updated Age: {}", p.age());
}
```

In this example, the `define_struct!` macro creates a struct with the given fields and also generates getter and setter methods for each field.

### Types of Macro Variables

Macro variables in Rust's `macro_rules!` system are placeholders that capture parts of the input and allow you to reuse them in the generated code. These variables are defined using specific patterns that match various kinds of syntax elements in Rust. Here's a detailed explanation of the different types of macro variables:



1. **Identifiers (`ident`)**:
   - Matches identifiers like variable names, function names, or type names.
   - Example:
     ```rust
     macro_rules! create_function {
         ($name:ident) => {
             fn $name() {
                 println!("You called {:?}()", stringify!($name));
             }
         };
     }

     create_function!(foo);

     fn main() {
         foo(); // prints "You called foo()"
     }
     ```

2. **Expressions (`expr`)**:
   - Matches any Rust expression.
   - Example:
     ```rust
     macro_rules! print_expr {
         ($e:expr) => {
             println!("{:?}", $e);
         };
     }

     fn main() {
         print_expr!(1 + 2); // prints "3"
     }
     ```

3. **Types (`ty`)**:
   - Matches type names.
   - Example:
     ```rust
     macro_rules! create_struct {
         ($name:ident, $type:ty) => {
             struct $name {
                 value: $type,
             }
         };
     }

     create_struct!(MyStruct, i32);

     fn main() {
         let s = MyStruct { value: 10 };
         println!("{}", s.value); // prints "10"
     }
     ```

4. **Patterns (`pat`)**:
   - Matches patterns used in `match` statements or `let` bindings.
   - Example:
     ```rust
     macro_rules! match_value {
         ($value:expr, $pattern:pat) => {
             match $value {
                 $pattern => println!("Matched!"),
                 _ => println!("Not matched"),
             }
         };
     }

     fn main() {
         match_value!(Some(3), Some(x)); // prints "Matched!"
         match_value!(None, Some(x));    // prints "Not matched"
     }
     ```

5. **Token Trees (`tt`)**:
   - Matches any sequence of tokens. This is the most flexible and can be used for nested patterns.
   - Example:
     ```rust
     macro_rules! log {
         ($msg:tt) => {
             println!("{}", $msg);
         };
     }

     fn main() {
         log!("Hello, world!"); // prints "Hello, world!"
     }
     ```

6. **Block (`block`)**:
   - Matches a block of code, typically wrapped in curly braces `{}`.
   - Example:
     ```rust
     macro_rules! repeat_block {
         ($b:block) => {
             $b
             $b
         };
     }

     fn main() {
         repeat_block!({
             println!("This will print twice!");
         });
     }
     ```

7. **Paths (`path`)**:
   - Matches a path to a module, struct, enum, or other item.
   - Example:
     ```rust
     macro_rules! call_function {
         ($func:path) => {
             $func();
         };
     }

     fn greet() {
         println!("Hello!");
     }

     fn main() {
         call_function!(greet); // prints "Hello!"
     }
     ```

8. **Literal (`lit`)**:
   - Matches literal values like numbers, strings, and characters.
   - Example:
     ```rust
     macro_rules! print_literal {
         ($l:lit) => {
             println!("{}", $l);
         };
     }

     fn main() {
         print_literal!(42); // prints "42"
         print_literal!("Hello, world!"); // prints "Hello, world!"
     }
     ```

### Repetition and Separator

Repetition allows you to match and reuse multiple occurrences of a pattern. The syntax `$(...),*` is used for this purpose, where `*` means zero or more times, and `+` means one or more times. A separator can also be included.

Example with repetition and separator:
```rust
macro_rules! create_struct {
    ($name:ident, $( $field:ident: $type:ty ),* ) => {
        struct $name {
            $( $field: $type ),*
        }
    };
}

create_struct!(Person, name: String, age: u32);

fn main() {
    let p = Person { name: "Alice".to_string(), age: 30 };
    println!("{} is {} years old", p.name, p.age);
}
```

In this example, `$( $field:ident: $type:ty ),*` matches zero or more field definitions, separated by commas. This allows you to define structs with an arbitrary number of fields.

### Advanced Example: Combining Multiple Variable Types

Here’s a more complex example that demonstrates using multiple types of macro variables and repetition:

```rust
macro_rules! define_struct_and_methods {
    ($name:ident, $( $field:ident: $type:ty ),* ) => {
        struct $name {
            $( $field: $type ),*
        }

        impl $name {
            $(
                fn $field(&self) -> &$type {
                    &self.$field
                }

                paste::paste! {
                    fn [<set_ $field>](&mut self, value: $type) {
                        self.$field = value;
                    }
                }
            )*
        }
    };
}

define_struct_and_methods!(Person, name: String, age: u32);

fn main() {
    let mut p = Person { name: "Alice".to_string(), age: 30 };
    
    println!("Name: {}", p.name());
    println!("Age: {}", p.age());
    
    p.set_name("Bob".to_string());
    p.set_age(25);
    
    println!("Updated Name: {}", p.name());
    println!("Updated Age: {}", p.age());
}
```

In this example, we define a struct and its getter and setter methods using the macro. The macro uses `ident`, `ty`, and repetition to generate the struct and methods dynamically.

By understanding and using macro variables effectively, you can write powerful and flexible macros in Rust that reduce boilerplate and enhance code expressiveness.

### More to know
There are a few additional aspects of declarative macros in Rust that are worth understanding to make the most of them:

### 1. **Macro Hygiene**

Macro hygiene refers to the concept that macros avoid unintentional name conflicts by isolating the scope of identifiers. Rust's macros use a hygiene system to ensure that identifiers generated within a macro don’t clash with those in the calling code.

#### Key Points:
- **Scope Isolation**: Macros create a new scope for identifiers, preventing conflicts with names in the surrounding code.
- **Debugging Hygiene Issues**: If you encounter unexpected behavior, it might be due to hygiene. To troubleshoot, consider using `#[macro_use]` to import macros properly and verify the macro's expansion using `cargo expand`.

### 2. **Macro Expansion**

You can inspect the code generated by macros using tools like `cargo expand`, which shows the expanded code. This is helpful for understanding what the macro generates and debugging issues.

```bash
cargo install cargo-expand
cargo expand
```

### 3. **Compile-Time Evaluation**

Macros are expanded at compile time, meaning they do not affect runtime performance but can significantly affect compile times if used extensively. Be mindful of macro complexity and try to keep macros readable and maintainable.

### 4. **`compile_error!` for Diagnostics**

You can use `compile_error!` within a macro to provide helpful error messages when the macro is used incorrectly.

```rust
macro_rules! ensure_positive {
    ($x:expr) => {
        if $x <= 0 {
            compile_error!("Value must be positive");
        }
    };
}

fn main() {
    ensure_positive!(-1); // This will trigger a compile-time error
}
```

### 5. **Variable Capture and Matching**

- **Variable Capture**: Macro variables can capture and reuse parts of the input. For example, `$x` captures a piece of syntax that can be reused later in the macro expansion.
- **Matching**: You can match patterns in the macro arguments to handle different cases or configurations.

### 6. **Macros with Multiple Arms**

Macros can have multiple arms to handle different patterns. This allows you to create versatile macros that can adapt to various inputs.

```rust
macro_rules! print_info {
    ($x:expr) => {
        println!("Expression: {:?}", $x);
    };
    ($x:expr, $y:expr) => {
        println!("Expression 1: {:?}", $x);
        println!("Expression 2: {:?}", $y);
    };
}

fn main() {
    print_info!(42);
    print_info!(42, "hello");
}
```

### 7. **Recursive Macros**

Macros can call themselves recursively to handle more complex patterns. However, this can be tricky and should be used carefully to avoid infinite recursion.

```rust
macro_rules! factorial {
    (1) => { 1 };
    ($n:expr) => { $n * factorial!($($n - 1)*) };
}

fn main() {
    println!("{}", factorial!(5)); // prints "120"
}
```

### 8. **Attribute Macros**

Although not part of `macro_rules!`, attribute-like procedural macros (using `#[derive]`, `#[cfg]`, etc.) are another type of macro that provides more powerful capabilities. They are used to define custom derive traits, attributes, and functions.

### 9. **Declarative Macros and Code Generation**

Declarative macros are useful for code generation, such as implementing boilerplate code for struct fields, traits, or repetitive patterns. They reduce the need for manual coding and improve maintainability.

### 10. **Macro Expansion Limits**

Be cautious of deeply nested or overly complex macros, as they can lead to long compile times or errors. Keeping macros simple and understandable will help maintain their usability.

By understanding these additional aspects, you can use declarative macros more effectively and avoid common pitfalls. They are a powerful feature in Rust, enabling more expressive and reusable code.

### Conclusion

Declarative macros are a powerful feature in Rust that can greatly reduce code duplication and make your code more expressive and flexible. While they can be complex to write and understand, they provide a significant advantage in many scenarios. Start with simple macros and gradually work towards more complex patterns as you become more comfortable with the syntax and capabilities.

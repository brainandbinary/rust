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

### Conclusion

Declarative macros are a powerful feature in Rust that can greatly reduce code duplication and make your code more expressive and flexible. While they can be complex to write and understand, they provide a significant advantage in many scenarios. Start with simple macros and gradually work towards more complex patterns as you become more comfortable with the syntax and capabilities.

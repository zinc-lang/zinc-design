
# Basic Data Types

Naming convention: all named types use UpperCamelCase. (`fn` is an exception, because that keyword is mainly used in function definitions.)

## Bool type

The `Bool` type has two possible values, `true` and `false`.

Negation of a `Bool` uses the `!` operator.

## Integers

|  Width |  Signed  |  Unsigned  |
|-------|---------|----------|
| 8 bit | Int8      |  UInt8      |
| 16 bit | Int16      |  UInt16      |
| 32 bit | Int32      |  UInt32      |
| 64 bit | Int64      |  UInt64      |
| arch-dependent | Int      |  UInt  |
| arch-dependent | Short    | UShort  |

The size of Int/UInt depends on the target platform. On 64-bit platforms they are 64 bits; on 32-bit platforms they are 32 bits. They are equivalent to C's `intptr_t` / `uintptr_t`.
The size of Short/UShort depends on the target platform and is always half of Int/UInt. On 64-bit platforms they are 32 bits; on 32-bit platforms they are 16 bits.


Integer literals allow `_` as a separator.

Prefixes distinguish bases: `0o` for octal, `0x` for hexadecimal, `0b` for binary.

Integer literals allow a type suffix. Suffixes are lowercase, for example `123_i8`  `100_i`  `0x64_u`.
Without other contextual information, an integer literal cannot be inferred to a type. Users are encouraged to add a type suffix to integer literals to make the type explicit.

Integer arithmetic always uses wrapping on overflow. If you need other behavior, call the standard library. Integers provide `checked_add` / `wrapping_add` / `saturating_add` and the matching `sub`/`mul` methods.

Zinc does not provide "bitwise" operators, including left shift, right shift, bitwise and, bitwise or, bitwise not, and bitwise xor.
Instead, the standard library provides member functions on integer types for bitwise operations, such as `bit_and`, which is enough for the needed functionality.

Reason: bitwise operations are used infrequently and apply only to integer types, so dedicating operators to them is a bit wasteful.
Besides, bitwise operations are not limited to the ones above; there are also rotate, reverse, count_ones, and a dozen other uncommon bitwise operations. Those can only be provided as member functions anyway. It feels more consistent to provide all bitwise operations as member functions.
ASCII symbols are already in short supply; it is more reasonable to save those symbols for more important features.


## Floating-point numbers

Floating-point numbers are split into the `F32` and `F64` types.

Floating-point literals allow `_` as a separator.
Floating-point literals allow type suffixes, `f32` and `f64` respectively.

Floating-point literals may not omit the 0 before the decimal point. `.123` or `-.1` are both errors.

## Character type Char

The `Char` type is 32 bits and represents a Unicode code point.

`Char` literals are enclosed in single quotes. For example `'好'`.

`Char` literals allow escapes. `'\u{3456}'`.

Byte literals are supported: `b'a'`. A byte literal's value can only be an ASCII character, and its type is `UInt8`.

The standard library defines a type alias `type Byte = UInt8;`.

## Arrays

Array elements are all the same type. An array's length must be a compile-time constant.

An array type whose elements are `T` and whose length is `N` is written `[T; N]`. Arrays are value types; when an array type is assigned, passed as an argument, or returned, all elements are copied.

Example:

```rust
fn main() {
    let x: [Int; 3] = [1, 2, 3];
}
```

Array literals have two forms:

1. List all elements in brackets, separated by commas.

    `[1, 2, 3]`

2. Separate the element and the count with a semicolon in brackets, meaning several repeated elements.

    `[1; 3]`

Accessing array elements uses an index expression; out-of-bounds access panics. Example:

```rust
fn main() {
    let mut x: [Int; 3] = [1, 2, 3];
    x[1] = 10;
    println(x[5]); // compiles, panics at runtime
}
```

## Slices

An array slice is a borrow pointer to part of an array. There are two kinds of array slices, `Slice<'a, T>` and `SliceMut<'a, T>`, representing read-only and read-write permission respectively.

```rust
fn main() {
    let x: [Int; 3] = [1, 2, 3];
    let s: Slice<Int> = &x;
    println(s.len());
    println(s[1]);
}
```

An array slice is a struct with a lifetime; it can be seen as a borrow pointer with extra metadata. In the example above, the variable `s` is two pointers in size, storing the start address and the length respectively.

Note: the reason we do not use Rust's `&[T]` syntax is to simplify the type system and the implementation.
In Rust, `[T]` is a legal type that can be used independently and is dyn sized. That introduces a special case in the type system and is troublesome when implementing generics. Zinc wants users to only use `&[T]/&mut [T]` types, with no standalone dyn-sized `[T]` type, and no `Box<[T]>` `Rc<[T]>` and similar types.
Therefore the `&[T]/&mut[T]` syntax was replaced, and only `Slice/SliceMut` types are provided in the standard library. Their semantics are equivalent to forcing Rust's `[T]` to be used only together with a borrow pointer. Zinc's `Slice<'a, T>` is equivalent to Rust's `&'a [T]`.

> Range is implemented: half-open `..<` is `RangeExclusive<T>`, closed `..=` is `RangeInclusive<T>`. Integer types implement `Step`, so `for i in 0_i ..< 10_i` works. `RangeFull` is dropped.
Slicing cannot yet be done with the index operator `[]` together with a Range type, because operator overloading is not implemented yet.
```
let s: Slice<Int> = &x[1..<3]; // to be supported later
```

## Strings

There are two string types, `Str` and `String`.

`Str` is a struct with a lifetime defined in the standard library, `struct Str<'a>`. The `String` type is a struct without a lifetime defined in the standard library, implemented on top of `Vec<UInt8>`.

String literals have type `Str<'static>`.

Note: similar to array slices, the reason we do not use Rust's `str` keyword is to simplify the type system and the implementation.
In Rust, `str` is a legal type that can be used independently and is dyn sized. That introduces a special case in the type system and is troublesome when implementing generics. Zinc wants users to only use the `&str` type, with no standalone dyn-sized `str` type, and no `Box<str>` `Rc<str>` and similar types.
Therefore the `str` keyword was replaced, and only the `Str` type is provided in the standard library. Its semantics are equivalent to forcing Rust's `str` to be used only together with a read-only borrow pointer. Zinc's `Str<'a>` is equivalent to Rust's `&'a str`.

Strings are encoded as UTF-8 internally.

Thinking in C++ terms, Zinc's `String` corresponds to C++'s `std::string`, except the internal implementation uses copy-on-write. Zinc's `Str` corresponds to C++'s `std::string_view`, plus compile-time lifetime checking.

String literals support escapes.

Raw string literals are supported: `r"content"`, `r#" content "#`, `r##" content "##`. C string literals `c"..."` are also supported (typed as `Str`, with a trailing NUL byte for FFI).

todo: Support string interpolation, i.e. nested expressions inside string literals. Example:

```
let name: Str = "Amy";
let s: String = f"hello $(name), how are you?";
```

Parentheses are used instead of braces because in Zinc, parentheses are generally used for expression contexts. Expressions have a type and a value. Braces are generally used for statement contexts; statements have no type and no value.

TODO: Embedded expressions currently only support the simplest case of a single variable name; later this will support all expressions.

## Tuples

A tuple with 0 elements is called the unit type. The unit type is written `()`. The type of the empty-tuple `()` expression is the unit type.
If a function's return type is omitted, the function returns `()`.

A tuple with 1 element needs an extra trailing comma, otherwise the compiler treats it as a "parenthesized expression" rather than a tuple. `(1,)` is a tuple; `(1)` is not.

A tuple with 2 or more elements may have elements of different types.

## Structs

Structs are defined with the `struct` keyword. Example:

```rust
struct User {
    active: Bool,
    username: String,
    email: String,
}
```

Structs support generics.
Both the struct itself and its fields may be marked `pub`.
A struct may have no fields inside the braces.

Struct fields support default values; a default value must be a constant expression.
```rust
struct User {
    active: Bool = true, // default value
    username: String = f"",
    email: String = f"",
}
```

Struct initialization expressions are similar to C:

```rust
let user: User = {  // specify the type of the struct initialization expression by annotating the variable binding
    .active = false,
    .username = "John".to_string(),
    .email = "john@gmail.com".to_string(),
};
// or
let user = {  // specify the type of the struct initialization expression with a type suffix
    .active = false,
    .username = "John".to_string(),
    .email = "john@gmail.com".to_string(),
} : User;
```

Structs do not support C-style anonymous initialization of fields in order:
```
let user: User = { false, "John".to_string(), "john@gmail.com".to_string() };  // error
```

Fields with default values do not have to be initialized explicitly.

You can initialize one struct from another. Note that this uses three dots `...`, not Rust's two dots `..`.
In Zinc, three dots `...` are used uniformly to mean ellipsis. Ellipsis syntax is used in several contexts. An expression may or may not follow the ellipsis.

```rust
// active can be omitted, meaning it uses the default value
let user1: User = { .username = "smith".to_string(), .email = String::new() };

// initialize user2 from user1, changing only the active field
let user2: User = { .active = true, ...user1 }; 

// initialize user3 from the default values in the User type definition, changing only the active field
let user3: User = { .active = true, ... }; 

// this is equivalent to initializing entirely from the default values in the User type definition.
// the ellipsis is required; otherwise lexical analysis cannot distinguish an empty struct initializer from an empty statement block
let user4: User = { ... }; 
```

Structs do not support inheritance. Zinc plans to support the object-oriented programming paradigm through trait inheritance and the `reuse` mechanism.

Zinc does not plan to support tuple structs, i.e. structs whose fields have no names. For simple structs, the suggestion is to provide a corresponding snake_case global function as a "constructor."
That keeps the language rules concise and uniform, and from a usage perspective the naming convention is also uniform. There will be no "function-like" things that start with a capital letter.
For example:

```rust
struct Weight {
    w: UInt32
}

fn weight(w: UInt32) -> Weight {
    return { .w = w };
}

fn main() {
    // a bit more work on the definition side; on the use side it is basically the same as a tuple struct
    let w = weight(1); 
}
```

Zinc supports "unit structs." A struct with no fields can end with a semicolon, or have empty braces:
```rust
// the following two spellings are the same
struct Unit{} 
struct Unit;
```
But note that initializing a unit struct still requires braces:
```
let x: Unit = { ... };
```

## Enums

Enums are defined with the `enum` keyword:

```rust
enum IpAddrKind {
    V4,
    V6,
}

fn main() {
    let k: IpAddrKind = IpAddrKind::V4;
}
```

Enum variants support associated values.
```rust
enum IpAddr {
    V4(Byte, Byte, Byte, Byte),
    V6(String),
}
```

Brace-style associated values are not supported. If the user needs named parameters, the suggestion is to wrap all parameters in a struct, which has the same effect:

```rust
enum E {
    A{ x:Byte, y: String } // error
}
// alternative:
struct EA {
    x: Byte,
    y: String,
}
enum E {
    A(EA),
}
```

Enum variant overloading is supported, with the same rules as function overloading: only overloading by different argument counts is supported.

```rust
enum Arg {
    Integer(UInt),
    Integer(UInt, UInt),
    Integer(UInt, UInt, UInt),
    Float(F64),
    Float(F64, F64),
    Float(F64, F64, F64),
}
```

An enum is essentially a tagged union. Users may specify which integer type the tag of a custom enum uses, and the tag value corresponding to each variant.
Tag values must be constant expressions, must be representable in the corresponding integer type, and must not be duplicated across variants.

Only enums without associated values may specify tag values.

```rust
enum IpAddr : UInt8 {
    V4 = 1,  //
    V6 = 2,
}
```

Enums without associated values may be converted directly to integers with the `as` operator.

Enum type definitions support generics. Enum variants may not introduce new generics.

> **Planned feature**: Support defining non-exhaustive enums, with syntax that uses an ellipsis after the last variant:
```rust
enum Puctuation {
    Comma,
    Dot,
    Semi,
    ...  // ellipsis, equivalent to marking the enum with #[non_exhaustive] in Rust
}
```

### Pointer types

Covered in Chapter 4.

### A few special enum types defined in the standard library

1. `Option`

    ```rust
    enum Option<T> { None, Some(T) }
    ```

    The Option type has a special memory-layout optimization.

2. `Result`

    ```rust
    enum Result<T, E> { Ok(T), Err(E) }
    ```

    Result supports the `?` operator, which makes error handling convenient.

## Unions

Unions are defined with the `union` keyword. A union's overall size is determined by the size of its largest field.

```
union U {
    data1: Byte,
    data2: UInt,
}
```

Union types are unsafe; reading and writing union fields must be done in an unsafe context.

Union initialization syntax is similar to struct initialization syntax. But you must choose exactly one field to initialize.

Non-trivial types cannot be used as union fields.
Non-trivial types include:
1. Types with a destructor, such as Arc/Weak
2. Types with lifetime parameters, such as &/&mut
3. If a struct's fields include a non-trivial type, the struct is non-trivial
4. If an enum's associated values include a non-trivial type, the enum is non-trivial
5. If an array's element type is non-trivial, the array is non-trivial
6. If a tuple's elements include a non-trivial type, the tuple is non-trivial

Union type definitions may not have generic parameters.

Unions have two main use cases:
1. Interop with C
2. Bitwise type conversion (similar to Rust's `std::mem::transmute`)

For example, if you want to bit-cast an `F32` to a `UInt32`, a union can help:

```rust
union FloatOrInt {
    f: F32,
    i: UInt32,
}

fn bit_cast_f32_to_uint32(f: F32) -> UInt32 {
    let u: FloatOrInt = { .f = f };
    return unsafe(u.i);
}
```

## Type sizes

Types are divided into Sized and UnSized. Sized types can be used as local variables, global variables, and fields, while UnSized types cannot; they can only be accessed indirectly through a pointer.

UnSized types include:
1. Types declared with `extern type`
    * `extern type` is mainly used for interop. It means the type's definition is in another language, and Zinc does not have access to that definition. Therefore its size is unknown, and its fields cannot be accessed directly.

2. All traits
    * Suppose `trait R` is defined; you cannot declare a variable of type `R`, but you can use `&R / &mut R / *R` and similar

3. Custom types `S` that explicitly `impl UnSized for S {}`
    * `UnSized` is a built-in trait that lets users mark a custom type as Unsized, via `impl UnSized for MyType {}`.
        The purpose is to forbid using this type as a "value-semantic type"; it may only be used through a pointer, as a "reference-semantic type."
        A typical case is the `File` type in the standard library. Users are not allowed to use this type as a "value-semantic type"; they can only obtain a pointer type pointing to `File`.

In generic contexts, UnSized types are currently not allowed as generic arguments.

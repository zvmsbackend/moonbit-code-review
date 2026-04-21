---
name: moonbit-code-review
description: Common pitfalls and best practices for code review in the Moonbit project. Used when asked to review an existing MoonBit codebase or implementing a new feature.
---

# Project Structure

## Split code into multiple files and packages

The basic scope unit of MoonBit project is package, not file. A package can contain multiple files, and the files all share a single scope. It's safe to split code into multiple files, and it's often a good idea to do so for better organization and readability.

Modules are the distribution unit of MoonBit project. A module can contain multiple packages. It's a good practice to split a module into multiple packages, especially if the module is large or has distinct components. This can help improve code organization and maintainability while reducing coupling between different parts of the codebase.

When splitting code into multiple files and packages, it's important to ensure that the code is organized in a logical and consistent manner. This can help make it easier for developers to navigate the codebase and understand how different components fit together. Additionally, it's important to ensure that the code is well-documented and follows established coding conventions to improve readability and maintainability.

## Package with Multiple Targets

MoonBit has four target backends: `wasm` (including `wasm-gc`), `native`, `js` and `llvm`, among which `wasm`, `native` and `js` are the most commonly used ones. Some language features are restricted to specific targets, notably `extern` FFI functions. To support multiple targets in a single package, we can use conditional compilation with `#cfg` attributes to include or exclude code based on the target platform. This allows us to maintain a single codebase while still supporting multiple backends. As the number of `#cfg` attributes increases, it's worth considering splitting the code into separate files and conditionally compile them with `targets` option in `moon.pkg` file. Here is an example of how to use conditional compilation to support multiple targets in a single package:

```
- lib
  - lib_js.js // contains code specific to the js target
  - lib_native.rs // contains code specific to the native target
  - lib_wasm.rs // contains code specific to the wasm target
  - moon.pkg
```

and in `moon.pkg` file:

```
options(
  targets: {
    "lib_js.js": ["js"],
    "lib_native.rs": ["native"],
    "lib_wasm.rs": ["wasm", "wasm-gc"],
  }
)
```

When the package support exactly one target, it's still recommended to conditionally compile the files as mentioned above, to make packages that import this package compile on other targets, as MoonBit does not support conditional import yet.

Be careful when the public interface of the package diverges between different targets, as this can lead to confusion and maintenance issues. In such cases, it's important to clearly document the differences in the public interface for each target and ensure that the code is organized in a way that makes it easy for developers to understand which parts of the code are relevant for each target. Avoid such inconsistency unless you have a good reason to do so.

## Internal Packages

Sometimes a package is only intended to be used internally within a module, and should not be exposed as part of the public API. In such cases, it's a good practice to place the package in an `internal` directory within the module. This can help signal to developers that the package is not intended for external use and should only be used within the module.

# Package Interface

## Restrict the public interface of a package

In MoonBit, there is a rich set of visibility levels. Take structs for example. There are six levels of visibility for structs:

- `priv struct`: the struct is only visible within the current package.
- `struct`: the struct is visible to the current package and all packages that import it, but the fields of the struct are private and not visible outside the package.
- `pub struct`: the struct and its fields are visible to the current package and all packages that import it, but the fields are read-only and cannot be modified outside the package. Also, the struct cannot be instantiated outside the package, as the constructor is private.
- `pub(all) struct`: the struct and its fields are visible to all packages, and the fields can be modified outside the package.
- `pub struct` with `priv` fields: the struct is visible to the current package and all packages that import it, but the fields marked as `priv` are only visible within the current package.
- `pub(all) struct` with `priv` fields: the struct and its fields are visible to all packages, but the fields marked as `priv` are only visible within the current package. These `struct`s are similar to `pub struct` with `priv` fields, but the `mut` fields can be modified outside the package.

enums are similar to structs, but they only have the first four visibility levels, as it's not possible to have private constructors in enums.

Traits have four visibility levels too:

- `priv trait`: the trait is only visible within the current package.
- `trait`: the trait is visible to the current package and all packages that import it, but the methods of the trait are private and not visible outside the package.
- `pub trait`: the trait and its methods are visible to the current package and all packages that import it, but new instances of the trait cannot be implemented outside the package.
- `pub(open) trait`: the trait and its methods are visible to all packages, and new instances of the trait can be implemented outside the package.

Functions, let bindings, constants and type synonyms have only two visibility levels: `pub` and default (private).

When designing the public interface of a package, it's important to carefully consider which components should be exposed and which should be kept private. This can help improve encapsulation and reduce coupling between different parts of the codebase. Additionally, it's important to ensure that the public interface is well-documented and follows established coding conventions to improve readability and maintainability.

## Avoid Unnecessary Mutability and Nullability

When designing the structure of a MoonBit type, it's tempting to leave room for potential modification and nullability. Here is a snippet that follows this approach:

```moonbit nocheck
struct User {
  mut name : String?
}

pub fn User::name(self : User) -> String? {
  self.name
}

pub fn User::set_name(self : User, name : String?) -> Unit {
  self.name = name
}
```

This is usually a bad practice. Unless you are sure the `User` is possible to be anonymous and the name can be changed during the object's lifetime, it's better to design the struct like this:

```moonbit nocheck
struct User {
  name : String
}

pub fn User::name(self : User) -> String {
  self.name
}
```

## Use Named Parameters And Enum Payloads

When a function has many parameters and their meanings cannot be easily inferred from their types, it's a good practice to use named parameters or enum payloads to improve readability and maintainability. This can help make it clear what each parameter represents and how it should be used. Additionally, it can help reduce the likelihood of errors caused by passing parameters in the wrong order or with the wrong values.

Besides function parameters, it's worth noting that MoonBit enums support named payloads, too. Below is an example of named payloads in enums:

```moonbit nocheck
enum Shape {
  Circle(radius~ : Double)
  Rectangle(width~ : Double, height~ : Double)
}

fn init {
  let rectangle = Rectangle(width: 10.0, height: 5.0)
}
```

## Optional Parameters

MoonBit confusingly has two styles of optional parameters, as shown below:

```moonbit nocheck
fn foo(bar? : Int, baz? : Int? = None) -> Unit {
  // function body
}
```

The `bar` and `baz` parameters look pretty similar at first glance, but there's a subtle difference in the way to call the `foo` function:

```moonbit nocheck
fn init {
  foo(
    bar=42,
    baz=Some(67)
  )
}
```

From a language-mechanism perspective, `arg? : T` and `arg? : T? = None` have the same in-function semantics (`T?` with default `None`), but they are not equivalent at call sites: `arg? : T` supports auto-wrapping (`arg=v`), while `arg? : T? = None` forces explicit `Some(v)` in common value-passing cases. Therefore, `arg? : T? = None` is discouraged unless explicit `T?` passing is truly required by the API surface.

## Prefer Checked Errors Over `Result`s

When a function can fail, it's generally better to use checked errors instead of returning a `Result` type. Checked errors and `Result`s essentially compile to the same representation, but the former propagates errors implicitly in nested control flow, while the latter requires explicit handling of the `Result` type at each level. This can lead to more concise and readable code when using checked errors, as it reduces the amount of boilerplate code needed to handle errors.

## Implement Standard Traits Whenever Possible

MoonBit provides a set of standard traits that can be implemented for custom types, notably `Eq`, `Compare`, `Hash` and `Show`. When you find some methods like `Type::equal`, `Type::compare`, `Type::hash` or `Type::to_string` in your code, it's a good idea to consider implementing the corresponding standard traits for the type instead. This can help improve code reuse and make it easier for other developers to work with your types, as they can take advantage of the standard trait implementations.

## Prefer Trait Objects For Dynamic Dispatch

In MoonBit there are mainly two ways to achieve dynamic dispatch: using trait objects or using a struct of higher-order functions. While the latter is more flexible, trait objects are generally preferred for they are more idiomatic and usually more efficient.

## Re-export Declarations In The Root Package

When your module contains multiple packages, and the users need to import multiple packages to use the module, it's a good practice to re-export the necessary components in the root package with `pub using` statement. This can help simplify the import statements for users and make it easier for them to access the functionality provided by the module. Additionally, it can help improve code organization and maintainability by centralizing the public interface of the module in a single location.

## Prefer Methods Over Functions

When you find yourself needing to define a function that operates on a specific type, it's generally better to define it as a method on that type rather than as a standalone function. This can help improve readability and maintainability, as it makes it clear that the function is associated with the type and can be called using method syntax. Additionally, defining functions as methods can help reduce the likelihood of naming conflicts and make it easier for developers to discover and use the functionality provided by the type.

Note that in MoonBit, files in the same package share the same scope, so you can define methods for a type in a different file as long as they are in the same package.

## Provide Struct Constructors

MoonBit structs may have an optional constructor declaration. In this way you can instantiate a struct like `TypeName(...)` instead of `TypeName::new(...)`. This can help improve readability and make it easier for developers to create instances of the struct.

```moonbit nocheck
pub struct User {
  name : String

  fn new(name : String) -> User
}

pub fn User::new(name : String) -> User {
  { name }
}
```

Pay attention that the visibility of the constructor follows that of the struct, not the associated function, so if the `struct` is not declared as `pub`, the constructor will not be accessible outside the package even if the associated function is `pub`. To make the constructor accessible while keeping the fields private, you can declare the struct as `pub` and the fields as `priv`:

```moonbit nocheck
pub struct User {
  priv name : String

  fn new(name : String) -> User
}

pub fn User::new(name : String) -> User {
  { name }
}
```

# Code Style

## Prefer Standard Library Over Custom Implementations

MoonBit provides a rich standard library with diverse functions and data structures, including utility functions on primitive types(`@string`, `@cmp`, etc.), collections (`@queue`, `@deque`, `@immut/vector`, etc.). When you find yourself needing to implement a common data structure or utility function, it's a good idea to first check if there is already a suitable implementation in the standard library. MoonBit toolchain provides a `moon ide doc` command that can query the documentation of the standard library and beyond, which can help you find existing implementations and avoid reinventing the wheel. Using the standard library can help improve code reuse and maintainability, as well as reduce the likelihood of bugs caused by custom implementations.

## Reduce Indentation Levels With Early Returns And `guard` Statements

When a function's body involves deep nested pattern matching, the increasing indentation levels can make the code harder to read and understand. To mitigate this issue, it's a good practice to use early returns or `guard` statements to handle edge cases and error conditions at the beginning of the function. This can help reduce the overall indentation levels and improve readability. Here's an example of how to use early returns and `guard` statements to reduce indentation levels:

```moonbit nocheck
fn process_data(data: Data) -> ProcessedData? {
  if data.is_empty() {
    match (data.prop_a(), data.prop_b()) {
      (Some(a), Some(b)) => process_a_and_b(a, b)
      _ => None
    }
  } else {
    None
  }
}
```

Using early returns and `guard` statements, we can simplify the function to

```moonbit nocheck
fn process_data(data: Data) -> ProcessedData? {
  if data.is_empty() {
    return None
  }
  guard data.prop_a() is Some(a) && data.prop_b() is Some(b) else {
    return None
  }
  process_a_and_b(a, b)
}
```

## Avoid Unnecessary Copying With View Types

MoonBit has three built-in view types: `StringView`, `BytesView` and `ArrayView`. They work with most of the standard library functions that take `String`, `Bytes` and `Array` as parameters, but they do not own the underlying data and are much cheaper to copy. When you find yourself needing to pass a `String`, `Bytes` or `Array` to a function, but you don't need to modify it, it's a good idea to consider using the corresponding view type instead. This can help reduce unnecessary copying and improve performance, especially when dealing with large data structures.

## Avoid Unnecessary Qualification On Constructors

Unlike Rust but similar to OCaml, MoonBit does not require enum constructors to be qualified with the enum name. For example, if you have an enum `Option` defined as follows:

```moonbit nocheck
enum Option[T] {
  Some(T)
  None
}
```

You can construct an `Option` value without qualifying the constructor with the enum name unless there is a name conflict. So instead of writing `Option::Some(42)`, you can simply write `Some(42)`:

```moonbit nocheck
fn init {
  let x = Some(42) // instead of Option::Some(42)
}
```

If the enum comes from another package and is not imported with `using`, a package prefix might be required, but the enum name is still not necessary:

```moonbit nocheck
fn init {
  let x = @pkg.Some(42) // instead of Option::Some(42)
}
```

Moreover, if the expected type is known, the package prefix can be omitted as well:

```moonbit nocheck
fn accept_option(x : @pkg.Option[Int]) -> Unit {
  // function body
}

fn init {
  accept_option(Some(42)) // instead of @pkg.Some(42)
}
```

The same applies to struct constructors.

## Use `is` Expressions To Simplify Pattern Matching

When you find yourself needing to match on a value and check if it is of a certain type or variant, it's often more concise and readable to use an `is` expression instead of a full pattern match. This can help simplify the code and make it easier to understand. Here's an example of how to use an `is` expression to simplify pattern matching:

```moonbit nocheck
// bad
fn Expr::is_atom(self : Expr) -> Bool {
  match self {
    Int(_) | String(_) | Ident(_) => true
    _ => false
  }
}

// good
fn Expr::is_atom(self : Expr) -> Bool {
  self is (Int(_) | String(_) | Ident(_))
}
```

## Use View Patterns To Deconstruct Arrays, Strings And Bytes

When you find yourself needing to deconstruct an array, string or bytes value in a pattern match, it's often more concise and readable to use view patterns instead of manually indexing into the value. This can help simplify the code and make it easier to understand. Here's an example of how to use view patterns to deconstruct an array:

```moonbit nocheck
// bad
fn first_two_elements(arr: Array[Int]) -> (Int, Int)? {
  if arr.length() >= 2 {
    let first = arr[0]
    let second = arr[1]
    Some((first, second))
  } else {
    None
  }
}

// good
fn first_two_elements(arr: Array[Int]) -> (Int, Int)? {
  match arr {
    [first, second, ..] => Some((first, second))
    _ => None
  }
}
```

## Prefer `let` Or `const` Over Constant Functions

When you find yourself needing to define a constant value, it's generally better to use a `let` binding or a `const` declaration instead of a constant function. This can help improve readability and maintainability, as it makes it clear that the value is constant and does not require any computation to obtain. Additionally, using `let` or `const` can help reduce the likelihood of bugs caused by unintended side effects or changes to the constant value.

## Use `StringBuilder` To Build Strings

When you find yourself needing to build a string by concatenating multiple parts together, it's often more efficient and readable to use a `StringBuilder` instead of using the `+` operator or string interpolation. This can help improve performance by reducing the number of intermediate string objects created during concatenation, and it can also make the code easier to understand by clearly indicating that you are building a string.

## Prefer Functional Loops Over Imperative Loops

MoonBit loops are functional i.e. they optionally yield a value when the iteration is done instead of always returning `Unit`. This allows you to use loops in a more functional style, which can lead to clearer and more concise code. When you find yourself needing to use a loop, it's a good idea to consider using a functional loop instead of an imperative loop. This can help improve readability and maintainability, as well as reduce the likelihood of bugs caused by mutable state. Here's an example of how to use a functional loop:

```moonbit nocheck
fn sum_of_squares(n: Int) -> Int {
  let mut sum = 0
  for i in 1..<=n {
    sum += i * i
  }
  sum
}
```

Using a functional loop, we can simplify the function to

```moonbit nocheck
fn sum_of_squares(n: Int) -> Int {
  for i in 1..<=n; sum = 0 {
    continue sum + i * i
  }
}
```

## Prefer Iterators Over Explicit Loops

When you find yourself needing to iterate over a collection, it's generally better to use iterators instead of explicit loops. Iterators provide a more concise and expressive way to work with collections, and they can help improve readability and maintainability. Additionally, iterators can often be more efficient than explicit loops, as they can take advantage of lazy evaluation and other optimizations. Common scenarios where iterators can replace explicit loops:

```moonbit nocheck
// any/all

// bad
fn all_available(items: Array[Item]) -> Bool {
  for item in items {
    if !item.is_available() {
      return false
    }
  }
  true
}

// good
fn all_available(items: Array[Item]) -> Bool {
  items.iter().all(item => item.is_available())
}
```

```moonbit nocheck
// map/filter

// bad
let result = []
for item in items {
  if item.is_available() {
    result.push(item.value())
  }
}

// good
items.filter_map(item => if item.is_available() {
  Some(item.value())
} else {
  None
})
```

```moonbit nocheck
// find

// bad
fn find_available(items: Array[Item]) -> Item? {
  for item in items {
    if item.is_available() {
      return Some(item)
    }
  }
  None
}

// good
fn find_available(items: Array[Item]) -> Item? {
  items.iter().find_first(item => item.is_available())
}
```

MoonBit supports the so-called "error polymorphism", so it doesn't matter if the higher-order function passed to iterator methods can fail or not. However, if the computation is async, MoonBit currently cannot automatically derive an async version of the method, so an explicit loop is still required in such cases.

## Prefer `Iter2` Over Manual Indexing

The left hand side of MoonBit foreach loops can optionally receive two values, in which case MoonBit calls the `iter2()` method on the iterated collection instead of `iter()`. This allows you to access both the index and the value of each item in the collection without needing to manually manage an index variable. When you find yourself needing to access both the index and the value of items in a collection, it's a good idea to consider using `Iter2` instead of manual indexing. This can help improve readability and maintainability, as well as reduce the likelihood of bugs caused by off-by-one errors or other issues related to manual indexing. Here's an example of how to use `Iter2`:

```moonbit nocheck
// bad
for i in 0..<items.length() {
  let item = items[i]
  // use i and item
}

// good
for i, item in items {
  // use i and item
}
```

## Use `rev_iter()` To Iterate Backwards

Most built-in collections provide a `rev_iter()` method that allows you to iterate over the collection in reverse order. When you find yourself needing to iterate over a collection backwards, it's a good idea to consider using `rev_iter()` instead of manually reversing the collection or using an index variable to iterate in reverse. This can help improve readability and maintainability, as well as reduce the likelihood of bugs caused by off-by-one errors or other issues related to manual reverse iteration. Here's an example of how to use `rev_iter()`:

```moonbit nocheck
// bad
for i = items.length() - 1; i >= 0; i = i - 1 {
  let item = items[i]
  // use item
}

// good
for item in items.rev_iter() {
  // use item
}
```

## Prefer `mut` Variables Over `Ref`s

In MoonBit, you can use `mut` variables or `Ref`s to achieve mutability. However, `mut` variables are generally preferred over `Ref`s for they are more concise and more efficient. `Ref`s can be useful in certain situations, such as when you need to share mutable state across multiple functions or when you need to implement interior mutability, but in most cases, using `mut` variables is sufficient and leads to clearer code.

## Write `...` Placeholder For Unimplemented Code

When you find some functionality partially implemented, it's a good practice to use `...` as a placeholder for the unimplemented code. This can help signal to developers that the code is not yet complete and should be implemented in the future by emitting compile-time warnings. Additionally, using `...` can help improve readability and maintainability by making it clear which parts of the code are still under development and need attention.

## Labelled/Optional Argument Evaluation Semantics

In MoonBit, labelled arguments can be supplied in any order at call sites, but argument evaluation still follows the parameter declaration order. For optional arguments, the default expression is evaluated every time that argument is omitted.

Because of this, code review should reject APIs that rely on call-site argument order side effects, and should avoid side effects in optional default expressions. Keep defaults pure whenever possible. If shared state is needed across calls, lift it to a toplevel `let` and use that binding as the default value.

```moonbit nocheck
let default_counter : Ref[Int] = { val: 0 }

fn incr(counter? : Ref[Int] = default_counter) -> Int {
  counter.val = counter.val + 1
  counter.val
}
```

# Testing

## Use `inspect` To Inspect Objects' Internal States

When writing tests, it's often necessary to inspect the internal state of objects to verify that they are behaving as expected. MoonBit provides an `inspect` function that can be used to print the string representation of an object in a human-readable format. This can be especially useful for debugging and understanding how your code is working. When you find yourself needing to write plenty of equality tests, it's a good idea to consider using the `inspect` function. This can help improve the effectiveness of your tests and make it easier to identify issues in your code. Here's an example of how to use the `inspect` function in a test:

```moonbit nocheck
test {
  let user = User::new("Alice")
  inspect(user, content="{name: \"Alice\"}")
}
```

Alternatively, you can leave the `content` parameter unfilled, and run `moon test --update` to update the expected content with the actual content, which can be useful when you are not sure about the expected content at the beginning:

```moonbit nocheck
test {
  let user = User::new("Alice")
  inspect(user)
}
```

Make sure the updated content is correct before committing the changes, as blindly updating the expected content with the actual content can lead to false positives in your tests.

Since MoonBit 0.9.0, deriving trait `Show` is deprecated. Instead, you can implement the `Debug` trait and use `debug_inspect` function which has a similar signature to `inspect` but uses the `Debug` implementation of the object to print its internal state.

## Use `json_inspect` To Inspect Objects In JSON Format

When you find yourself needing to inspect the internal state of an object in a structured format, it's a good idea to consider using the `json_inspect` function. This function allows you to print the string representation of an object in JSON format, which can be especially useful for debugging and understanding how your code is working. Here's an example of how to use the `json_inspect` function in a test:

```moonbit nocheck
test {
  let user = User::new("Alice")
  json_inspect(user, content={"name": "Alice"})
}
```

## Use `try?` To Simplify Error Handling In Tests

When writing tests, it's common to have functions that can fail and raise an error. In such cases, it's often more concise and readable to use the `try?` operator to convert the error into a `Result` value, which can then be easily handled in the test. This can help simplify error handling in your tests and make it easier to verify that your code is behaving as expected. Here's an example of how to use the `try?` operator in a test:

```moonbit nocheck
test {
  let result = try? some_function_that_can_fail()
  assert_true(result is Err(...))
}
```

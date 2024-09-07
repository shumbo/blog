+++
title = 'Union Types in Rust with Type-Level Lists'
date = 2024-09-06T22:37:50-07:00
draft = false
+++

In this article, I will discuss a technique to represent union types in Rust. With type-level lists, we can express a set of types, and through trait resolution, determine if a particular type is part of a set or if one set is a subset of another.

The core of this technique is a recursive operation on type-level lists. To avoid conflicting implementations of traits, we will add a marker type that express the depth of recursion. With this trick, various operations on type-level lists become possible.

## Motivation

### Union Types and Enums

Union types in languages like TypeScript and Scala allow combining multiple types into a single type. For example, in TypeScript, `string | number` represents a type that can be either a string or a number.

```typescript
function printValue(value: string | number) {
  console.log(value); // value is either string or number
}

printValue("hello"); // OK
printValue(42); // also OK
```

Rust doesn't have union types. In many cases, we can use enums to achieve similar functionality. For instance, to represent values that can be either a string or a number, we can define an enum:

```rust
enum Value {
    String(String),
    Number(i32),
}
```

Then, we can write a function that accepts `Value`:

```rust
fn print_value(value: Value) {
    match value {
        Value::String(s) => println!("{}", s),
        Value::Number(n) => println!("{}", n),
    }
}

print_value(Value::String("hello".to_string())); // OK
print_value(Value::Number(42)); // also OK
```

### Limitations of Enums

Although these two language features are similar, there are situations where enums are not as expressive as union types. One such case is writing functions that only accept certain cases. As an example, consider a cross-platform graphic library that supports rendering different shapes.

This library can render three types of shapes: `Circle`, `Square`, and `Text`. In TypeScript, we might define an abstract class `Shape` and three subclasses `Circle`, `Square`, and `Text`.

```typescript
abstract class Shape {}

class Circle extends Shape {
  constructor(public radius: number) {
    super();
  }
}

class Square extends Shape {
  constructor(public size: number) {
    super();
  }
}

class Text extends Shape {
  constructor(public text: string) {
    super();
  }
}
```

Suppose that our library supports the following operating systems:

- GeometryOS: specializes in rendering geometric shapes and only supports `Circle` and `Square`.
- TextOS: specializes in rendering text and only supports `Text`.
- MightyOS: supports all shapes.

In TypeScript, we can use union types to express supported shapes and write functions that only accept certain shapes:

```typescript
type GeometryShape = Circle | Square;
type TextShape = Text;
type MightyShape = GeometryShape | TextShape;

function renderOnGeometryOS(shape: GeometryShape) {
  // render circle or square
}

function renderOnTextOS(shape: TextShape) {
  // render text
}

function renderOnMightyOS(shape: MightyShape) {
  // render any shape
}
```

If we pass an unsupported shape to any of these functions, TypeScript will raise a type error.

```typescript
renderOnGeometryOS(new Circle(10)); // OK
renderOnGeometryOS(new Text("hello")); // Error
```

Now, let's try to implement the same functions in Rust. We model shapes as an enum:

```rust
enum Shape {
    Circle(radius: f64),
    Square(size: f64),
    Text(text: String),
}
```

However, we can't impose constraints on functions to only accept certain cases.

```rust
fn render_on_geometry_os(shape: ???) {
    // can only render circle or square
}
```

We can use pattern matching and crash the program if an unsupported shape is passed (or maybe return a `Result`), but we can't express this constraint at the type level.

In the rest of this article, I will show how to use type-level lists to represent union types in Rust and write functions that only accept certain cases.

## Representing Sets of Types with Type-Level Lists

To achieve union types in Rust, we need two things: (A) a way to define a type that represents a set of types and (B) a way to check if a type is a member of a set.

We can use a type-level list to represent a set of types. We define a trait `HList` and implement it on two types: `HNil` that represents an empty list and `HCons<Head, Tail>` that represents a list.

```rust
struct HNil;
struct HCons<Head, Tail>(Head, Tail);

trait HList {}

impl HList for HNil {}
impl<Head, Tail> HList for HCons<Head, Tail> {}
```

With this definition, we can represent type-level lists as follows:

```rust
struct Circle {
    radius: f64,
}
struct Square {
    size: f64,
}
struct Text {
    text: String,
}

// Geometric shapes
type GeometryShapes = HCons<Circle, HCons<Square, HNil>>;
// Text shapes
type TextShapes = HCons<Text, HNil>;
// All shapes
type AllShapes = HCons<Circle, HCons<Square, HCons<Text, HNil>>>;
```

The repeated use of `HCons` and `HNil` is verbose, so we define a macro to simplify this. Note that this macro is just to make the code more readable; it doesn't add any new functionality.

```rust
macro_rules! HList {
    () => { HNil };
    ($head:ty $(,)*) => { HCons<$head, HNil> };
    ($head:ty, $($tail:tt)*) => { HCons<$head, HList!($($tail)*)> };
}
```

With this macro, we can define type-level lists more concisely:

```rust
type GeometryShapes = HList!(Circle, Square);
type TextShapes = HList!(Text);
type AllShapes = HList!(Circle, Square, Text);
```

## Checking if a Type is a Member of a Set

Now that we have a way to represent sets of types, we need a way to check if a type is a member of a set. It would be nice if we could define a trait `Member` that is implemented for a type if it is a member of a set. For example, we would like to write:

```rust
type GeometryShapes = HList!(Circle, Square);

fn render_on_geometry_os<S>(shape: S)
where S: Member<GeometryShapes>
{
    // can only render circle or square
}
```

How might we implement such a trait? Given a type-level list `L`, we want to implement `Member<L>` for all types in `L`. This requires recursing over the list and implementing `Member` for each type. Specifically, we want to implement the following two cases:

- Base case: `Head` is `Member<HCons<Head, Tail>>`.
- Recursive case: If `T` is `Member<Tail>`, then `T` is `Member<HCons<Head, Tail>>`.

These two cases can be encoded as follows:

```rust
trait Member<L> {}

impl<Head, Tail>    Member<HCons<Head, Tail>> for Head {}
impl<T, Head, Tail> Member<Tail>              for T
    where T: Member<HCons<Head, Tail>> {}
```

Unfortunately, this code doesn't compile. The compiler will give the following error:

```
error[E0119]: conflicting implementations of trait `Member<HCons<_, _>>`
  --> src/main.rs:44:1
   |
43 | impl<Head, Tail> Member<HCons<Head, Tail>> for Head {}
   | --------------------------------------------------- first implementation here
44 | impl<T, Head, Tail> Member<Tail> for T where T: Member<HCons<Head, Tail>> {}
   | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ conflicting implementation
```

This error arises because there are multiple trait implementations for the same type. When the type being searched for is Head, the base case implements `Member<HCons<Head, Tail>>` for `Head`. In the recursive case, `Member<Tail>` overlaps with this, leading the compiler to be unable to distinguish between the two trait implementations.

While [Specialization](https://github.com/rust-lang/rust/issues/31844) might resolve this problem, this article assumes the use of stable Rust.

There is a way to solve this issue. It involves a trick I found through in the article [A Gentle Intro to Type-level Recursion in Rust: From Zero to HList Sculpting](https://beachape.com/blog/2017/03/12/gentle-intro-to-type-level-recursion-in-Rust-from-zero-to-frunk-hlist-sculpting/), to which I refer in this article as the "Index Trick." The core of this trick lies in separating the base case and recursive case by introducing a new type parameter (index) to `Member` that represents the depth of recursion.

First, we add a new type parameter `Index` to the `Member` trait:

```rust
trait Member<L, Index> {}
```

Next, we define two types, `Here` and `There<Index>`, to distinguish between the base case and the recursive case:

```rust
struct Here;
struct There<Index>(Index);
```

`Here` represents the base case, and `There` represents the recursive case. The base case implements `Member<T, Here>` while the recursive case implements `Member<T, There<Index>>` where `T` is `Member<Tail, Index>`. Since `Member<H, Here>` and `Member<T, There<Index>>` are distinct, we avoid conflicting implementations.

```rust
impl<Head, Tail: HList> Member<HCons<Head, Tail>, Here> for Head {}
impl<T, Head, Tail, Index> Member<HCons<Head, Tail>, There<Index>> for T
where T: Member<Tail, Index> {}
```

Here, `Index` represents the depth of recursion as a type parameter. `Here` indicates that the recursion depth is zero and does not recurse further. `There<Index>` indicates that recursion will continue for an additional `Index` times after the current level (for a total of `Index + 1` times). This ensures that the compiler treats the trait implementations as distinct.

For example, given

```rust
type AllShapes = HList!(Circle, Square, Text);
```

- `Circle` implements `Member<AllShapes, Here>`.
- `Square` implements `Member<AllShapes, There<Here>>`.
- `Text` implements `Member<AllShapes, There<There<Here>>>`.

Now, let's define `render_on_geometry_os`. The `Member` trait requires an `Index` parameter, but fortunately, Rust's type inference can deduce the `Index` parameter for us. We can write:

```rust
fn render_on_geometry_os<S, Index>(shape: S)
where S: Member<GeometryShapes, Index>
{
    // can only render circle or square
}
```

In this case, when Circle or Square is passed as an argument, the compiler will infer the Index and interpret it as Member`<GeometryShapes, Here>` or `Member<GeometryShapes, There<Here>>`. On the other hand, if Text is passed, the compiler cannot infer an Index that satisfies the `S: Member<GeometryShapes, Index>` constraint, resulting in a type error.

## Subsets

We showed that we can recursively implement `Member` for a type-level list using the Index Trick and `Here` and `There`. This technique provides a general method for implementing traits from type-level lists. Extending the `Member` trait, we can define a `Subset` trait that checks if one type-level set is a subset of another.

To motivate such a trait, consider the following case. We want to extend our `render` functions to accept multiple shapes. For example, we want to define `render_multiple_on_geometry_os` that accepts multiple shapes consisting of circles and squares.

```rust
// OK
render_multiple_on_geometry_os(HCons(
    Circle::new(),
    HCons(Square::new(), HNil),
));
// NG (Text is not a subset of GeometryShapes)
render_multiple_on_geometry_os(HCons(Text::new(), HNil));
```

Note that `HCons` and `HNil` here are used as constructors for a type-level list. We want to check if all types in the given list are members of `GeometryShapes`.

Let us consider how to implement this. Similar to `Member`, we define a `Subset` trait like so:

```rust
fn render_multiple_on_geometry_os<L>(shapes: L)
where
    L: Subset<GeometryShapes>
{
    // ...
}
```

How can we define `Subset`? `S1` is a subset of `S2` if all elements in `S1` are members of `S2`. Therefore, `Subset<S2>` must be implemented for `S1` when all elements in `S1` are members of `S2`. This can be expressed as the following two cases:

- Base case: `HNil` is a subset of any set, so we implement `Subset<S>` on `HNil` for any `S`.
- Recursive case: For `HCons<Head, Tail>`, if `Head` is `Member<S2>` and `Tail` is `Subset<S2>`, then `HCons<Head, Tail>` is `Subset<S2>`.

If we implement this directly, we will encounter the same conflicting implementation issue as with `Member`. To work around this, we use the Index Trick. Since `Subset` involves both recursion on `Head` (for checking membership) and recursion on `Tail` (for checking the subset for the rest of the list), we need two indices. To handle this, we define a `Pair` type that combines two `Index` types.

```rust
struct Pair<Index1, Index2>(Index1, Index2);
```

`Pair` combines two `Index` values to construct a new index. We use this to define the `Subset` trait:

```rust
trait Subset<S, Index> {}

// Base case: `HNil` is a subset of any collection
impl<S> Subset<S, Here> for HNil {}

// Recursive case: `Head` is a `Member<S, Index1>` and `Tail` is a `Subset<S, Index2>`
impl<Head, Tail, S, Index1, Index2> Subset<S, Pair<Index1, Index2>> for HCons<Head, Tail>
where
    Head: Member<S, Index1>,
    Tail: Subset<S, Index2>,
{}
```

Finally, we define the `render_multiple_on_geometry_os` function to let Rust infer the Index for `Subset`, just like with `Member`.

```rust
fn render_multiple_on_geometry_os<L>(shapes: L)
where
    L: Subset<GeometryShapes>
{
    // ...
}
```

This allows the render_multiple_on_geometry_os function to accept lists composed only of geometric shapes.

When specifying trait bounds with where, multiple bounds can be specified, and type parameters can appear on the right-hand side. By leveraging this feature, we can even determine whether two collections are equal by checking subset relations in both directions.

To do something meaningful with `HList` as values (the argument of `render_multiple_on_geometry_os`), we need to constrain the types of `Head` to implement some trait so we can interact with them.

When specifying trait bounds with `where`, multiple bounds can be specified, and type parameters can appear on the right-hand side. By leveraging this feature, we can even determine whether two collections are equal by checking subset relations in both directions.

## Conclusion

In this article, we saw how to represent type collections similar to Union Types using Rust's trait resolution. By using type-level lists, we can represent collections of types, and through trait resolution, we can check whether a particular type is part of a collection or whether one collection is a subset of another. With the `Index` trick, we can perform recursive operations on type-level lists, enabling various operations such as subset checking and equality checking. This technique can be useful and gives us more expressive power in Rust and gives us opportunities to design type-safe APIs or DSLs.

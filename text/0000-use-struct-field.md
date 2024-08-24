- Feature Name: `use` struct fields
- Start Date: 2024-08-24
- RFC PR:
  [rust-lang/rfcs#0000](https://github.com/aapoalas/use-struct-field-rfc)
- Rust Issue:
  [rust-lang/rust#0000](https://github.com/aapoalas/use-struct-field-rfc)

# Summary

[summary]: #summary

Allow `use` keyword on struct fields to flatten the struct field namespace.

# Motivation

[motivation]: #motivation

Rust offers very effective ways to control the layout of structs. A struct can
contain fields that themselves are structs, and those fields can for instance be
boxed to move the inner struct's memory out of line at the cost of a single
pointer indirection. This is a very useful feature for improving the cache line
efficiency of structs. However, this transformation is trivial only when the
inner struct already exists. When starting with a struct of data the act of
moving some fields into an inner, boxed struct, all code accessing those fields
needs to be changed from acchessing the field name, to accessing first the inner
struct's field name and then the actual data field.

These sorts of refactorings are common during prototyping and performance
optimisation work. In both cases the speed of iteration is imperative, and
having to change each file where the moved fields are accessed can have a
debilitating effect. The optimal case would be that changes only happen at the
struct level: After all, how the struct's data is laid out should not really
matter to the code acessing the struct's data, as long as the data can be
accessed.

Allowing the `use` keyword on struct fields would enable this optimal case: A
`use` on a struct field brings all the public fields of the `use`'d inner
structs into the namespace of the parent struct, removing the need to make any
changes outside of the struct declaration itself.

# Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

In a game prototype we might have the following kind of struct:

```rust
struct PlayerCharacter {
    pub x: f64,
    pub y: f64,
    pub hp: u32,
    pub dead: bool,
    pub items: Vec<Item>,
}
```

We decide that we want to move the x-y position of the character into a struct
that is shared with enemy character structs. Our `PlayerCharacter` struct now
changes to:

```rust
struct Position {
    pub x: f64,
    pub y: f64,
}

struct PlayerCharacter {
    pub pos: Position,
    pub hp: u32,
    pub dead: bool,
    pub items: Vec<Item>,
}
```

Now, we need to change all code that accesses the `x` and `y` fields of the
`PlayerCharacter` struct to access `pos.x` and `pos.y` instead. This may touch a
large number of files that isn't really central or even helpful to this
refactoring. The `use` keyword becomes beneficial here:

```rust
struct PlayerCharacter {
    pub use pos: Position,
    pub hp: u32,
    pub dead: bool,
    pub items: Vec<Item>,
}
```

This makes all code that accesses `x` and `y` fields of the `PlayerCharacter`
struct to access `pos.x` and `pos.y` directly, as if the `Position` struct was
never introduced. The refactoring and the memory layout of the struct is now an
internal detail of the struct, as it should be. (Note that it is possible to
also access the `pos` field explicitly, as well as `pos.x` and `pos.y`.)

This is also useful when moving fields from one inner struct to another. When
doing performance optimisation it is sometimes useful to move some fields from
being held "inline" in the struct to being held in a boxed struct. Using the
`use` keyword, this can be done without changing any code outside of the struct
itself.

We start with the following structs:

```rust
struct Hot {
    pub a: u32,
    pub b: i32,
    pub c: u64,
}

struct Cold {
    pub x: f64,
    pub y: f64,
    pub z: f64,
}

struct DataStruct {
    pub use hot: Hot,
    pub use cold: Box<Cold>,
}
```

The fields in `Hot` and `Cold` structs are accessed through `DataStruct` through
the `use` keyword's namespace flattening. When accessing `x`, `y` or `z` fields,
the field access automatically dereferences the `Box<Cold>` pointer and accesses
the corresponding field in the `Cold` struct. Accessing the `a`, `b` or `c`
fields naturally accesses those fields in the `Hot` struct.

Now, if we want to perform a test of moving the field `c` from `Hot` to `Cold`,
we only need to remove the field from `Hot` and add it to `Cold`:

```rust
struct Hot {
    pub a: u32,
    pub b: i32,
}

struct Cold {
    pub x: f64,
    pub y: f64,
    pub z: f64,
    pub c: u64,
}
```

The `DataStruct` struct does not need to be changed at all as accessing the
field `c` on `DataStruct` is still possible, it's memory is just accessed
differently.

## Mental model

The mental model for the `use` keyword on structs is akin to how `use mod::*;`
works for imports: It brings in all public members of the `use`'d struct field
to the parent struct's namespace. It does not change the parent struct's memory
layout in any way, but is only a syntactic shorthand for inner field access.

The keyword cannot be used to smuggle private data into a more public namespace:
A `pub(crate)` field in an inner struct does not become public on the parent
struct when the parent struct declares the field as `pub use field: T`.

## Readability and maintainability of code

This feature has both positive and negative effects on readability of Rust code.
On the positive side, the `use` keyword allows for the exact memory layout and
inner structure of a struct to be abstracted away from the code that uses the
struct. This meshes well with Rust's ownership rules: A struct owns all fields
that it contains, regardless of where that field is in the "inner structure" of
the struct. If the field is found behind a boxed pointer, it only means that
memory layout of the struct is indirect while the ownership is direct; the field
is still owned by the struct. Maintainability is also positively affected:
Refactoring code is made easier as vast amounts of changes are not necessarily
needed when a single field moves.

That is not to say that everything is positive: Given a reference
`data: &DataStruct` it is clear that `data.hot.a` is not a direct field of
`data`, and it is immediately obvious how the field can be found. Referring to
`data.a` looks like a direct field access, but is not. If one were to look at
`DataStruct`'s definition they would not find a field named `a`. If the
`DataStruct` field is large and contains many fields that are `use`'d, it is not
immediately obvious where the field `a` is indeed coming from. This can be
mitigated with LSP tooltips and go-to-definition links, but it is still a real
downside.

# Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

At its core, the `use` keyword is very simple: Given a `use`'d inner struct in a
struct declaration, define syntactic sugar to access that inner struct's fields
from the parent struct directly.

## Name clashes

The `use` keyword does not allow for name clashes. Similarly to a struct cannot
contain the same field name twice, `use`'d fields that cause name clashes to
occur in the parent struct's namespace are a compile time error. This applies
both to other `use`'d fields and fields defined in the parent struct.

```rs
struct A {
    data: usize,
}
struct B {
    data: u8,
}
// error[E0000]: field `data` is already in struct namespace
struct Parent {
    use a: A,
//  -------- `data` first brought into namespace here
    use b: B,
//  ^^^^^^^^ `data` already in namespace
}
```

```rs
struct A {
    a: usize,
}

// error[E0000]: field `a` is already in struct namespace
struct Parent {
    use a: A,
//  -------- `a` first declared here
//  ^^^^^^^^ `a` already in namespace
}
```

## Privacy and publicity, `pub(self)`, `pub(super)`, `pub(crate)`, and `pub`

The `use` keyword respects the privacy of the inner struct's fields. If an inner
struct's field is `pub(self)`, it is only accessible in the module that defined
the inner struct. If the field is `pub(super)`, it is accessible in the parent
module and all child modules. If the field is `pub(crate)`, it is accessible in
the current crate. If the field is `pub`, it is accessible everywhere.

Likewise, the `use` keyword respects the privacy of the parent struct's `use`'d
field. As an example, if the `use`'d field is `pub(crate)` then the inner
struct's fields are only accessible in the current crate, unless those fields
are themselves more private.

```rs
struct Inner {
    pub(self) a: u32,
    pub(super) b: u32,
    pub(crate) c: u32,
    pub d: u32,
}

struct SelfParent {
    // All fields are only accessible in this module.
    use inner: Inner,
}

struct SuperParent {
    // `a` is only accessible in this module, `b` is accessible in parent module and all child modules.
    pub(super) use inner: Inner,
}

struct CrateParent {
    // `a` is only accessible in this module, `b` is accessible in parent module and all child modules, `c` is accessible in the current crate.
    pub(crate) use inner: Inner,
}

struct PubParent {
    // `a` is only accessible in this module, `b` is accessible in parent module and all child modules, `c` is accessible in the current crate, `d` is accessible everywhere.
    pub use inner: Inner,
}
```

## Multi-level usage

The `use` keyword can be used "recursively" to access fields of fields of
fields, as long as name clashes do not occur. The following is valid:

```rs
struct InnerMost {
    a: u32,
}

struct Inner {
    use most: InnerMost,
}

struct Parent {
    use inner: Inner,
}

fn print(parent: &Parent) {
    println!("{}", parent.a);
}
```

## Tuples

It is unclear whether tuple field access with `use` should be supported, and
whether `use`'ing tuple fields should be supported. On a conceptual level it is
very much the same, but the likelihood of collisions is 100% whenever two tuples
end up `use`'d in the same parent struct and thus the usefulness of this feature
is somewhat lesser there.

It may still be perfectly possible to support the following kind of code:

```rs
struct Inner(u32);

struct Parent {
    use inner: Inner,
}

struct Inner2 {
    data: u32,
}

struct Parent2(use Inner2);

fn print(parent: &Parent, parent2: &Parent2) {
    println!("{}, {}", parent.0, parent2.data);
}
```

## `Borrow` and `BorrowMut`

If a `use`'d field implements `Borrow` (and optionally `BorrowMut`), then the
`use`'d field can be accessed through the `Borrow` trait. This is the basis of
the `use` keyword's functionality, as the inner struct implements `Borrow` to
itself.

Using a hypothetical constant string type parameter, we can define that a
`use`'ing a field as defining a trait implementation in the following veing:

```rs
trait<const F: &'static str, const S: &'static str> Accessor: Borrow {
    type Target: ?Sized;

    fn get(&self) -> &Self::Target {
        &self[F].borrow()[S]
    }
}
```

and similarly for `BorrowMut`.

```rs
trait<const F: &'static str, const S: &'static str> AccessorMut: Accessor<F, S>, BorrowMut {
    fn get(&mut self) -> &mut Self::Target {
        &mut self[F].borrow_mut()[S]
    }
}
```

### References

References obviously implement `Borrow` and `BorrowMut`, so it is possible to
define structs like:

```rs
struct ECSCharacterRef<'a, 'b, 'c> {
    use pos: &'a Position,
    use vel: &'b Velocity,
    use status: &'c mut Status,
}
```

and then use it as if it were an object-oriented `Character` struct:

```rs
// We're channeling a struct like this:
// struct Character {
//     x: usize,
//     y: usize,
//     d_x: isize,
//     d_y: isize,
//     hp: u32,
// }

fn update_character(character: ECSCharacterRef) {
    if character.x == 0 && character.d_x < 0 {
        // Fell to the bottom of the map!
        character.hp = 0;
    }
}
```

## `Deref` and `DerefMut`

If a `use`'d field implements `Deref` (and optionally `DerefMut`), then any
public fields of the `Deref` target are brought into the parent struct's
namespace. This is the enabling condition for the `Hot` and `Cold` structs in
the example above. The `Box<Cold>` field implements `Deref` and `DerefMut` to
`Cold`, which then allows `DataStruct` to access `Cold`'s fields with the `use`
keyword.

At least the following container types benefit from this:

```rs
struct Parent<T, U, V> {
    use t: Box<T>,
    use u: Rc<U>,
    use v: Arc<V>,
}
```

Again we can use the hypothetical constant string type parameter to define these
accessors:

```rs
trait<const F: &'static str, const S: &'static str> Accessor, Deref {
    type Target: ?Sized;

    fn get(&self) -> &Self::Target {
        &self[F].deref()[S]
    }
}

trait<const F: &'static str, const S: &'static str> AccessorMut: Accessor<F, S>, DerefMut {
    fn get(&mut self) -> &mut Self::Target {
        &mut self[F].deref_mut()[S]
    }
}
```

## Both `Borrow` and `Deref`

Essentially all types implement `Borrow`, including those that implement
`Deref`. As long as no name clashes occur, it is permitted for a single `use`
keyword to bring in fields from multiple "logical" sources. As a silly example,
the following is valid:

```rs
struct Inner<T> {
    pub data: usize,
    private: Box<T>,
}

impl<T> Deref<T> for Inner<T> {
    /// Deref to the private field
}

struct Example {
    pub example: u8,
}

struct Parent {
    pub use inner: Inner<Example>,
}

fn print(parent: &Parent) {
    println!("{}, {}", parent.data, parent.example);
}
```

The fact that access to `example` goes through a `Deref` while `data` is
accessed directly on the `Inner` struct is not visible or material to the code.

## Interior mutability

Interior mutability is never available directly using a `Borrow` or `Deref`
trait. Therefore, the `use` keyword cannot be used to hide away interior
mutability. Namely, `Rc<RefCell<T>>` and `Arc<Mutex<T>>` do not dereference to
their contained `T` and therefore the following code is not valid:

```rs
struct Data {
    data: usize,
}

struct Parent {
    pub use interior: Rc<RefCell<Data>>,
}

// error[E0609] no field 'data' on type 'Parent'
fn func(parent: &Parent) {
    parent.data = 5;
//         ^^^^ unknown field
}
```

## `Vec`

`Vec<T>` is an interesting case. `Vec<T>` dereferences into `&[T]`, which is a
slice. This means that `Vec<T>` does not bring in the fields of `T` into the
namespace of the parent struct. It does bring into the namespace all the public
fields of a slice, but there are none. It would be interesting to suggest that
it should bring in the trait implementations or possibly methods of `&[T]` into
the namespace, but I will avoid that argument in this RFC. It is a potentially
useful extension of the `use` keyword, but it also seems quite hazardous and too
ambitious for this first step.

# Drawbacks

[drawbacks]: #drawbacks

Extending the `use` keyword in this way is not fully trivial, requiring new
syntax and special handling by the compiler. Alternatively, if the feature is
implemented in a generic way using blanket trait implementations then it also
requires support for const generic string parameters _and_ a way to access
struct fields by const generic string parameters.

The feature also hides some of inner-but-public details of structs, making it
less obvious where a particular field access is actually loading its data from.

# Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

## Why do this?

This feature is (from the user's point of view, at least) a small but powerful
method to enable quick iteration on code during prototyping and performance
work. It enables structuring code with the ease of object-oriented programming
(Yes, it does have its upsides!) while allowing struct memory layout to avoid
the pitfalls of object-oriented programming. This pairs well with Rust's traits:
Inheritance of interfaces is useful when it is not hamstrung by forced
structural inheritance. The `use` keyword allows Rust code to be as easy to
write as structural inheritance is without forcing actual structural
inheritance.

## Other choices

The `Parent` struct can implement getter functions for the inner struct's field.
This is fairly common pattern, but it is more verbose and requires much more
code changes when iterating on memory layouts, as not only the field's location
needs to be changed but also all of its getter functions. And if the field has
originally been a directly accessible field, changing it to be accessed through
getters is a big change requiring each access point to change.

When working with multiple reference collection structs like `ECSCharacterRef`
above, changing the location of a field may require adding and removing getters
from multiple different structs. This grinds a quick, iterative approach to a
halt very quickly.

Implementing macros for this sort of work is not very reasonable either.

An alternative syntax would reuse destructuring patterns to bring fields into
the parent namespace:

```rs
struct Parent {
    a @ Inner {
      data,
      ..
    }: Inner;
}
```

The downside here is that moving a field between the `Parent` and `Inner`, or
`Inner` and `Inner2` requires changing multiple places, and there is no
(obvious) way to explicitly bring all of the fields of `Inner` into the `Parent`
namespace even if that was what one wanted to do.

## `use` keyword

The `use` keyword is chosen based on prior art and the existing `use` keyword of
Rust having a similarity to the feature proposed here.

It is absolutely up for bikeshedding.

# Prior art

[prior-art]: #prior-art

## Jai programming language

The Jai programming language is the inspiration for this feature proposal. The
language is not open source and is not publicly available, so I cannot directly
link to any resource describing the feature there, but a
[YouTube video](https://www.youtube.com/watch?v=ZHqFrNyLlpA) describes the
feature. The keyword in Jai is `using`.

A
[Rust reddit thread](https://www.reddit.com/r/rust/comments/2t6xqz/jai_demo_dataoriented_features_soa_crazy_using/)
analyses the feature from Rust's standpoint. Opinions vary.

### Note on `using` in Jai

Jai allows `using` in function parameters and bodies. This is not proposed here.
The feature proposed here is only for struct fields. The reasoning for this is
two-fold:

1. Rust already supports this sort usage with destructuring patterns. Having two
   things do the same or similar thing is not a good thing. (See: C++)
2. Name clashes from `use` in structs are generally subject to local reasoning.

Name clashes cannot occur without `use`appearing in a struct, and if it is only
used to access fields that are defined in this module then all possible name
clashes happen because of module-level local own changes. `use`'ing a foreign
struct means that a new name clash occur when updating that foreign dependency.
This can be thought of being fairly rare, though.

But if `use` is allowed in function parameters and bodies, then its usage can
balloon massively. In these positions it is too dangerous for the convenience it
provides, and it seems like it would become a very easy footgun. Hence, I do not
propose that addition.

## Anonymous struct fields

This feature shares some similarities with the
[unnamed_fields](https://rust-lang.github.io/rfcs/2102-unnamed-fields.html) RFC
and a few
pre-RFC[1](https://internals.rust-lang.org/t/pre-rfc-unnamed-struct-types/3872)
[2](https://internals.rust-lang.org/t/pre-rfc-anonymous-struct-and-union-types/3894/25).
An anomyous struct field can be thought of having `use` automatically defined on
it. As such, this `use` proposal partially subsumes these proposals.

# Unresolved questions

[unresolved-questions]: #unresolved-questions

- Is the `use` keyword the right choice?
- Should methods and trait implementations of a `use`'d type also appear in the
  parent struct's namespace? (I think that is too abitious and prone to
  namespace clashes, but someone may offer a different opinion.)
- Should the implementation be an internal part of rustc, or would a trait based
  implementation be preferable? Is expanding the trait and const generics system
  worth it?

# Future possibilities

[future-possibilities]: #future-possibilities

## Partial `use`

The key point of the `use` keyword is refactorability, but it comes at the cost
of explicitness. It might be nice to have more control over which fields are
brought into the namespace with something like:

```rust
struct A {
    pub use::{a, b} data: Inner;
}
```

or alternatively have a way to exclude parts:

```rust
struct A {
    pub use::{self, a as _} data: Inner;
}
```

The already mentioned possibility of using destructuring expressions offers this
control, but loses the ease of refactoring.

## Method inheritance

As mentioned multiple times, it could be an option to bring methods (including
trait methods) into the namespace as well. This would enable for instance
easy-to-define custom `Vec` wrappers:

```rust
struct MyWrapper<T> {
    pub data: Vec<T>,
}

fn foo(item: &MyWrapper<usize>, index: usize) -> usize {
    item[index]
}
```

But this can very quickly lead into massive struct namespaces with multiple
overlapping trait implementations. I do not consider this a particularly great
idea.

## Custom-allocator side tables

This is not a very fully fleshed out idea, but basically: Instead of putting the
`Cold` side of a struct behind a boxed pointer, it can be put behind a
reference. The reference would point to some custom-allocated arena (or possibly
just a `Vec`) which would be used to allocate the cold side data. This would
allow not only the cold data to be "out of the way" of the hot data, but more
importantly it would still make the cold data have a reasonably good cache
locality with one another.

```rs
pub(crate) struct DataStruct<'a> {
    pub(crate) hot: Hot,
    pub(crate) use cold: &'a Cold,
}
```

But let's say we always have the arena pointer with us when we're working with a
`DataStruct`: Then we could save memory by turning `&'a Cold` into a 32-bit
offset from the arena pointer (assuming we don't need more than 4 gigabytes to
store all the cold data we have).

```rs
pub(crate) struct ColdRef<'a> {
    offset: u32,
    _marker: PhantomData<&'a Cold>,
}

pub(crate) struct DataStruct<'a> {
    pub(crate) hot: Hot,
    pub(crate) use cold: ColdRef<'a>
}
```

With this we no longer benefit from the `use` on our `cold` field: We can only
use it to access the offset but that is not very useful on its own. But now
let's say Rust gets some sort of magic `Context` parameter that can be
(optionally) passed through call stacks. Then, let's say a `DerefWithContext`
trait is added. Now we can implement that on `ColdRef` and again benefit from
the `use` keyword:

```rs
pub(crate) struct Cold {
    pub(crate) some_cold_data: usize,
}

pub(crate) struct ColdRef<'a> {
    offset: u32,
    _marker: PhantomData<&'a Cold>,
}

impl<'a> DerefWithContext<ColdContext, Cold> for ColdRef<'a> {
    fn deref<context: ColdContext>(&self) -> &'a Cold {
        context.arena.get(self.offset)
    }
}

pub(crate) struct DataStruct<'a> {
    pub(crate) hot: Hot,
    pub(crate) use cold: ColdRef<'a>
}

fn print_some_cold_context_data<context: ColdContext>(data: DataStruct<'context>) {
    println!("{}", data.some_cold_data);
}
```

This is now pretty close to an automatic Entity-Component-System. Some extra
steps (like maybe extending `use` support for `AsRefWithContext` and
`AsMutWithContext`?) are probably needed but something like this could be
imagined:

```rs
pub(crate) struct ColdRef<'a> {
    _marker: PhantomData<&'a Cold>,
}

pub(crate) struct HotRef<'a> {
    _marker: PhantomData<&'a Hot>,
}

pub(crate) struct DataRef<'a> {
    index: u32,
    pub(crate) use cold: ColdRef<'a>
    pub(crate) use hot: HotRef<'a>
}

impl AsRefWithContext<DataContext, Cold> for DataRef {
    fn as_ref<context: DataContext>(&self) -> &Cold {
        context.arena.cold.get(self.index)
    }
}

impl AsRefWithContext<DataContext, Hot> for DataRef {
    fn as_ref<context: DataContext>(&self) -> &Hot {
        context.arena.hot.get(self.index)
    }
}

pub fn use_data_ref<context: DataContext>(data: DataRef<'context>) {
    // AsRefWithContext<DataContext, T> allows accessing fields of T
    data.some_cold_data;
    data.some_hot_data;
}
```

And that would be awesome.

- Feature Name: autoreborrow-traits
- Start Date: 2025-04-15
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue:
  [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary

[summary]: #summary

Enable users to opt into automatic reborrowing of custom reference-like types.

# Motivation

[motivation]: #motivation

Reborrowing of exclusive references is an important feature in Rust, enabling
much of the borrow checker's work. This important feature is not available for
reference wrappers or user-defined reference-like types, which hampers the
usefulness of the language when working with these kinds of types. Manual
reborrowing-like functions can be implemented but they must be explicitly called
making them verbose, and are importantly always limited to creating
reference-like types bound to the local function that cannot be returned to the
parent lifetime.

Examples of features unlocked by this RFC are:

- Automatic reborrowing of `Option<&mut T>`.
- Automatic reborrowing of custom exclusive reference types like `Mut<'_, T>`.
- Returning of values deriving from reborrowed parameters, extending to the
  source lifetime.

# Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

A reference _reborrow_ is effectively a _copy_ of a pointer with extra lifetime
analysis. The lifetime of the reborrowed reference is a subset of the source
reference (not a strict subset, the reborrowed reference's lifetime is allowed
to be equal to the source reference's lifetime). For exclusive reborrows, the
source reference is disabled for the duration of the borrow. For shared
reborrows, the source reference is disabled for writes for the duration of the
borrow.

## Automatic reference reborrowing

This section gives basic Rust reference usage examples with reborrowing. This is
to serve as a comparison for later examples of autoreborrow traits usage. The
following kind of methods are assumed to exist on all reference-like types in
these examples.

```rust
impl<T, U> T {
    /// Requires exclusive access to self and returns some derived exclusive reference.
    fn mutate(self) -> U;
    /// Requires shared access to self and returns some derived shared reference.
    fn observe(self) -> U;
}
```

Note that the methods are written as taking `self` by value; because we're
talking about reference-like types, this is intended to be read as
`fn mutate(&mut self)` and `fn observe(&self)` for normal references and eg.
`fn mutate(self: MyMut)` and `fn observe(self: MyRef)` for custom reference
types.

The following examples of reborrowing will be used later:

```rust
fn mixed_reborrowing(source: &mut T) {
    // Exclusive reborrow disables source.
    let result: &mut U = source.mutate();
    // This cannot be called because of result usage keeping source disabled.
    // source.observe();
    result.observe();
    // Note: source reference can be reborrowed as shared after the exclusive
    // reborrow is dropped.
    let result1: &U = source.observe();
    // Note: source can still be reborrowed as shared while a shared reborrow
    // result is active.
    let result2: &U = source.observe();
    // Note: both shared reborrow results can be kept and used simultaneously.
    result1.observe();
    result2.observe();
    // Note: after all shared reborrow results are dropped, source can be
    // reborrowed as exclusive again.
    source.mutate();
}

fn returning_derived_mut(source: &mut T) -> &mut U {
    // Note: the reborrow that happens when calling mutate() extends its
    // lifetime to the lifetime of source, not to some local value conceptually
    // create here:
    // let reborrow = &mut *source;
    // reborrow.mutate()
    source.mutate()
}

fn returning_derived_mut(source: &mut T) -> &U {
    // Note: exclusive reference is reborrowed as shared for the return value.
    source.mutate()
}

// Just some function returning an error where both variants refer to the
// source data; think eg. a parser that wants to return a structure containing
// references to source text slice or an error pointing to invalid source text.
fn result_function(source: &T) -> Result<&X, &Y>;

fn branching_function(source: &mut T) -> Result<&U, &Y> {
    // Note: the early Err return requires the Err value's lifetime to extend
    // to the source lifetime, ie. to the end of the call. The mutate call
    // later requires the Ok value's lifetime to end before that call. The
    // compiler is currently not smart enough to split Ok and Err from one
    // another, meaning that this CANNOT be done today.
    let result = result_function(source)?;
    println!("Example: {:?}", result);
    source.mutate()
}
```

The important things to note here are:

1. There are two types of reborrowing, shared and exclusive.
2. Reborrowing happens implicitly at each
   [coercion-site](https://doc.rust-lang.org/stable/reference/type-coercions.html#coercion-sites).
3. If the function being called returns a value derived from the reborrowed
   reference, the returned value's lifetime extends in the caller's scope to
   match the original reference's lifetime.
4. Branching function returns run into issues with lifetime extension.

## Automatic wrapper type reborrowing

Wrapper types like `Option<T>` cannot be reborrowed today. This makes using
optional exclusive references tedious as they are automatically moved when used.
We'll revisit the above examples:

```rust
fn mixed_reborrowing(source: Option<&mut T>) {
    // Note: In current Rust this would work, but after this the source has
    // been moved out of this function and could no longer be used.
    let result: Option<&mut U> = source.mutate();
    result.observe();

    // We want to enable the following calls to still work: this means that
    // the first mutate() call must reborrow source instead of moving it.
    // The following calls must also implicitly reborrow source as shared.
    let result1: Option<&U> = source.observe();
    let result2: Option<&U> = source.observe();
    result1.observe();
    result2.observe();
    // The source must still be available for exclusive reborrowing afterwards.
    source.mutate();
}

fn returning_derived_mut(source: Option<&mut T>) -> Option<&mut U> {
    // This does work today, but mutate() must map() the option instead of eg.
    // using as_deref_mut().
    source.mutate()
}

fn returning_derived_mut(source: Option<&mut T>) -> Option<&U> {
    // Note: This does not work today, and suggests using as_deref() which also
    // doesn't work, citing "returns a value referencing data owned by the
    // current function". Using Option::map() to inject a coercion site fixes
    // the issue:
    // source.map(|v| -> &T { v })
    source.mutate()
}

// Just some function returning an error where both variants refer to the
// source data; think eg. a parser that wants to return a structure containing
// references to source text slice or an error pointing to invalid source text.
fn result_function(source: Option<&T>) -> Result<&X, &Y>;

fn branching_function(source: Option<&mut T>) -> Result<&U, &Y> {
    // Note: this would again move source and hence does not work.
    // let result = result_function(source)?;
    // What about manually reborrowing? No, this does not work either!
    // The Option<&T> now has a lifetime bound on the temporary as_ref()
    // borrow and cannot be returned from this function (via the Err variant)
    // as it "returns a value referencing data owned by the current function".
    let result = result_function(source.as_ref().map(|v| -> &T { v }))?;
    // Do something with the result.
    println!("Example: {:?}", result);
    // Then mutate the source again.
    Ok(source.mutate())
}
```

The important thing to note here is that while manual unwrapping and rewrapping
of an Option does help through many issues, it too fails in the branching
function returns case, though it gives a different error: the error is now
related to the lack of true reborrowing as opposed to the compiler not knowing
how to split Ok and Err branch lifetimes from one another.

## Automatic reborrowing of custom reference types

Custom reference types are as many as there are users, but a common theme
amongst them is that they usually hold one (though sometimes multiple or none)
data field and a PhantomData field (or multiple) for holding a lifetime in.
Examples from above would now look as follows. In comments ways to make these
functions compile today are given, as possible.

```rust
/// Exclusive reborrowable reference to some data. Can be turned into a MyRef
/// by reborrowing as shared.
struct MyMut<'a>(...);
impl Reborrow for MyMut { /* ... */ }

/// Shared reference to some data.
#[derive(Clone, Copy)]
struct MyRef<'a>(...);

fn mixed_reborrowing(source: MyMut) {
    // Note: mutate() takes self by value, so in current Rust MyMut be moved
    // here. Instead we want it to be reborrowed as exclusive.
    let result = source.mutate(); // source.rb_mut().mutate();
    result.observe();

    // We want to enable the following calls to still work: this means that
    // the first mutate() call must reborrow source instead of moving it.
    // The following calls must also implicitly reborrow source as shared.
    let result1: Option<&U> = source.observe(); // source.rb().observe();
    let result2: Option<&U> = source.observe(); // source.rb().observe()
    // Note: source can be reborrowed as shared multiple times.
    result1.observe();
    result2.observe();
    // The source must still be available for exclusive reborrowing afterwards.
    // Note: here we don't need to reborrow, as source is not used after this.
    source.mutate(); // or source.rb_mut().mutate();
}

fn returning_derived_mut(source: MyMut) -> MyMut {
    // This does work today, but only if source is moved into mutate() instead
    // of (explicitly) reborrowed.
    source.mutate() // NOT source.rb_mut().mutate()
}

fn returning_derived_mut(source: MyMut) -> MyRef {
    // This does not work today, but can be made to work with a helper, such as
    // the Into trait, that turns an owned MyMut into an owned MyRef.
    source.mutate() // source.mutate().into()
}

// Just some function returning an error where both variants refer to the
// source data; think eg. a parser that wants to return a structure containing
// references to source text slice or an error pointing to invalid source text.
fn result_function<'a>(source: MyRef<'a>) -> Result<MyRef<'a>, MyRef<'a>>;

fn branching_function<'a>(source: MyMut<'a>) -> Result<MyRef<'a>, MyRef<'a>> {
    // Note: here again we'd want automatic reborrowing as shared. But if we
    // explicitly reborrow source as shared, then the resulting MyRef has a
    // lifetime strictly bound to the temporary borrow of source. That is what
    // we want for the Ok value, as we must drop result before source gets
    // reused as exclusive, but for the Err value we conversely want the
    // lifetime to be extended to the source lifetime so that we can return it
    // from the function. This cannot not work today as explained above in the
    // references section.
    let result = result_function(source)?; // result_function(source.rb())?;
    // Do something with the result.
    println!("Example: {:?}", result);
    // Then mutate the source again.
    Ok(source.mutate())
}
```

A smattering of custom reference-like type usage examples are given next.

### [PeripheralRef](https://docs.rs/embassy-nrf/latest/embassy_nrf/struct.PeripheralRef.html)

- Size: Generic type dependent, but commonly ZST or 1 byte.
- Kind: Zero-sized marker type, or a small value for runtime decision making.
- Usage: Provide exclusive access to statically knowable (or mostly knowable)
  hardware peripherals in an embedded system codebase.

### [GcScope and NoGcScope](https://github.com/trynova/nova/blob/260683fccbad913491c758781965eb3bf67b531e/nova_vm/src/engine/context.rs#L40-L78)

- Size: ZST.
- Kind: Marker types with no runtime existence. `GcScope` has two lifetimes, one
  lifetime is exclusive and one is shared. `NoGcScope` has two shared lifetimes.
  The always-shared lifetime takes no part in reborrowing and can be ignored
  here.
- Usage: `GcScope` provides exclusive access to a garbage collector, while its
  shared reborrow `NoGcScope` is used to create handles to garbage collected
  data. Handles are thus invalidated if `GcScope` is used as exclusive.

### [MutableHandle](https://doc.servo.org/mozjs/gc/struct.MutableHandle.html) and [Handle](https://doc.servo.org/mozjs/gc/struct.Handle.html)

- Size: Pointer-sized.
- Kind: `MutableHandle` is conceptually an exclusive reference type. `Handle` is
  a wrapper around a shared reference.
- Usage: `MutableHandle` is a reference type wrapping a raw pointer that can
  unsafely be turned into an exclusive reference on the condition that the
  garbage collector does not run while the exclusive reference exists. `Handle`
  is a plain shared reference wrapper; the pointed-to value is (aspirationally)
  only mutated by the GC where it is internally mutable.

### [Mat](https://docs.rs/faer/latest/faer/mat/struct.Mat.html), [MatMut](https://docs.rs/faer/latest/faer/mat/struct.MatMut.html), [MatRef](https://docs.rs/faer/latest/faer/mat/struct.MatRef.html), [ColMut](https://docs.rs/faer/latest/faer/col/struct.ColMut.html), and [ColRef](https://docs.rs/faer/latest/faer/col/struct.ColRef.html)

`Mat` is an owned rectangular matrix while `MatMut`, `MatRef`, `ColMut`, and
`ColRef` are various kinds of custom reference types over the source `Mat`.

- Size: Usually 5 pointer-sizes for `MatMut` and `MatRef`, 3 for `ColMut` and
  `ColRef`.
- Kind: `MatMut` and `ColMut` are exclusive reference types. `MatRef` and
  `ColRef` are shared reference types.
- Usage: Enable various near-zero-cost operations over matrices by binding the
  iteration to the original `Mat` through the reference types while statically
  ensuring that data access within the bounds of the borrow is valid.

Note: `Mat` is also reborrowable into `MatMut` and `MatRef`, but also has a
`Drop` implementation. If reborrowing is understood to act on lifetimes of
reference-like types, then reborrowing `Mat` is outside that definition as it
neither has a lifetime nor is a reference-like type. It could be argued that
`Mat` does have an implied `'static` lifetime, but the lack reference-like
aspects of it at least cannot be denied. This use-case might be better thought
of as a generalised `AsRef/AsMut` or perhaps `Deref[Mut]` implementation.

### [reborrow crate](https://docs.rs/reborrow/0.5.5/reborrow/index.html)

This is a library used in eg. faer (above `Mat` and friends) that provides
traits for emulating reborrowing.

### Honorable mention: [CppRef and friends](https://docs.rs/autocxx/latest/autocxx/struct.CppRef.html)

These types do not implement reborrowing and do not always carry lifetimes, but
cxx is interested in automatic coercion between the different C++ reference
types. Since these are pointer wrapping types, some forms of autoreborrowing
would make these coercions possible.

# Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

Reborrowing itself should be a fairly direct and "easy-to-grasp" feature in how
it works; reference wrappers and custom reference types should be able to opt-in
to exclusive reference-like reborrowing. This means that:

1. When such a type is used at a coercion site, the type is implicitly
   reborrowed as either exclusively or as shared, depending on the target type
   required by the site.
2. A copy of the source type is produced and converted into the target type.
3. Lifetime extension is applied to the reborrowed lifetime as necessary.

A reborrowable type is declared to the compiler using the following methodless
trait: note that implementing the trait makes a type reborrowable both
exclusively and as shared, and it is the target type of a coercion site that
implicitly decides which type of reborrowing is performed.

```rust
trait Reborrow<'a>: !Clone + !Drop {
    type Ref: Copy;
}
```

Note: the `!Drop` requirement isn't exactly correct; reborrowable types must not
implement `Drop` or return `true` for `std::mem::needs_drop`. The `Copy` bound
on `Borrow::Ref` is a logical result of shared reborrowing: the source type can
be reborrowed as shared multiple times and each shared reborrow should produce
logically interchangeable target types (order or number of reborrows should not
matter); hence reborrowing twice as shared should be equal to reborrowing once
as shared and copying the result.

These trait would be implemented in the following way:

```rust
impl<'a> Reborrow<'a> for MatMut<'a> {
    type Ref = MatRef<'a>;
}
```

or for types with multiple lifetimes:

```rust
impl<'a, 'b> Reborrow<'a> for GcScope<'a, 'b> {
    type Ref = NoGcScope<'a, 'b>;
}
```

Implementing `Reborrow<'a> for T<'a> { type Ref; }` means that

1. At coercion sites taking `T` by value, the source `T` is copied via an
   exclusive reference (exclusive access is asserted) and its `'a` lifetime is
   replaced with a reborrowed lifetime on which normal lifetime extension rules
   can be applied. While the reborrowed lifetime lives, the source `T` is
   disabled.
2. At coercion sites taking `T::Ref` by value, the source `T` is copied via a
   shared reference (shared access is asserted) and the copy converted into
   `T::Ref`. The `'a` lifetime in `T::Ref`, if found, is replaced with a
   reborrowed lifetime as above. While the reborrowed lifetime lives, the source
   `T` is disabled for exclusive access.

# Drawbacks

[drawbacks]: #drawbacks

The main drawback of autoreborrowing is how to convert the `T` to `T::Ref`: the
most obvious way to do this would be with an `fn reborrow` method on the
`Reborrow` trait, but this opens up a world where each coercion site may
implicitly inject a call to user-code. For this reason the `Reborrow` traits
both lack methods entirely, as we'd prefer for reborrowing to be devoid of any
user-code. But if user-code is not involved in the conversion, then the compiler
is forced to perform the conversion on its own.

The best-case for compiler-injected conversion would be the
[safe transmute project](https://rust-lang.github.io/rfcs/2835-project-safe-transmute.html)'s
mechanisms, but those can only detect grievous errors like transmuting padding
bytes to data. They would not detect logical errors, such as swapping two
`usize` fields with one another (think swapping `length` and `cap` in `Vec`).

If the `Reborrow` trait relies on compiler-injected transmutes, then it should
be marked `unsafe`.

The second drawback of autoreborrowing is that it of course requires a generic
implementation of reborrowing relying on the `Reborrow` trait in rustc. A
version of reference wrapper autoreborrowing already exists in rustc for
`Pin<&mut T>`, so the basic building blocks needed for autoreborrowing are
already present.

# Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

## Rationale

The rationale for choosing a single trait with a single target GAT and no
methods is as follows:

- Reborrowing only makes sense for exclusive reference types: reborrowing of
  shared reference types as shared is exactly equal to copying the shared
  reference type.
- An exclusive reference should only really be exclusively reborrowable into
  itself, ie. `&mut T -> &mut T` or `MatMut<'_> -> MatMut<'_>`. a
  `&mut T -> &mut U` should be thought of as a reborrow followed by a
  dereference. Hence, a separate `Reborrow::Mut` GAT is not needed.
- An exclusive reference type that shouldn't be reborrowable as shared may
  exist: the proposed `Reborrow` trait does not support this directly, but it
  can implemented by creating a dummy type (with same layout as the exclusive
  reference) which is never used and has no methods on it. The exclusive
  reference type will thus be reborrowable as shared, but into a type that
  cannot be used for anything.
- Adding a method to reborrowing would make all coercion sites into implicit
  injection points into user-code. This seems both too powerful and too magical
  to allow, hence a no method is suggested.

### Compiler-level reasoning around custom type autoreborrowing

The exclusive reborrowing is fairly straightforward as the only complexity lies
in implementing the lifetime replacement and extension, and the disabling of the
source `T`. The shared reborrowing is less so, as it requires the compiler to
convert `T` into `<T as Reborrow>::Ref`. The usual case for reborrowable types
is that they have only one (or no) data field of the same kind for both the
exclusive and shared reference types, meaning that a transmute would be
trivially sound (at least for `#[repr(transparent)]` types), but for multi-field
types the compiler would need to reason about the transmute safety. The basic
reasoning mechanism for this is provided by the
[safe transmute project](https://rust-lang.github.io/rfcs/2835-project-safe-transmute.html),
which can detect grievous mistakes like transmuting padding bytes into data.
More complex reasoning, like making sure that two `usize` fields do not
logically flip in the transmute, cannot be performed by the compiler and would
require `#[repr(C)]` usage on both sides of the transmute. As this requires
manual checking by the user of the `Reborrow` trait, the trait might best be
marked as `unsafe`.

### Mitigating the unsafety of an injected transmute

As the transmuting of `T` to `T::Ref` is unsafe and requires manual checking,
we'd like to avoid the injected transmute. The obvious thing to do would be to
add a method `fn reborrow(self) -> T::Ref` (note: the method must take self by
value, not by reference) to the `Reborrow` method but as discussed above, this
is not suggested because it would mean injecting implicit user-controlled code
with arbitrary side-effects to every coercion site. This runs counter to what
reborrowing is generally though of being, "copy of a pointer with extra lifetime
analysis", and we'd thus want to avoid it. (Contrast this with adding an
`fn copy()` method to the `Copy` trait.)

With the safe transmute project, the implicit transmute could be improved by
requiring a `type Ref: TransmuteFrom<Self, _>` trait bound on the GAT: with this
the source and target would at least be confirmed to be safely transmutable.
This would not guarantee that the transmute is sound, however. Eg. an mixup of
the declaration order of two pointer fields would still be catastrophic.

An alternative to an explicit `reborrow` method would be to add a
`type Ref: From<T>` bound to the GAT and have the compiler inject a
`<T as Reborrow>::Ref::from(T)` call at the reborrow site. The downside here is
that the `From` impl is still user-controlled code and could likewise have
arbitrary side-effects. That being said, `From` trait implementations are well
understood (as compared to a custom `reborrow` method, or even eg. `Deref` vs
`AsRef` vs `Borrow`) and held to a higher standard; they are unlikely to contain
side-effectful code.

Finally, in the future if/when `const fn`s in traits become available,
improvements on the above method-based approaches appear: the custom `reborrow`
method could be made into a `const fn reborrow()` or, if trait bounds requiring
`const` implementations of traits or trait methods become possible, the `From`
bound could be turned into `Target: const From<Self>::from`. With this the fear
of entirely arbitrary side-effects is mitigated, though not entirely removed.

With any of these mitigations in place, the `Reborrow` trait could arguably be
implemented without requiring an `unsafe` marking.

### The reborrowed lifetime

In the above reborrow formulation, the `Reborrow` trait itself has a lifetime
parameter that the compiler uses to detect which lifetime is being reborrowed,
in case there are multiple (or none). This seems like a neat solution, but if it
is deemed problematic then an alternative would be to require that reborrowable
types and their `Ref` targets have exactly one lifetime.

## Alternatives

### Alternative 1: `Reborrow` trait with a GAT and method

Implement trait or traits in the vein of the `reborrow` crate:

```rust
/// Immutable reborrowing.
trait Reborrow<'short, _Outlives = &'short Self> {
    type Target;

    #[must_use]
    fn reborrow(&'short self) -> Self::Target;
}

/// Mutable reborrowing.
trait ReborrowMut<'short, _Outlives = &'short Self> {
    type Target;

    #[must_use]
    fn reborrow_mut(&'short mut self) -> Self::Target;
}
```

The main benefit with this approach is that this follows the current userland
state-of-the-art way of creating a facsimile of reborrowing and would
effectively bless a particular, well-liked userland implementation into the
standard library together with compiler support for automatically injecting the
trait method calls as opposed to the current userland best-case scenario where
the calls must be written out manually.

The main issue with this approach is that this does not implement true
reborrowing: the `'short` lifetime gets bound to the temporary borrow created
when calling these methods, meaning that lifetime extension does not work out of
the box. The `'short` lifetime is a strict subset of the source lifetime (call
it `'long`), instead of being able to extend to equal it. To overcome this
issue, rustc would need to special-case this trait implementation to allow
lifetime extension of `'short` into `'long`.

A secondary issue is that the implementation of these methods is
user-controlled, and users are not known to always fully live by the spirit of
the trait if they find a neat way to reinterpret its words. As rustc would have
to implicitly inject these method calls in various locations, some enterprising
users might find clever tricks they can perform with these extra control points
they receive. Especially if rustc were to special-case the trait implementation
to allow lifetime extension of `'short` into `'long` unconditionally, it would
lead to the trait being unsound!

As an example, while in a purely reborrow frame of mind turning
`&'short Self<'long>` into `Self<'short>` and then allowing that `Self<'short>`
to lifetime extend to be up to or equal to `'long` seems sound, the fact of the
matter is that it isn't: the `&Self` reference is a pointer onto the stack (most
likely) and allowing lifetime extension from `Self<'short>` to `Self<'long>`
would mean that rustc would allow the `&Self` reference to escape its call
stack. This is equal to returning a pointer pointing to released stack memory,
ie. undefined behaviour.

Thus, in current Rust syntax a `Reborrow` trait taking `self` by reference but
allowing lifetime extension would be unsound. To solve this issue, a new syntax
for taking a value by reference that cannot be used as a place or turned into an
address would be needed, perhaps something like:

```rust
trait Reborrow<'short, _Outlives = &'short Self> {
    type Target;
    fn reborrow(^'short self) -> Self::Target;
}

trait ReborrowMut<'short, _Outlives = &'short Self> {
    type Target;
    fn reborrow_mut(^'short mut self) -> Self;
}
```

As this sort of a reference would be something entirely new, it was deemed too
complicated a change for this RFC.

### Alternative: `Reborrow` trait with a generic type parameter and method

Implementing the above trait(s) with a type parameter would be another option,
though it would probably require more rustc work as well. The same issues as
above apply equally but it has a further issue lurking within it as well: a
`Reborrow` trait that can be implemented multiple times over gives users the
power needed to implement eg. automatic integer coercion. You need only to
implement `Reborrow<uN> for MyInt` and `Reborrow<iN> for MyInt` for all `N` of
your preference and now your `MyInt` type will be reborrow-coerced into whatever
integer type is needed by a given call.

For this reason, this RFC does not support allowing multiple `Reborrow` or
`ReborrowMut` implementations.

### Alternative: `Reborrow` trait with GAT, methods, and explicit usage

A variant on the above, if user-land reborrowing was made explicit in the source
code then the compiler would not need to implicitly inject user-controlled code
at potentially every coercion site. Instead, an explicit syntax would be present
in user-code in these locations.

The
[ergonomic clones](https://github.com/rust-lang/rust-project-goals/issues/107)
feature has added a `foo.use` suffix operator for calling `clone()` in an
ergonomic way. A similar thing could be done for reborrowing using eg. the `ref`
and `mut` keywords:

```rust
fn mixed_reborrowing(source: MyMut) {
    let result = source.mut.mutate();
    result.ref.observe();
    let result1 = source.ref.observe();
    let result2 = source.ref.observe();
    result1.ref.observe();
    result2.ref.observe();
    source.mut.mutate();
}
```

This would be a fairly neat solution to the issue of autoreborrowing being used
to do scary implicit type conversions or such. However, this does not solve the
problem of lifetime extension in any way.

Additionally, this syntax might be better used for other purposes, such as
generalised `Deref` / `DerefMut` or `AsRef` / `AsMut` calls.

## Related efforts

### Generalised `Deref` and `DerefMut` traits

The `Deref` and `DerefMut` traits are quite magical and powerful, but they are
not quite powerful enough to implement the user-land fascimile of reborrowing
because they must always return references. A generalised `GeneralDeref(Mut)`
trait would be one where the return value is the `Target` GAT directly and not a
reference of it. These traits would be strong enough to replace the `reborrow`
crate and would have the benefit that a generalised `Deref` trait enables
passing reborrowable parameters with a simple `&my_ref` or `&mut my_mut`. This
would also be visible in the source code (usually) and thus allowing the trait
methods to be user-implemented would not be as scary, compared to implicitly
inserted user-implemented code.

Again, these traits on their own could not solve the lifetime extension issue
and absolutely should not: a `Deref` target is generally not allowed to escape
its call site, eg. dereferencing `Box<T>` into `&T` must yield a reference that
cannot be returned from the function that performs the dereferencing. Any
unconditional lifetime extension on these methods would be unsound.

### Reborrowing of reference counted references

The standard library's `RefCell<T>` gives out `Ref<'a, T>` and `RefMut<'a, T>`
custom reference types; each of these carries with it a pointer to the reference
counted data. `Ref`s are `Clone` and increment the `RefCell`'s ref counter,
while `RefMut` have an internal `clone` method that gets used in eg. the
`RefMut::map_split` method and increments the `RefCell`'s mut counter. Both
types then have `Drop` impls that decrement their respective counters again at
the end of their lifetimes. These types of references are not reborrowable in
the traditional sense as they need to run custom code during reborrowing, but
they are reborrowable in the sense that a `Ref` or `RefMut` created based on an
earlier `RefMut` should be allowed to outlive its source.

eg. Given the following kind of code:

```rust
fn example() {
    let rc = RefCell::new(0u32);
    let result = {
        let ref_mut = rc.borrow_mut(); // mut counter +1
        {
            let (ref_mut_1, ref_mut_2) = ref_mut.map_split(...); // mut counter +1
            drop(ref_mut_1); // mut counter -1
            ref_mut_2
        }
    };
    println!("ref_mut_2: {:?}", result);
    // mut counter -1
}
```

On the type level, the `ref_mut` is moved out of in `map_split`, meaning that
`ref_mut_1` and `ref_mut_2` are both dependent on a borrow that, in a sense,
dropped already. Despite this, they can not only be used but also returned from
their current scope and moved to parent scopes up until the scope where the
originating `RefCell` is.

This is very similar to how reborrowed references can use lifetime extension to
return to their "true source of origin" despite being borrowed deep inside a
call graph. If `RefMut`'s clone method were to be rewritten in terms of a
`Reborrow` trait with a method incrementing the mut counter, and if the
`Reborrow` trait could be extended to allow types containing data with `Drop`
impls, then the `RefMut::map_split` could be generalised further.

There is a complication to this: since the `borrow: BorrowRefMut` field of
`RefMut` implements `Drop`, the above code could see three drop calls instead of
the current two. Right now `ref_mut` is never dropped as it is moved out of and
into `map_split` but if that move was made into a reborrow, then an extra
reborrow-injected clone would happen at the `map_split` call site and an extra
drop would happen when the scope in which `ref_mut` was create is left. As an
optimisation, the compiler could choose to only perform a reborrow when a move
isn't possible, but that might be very complicated to implement.

# Prior art

[prior-art]: #prior-art

- Formalise reborrows RFC:
  [rust-lang#2364](https://github.com/rust-lang/rfcs/pull/2364)
- Feature request to simulate exclusive reborrows in user code:
  [rust-lang#1403](https://github.com/rust-lang/rfcs/issues/1403)
- Experimental support for Pin reborrowing:
  [rustc#130526](https://github.com/rust-lang/rust/pull/130526)

# Unresolved questions

[unresolved-questions]: #unresolved-questions

No direct unresolved questions at this moment.

# Future possibilities

[future-possibilities]: #future-possibilities

It seems possible that autoreborrowing will open wide the doors for `AsRef`,
`Deref`, `Borrow`, and similar traits for re-examination and extension in a
world where custom reference types are commonplace. For some custom reference
types, any generalisations of these methods will necessarily require a context
pointer (so that eg. dereferencing of handles can be performed).

It is unclear whether these sorts of traits deserve to exist in the standard or
core library, or if they would better be left as user-land implementations.

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

Reborrowing of shared references and exclusive references is an important
feature in Rust, enabling much of the borrow checker's work. This important
feature is not available for reference wrappers or user-defined reference-like
types, which hampers the usefulness of language when working with these kinds of
types. Manual reborrowing-like functions can be implemented but they must be
explicitly called making them verbose, and are importantly always limited to
creating local sub-lifetimes that cannot expand to the parent lifetime.

Examples of features unlocked by this RFC are:

- Automatic reborrowing of `Option<&mut T>`.
- Automatic reborrowing of custom reference types like `Ref<'_, T>` and
  `Mut<'_, T>`.
- Returning of values deriving from reborrowed parameters, expanding to the
  source lifetime.

# Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

A reference _reborrow_ is essentially a _copy_ of a pointer with extra lifetime
analysis. The lifetime of the reborrowed reference is a subset of the source
reference, including any lifetime expansion. For exclusive references, the
reborrowed reference disables the source reference for the duration of its own
lifetime.

## Automatic reference reborrowing

This section gives basic Rust reference usage examples with reborrowing. This is
to serve as a comparison for later examples of autoreborrow traits usage. The
following kind of methods are assumed to exist on all reference-like types in
these examples.

```rust
impl<T, U> T {
    /// Requires exclusive access to self and returns some derived exlusive reference.
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
fn mixed_reborrowing(soure: &mut T) {
    let result: &mut U = source.mutate();
    // This cannot be called because of result usage keeping source disabled.
    // source.observe();
    result.observe();
    // Note: exclusive reference can be reborrowed as shared.
    let result1: &U = source.observe();
    // Note: the same exclusive reference can again be reborrowed as shared
    // while a shared reborrow result is still active.
    let result2: &U = source.observe();
    result1.observe();
    result2.observe();
    source.mutate();
}

fn returning_derived_mut(source: &mut T) -> &mut U {
    // Note: the reborrow that happens when calling mutate() expands its
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
```

The important things to note here are:

1. There are two types of reborrowing, shared and exclusive.
2. Only exclusive references can be reborrowed exclusively.
3. Reborrowing happens implicitly at each call-site.
4. If the function being called returns a value derived from the reborrowed
   reference, the returned value's lifetime expands in the caller's scope to
   match the original reference's lifetime.

## Automatic wrapper type reborrowing

Wrapper types like `Option<T>` cannot be reborrowed today. This makes using
optional exclusive references tedious as they are automatically moved when used.
We'll revisit the above examples:

```rust
fn mixed_reborrowing(soure: Option<&mut T>) {
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
    // The source must still be available for exlusive reborrowing afterwards.
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
    // current function". Using map() and &* to explicitly reborrow the
    // contained reference does work.
    source.mutate()
}
```

## Automatic reborrowing of custom reference types

Custom reference types are as many as there are users, but a common refrain
amongst them is that they usually hold one (though sometimes multiple) data
fields and a PhantomData field for holding a lifetime in. Examples from above
would now look like this:

```rust
/// Exclusive reborrowable reference to some data. Can be turned into a MyRef
/// by reborrowing as shared.
struct MyMut<'a>(...);
/// Shared reborrowable reference to some data.
struct MyRef<'a>(...);

fn mixed_reborrowing(soure: MyMut) {
    // Note: mutate() takes self by value, so MyMut would currently be moved
    // here.
    let result = source.mutate();
    result.observe();

    // We want to enable the following calls to still work: this means that
    // the first mutate() call must reborrow source instead of moving it.
    // The following calls must also implicitly reborrow source as shared.
    let result1: Option<&U> = source.observe();
    let result2: Option<&U> = source.observe();
    result1.observe();
    result2.observe();
    // The source must still be available for exlusive reborrowing afterwards.
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
    // current function". Using map() and &* to explicitly reborrow the
    // contained reference does work.
    source.mutate()
}
```

A smattering of examples are:

### [PeripheralRef](https://docs.rs/embassy-nrf/latest/embassy_nrf/struct.PeripheralRef.html)

- Size: Generic type dependent, but commonly ZST or 1 byte.
- Kind: Zero-sized marker type, or a small value for runtime decision making.
  The reference type is exclusive.
- Usage: Provide exclusive access to statically knowable (or mostly knowable)
  hardware peripherals in an embedded system codebase.

### [GcScope and NoGcScope](https://github.com/trynova/nova/blob/260683fccbad913491c758781965eb3bf67b531e/nova_vm/src/engine/context.rs#L40-L78)

- Size: ZST.
- Kind: Marker types with no runtime existence. `GcScope` has two lifetimes, one
  lifetime is exclusive and one is shared. `NoGcScope` has two shared lifetimes.
  The always-shared lifetime is not very relevant to reborrowing and can be
  ignored here.
- Usage: `GcScope` provides exclusive access to a garbage collector, while its
  shared reborrow `NoGcScope` is used to create handles to garbage collected
  data. Handles are thus invalidated if `GcScope` is reborrowed as exclusive (or
  moved).

### [MutableHandle](https://doc.servo.org/mozjs/gc/struct.MutableHandle.html) and [Handle](https://doc.servo.org/mozjs/gc/struct.Handle.html)

- Size: Pointer-sized.
- Kind: `MutableHandle` is conceptually an exclusive reference type. `Handle` is
  a wrapper around a shared reference.
- Usage: `MutableHandle` is a reference type wrapping a raw pointer that can
  unsafely be turned into an exclusive reference on the condition that the
  garbage collector does not run while the exclusive reference exists. `Handle`
  is a plain shared reference wrapper; the pointed-to value is either a pointer
  to garbage collected data, is internally mutable, or is never mutated by the
  garbage collector. (Or so I assume.)

### [Mat](https://docs.rs/faer/latest/faer/mat/struct.Mat.html), [MatMut](https://docs.rs/faer/latest/faer/mat/struct.MatMut.html), [MatRef](https://docs.rs/faer/latest/faer/mat/struct.MatRef.html), [ColMut](https://docs.rs/faer/latest/faer/col/struct.ColMut.html), and [ColRef](https://docs.rs/faer/latest/faer/col/struct.ColRef.html)

`Mat` is an owned rectangular matrix while `MatMut`, `MatRef`, `ColMut`, and
`ColRef` are various kinds of custom reference types over the source `Mat`.

- Size: Usually 5 pointer-sizes for `MatMut` and `MatRef`, 3 for `ColMut` and
  `ColRef`.
- Kind: `MatMut` and `ColMut` are exclusive reference types. `MatRef` and
  `ColRef` are shared reference types.
- Usage: Enable various near-zero-cost operations over matrices by binding the
  iteration to the original `Mat` through the reference types while statically
  ensuring that indexed access is not required.

Note: `Mat` is also reborrowable into `MatMut` and `MatRef`, but also has a
`Drop` implementation. If reborrowing is understood to act on lifetimes of
reference-like types, then reborrowing `Mat` is outside that definition as it
neither has a lifetime nor is a reference-like type. It could be argued that
`Mat` does have an implied `'static` lifetime, but the lack reference-like
aspects of it at least cannot be denied. It might be that this use-case would be
better understood as a generalised `AsRef` implementation.

### [reborrow crate](https://docs.rs/reborrow/0.5.5/reborrow/index.html)

This is a library used in eg. faer (above `Mat` and friends) that provides
traits for emulating reborrowing.

### Honorable mention: [CppRef and friends](https://docs.rs/autocxx/latest/autocxx/struct.CppRef.html)

These types do not implement reborrowing, but cxx is interested in automatic
coercion between the different C++ reference types. Since these are pointer
wrapping types, some forms of autoreborrowing would make these coercions
possible.

## A note on `Reborrow` and `Copy`

When dealing with shared references, reborrowing is not actually necessary. A
shared reference ought to be `Copy` by virtue of being shareable, so for them
implementing `Reborrow` is mostly unnecessary. The only reason that the
`Reborrow` trait exists alongside `ReborrowMut` is because of the need to coerce
exclusive references into shared references.

# Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

Reborrowing itself should be a fairly direct and "easy-to-grasp" feature in how
it works; reference wrappers and custom reference types should be able to opt-in
to reference-like reborrowing. This means that:

1. When such a type is used as a function parameter, the type is implicitly
   reborrowed.
2. When such a type is used as a function parameter and a value bound to the
   reborrowed type is returned, the lifetime is automatically expanded to the
   calling context.
3. When such a type `T` is returned from a function that returns a different
   type `U` that `T` can be reborrowed into, the type is implicitly reborrowed.

The implementation of reborrowing, however, is not an immediately obvious
question. At least three main possibilities exist:

## `Reborrow` and `ReborrowMut` traits with a GAT and methods

Implement traits in the vein of the `reborrow` crate:

```rust
/// Immutable reborrowing.
trait Reborrow<'short, _Outlives = &'short Self> {
    type Target;

    #[must_use]
    fn rb(&'short self) -> Self::Target;
}

/// Mutable reborrowing.
trait ReborrowMut<'short, _Outlives = &'short Self> {
    type Target;

    #[must_use]
    fn rb_mut(&'short mut self) -> Self::Target;
}
```

The main issue with this approach is that this does not work as-is. The `'short`
lifetime gets bound to the temporary borrow created when calling these methods,
meaning that lifetime expansion does not work when eg. returning values. To
overcome this issue, rustc would need to special-case this trait implementation
to allow lifetime expansion to work. Alternatively, new syntax would be needed
that allows for lifetime expansion.

A secondary issue is that the implementation of these methods is
user-controlled, and users are not known to always fully live by the spirit of
the trait if they find a neat way to reintepret its words. As rustc would have
to implicitly inject these method calls in various locations, some enterprising
users might find clever tricks they can perform with these extra control points
they receive. Especially if rustc were then to special-case the trait
implementation to allow lifetime expansion, it might perhaps even lead to
unsoundness with interesting enough user implementations.

### Alternative: `Reborrow` and `ReborrowMut` traits with a generic type parameter and methods

Implementing the above traits with a type parameter would be another option,
though it would probably require more rustc work as well. This choice has a
worse issue lurking within it, though: a `Reborrow` trait that can be
implemented multiple times over gives users the power needed to implement eg.
automatic integer coercion. You need only to implement `Reborrow<uN> for MyInt`
and `Reborrow<iN> for MyInt` and now your `MyInt` type will be reborrow-coerced
into whatever integer type is needed by a given call.

In short, this RFC does not support allowing multiple `Reborrow` and
`ReborrowMut` implementations.

## `Reborrow` and `ReborrowMut` traits with GAT, methods, and explicit usage

A variant on the above, if user-land reborrowing was made explicit in the source
code then the secondary issue of the compiler implicitly injecting
user-controlled code in various locations would be replaced with extra syntax
being explicitly injected at various locations.

The
[ergonomic clones](https://github.com/rust-lang/rust-project-goals/issues/107)
feature has added a `foo.use` suffix operator for calling `clone()` in an
ergonomic way. A similar thing could be done for reborrowing using eg. the `ref`
and `mut` keywords:

```rust
fn mixed_reborrowing(soure: MyMut) {
    let result = source.mut.mutate();
    result.ref.observe();
    let result1 = source.ref.observe();
    let result2 = source.ref.observe();
    result1.ref.observe();
    result2.ref.observe();
    source.mut.mutate();
}
```

This would be a fairly neat solution to the possible issue of autoreborrowing
being used to do scary implicit type conversions or such. However, this does not
solve the problem of lifetime expansion, as the syntax to define or declare the
lifetime expansion in the `Reborrow` and `ReborrowMut` trait methods does not
exist.

## Generalised `Deref` and `DerefMut` traits

The `Deref` and `DerefMut` traits are quite magical and powerful, but they are
not quite powerful enough to implement reborrowing because they must always
return references. A generalised `GeneralDeref(Mut)` trait would be one where
the return value is the `Target` GAT directly and not a reference of it. These
traits would be strong enough to replace the `reborrow` crate and would have the
benefit that a generalised `Deref` trait enables passing reborrowable parameters
with a simple `&my_ref` or `&mut my_mut`. This would also be visible in the
source code (usually) and thus allowing the trait methods to be user-implemented
would not be as scary, compared to implicitly inserted user-implemented code.

Unfortunately, these traits on their own could not solve the lifetime expansion
issue. Specialisation in rustc for these traits would still be needed, or a new
kind of syntax for expressing this expansion.

## `Reborrow` and `ReborrowMut` traits with GAT but without methods

An alternative to implicitly injected, user-controlled reborrow method calls
would be to implement the traits as special, method-less traits:

```rust
/// Immutable reborrowing.
trait Reborrow<'a> {
    type Target;
}

/// Mutable reborrowing.
trait ReborrowMut<'a> {
    type Target;
}
```

The compiler would then automatically implement the type conversion between
`Self` and `Target` using a transmute (with at least basic validity checks?).
Alternatively, a type bound of `Target: From<Self>` could be placed on the GAT
which the compiler could then use to perform the type conversion. The downside
here would of course be that this would again be user-controlled code, but the
`From` trait is at least quite well understood and its usage held to a fairly
high standard generally.

The `'a` lifetime on the trait would be the one that the compiler would replace
with the "temporary lifetime" of the reborrow. Alternatively, the compiler could
just require that reborrowable types have exactly one (or at most one) lifetime.

# Drawbacks

[drawbacks]: #drawbacks

The main drawback of autoreborrowing is that either it needs some kind of
special casing in rustc, or it requires a whole new syntax extension.

The special casing, however, already exists in limited circumstances and as such
it seems like the drawbacks are small or insignificant indeed.

# Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

Above four alternatives for autoreborrowing have been suggested. This RFC
proposes to use the GAT-but-method-less implementation. It most closely matches
the natural language definition and prior art of reborrowing being a "copy of a
pointer with extra lifetime analysis".

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
`Deref`, `Borrow`, and similar traits for re-examination and expansion in a
world where custom reference types are commonplace. For some custom reference
types, any generalisations of these methods will necessarily require a context
pointer (so that eg. dereferencing of handles can be performed).

It is unclear whether these sorts of traits deserve to exist in the standard or
core library, or if they would better be left as user-land implementations.

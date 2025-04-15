- Feature Name: autoreborrow-traits
- Start Date: 2025-04-15
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue:
  [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary

[summary]: #summary

Enable users to opt into automatic reborrowing of custom types with exclusive
reference semantics.

# Motivation

[motivation]: #motivation

Reborrowing of exclusive references is an important feature in Rust, enabling
much of the borrow checker's work. This important feature is not available for
reference wrappers or user-defined reference-like types, which hampers the
usefulness of the language when working with these kinds of types. Manual
reborrow-like functions can be implemented but they must be explicitly called
making them verbose, and are importantly always limited to creating
reference-like types bound to the local function that cannot be returned to the
caller.

Examples of features unlocked by this RFC are:

- Automatic reborrowing of `Option<&mut T>` and `Pin<&mut T>`.
- Automatic reborrowing of custom exclusive reference types like `Mut<'_, T>`.
- Returning of values deriving from reborrowed parameters, extending to the
  source lifetime.

# Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

A _reborrow_ is a bitwise _copy_ of a type with extra lifetime analysis
performed by the compiler. The lifetime of the reborrow result is a subset of
the source value's lifetime (not a strict subset, the result's lifetime is
allowed to be equal to the source's lifetime). When reborrowing as exclusive,
the source value is disabled for the lifetime of the result. When reborrowing as
shared, the source value is disabled for mutations for the lifetime of the
result.

## Status quo - reference reborrowing

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
    {
        let result: &mut U = source.mutate();
        // The following line would not compile because source is disabled for
        // the duration of the result's lifetime:
        // source.observe();

        result.observe();
    }
    // Note: source can be reborrowed as shared or exclusive after the
    // previous exclusive reborrow is dropped.

    // Shared reborrow disables source for mutations. Source can be reborrowed
    // as shared multiple times.
    {
        let result1: &U = source.observe();
        let result2: &U = source.observe();
        // The following line would not compile because source is disabled
        // for mutations for the duration of the results' lifetime:
        // source.mutate();

        // Note: both shared reborrow results can be kept and used
        // simultaneously.
        result1.observe();
        result2.observe();
    }
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

fn returning_derived_ref(source: &mut T) -> &U {
    // Note: exclusive reference is reborrowed as shared for the return value.
    source.mutate()
}

// Some function returning a result where the both variants refer to the
// source data; think eg. a parser that wants to return a structure containing
// references to source text slice or an error pointing to invalid source text.
fn result_function(source: &T) -> Result<&X, &Y>;

// A function taking an exclusive reference, reborrowing it as shared to call
// other functions that return derived references, and then conditionally
// returning some results of those calls.
fn branching_function(source: &mut T) -> Result<&U, &Y> {
    // Note: the early Err return requires the Err value's lifetime to extend
    // to the source lifetime, ie. to the end of the call. The mutate call
    // later requires the Ok value's lifetime to end before that call. The
    // compiler is currently not smart enough to split Ok and Err from one
    // another, meaning that this does not work in current stable Rust.
    // However, this does compile on Polonius!
    let result = result_function(source)?;
    println!("Example: {:?}", result);
    source.mutate()
}
```

The important things to note here are:

1. Reborrowing is done on exclusive references only; shared references are Copy
   and need no reborrowing semantics.
2. Reborrowing has two flavours, shared and exclusive.
3. Reborrowing happens implicitly at each
   [coercion-site](https://doc.rust-lang.org/stable/reference/type-coercions.html#coercion-sites).
4. If the function being called returns a value derived from the reborrowed
   reference, the returned value's lifetime extends in the caller's scope to
   match the original reference's lifetime.
5. Branching function returns run into issues with lifetime extension that needs
   Polonius to resolve.

## Wrapper type reborrowing

Wrapper types like `Option<T>` cannot be reborrowed today. This makes using
optional exclusive references tedious as they are automatically moved when used.
Reborrowing of `Pin<&mut T>` has been provisionally added into the nightly
compiler, but it is done via custom handling and relies on internal knowledge of
the type.

We'll revisit the earlier examples, now with `Option<&mut T>` as our example of
a wrapper type with exclusive reference semantics. Note that this means that the
function signatures for our helper functions are now
`fn mutate(self: Option<&mut T>) -> Option<&mut U>` and
`fn observe(self: Option<&T>) -> Option<&U>`.

```rust
fn mixed_reborrowing(source: Option<&mut T>) {
    // Exclusive reborrowing
    {
        // Note: In current Rust this would work, but after this the source has
        // been moved out of this function and could no longer be used. This
        // call site must change from using a move to using a reborrow.
        let result: Option<&mut U> = source.mutate();
        // Again, we want the following line to fail to compile, and it does
        // but currently for the wrong reasons:
        // source.mutate();

        result.observe();
    }

    // Shared reborrowing
    {
        // We want to enable the following calls to still work: this means that
        // the first mutate() call must reborrow source instead of moving it.
        // The following calls must also implicitly reborrow source as shared.
        let result1: Option<&U> = source.observe();
        let result2: Option<&U> = source.observe();
        result1.observe();
        result2.observe();
    }
    // After all shared reborrow results are dropped, source must be
    // reborrowable as exclusive again.
    source.mutate();
}

fn returning_derived_mut(source: Option<&mut T>) -> Option<&mut U> {
    // This does work today, but mutate() must internally map() the Option
    // instead of eg. using as_deref_mut() to compile. See below.
    source.mutate()
}

fn returning_derived_ref(source: Option<&mut T>) -> Option<&U> {
    // Note: This does not work today, and suggests using as_deref() which also
    // doesn't work, citing "returns a value referencing data owned by the
    // current function". Using eg. Option::map() to inject a coercion site
    // fixes the issue:
    // source.map(|v| -> &T { v })
    source.mutate()
}

// Some function returning a result where the both variants refer to the
// source data; think eg. a parser that wants to return a structure containing
// references to source text slice or an error pointing to invalid source text.
fn result_function(source: Option<&T>) -> Result<&X, &Y>;

// A function taking an exclusive reference, reborrowing it as shared to call
// other functions that return derived references, and then conditionally
// returning some results of those calls.
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
how to split Ok and Err branch lifetimes from one another. Even Polonius does
not help in this case, as the `source.as_ref()` call has made the result
lifetime a strict subset of the source lifetime: even though we can see that the
transformation from `Option<&mut T>` to `Option<&T>` does not retain any
references to local values, the compiler cannot be quite so sure and thus does
not allow this code to compile.

## Custom reference type reborrowing

Custom reference types are as many as there are users, but a common theme
amongst them is that they usually hold one (though sometimes multiple or none)
data field and a PhantomData field (or multiple) for holding a lifetime in. The
previous examples are once again revisited with custom `MyMut` and `MyRef`
reference types. The function signatures of our helper functions are now
`fn mutate<'a>(self: MyMut<'a>) -> MyMut` and
`fn observe<'a>(self: MyRef<'a>) -> MyRef`.

In the examples, "FIX" and "NOT" comments are given to show how some of these
examples can be made to work today and what sort of issues are related to those
fixes. These fixes use `rb()` and `rb_mut()` helper methods that are
"reborrow-like" but do not perform true reborrowing, as will become apparent in
the examples.

```rust
/// Exclusive reborrowable reference to some data. Can be turned into a MyRef
/// by reborrowing as shared.
///
/// Note: this type is not Copy or Clone.
struct MyMut<'a>(...);
impl Reborrow for MyMut { /* ... */ }

/// Shared reference to some data.
#[derive(Clone, Copy)]
struct MyRef<'a>(...);

/// Reborrow-like helper methods, in the form of a trait.
trait Reborrow {
    /// Type that represents the Self exclusive reference reborrowed as shared.
    type Target;

    fn rb_mut(&mut self) -> Self;
    fn rb(&self) -> Self::Target;
}

fn mixed_reborrowing(source: MyMut) {
    // Exclusive reborrowing
    {
        // Note: mutate() takes self by value, so in current Rust MyMut would be
        // moved here. Instead we want it to be reborrowed as exclusive.
        // Note: explicit "reborrow-like" function works today:
        // FIX: let result = source.rb_mut().mutate();
        let result = source.mutate();
        // Again, we want the following line to fail to compile, and it does
        // but currently for the wrong reasons:
        // source.mutate();

        result.observe();
    }

    // Shared reborrowing
    {
        // We want to enable the following calls to still work: this means that
        // the first mutate() call must reborrow source instead of moving it.
        // The following calls must also implicitly reborrow source as shared.
        // Note: the explicit "reborrow-like" function works today:
        // FIX: let result1 = source.rb().observe();
        // FIX: let result2 = source.rb().observe();
        let result1 = source.observe();
        let result2 = source.observe();
        // Note: source can be reborrowed as shared multiple times.
        result1.observe();
        result2.observe();
    }
    // After all shared reborrow results are dropped, source must be
    // reborrowable as exclusive again.
    // Note: even in current Rust we don't need to use the "reborrow-like"
    // function here, as source is not used after this and we can thus move it
    // out of the function.
    source.mutate();
}

fn returning_derived_mut(source: MyMut) -> MyMut {
    // This does work today, but only if source is moved into mutate() instead
    // of using the "reborrow-like" function:
    // NOT: source.rb_mut().mutate()
    source.mutate()
}

fn returning_derived_ref(source: MyMut) -> MyRef {
    // This does not work today, but can be made to work with a helper, such as
    // the Into trait, that turns an owned MyMut into an owned MyRef. Again,
    // using the explicit "reborrow-like" function does not work:
    // FIX: source.mutate().into()
    // NOT: source.rb_mut().mutate().into()
    source.mutate()
}

// Some function returning an error where both variants refer to the source
// data; think eg. a parser that wants to return a structure containing
// references to source text slice or an error pointing to invalid source text.
fn result_function(source: MyRef) -> Result<MyRef, MyRef>;

// A function taking an exclusive reference, reborrowing it as shared to call
// other functions that return derived references, and then conditionally
// returning some results of those calls.
fn branching_function(source: MyMut) -> Result<MyRef, MyRef> {
    // Note: here again we'd want automatic reborrowing as shared. If we use
    // the explicit "reborrow-like" function then the resulting MyRef has a
    // lifetime strictly bound to the temporary borrow of source. That is what
    // we want for the Ok value, as we must drop result before source gets
    // reused as exclusive, but for the Err value we conversely want the
    // lifetime to be extended to the source lifetime so that we can return it
    // from the function. This could work with Polonius, but only with true
    // reborrowing instead of the "reborrow-like" functions. In effect, this
    // function cannot currently be compiled without using lifetime transmutes.
    // NOT: result_function(source.rb())?;
    // This also does not work, since source is moved here but reused below.
    let result = result_function(source)?;
    // Do something with the result.
    println!("Example: {:?}", result);
    // Then mutate the source again.
    Ok(source.mutate())
}
```

The most important point here is that just like with `Option<&mut T>`, custom
exclusive reference-like types cannot be used in functions returning values with
derived lifetimes from branches. The derived lifetime is long enough (equal to
the source lifetime) if the reference-like type is moved out of the function and
into the subroutine, but then it cannot be reused. If reference-like type is
explicitly "reborrowed" using the helper functions, then the resulting derived
lifetime is strictly bound within the local function and cannot be returned from
it.

Let's look at usage of reborrowing, or the "reborrow-like" functions, of custom
reference-like types in the wild. The following are a smattering of custom
reference-like types and traits that either make use of "reborrow-like"
functions, implement the "reborrow-like" function APIs, or are interested in
using reborrowing in the future.

### [PeripheralRef](https://docs.rs/embassy-nrf/latest/embassy_nrf/struct.PeripheralRef.html)

`PeripheralRef` is an custom exclusive reference-like type that provides
exclusive access to hardware peripherals in an embedded system codebase. Of note
is that the size is usually either a ZST or a single byte in size as many
peripherals are either statically known and need no runtime behaviour, or have
only a small number of possible values to refer to.

Also note that `PeripheralRef` does not have a corresponding shared
reference-like type.

- Size: Generic type dependent, but commonly ZST or 1 byte.
- Kind: Zero-sized marker type, or a small value for runtime decision making.
- Usage: Provide exclusive access to statically knowable (or mostly knowable)
  hardware peripherals in an embedded system codebase.

### [GcScope and NoGcScope](https://github.com/trynova/nova/blob/260683fccbad913491c758781965eb3bf67b531e/nova_vm/src/engine/context.rs#L40-L78)

`GcScope` and `NoGcScope` are a custom exclusive and shared reference-like type,
respectively, that provide access to the Nova JavaScript engine's garbage
collector. The types are zero-sized and have no runtime existence, serving only
as a means of compile-time verification of garbage collector safety, or
use-after-free checking in other words. The "garbage collector lifetime" of the
type is observed by all garbage-collectable data handles, and is taken as
exclusive by any method that may trigger garbage collection. The result is that
methods that may trigger garbage collection always invalidate all
garbage-colletable data handles on call. Of note is that these types actually
carry a second lifetime as well, the "scope lifetime", which is used when
handles are rooted to the current scope. This second lifetime is always shared
reference-like, and thus does not participate in reborrowing.

- Size: ZST.
- Kind: Marker types with no runtime existence. `GcScope` has two lifetimes, one
  lifetime is exclusive and one is shared. `NoGcScope` has two shared lifetimes.
  The always-shared lifetime takes no part in reborrowing and can be ignored
  here.
- Usage: `GcScope` provides exclusive access to a garbage collector, while its
  shared reborrow `NoGcScope` is used to create handles to garbage collected
  data. Handles are thus invalidated if `GcScope` is used as exclusive.

### [MutableHandle](https://doc.servo.org/mozjs/gc/struct.MutableHandle.html) and [Handle](https://doc.servo.org/mozjs/gc/struct.Handle.html)

`MutableHandle` and `Handle` are custom reference-like types to garbage
collectable data in the Servo browser's SpiderMonkey `mozjs` wrapper.

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
`Mat` does have an implied `'static` lifetime, but the lack of reference-like
aspects of it at least cannot be denied. This use-case might be better thought
of as a generalised `AsRef/AsMut` or perhaps `Deref[Mut]` implementation.

### [reborrow crate](https://docs.rs/reborrow/0.5.5/reborrow/index.html)

This is a library used in eg. faer (above `Mat` and friends) that provides
traits for emulating reborrowing.

### Honorable mention: Rust for Linux smart pointers

The Rust for Linux project has many custom smart pointer types with peculiar
requirements beyond normal Rust usage. At least
[one type](https://rust.docs.kernel.org/core/io/struct.BorrowedCursor.html#method.reborrow)
implements reborrowing, and according to some
[discussion on Zulip](https://rust-lang.zulipchat.com/#narrow/channel/425075-rust-for-linux/topic/Autoreborrowing.20of.20RFL.20.28smart.29.20pointers/with/514896274),
there is some interest in having more of such types. One of the reasons why more
of such types do not exist is said to be bad ergnomics around automatic
reborrowing.

### Honorable mention: [CppRef and friends](https://docs.rs/autocxx/latest/autocxx/struct.CppRef.html)

These types do not implement reborrowing and do not always carry lifetimes, but
cxx is interested in automatic coercion between the different C++ reference
types. Since these are pointer wrapping types, some forms of autoreborrowing
would make these coercions possible.

# Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

Reborrowing itself should be a fairly direct and "easy-to-grasp" feature in how
it works; exclusive reference wrappers and custom exclusive reference-like types
should be able to opt-in to exclusive reference-like reborrowing. This means
that:

1. When such a value of this type is used at a coercion site, the value is
   implicitly reborrowed as either exclusively or as shared, depending on the
   target type required by the site.
2. A copy of the source value is produced and converted into the target type.
3. Lifetime extension is applied to the reborrowed lifetime as necessary.

A reborrowable type is declared to the compiler using the following methodless
trait: note that implementing the trait makes a type reborrowable both
exclusively and as shared, and it is the target type of a coercion site that
implicitly decides which type of reborrowing is performed.

```rust
unsafe trait Reborrow<'a>: !Clone + !Drop {
    type Ref: Copy;
}
```

Note: the `!Drop` requirement isn't sufficient; reborrowable types must neither
implement `Drop` or return `true` for `std::mem::needs_drop`; this requirement
cannot be specified as a trait bound. The `Copy` bound on `Borrow::Ref` is a
logical result of shared reborrowing: the source type can be reborrowed as
shared multiple times and each shared reborrow should produce logically
interchangeable target types (order or number of reborrows should not matter);
hence reborrowing twice as shared should be equal to reborrowing once as shared
and copying the result.

The trait would be implemented in the following way:

```rust
unsafe impl<'a> Reborrow<'a> for MatMut<'a> {
    type Ref = MatRef<'a>;
}
```

or for types with multiple lifetimes:

```rust
unsafe impl<'a, 'b> Reborrow<'a> for GcScope<'a, 'b> {
    type Ref = NoGcScope<'a, 'b>;
}
```

Implementing `Reborrow<'a> for T<'a> { type Ref; }` means that

1. At coercion sites taking `T` by value, the source `T` is copied via an
   exclusive reference (exclusive access is asserted) and its `'a` lifetime is
   replaced with a reborrowed lifetime on which normal lifetime extension rules
   can be applied. While the result `T` lives, the source `T` is disabled.
2. At coercion sites taking `T::Ref` by value, the source `T` is copied via a
   shared reference (shared access is asserted) and the copy converted into
   `T::Ref`. The `'a` lifetime in `T::Ref`, if found, is replaced with a
   reborrowed lifetime as above. While the result `T::Ref` lives, the source `T`
   is disabled for mutations.

# Drawbacks

[drawbacks]: #drawbacks

The main drawback of autoreborrowing is how to convert the `T` to `T::Ref`: the
most obvious way to do this would be with an `fn reborrow` method on the
`Reborrow` trait, but this opens up a world where each coercion site may
implicitly inject a call to user-code. For this reason the proposed `Reborrow`
trait lack methods entirely, as we'd prefer for reborrowing to be devoid of any
user-code. But if user-code is not involved in the conversion, then the compiler
is forced to perform the conversion on its own.

The best-case for compiler-injected conversion would be the
[safe transmute project](https://rust-lang.github.io/rfcs/2835-project-safe-transmute.html)'s
mechanisms, but those can only detect grievous errors like transmuting padding
bytes to data. They would not detect logical errors, such as swapping two
`usize` fields with one another (think swapping `length` and `cap` in `Vec`).
Thus, as the `Reborrow` trait relies on compiler-injected transmutes, it is
marked `unsafe`.

The second drawback of autoreborrowing is that it of course requires a generic
implementation of reborrowing relying on the `Reborrow` trait in rustc. A
version of reference wrapper autoreborrowing already exists in rustc for
`Pin<&mut T>`, so the basic building blocks needed for autoreborrowing are
already present. This drawback is thus considered to be minor.

# Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

## Rationale

The rationale for choosing a single trait with a single target GAT and no
methods is as follows:

- Reborrowing only makes sense for exclusive reference types: reborrowing of
  shared reference types as shared is exactly equal to copying the shared
  reference type.
- An exclusive reference should only be exclusively reborrowable into itself,
  ie. `&mut T -> &mut T` or `MatMut<'_> -> MatMut<'_>`. A `&mut T -> &mut U`
  should be thought of as a reborrow followed by a dereference. Hence, a
  separate `Reborrow::Mut` GAT is not needed.
- An exclusive reference type that doesn't need to be reborrowable as shared may
  exist: the proposed `Reborrow` trait does not directly support this, but it
  can implemented by creating a dummy type (with same layout as the exclusive
  reference) which is never used and has no methods on it. The exclusive
  reference type will thus be reborrowable as shared, but into a type that
  cannot be used for anything.
- Adding a method to reborrowing would make all coercion sites into implicit
  injection points into user-code. This seems both too powerful, too magical,
  and too subtle to allow, hence no method is proposed.

### Compiler-level reasoning around custom type autoreborrowing

Exclusive reborrowing of custom types is fairly straightforward as the only
complexity lies in implementing the lifetime replacement and extension, and the
disabling of the source `T`. This is already implemented in the compiler for
`Pin<&mut T>` specifically, and extending this for all types implementing the
`Reborrow` trait ought not be too complicated.

Shared reborrowing is much less so, as it requires the compiler to copy the
source `T` into `<T as Reborrow>::Ref`. The usual case for reborrowable types is
that they have only one (or no) data field of the same kind for both the
exclusive and shared reference types, meaning that a transmute would be
trivially sound (at least for `#[repr(transparent)]` types), but for multi-field
types the soundness of the transmute cannot be guaranteed. A basic reasoning
mechanism safe transmutes is provided by the
[safe transmute project](https://rust-lang.github.io/rfcs/2835-project-safe-transmute.html),
which can detect grievous mistakes like transmuting padding bytes into data.
More complex reasoning, like making sure that two `usize` fields do not
logically flip in the transmute, cannot be performed by the compiler and would
require `#[repr(C)]` usage on both sides of the transmute. As this requires
manual checking by the user of the `Reborrow` trait, the trait is proposed as
`unsafe` with the requirement being that `T` is transmutable into `T::Ref`.
Hence, the trait's safety requirements are inherited from `std::mem::transmute`.

### Mitigating the unsafety of an injected transmute

As the transmuting of `T` to `T::Ref` is unsafe and requires manual checking,
we'd like to avoid the injected transmute. The obvious thing to do would be to
add a method `fn reborrow(self) -> T::Ref` (note: the method must take self by
value, not by reference) to the `Reborrow` method but as discussed above, this
is not proposed because it would mean injecting implicit user-controlled code
with arbitrary side-effects to every coercion site. This runs counter to what
reborrowing is generally though of being, "copy of a pointer with extra lifetime
analysis", and we'd thus want to avoid it. (Contrast this with adding an
`fn copy()` method to the `Copy` trait.)

With the safe transmute project, the implicit transmute could be improved by
requiring a `type Ref: TransmuteFrom<Self, _>` trait bound on the GAT: with this
the source and target would at least be confirmed to be safely transmutable. As
mentioned earlier, this would still not guarantee that the transmute is sound
but would at least go a long way in mitigating the issue.

An alternative to an explicit `reborrow` method would be to add a
`type Ref: From<T>` bound to the GAT and have the compiler inject a
`<T as Reborrow>::Ref::from(T)` call at the reborrow site. The downside here is
that the `From` impl is still user-controlled code and could likewise have
arbitrary side-effects. That being said, `From` trait implementations are well
understood (as compared to a custom `reborrow` method, or even eg. `Deref` vs
`AsRef` vs `Borrow`), are held to a high standard, and misusing said traits is
likely to cause various other issues; they are thus unlikely to contain
side-effectful code.

Finally, in the future if/when `const fn`s in traits become available,
improvements on the above method-based approaches appear: the custom `reborrow`
method could be made into a `const fn reborrow()` or, if trait bounds requiring
`const` implementations of traits or trait methods become possible, the `From`
bound could be turned into `Target: const From<Self>::from`. With this the fear
of entirely arbitrary side-effects is further mitigated, though not entirely
removed.

The safe transmute trait bound would not be enough for the `Reborrow` trait to
be marked as safe, due to the soundness issue. Hence, it is not a complete
mitigation.

Of the possible complete mitigations, this RFC considers only the
`type Ref: From<T>` bound worthy of further study. An explicit reborrow method
would have high chance of misuse, but a `From` trait conversion can be fairly
well trusted to be valid and safe to call implicitly. Adding `const` bounds
would restrict the possible misuse of a `reborrow` or `from` method, but as the
capabilities of `const` expand the restrictions placed by this bound would
slowly erode. Hence, those bounds would not really be reliable soundness
guarantees.

### Detecting the reborrowed lifetime

In the above reborrow formulation, the `Reborrow` trait itself has a lifetime
parameter that the compiler uses to detect which lifetime is being reborrowed,
in case there are multiple (or none). This seems like a neat solution, but if it
is deemed problematic then an alternative would be to require that reborrowable
types and their `Ref` targets have exactly one lifetime.

This would cause some trouble for some users (including the author's `GcScope`),
but if it is the price that has to be paid for true reborrowing of custom types,
then so be it.

## Alternatives

### Alternative 1: Reference-based method(s)

The most obvious alternative to the proposed `Reborrow` trait would be to take
what we already have in userland and make it part of the core library and rustc.
This would mean implementing a trait or traits in the vein of the `reborrow`
crate:

```rust
/// Shared reborrowing.
trait Reborrow<'short, _Outlives = &'short Self> {
    type Target;

    #[must_use]
    fn rb(&'short self) -> Self::Target;
}

/// Exclusive reborrowing.
trait ReborrowMut<'short, _Outlives = &'short Self> {
    type Target;

    #[must_use]
    fn rb_mut(&'short mut self) -> Self::Target;
}
```

Note that two traits are given here to match what the `reborrow` crates does,
not as an explicit alternative proposal.

The main benefit with this approach is that this follows the current userland
state-of-the-art way of creating "reborrow-like" functions and would effectively
bless a particular, well-liked userland implementation into the language
together with compiler support for automatically injecting the trait method
calls, relieving the user of having to call the methods manually.

The main issue with this approach is that this does not implement true
reborrowing: the `'short` lifetime gets bound to the temporary borrow created
when calling these methods, meaning that lifetime extension does not work out of
the box. The `'short` lifetime is a strict subset of the source lifetime (call
it `'long`), instead of being able to extend to equal it. To overcome this
issue, rustc would need to special-case this trait implementation to allow
lifetime extension of `'short` into `'long`. But doing so is unsound!

As the implementation of these methods is user-controlled, users often find neat
ways to stretch implementations beyond the limits of what was intended but not
explicitly made impossible. In a pure reborrow frame of mind, turning
`&'short Self<'long>` into `Self<'short>` with `'short` being allowed to
lifetime extend to equal `'long` seems sound, but it is not. The `&Self`
reference is a pointer onto the stack (most likely) and allowing lifetime
extension from `Self<'short>` to `Self<'long>` would mean that rustc would allow
the `&Self` reference to escape its call stack. This is equal to returning a
pointer pointing to released stack memory, ie. undefined behaviour.

Thus, in current Rust syntax a `Reborrow` trait taking `self` by reference while
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

As this sort of a reference would be something entirely new (though it probably
relates to `Pin<& (mut) T>` somehow), it is deemed too complicated a change for
this RFC. Because true reborrowing is one of the focal aims of this RFC, any
reference-based trait method is thus considered a non-starter due to the
soundness issue.

### Alternative: Generic type parameter

The `Reborrow` trait (any version of it) could be defined with a generic type
parameter. This would be more generic than having a generic associated type,
though it would likely make the implementation correspondingly more complex as
well. This doesn't help with the soundness issues of a reference-based method or
anything else related to the trait. It also has a new issue lurking within it: a
`Reborrow` trait that can be implemented multiple times over gives users the
power needed to implement automatic type coercion. Eg. for integer coercion, you
need only to implement `Reborrow<uN> for MyInt` and `Reborrow<iN> for MyInt` for
all `N` of your preference and now your `MyInt` type will be reborrow-coerced
into whatever integer type is needed by a given call.

For this reason, this RFC does not propose a `Reborrow` trait with a generic
type parameter.

### Alternative: Special reborrow syntax

If userland reborrowing was made explicit in the source code using special
syntax, then the compiler would not need to implicitly inject user-controlled
code at coercion sites. This would make user-controlled reborrow methods
perfectly acceptable, as they would be visible in user code.

The
[ergonomic clones](https://github.com/rust-lang/rust-project-goals/issues/107)
feature has added a `foo.use` suffix operator for (effectively) calling
`clone()` in an ergonomic way. A similar thing could be done for reborrowing
using eg. the `ref` and `mut` keywords:

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

This does not solve the problem of lifetime extension in any way, ie. this does
enable implementing true reborrowing. It also takes away the "auto" from
"autoreborrowing". (The `ref` and `mut` suffix syntax might also be better used
for other purposes, such as generalised `Deref` / `DerefMut` or `AsRef` /
`AsMut` calls.) For these reasons, this RFC does not propose adding a custom
syntax for reborrowing.

### Alternative: "Event handler" methods

The soundness issue related with returning a new value from a reference-based
reborrow method makes reference-based reborrow methods a non-starter. If
user-controlled code during reborrowing is still wanted, an alternative to the
conversion methods is to have "event handler" methods like `on_reborrow` and
`on_reborrow_mut`. These methods would take the reborrow result by reference but
would have no return value, thus avoiding the soundness issue at the cost of
still requiring the compiler to inject bitwise copies and transmutes.

These would allow compiler-injected reborrowing to trigger user-controlled code,
which would be beneficial for eg. reference counted types. This would be useful
for eg. `RefCell<T>`'s `Ref<'a, T>` and `RefMut<'a, T>` types, which perform
(cheap) reference incrementing/decrementing on Clone and Drop. If the reference
counting of these types could be performed as part of reborrowing, then methods
like `RefMut::map_split` could be generalised over both reference counting and
reborrowing types.

The downside here is that cloning is explicit, and this explicitness and clippy
linting makes it easy for users to opt out of cloning when it is not needed.
With compiler-injected reborrowing, this becomes much more complicated. Consider
the following code:

```rust
fn method(r: RefMut) {
    method_a(r);
    method_b(r);
}
```

In current Rust, this will not compile as `r` is moved into the `method_a` call.
To fix this, an explicit `.clone()` must be inserted:

```rust
fn method(r: RefMut) {
    method_a(r.clone());
    method_b(r);
}
```

Now imagine the original code being compiled with `RefMut` implementing
`Reborrow` (and still being `Drop` like previously): both method calls will
implicitly reborrow `r`, effectively performing two clone calls and one `drop`:

```rust
fn method(r: RefMut) {
    method_a(r.reborrow()); // calls on_reborrow() on the result
    method_b(r.reborrow()); // cals on_reborrow() on the result
    // implicit `drop(r);` happens here
}
```

A sufficiently smart compiler could theoretically optimise the extra clone out,
but because of the `on_reborrow` method it would be very hard or impossible to
prove that the combination of `r.reborrow()` and `drop(r)` is equal to moving
`r` into the second call. The "ergonomic clones" RFC
[does include a provision](https://github.com/joshtriplett/rfcs/blob/use/text/3680-use.md#the-implementation-and-optimization-of-use)
for allowing the compiler to optimise out extra clones; the same kind of
provision could be asserted for `Reborrow` to allow the compiler to remove the
extra calls, but it is an added complexity.

Therefore, this RFC does not propose any `on_reborrow` like trait methods and
instead suggests keeping reference counting and reborrowing separate from one
another.

## Related efforts

### Generalised `Deref` and `DerefMut` traits

The `Deref` and `DerefMut` traits are quite magical and powerful, but they are
not quite powerful enough to implement the userland reborrow-like functions
because they must always return references. A generalised
`Deref(Mut)Generalised` trait would be one where the return value is the
`Target` GAT directly and not a reference of it. These traits would be strong
enough to replace the `reborrow` crate and would have the benefit that a
`DerefGeneralised` trait enables passing reborrowable parameters with a simple
`&my_ref` or `&mut my_mut` to explicitly trigger dereferencing. This would be
visible in the source code (usually; method calls wouldn't show them) and thus
the trait methods would not be as scary, compared to implicitly inserted
user-controlled code.

Again, these traits on their own do not and cannot solve the lifetime extension
issue. In this sense, the current userland's "reborrow-like" functions could
actually be more accurately called "generalised deref functions" as the two are
exactly equivalent.

### Reborrowing of reference counted references

Briefly discussed above in the "event handler" methods alternative, this section
expands on how and why reference counted references might want to tap into
method-based reborrowing, and why this is probably not the best of ideas.

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
- Reborrow trait pre-RFC:
  [Rust Internals Forum #20774](https://internals.rust-lang.org/t/reborrow-trait/20774)
- Feature request to simulate exclusive reborrows in user code:
  [rust-lang#1403](https://github.com/rust-lang/rfcs/issues/1403)
- Experimental support for Pin reborrowing:
  [rustc#130526](https://github.com/rust-lang/rust/pull/130526)
- Obscure Rust: reborrowing is a half-baked feature
  [haibane_tenshi's blog](https://haibane-tenshi.github.io/rust-reborrowing/)

# Unresolved questions

[unresolved-questions]: #unresolved-questions

Matching the custom type's lifetimes to the reborrowed lifetime can be done via
a lifetime bound on the trait, but it is a magical feature that doesn't really
otherwise have much of a use. Whether this matching could be done in a better
way is an unresolved question, possibly in a manner allowing multiple reborrowed
lifetimes if that happens to be reasonable as that would make passing of
"collection of exclusive references" kind of context structs easier. This is not
considered a high priority for this RFC, however.

# Future possibilities

[future-possibilities]: #future-possibilities

It seems probable that autoreborrowing will open wide the doors for `AsRef`,
`Deref`, `Borrow`, and similar traits to be re-examined and extended in a world
where custom reference types are commonplace. As an example, currently a
`&mut T` can be passed to a method that takes `&mut U` where `T` dereferences
into a `U`. With the proposed `Reborrow` trait, this would not be possible. A
`Reborrow`-aware generalised `Deref` trait that enables this would need to take
the custom reference type `T` by value and produce a `U` as a result, ie. an
`Into` trait. Allowing `Into::into` calls to be injected at reborrow sites seems
like a possibly bad idea, and is thus not proposed in this RFC. Revisiting this
is, however, possibly worthwhile at a later time if eg. `^(mut)`-like references
that cannot be used as places make it into the language.

Additionally, some custom reference types require a context parameter for
performing actual dereferencing. Far into the future, a true generalised `Deref`
trait might thus look something like this:

```rust
impl<'a, T> DerefGeneralised for ArenaHandle<'a, T> {
    type Context = &'a Arena;
    type Target = &'a T;

    fn deref_generalised(^self, ctx: Self::Arena) -> Self::Target {
        // ...
    }
}
```

But, as the traits become more generic they become more and more complex and
error-prone, and thus it is unclear if these sorts of generalisations truly
deserve to make it into the language or if they should forever stay in userland
only.

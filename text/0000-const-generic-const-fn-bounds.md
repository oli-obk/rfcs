- Feature Name: const_generic_const_fn_bounds
- Start Date: 2018-10-05
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

Allow `impl const Trait` for trait impls where all method impls are checked as const fn.

Make it legal to declare trait bounds on generic parameters of const functions and allow
the body of the const fn to call methods on the generic parameters that have a `const` modifier
on their bound.

# Motivation
[motivation]: #motivation

Without this RFC one can declare const fns with generic parameters, but one cannot add trait bounds to these
generic parameters. Thus one is not able to call methods on the generic parameters (or on objects of the
generic parameter type), because they are fully unconstrained.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

You can mark trait implementations as having only `const fn` methods.
Instead of adding a `const` modifier to all methods of a trait impl, the modifier is added to the trait
of the `impl` block:

```rust
struct MyInt(i8);
impl const Add for MyInt {
    fn add(self, other: Self) -> Self {
        MyInt(self.0 + other.0)
    }
}
```

You cannot implement both `const Add` and `Add` for any type, since the `const Add`
impl is used as a regular impl outside of const contexts. Inside a const context, you can now call
this method, even via its corresponding operator:

```rust
const FOO: MyInt = MyInt(42).add(MyInt(33));
const BAR: MyInt = MyInt(42) + MyInt(33);
```

You can also call methods of generic parameters of a const function, because they are implicitly assumed to be
`const fn`. For example, the `Add` trait bound can be used to call `Add::add` or `+` on the arguments
with that bound.

```rust
const fn triple_add<T: Add<Output=T>>(a: T, b: T, c: T) -> T {
    a + b + c
}
```

The obligation is passed to the caller of your `triple_add` function to supply a type which has a
`const Add` impl.

The const requirement is inferred on all bounds of the impl and its methods,
so in the following `H` is required to have a const impl of `Hasher`, so that
methods on `state` are callable.

```rust
impl const Hash for MyInt {
    fn hash<H>(
        &self,
        state: &mut H,
    )
        where H: Hasher
    {
        state.write(&[self.0 as u8]);
    }
}
```

The same goes for associated types' bounds: all the bounds require `impl const`s for the type used
for the associated type:

```rust
trait Foo {
    type Bar: Add;
}
impl const Foo for A {
    type Bar = B; // B must have an `impl const Add for B`
}
```

If an associated type has no bounds in the trait, there are no restrictions to what types may be used
for it.

These rules for associated types exist to make this RFC forward compatible with adding const default bodies
for trait methods. These are further discussed in the "future work" section.

## Generic bounds

The above section skimmed over a few topics for brevity. First of all, `impl const` items can also
have generic parameters and thus bounds on these parameters, and these bounds follow the same rules
as bounds on generic parameters on `const` functions: all bounds can only be substituted with types
that have `impl const` items for all the bounds. Thus the `T` in the following `impl` requires that
when `MyType<T>` is used in a const context, `T` needs to have an `impl const Add for Foo`.

```rust
impl<T: Add> const Add for MyType<T> {
    /* some code here */
}
const FOO: MyType<u32> = ...;
const BAR: MyType<u32> = FOO + FOO; // only legal because `u32: const Add`
```

Furthermore, if `MyType` is used outside a const context, there are no constness requirements on the
bounds for types substituted for `T`.

## Drop

A notable use case of `impl const` is defining `Drop` impls.
Since const evaluation has no side effects, there is no simple example that
showcases `const Drop` in any useful way. Instead we create a `Drop` impl that
has user visible side effects:

```rust
let mut x = 42;
SomeDropType(&mut x);
// x is now 41

struct SomeDropType<'a>(&'mut u32);
impl const Drop for SomeDropType {
    fn drop(&mut self) {
        *self.0 -= 1;
    }
}
```

You are now allowed to actually let a value of `SomeDropType` get dropped within a constant
evaluation. This means that

```rust
(SomeDropType(&mut 69), 42).1
```

is now allowed, because we can prove
that everything from the creation of the value to the destruction is const evaluable.

### const Drop in generic code

`Drop` is special in Rust. You don't need to specify `T: Drop`, but `T::drop` will still be called
if an object of type `T` goes out of scope. This means there's an implicit assumption, that given
an arbitrary `T`, we might call `T::drop` if `T` has a drop impl. While we can specify
`T: Drop` to allow calling `T::drop` in a `const fn`, this means we can't pass e.g. `u32` for
`T`, because `u32` has no `Drop` impl. Even types that definitely need dropping, but have no
explicit `Drop` impl (like `struct Foo(String);`), cannot be passed if `T` requires a `Drop` bound.

To be able to know that a `T` can be dropped in a `const fn`, this RFC proposes to make `T: Drop`
be a valid bound for any `T`, even types which have no `Drop` impl. In non-const functions this
would make no difference, but `const fn` adding such a bound would allow dropping values of type
`T` inside the const function. Additionally it would forbid calling a `const fn` with a `T: Drop`
bound with types that have non-const `Drop` impls (or have a field that has a non-const `Drop` impl).

```rust
struct Foo;
impl Drop for Foo { fn drop(&mut self) {} }
struct Bar;
impl const Drop for Bar { fn drop(&mut self) {} }
struct Boo;
// cannot call with `T == Foo`, because of missing `const Drop` impl
// `Bar` and `Boo` are ok
const fn foo<T: Drop>(t: T) {}
```

Note that one cannot implement `const Drop` for structs with fields with just a regular `Drop` impl.
While from a language perspective nothing speaks against that, this would be very surprising for
users. Additionally it would make `const Drop` pretty useless.

```rust
struct Foo;
impl Drop for Foo { fn drop(&mut self) {} }
struct Bar(Foo);
impl const Drop for Bar { fn drop(&mut self) {} } // not ok
// cannot call with `T == Foo`, because of missing `const Drop` impl
const fn foo<T: Drop>(t: T) {
    // Let t run out of scope and get dropped.
    // Would not be ok if `T` is `Bar`,
    // because the drop glue would drop `Bar`'s `Foo` field after the `Bar::drop` had been called.
    // This function is therefore not accept by the compiler.
}
```

## Runtime uses don't have `const` restrictions

`impl const` blocks are treated as if the constness is a generic parameter
(see also effect systems in the alternatives).

E.g.

```rust
impl<T: Add> const Add for Foo<T> {
    fn add(self, other: Self) -> Self {
        Foo(self.0 + other.0)
    }
}
#[derive(Debug)]
struct Bar;
impl Add for Bar {
    fn add(self, other: Self) -> Self {
        println!("hello from the otter side: {:?}", other);
        self
    }
}
impl Neg for Bar {
    fn neg(self) -> Self {
        self
    }
}
```

allows calling `Foo(Bar) + Foo(Bar)` even though that is most definitely not const,
because `Bar` only has an `impl Add for Bar`
and not an `impl const Add for Bar`. Expressed in some sort of effect system syntax (neither
effect syntax nor effect semantics are proposed by this RFC, the following is just for demonstration
purposes):

```rust
impl<c: constness, T: const(c) Add> const(c) Add for Foo<T> {
    const(c) fn add(self, other: Self) -> Self {
        Foo(self.0 + other.0)
    }
}
```

In this scheme on can see that if the `c` parameter is set to `const`, the `T` parameter requires a
`const Add` bound, and creates a `const Add` impl for `Foo<T>` which then has a `const fn add`
method. On the other hand, if `c` is "may or may not be `const`", we get a regular impl without any
constness anywhere.
For regular impls one can still pass a `T` which has a `const Add` impl, but that won't
cause any constness for `Foo<T>`.

This goes in hand with the current scheme for const functions, which may also be called
at runtime with runtime arguments, but are checked for soundness as if they were called in
a const context. E.g. the following function may be called as
`add(Bar, Bar)` at runtime.

```rust
const fn add<T: Neg, U: Add<T>>(a: T, b: U) -> T {
    -a + b
}
```

Using the same effect syntax from above:

```rust
<c: constness> const(c) fn add<T: const(c) Neg, U: const(c) Add<T>>(a: T, b: U) -> T {
    -a + b
}
```

Here the value of `c` decides both whether the `add` function is `const` and whether its parameter
`T` has a `const Add` impl. Since both use the same `constness` variable, `T` is guaranteed to have
a `const Add` iff `add` is `const`.

This feature could have been added in the future in a backwards compatible manner, but without it
the use of `const` impls is very restricted for the generic types of the standard library due to
backwards compatibility.
Changing an impl to only allow generic types which have a `const` impl for their bounds would break
situations like the one described above.

## `?const` opt out

### Motivation

There is often desire to add bounds to a `const` function's generic arguments, without wanting to
call any of the methods on those generic bounds. Prominent examples are `new` functions:

```rust
struct Foo<T: Trait>(T);
const fn new<T: Trait>(t: T) -> Foo<T> {
    Foo(t)
}
```

<details><summary>Click here for effect system syntax description</summary>

```rust
struct Foo<T: Trait>(T);
<c: constness> const(c) fn new<T: const(c) Trait>(t: T) -> Foo<T> {
    Foo(t)
}
```
</details>

Unfortunately, with the given syntax in this RFC, one can now only call the `new` function in a const
context if `T` has an `impl const Trait for T { ... }`.

This `new` constructor example is simplified from the following real use cases:

1. Drop impls need to have the same generic bounds that the type declaration has.
    If you want to have a const Drop implementation,
    all bounds must be const Trait on the type and the Drop impl,
    even if the Drop impl does not use said trait bounds.

2. The standard library is full of cases where you have bounds on a generic type's declaration
    (e.g. because the Drop impl needs them or to have earlier, more helpful, errors).
    Any method (or its impl block) on that generic type will need to repeat those bounds.
    Repeating those bounds will restrict the impls further than actually required.
    We don't need `const Trait` bounds if all we do is store values of the generic
    argument type in fields of the result type.
    Examples from the standard library include, but are not limited to
    * [RawVec::new](https://github.com/rust-lang/rust/blob/a7cef0bf0810d04da3101fe079a0625d2756744a/src/liballoc/raw_vec.rs#L51)
    * [Iterator::map](https://github.com/rust-lang/rust/blob/a7cef0bf0810d04da3101fe079a0625d2756744a/src/libcore/iter/traits/iterator.rs#L558)
        * and essentially all other `Iterator` methods that take a closure arg
    * [HashMap::new](https://github.com/rust-lang/rust/blob/a7cef0bf0810d04da3101fe079a0625d2756744a/src/libstd/collections/hash/map.rs#L697)
        * and `HashSet` (which is implemented on top of `HashMap`)
    * [[T]::split](https://github.com/rust-lang/rust/blob/a7cef0bf0810d04da3101fe079a0625d2756744a/src/libcore/slice/mod.rs#L1045)
        * and all other slice methods that take a closure

### `?const` syntax

Thus an opt-out similar to `?Sized` is proposed by this RFC:

```rust
struct Foo<T: Trait>(T);
const fn new<T: ?const Trait>(t: T) -> Foo<T> {
    Foo(t)
}
```

<details><summary>Click here for effect system syntax description</summary>
```rust
struct Foo<T: Trait>(T);
// note the lack of `const(c)` before `Trait`
<c: constness> const(c) fn new<T: Trait>(t: T) -> Foo<T> {
    Foo(t)
}
```
</details>

This allows functions to have `T: ?const Trait` bounds on generic parameters without requiring users
to supply a `const Trait` impl for types used for `T`. This feature is added under a separate
feature gate and will be stabilized separately from (and after) `T: Trait` bounds on `const fn`.

## `const` default method bodies

Trait methods can have default bodies for methods that are used if the method is not mentioned
in an `impl`. This has several uses, most notably

* reducing code repetition between impls that are all the same
* adding new methods is not a breaking change if they also have a default body

In order to keep both advantages in the presence of `impl const`s, we need a way to declare the
method default body as being `const`. The exact syntax for doing so is left as an open question to
be decided during the implementation and following final comment period. For now one can add the
placeholder `#[default_method_body_is_const]` attribute to the method.

```rust
trait Foo {
    #[default_method_body_is_const]
    fn bar() {}
}
```

While we could use `const fn bar() {}` as a syntax, that would conflict
with future work ideas like `const` trait methods or `const trait` declarations.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The implementation of this RFC is (in contrast to some of its alternatives) mostly
changes around the syntax of the language (allowing `const` modifiers in a few places)
and ensuring that lowering to HIR and MIR keeps track of that.
The miri engine already fully supports calling methods on generic
bounds, there's just no way of declaring them. Checking methods for constness is already implemented
for inherent methods. The implementation will have to extend those checks to also run on methods
of `impl const` items.

## Precedence

A bound with multiple traits only ever binds the `const` to the next trait, so `const Foo + Bar`
only means that one has a `const Foo` impl and a regular `Bar` impl. If both bounds are supposed to
be `const`, one needs to write `const Foo + const Bar`. More complex bounds might need parentheses.

## Implementation instructions

1. Add an `maybe_const` field to the AST's `TraitRef`
2. Adjust the Parser to support `?const` modifiers before trait bounds
3. Add an `maybe_const` field to the HIR's `TraitRef`
4. Adjust lowering to pass through the `maybe_const` field from AST to HIR
5. Add a a check to `librustc_typeck/check/wfcheck.rs` ensuring that no generic bounds
    in an `impl const` block have the `maybe_const` flag set
6. Feature gate instead of ban `Predicate::Trait` other than `Sized` in
    `librustc_mir/transform/qualify_min_const_fn.rs`
7. Remove the call in https://github.com/rust-lang/rust/blob/f8caa321c7c7214a6c5415e4b3694e65b4ff73a7/src/librustc_passes/ast_validation.rs#L306
8. Adjust the reference and the book to reflect these changes.

## Const type theory

This RFC was written after weighing practical issues against each other and finding the sweet spot
that supports most use cases, is sound and fairly intuitive to use. A different approach from a
type theoretical perspective started out with a much purer scheme, but, when exposed to the
constraints required, evolved to essentially the same scheme as this RFC. We thus feel confident
that this RFC is the minimal viable scheme for having bounds on generic parameters of const
functions. The discussion and evolution of the type theoretical scheme can be found
[here](https://github.com/rust-rfcs/const-eval/pull/8#issuecomment-452396020) and is only 12 posts
and a linked three page document long. It is left as an exercise to the reader to read the
discussion themselves.
A summary of the result of the discussion can be found at the bottom of [this blog post](https://varkor.github.io/blog/2019/01/11/const-types-traits-and-implementations-in-Rust.html)

# Drawbacks
[drawbacks]: #drawbacks

* It is not a fully general design that supports every possible use case,
  but it covers the most common cases. See also the alternatives.
* It becomes a breaking change to add a new method to a trait, even if that method has a default
  impl. One needs to provide a `const` default impl to not make the change a breaking change.
* It becomes a breaking change to add a field (even a private one) that has a `Drop` impl which is
  not `const Drop` (or which has such a field).
* `?const` gives a lot of control to users and may make people feel an obligation to properly
  annotate all of their generic parameters so that they propagate constness as permissively as
  possible, but that this will create too much burden on the community in a variety of ways.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## `ConstDrop` trait to opt into const-droppability

Right now it is a breaking change to add a field (even a private one) that has a non-const `Drop`
impl. This makes `const Drop` a marker trait similar to `Send` and `Sync`. Alternatively we can
introduce an explicit `ConstDrop` (name bikesheddable) trait, that needs to be implemented for all
types, even `Copy` types. Users would need to add `T: ConstDrop` bounds instead of `T: Drop` bounds.

## Effect system

A fully powered effect system can allow us to do fine grained constness propagation
(or no propagation where undesirable). This is out of scope in the near future
and this RFC is forward compatible to have its background impl be an effect system.

## Fine grained `const` annotations

One could annotate methods instead of impls, allowing just marking some method impls
as const fn. This would require some sort of "const bounds" in generic functions that
can be applied to specific methods. E.g. `where <T as Add>::add: const` or something of
the sort. This design is more complex than the current one and we'd probably want the
current one as sugar anyway.

## Require `const` bounds everywhere

One could require `const` on the bounds (e.g. `T: const Trait`) instead of assuming constness for all
bounds. That design would not be forward compatible to allowing `const` trait bounds
on non-const functions, e.g. in:

```rust
fn foo<T: const Bar>() -> i32 {
    const FOO: i32 = T::bar();
    FOO
}
```

See also the [explicit `const` bounds](#explicit_const) extension to this RFC.

## Infer all the things

We can just throw all this complexity out the door and allow calling any method on generic
parameters without an extra annotation `iff` that method satisfies `const fn`. So we'd still
annotate methods in trait impls, but we would not block calling a function on whether the
generic parameters fulfill some sort of constness rules. Instead we'd catch this during
const evaluation.

This is strictly the least restrictive and generic variant, but is a semver
hazard as changing a const fn's body to suddenly call a method that it did not before can break
users of the function.

# Unresolved questions

* Is it possible to ensure that we are consistent about opt-in vs opt-out of
  constness in static trait bounds, function pointers, and `dyn Trait` while
  remaining backwards compatible? Also see [this discussion][dynamic_dispatch].

# Future work

This design is explicitly forward compatible to all future extensions the author could think
about. Notable mentions (see also the alternatives section):

* an effect system with a "notconst" effect
* const trait bounds on non-const functions allowing the use of the generic parameter in
  constant expressions in the body of the function or maybe even for array lenghts in the
  signature of the function
* fine grained bounds for single methods and their bounds (e.g. stating that a single method
  is const)

It might also be desirable to make the automatic `Fn*` impls on function types and pointers `const`.
This change should probably go in hand with allowing `const fn` pointers on const functions
that support being called (in contrast to regular function pointers).

## Deriving `impl const`

```rust
#[derive(Clone)]
pub struct Foo(Bar);

struct Bar;

impl const Clone for Bar {
    fn clone(&self) -> Self { Bar }
}
```

could theoretically have a scheme inferring `Foo`'s `Clone` impl to be `const`. If some time
later the `impl const Clone for Bar` (a private type) is changed to just `impl`, `Foo`'s `Clone`
impl would suddenly stop being `const`, without any visible change to the API. This should not
be allowed for the same reason as why we're not inferring `const` on functions: changes to private
things should not affect the constness of public things, because that is not compatible with semver.

One possible solution is to require an explicit `const` in the derive:

```rust
#[derive(const Clone)]
pub struct Foo(Bar);

struct Bar;

impl const Clone for Bar {
    fn clone(&self) -> Self { Bar }
}
```

which would generate a `impl const Clone for Foo` block which would fail to compile if any of `Foo`'s
fields (so just `Bar` in this example) are not implementing `Clone` via `impl const`. The obligation is
now on the crate author to keep the public API semver compatible, but they can't accidentally fail to
uphold that obligation by changing private things.

## RPIT (Return position impl trait)

```rust
const fn foo() -> impl Bar { /* code here */ }
```

does not allow us to call any methods on the result of a call to `foo`, if we are in a
const context. It seems like a natural extension to this RFC to allow

```rust
const fn foo() -> impl const Bar { /* code here */ }
```

which requires that the function only returns types with `impl const Bar` blocks.

## Specialization

Impl specialization is still unstable. There should be a separate RFC for declaring how
const impl blocks and specialization interact. For now one may not have both `default`
and `const` modifiers on `impl` blocks.

## `const` trait methods

This RFC does not touch `trait` methods at all, all traits are defined as they would be defined
without `const` functions existing. A future extension could allow

```rust
trait Foo {
    const fn a() -> i32;
    fn b() -> i32;
}
```

Where all trait impls *must* provide a `const` function for `a`, allowing

```rust
const fn foo<T: ?const Foo>() -> i32 {
    T::a()
}
```

even though the `?const` modifier explicitly opts out of constness.

The author of this RFC believes this feature to be unnecessary, since one can get the same effect
by splitting the trait into its const and nonconst parts:

```rust
trait FooA {
    fn a() -> i32;
}
trait FooB {
    fn b() -> i32;
}
const fn foo<T: FooA + ?const FooB>() -> i32 {
    T::a()
}
```

Impls of the two traits can then decide constness of either impl at their leasure.

### `const` traits

A further extension could be `const trait` declarations, which desugar to all methods being `const`:

```rust
const trait V {
    fn foo(C) -> D;
    fn bar(E) -> F;
}
// ...desugars to...
trait V {
    const fn foo(C) -> D;
    const fn bar(E) -> F;
}
```

## `?const` modifiers in trait methods

This RFC does not touch `trait` methods at all, all traits are defined as they would be defined
without `const` functions existing. A future extension could allow

```rust
trait Foo {
    fn a<T: ?const Bar>() -> i32;
}
```

which does not force `impl const Foo for Type` to now require passing a `T` with an `impl const Bar`
to the `a` method.

## `const` function pointers and `dyn Trait`
[dynamic_dispatch]: #const-function-pointers-and-dyn-trait

The RFC discusses const bounds on *static* dispatch. What about *dynamic* dispatch?
In Rust, that means function pointers and `dyn Trait`.

### `dyn Trait`

Treating `dyn Trait` similar to how this RFC treats static trait bounds, we could allow

```rust
const fn foo(bar: &dyn Trait) -> SomeType {
    bar.some_method()
}
```

with an opt out via `?const`

```rust
const fn foo(bar: &dyn ?const Trait) -> SomeType {
    bar.some_method() // ERROR
}
```

However, there is a problem with this. The following code is already allowed on
stable, without any check that the `Trait` implementation is `const`:

```rust
const F: &dyn Trait = ...;
```

We could instead make `dyn Trait` opt-in to constness with `dyn const Trait`,
but that would be inconsistent with how this RFC defines `const` to work around
static trait bounds. Or we could treat `dyn Trait` differently in `const` types
and `const fn` argument/return types.

### Function pointers

This is illegal before and with this RFC:

```rust
const fn foo(f: fn() -> i32) -> i32 {
    f()
}
```

To remain consistent with trait bounds as described in this RFC, it seems
reasonable to assume that a `fn` pointer passed to a `const fn` would implicitly
be required to point itself to a `const fn`, and to have an opt-out with
`?const` for cases where `foo` does not actually want to call `f` (such as
`RawWakerVTable::new`).

However, we have the same problem as with `dyn Trait`.  The following is already
legal in Rust today, even though the `F` doesn't need to be a `const` function:

```rust
const F: fn() -> i32 = ...;
```

Since we can't reuse this syntax, do we need a different syntax or should the
same syntax mean different things for `const` types and `const fn` types?

Alternatively one can prefix function pointers to `const` functions with `const`:

```rust
const fn foo(f: const fn() -> i32) -> i32 {
    f()
}
const fn bar(f: fn() -> i32) -> i32 {
    f() // ERROR
}
```

This opens up the curious situation of `const` function pointers in non-const functions:

```rust
fn foo(f: const fn() -> i32) -> i32 {
    f()
}
```

Which could be useful for ensuring some sense of "purity" of the function pointer ensuring that
subsequent calls will only modify global state if passed in via arguments.

However, as with `dyn Trait` above, this would be inconsistent with what the RFC proposes for traits.

## explicit `const` bounds
[explicit_const]: #explicit-const-bounds

`const` on the bounds (e.g. `T: const Trait`) requires an `impl const Trait` for any types used to
replace `T`. This allows `const` trait bounds on any (even non-const) functions, e.g. in

```rust
fn foo<T: const Bar>() -> i32 {
    const FOO: i32 = T::bar();
    FOO
}
```

Which, once `const` items and array lengths inside of functions can make use of the generics of
the function, would allow the above function to actually exist.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

## Resolve syntax for making default method bodies `const`
The syntax for specifying that a trait method's default body is `const` is left unspecified and uses
the `#[default_method_body_is_const]` attribute as the placeholder syntax.

## Resolve keyword order of `impl const Trait`

There are two possible ways to write the keywords `const` and `impl`:

* `const impl Trait for Type`
* `impl const Trait for Type`

The RFC favors the latter, as it mirrors the fact that trait bounds can be `const`. The constness
is not part of the `impl` block, but of how the trait is treated. This is in contrast to
`unsafe impl Trait for Type`, where the `unsafe` is irrelevant to users of the type.

## Implied bounds

Assuming we have implied bounds on functions or impl blocks, will the following compile?

```rust
struct Foo<T: Add> {
    t: T,
    u: u32,
}

/// T has implied bound `Add`, but is that `const Add` or `?const Add`
const fn foo<T>(foo: Foo<T>, bar: Foo<T>) -> T {
    foo.t + bar.t
}
```

In our exemplary effect syntax would need to add an effect
to struct definitions, too.

```rust
struct Foo<c: constness, T: const(c) Add> {
    t: T,
    u: u32,
}
// const Add
<c: constness> const(c) fn foo<T>(foo: Foo<c, T>, bar: Foo<c, T>) -> T {
    foo.t + bar.t
}
// ?const Add
<c: constness, d: constness> const(c) fn foo<T>(foo: Foo<d, T>, bar: Foo<d, T>) -> T {
    foo.t + bar.t // error
}
```

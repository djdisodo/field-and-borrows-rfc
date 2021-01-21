- Feature Name: (fill me in with a unique ident, `field_and_borrows`)
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

`field` is a item that represents field in `struct`.

`borrows` is a set of `field` that supports borrowing.

I'm not convinced about name `borrows`.
maybe there will be a better name for it

# Motivation
[motivation]: #motivation
## field as item
this allows traits to have field which needs to be implemented when implementing trait

[related thread](https://internals.rust-lang.org/t/fields-in-traits/6933/13)

and it is quite similar to [niko's rfc](https://github.com/nikomatsakis/fields-in-traits-rfc)

## borrows
this helps partial borrowing

[related issue](https://github.com/rust-lang/rfcs/issues/1215)

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

# `field`
`field` is is interface that represents field of struct
```rust
struct Foo {
    x: usize,
    y: usize
}
```
there's a field x, y in Foo so `Foo::x` and `Foo::y` is defined `field`

`field` can be accessed like this
```rust
fn field1() {
    let foo = Foo {
        x: 42,
        y: 39,
    };
    
    assert_eq!(
        foo.(Foo::x), //and the value of Foo::x can be accessed with Foo
        foo.x //in short
    );
}
```

`field` can be field of field of field of .......
```rust
struct Bar {
    foo: Foo,
}
```
so `Bar::foo.x` and `Bar::foo.y` is `field` of Bar

```rust
fn field_of_field {
    let bar = Bar {
        foo: Foo {
            x: 1,
            y: 0
        }
    };
    assert_eq!(1, bar.(Bar::foo.x));
}
```

## `borrows`
`borrows` is a set of `field` to borrow

`borrows` visualises partial borrow

it can be declared using `borrows` keyword

`borrows` can have `public` visibility

putting `borrows` in declaration of `borrows` will flatten it

so` Foo::{ .. }` and `Foo::{ Foo::{ .. } }` is same
```rust
pub borrows borrows1 = Foo::{
    //you can put mutability
    mut x,
    y
}
```

### remaining fields with `..`
this will borrow `x` as immutable and others as mutable
```rust
pub borrows negative_borrow1 = Foo::{
    x,
    mut ..
}
```
#### negative borrow
this will borrow others except `x`
```rust
pub borrows negative_borrow1 = Foo::{
    !x,
    ..
}
```

### borrowing field of field
this will borrow `foo.x`
```rust
pub borrows borrow3 = Bar::{
    foo.{
        x
    }
}
```

in short
```rust
pub borrows borrow3 = Bar::{
    foo.x
}
```

### examples
```rust
//this will borrow Bar::foo with full mutability
pub borrows borrows2 = Bar::{
    mut foo
}

//this will borrow Bar::foo as immutable and borrow Bar::foo.x as mutable
pub borrows borrow3 = Bar::{
    foo.{
        mut x,
        ..
    }
}

//this will borrow Bar::foo as mutable except Bar::foo.x(immutable)
pub borrows borrow4 = Bar::{
    foo.{
        x,
        mut ..
    }
}

pub borrows borrow5 = Foo::{
    mut x,
    ..
}

//this will borrow Bar::foo as immutable and borrow Bar::foo.x as mutable
pub borrows borrow6 = Bar::{
    foo.{ borrow5 }
}
```
### borrowing
`borrows` used in partial borrow

you don't need to do this as current borrow checker automatically implements partial borrow in this case
```rust
fn borrows_use() {
    let foo = Foo {
        x: 42,
        y: 39
    };
    
    let foo_part1 = &foo.{ x }; //borrow only x
    //using foo_part1.y will cause borrow check error
    
    let foo_part2: &Foo::{ y } = &foo; //borrow only y
    //using foo_part2.x will cause borrow check error
}
```

## `field` and `borrows` in `impl` block
```rust
impl Foo {
...
```
this example makes private field `Foo::x` accessible out of the module with `foo.x_another`

`field` can have `public` visibility

`field` can be declared with `field` keyword

declared name should not conlfict with field name in `Foo`
```rust
...
    pub field x_another = Self::x;
...
```

you can borrow all fields of `Foo` like this
```rust
...
    fn field2(&self.{ .. }) {
        self.field3(); //you can borrow all fields of Foo so you can call field3
        self.field4(); //you can borrow Foo::x so you can call field4
    }
...
```
borrowing all fields can be simplified
```rust
...
    fn field3(&self) {
    
    }
...
```
borrow only Foo::x
```rust
...
    fn field4(&self.{ x }) {
        field6(self) //you can borrow Foo::x so you can call field6
    }
...
```
borrowing fields in typed param
```rust
...
    fn field5(foo: &Foo::{ x }) {
        //foo.field5() //you can't call field5 because you can't borrow { !x, .. } which is { Foo::y }
    }
...
```
borrow except Foo::x
```rust
...
    fn field6(&self.{ !x, .. }) {
        
    }
}
```

## `field` and `borrows` in `impl` block
```rust
trait Baz {
...
```
you can declare `field` in trait block

it needs to be implemented when struct implements the trait
```rust
...
    field baz_x: usize;
...
```
borrow all fields of `Self`
```rust
...
    fn field7(&self.{ .. }) -> usize {
        self.baz_x //declared trait can be used here
    }
...
```
borrow all fields of `Baz`

so it's parital borrow
```rust
...
    fn field8(&self.{ Bar::{ .. } }) -> usize{
        self.baz_x
    }
...
```
you can declare `borrows` in trait block

like in this case, if you don't initialize it it will need to be implemented when implementing trait
```rust
...
    borrows f: Self;
...
```
you can add required `borrows` to `Self::f` when implementing `Baz::field9`
```rust
...
    fn field9(&self.{ Self::f }) -> usize;
...
```
```rust
you can borrowed `Self::f` so you can call field9
...
    fn field10(&self.{ bar_x, Self::f }) {
        self.field11();
    }
...
```
```rust
...
    fn field11(&self.{ Self::f }) -> usize;
}
```

## implementing `trait` with `field` and `borrows`
```rust
impl Baz for Foo {
...
```
if `Foo` already has field with same name this can be accessed like `foo.(Baz::field_name)`
```rust
...
    field baz_x: usize = Self::x;
...
```

implementing `borrows`
```rust
...
    borrows f = Self::{ y, x }
...
```
`Foo::f` has `Foo::y` so you can access `Foo::y`
```rust
...
    fn field9(&self.{ Self::f }) -> usize {
        self.y //Foo::f has Foo::y so you can access Foo::y
    }
...
```
you can borrow less than declared in trait
```rust
...
    fn field11(&self.{ y }) -> usize {
        self.y
    }
}
```

```rust
fn field13() {
    let foo = Foo {
        x: 42,
        y: 39,
    };
    
    assert_eq!(
        foo.(Baz::bar_x),
        foo.(Foo::bar_x)
    );
    
    assert_eq!(
        foo.(Baz::bar_x),
        foo.bar_x
    );
    
    assert_eq!(
        foo.field8(),
        foo.y
    );
    
    assert_eq!(
        foo.field8(),
        foo.field9()
    );
}
```


## example usage
```rust
trait DerefField {
   type FieldType;
   field deref_field: FieldType;
}

impl<T: DerefField> Deref for DerefField {
    type Target = Self::FieldType;
    fn deref(&self { deref_field }) -> &Self::Target {
        &self.deref_field
    }
}

struct Example<T> {
    value: T
}

impl<T> DerefField<T> for Example<T> {
    type FieldType = T;
    field deref_field = self.value;
}
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

field should work like re-exporting in module level

there can be a field with different ident pointing same field

field in trait object won't work but you can add a function that returns reference of field



`&T::{ .... }` will work like a type which `&T` or `&mut T` or `&T::{ ..... }` can be implicitally casted into

casting `&T` or `&mut T` into `&T::{ .... }` will borrow every field of it in braces

and won't work if can't borrow it

`&T::{ ... }` will work same as `&T` or `&mut T` in low level like `&mut T` does like `&T`

trait fields will work by storing relative position in the memory



# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

# Prior art
[prior-art]: #prior-art

[partial borrow](https://github.com/rust-lang/rfcs/issues/1215)

[fields in trait](https://github.com/rust-lang/rfcs/pull/1546)


# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

# Future possibilities
[future-possibilities]: #future-possibilities

something like partial move will be implemented in future
then name of `borrows` will need to be changed

we won't need to add `get()` or `get_mut()` for every field the trait is using
and now it will only borrow the exact field

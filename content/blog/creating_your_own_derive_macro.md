+++

title = "Creating your own custom derive macro"
description = "Harnessing the power of `syn` and `quote` to create custom rust macros"
date = 2023-07-23

[taxonomies]
tags = ["rust", "divedb"]
+++

One of the things I tackled early on with when creating [DiveDB](https://github.com/cetra3/divedb) was trying to import a lot of metrics from dive logs. At regular intervals, normally every 10 seconds, a dive computer will record metrics about the dive, such as depth, tank pressure, water temperature etc... While this isn't a *huge* amount of entries, nearly every SQL framework I used for it was painfully slow at importing this type of data.

The only crate that I managed to get to work with reasonable performance was [tokio-postgres](https://docs.rs/tokio-postgres/latest/tokio_postgres/).  Itself being lightweight: it doesn't include a lot of features you would expect from a full blown SQL framework like [diesel](http://diesel.rs/) or [SQLx](https://github.com/launchbadge/sqlx). I was OK with that, I don't mind writing a bit of raw SQL, but one thing I thought was missing was some sort of way to create a struct from a database row.

So I set out to make a really simple trait to hydrate a struct from a row. The obvious candidate for this is a procedural macro. Writing procedural macros has been written about before, but I haven't found a terse enough guide that I was happy with. I wanted to document my approach in the odd chance it helps other people.  At the very least I have something to refer to if I need to do this again!

## Aim of the Macro

The Aim of the macro is to be able to write `#[derive(MyTrait)]` and have it fill in the trait implementation at compile time.

In this case I will be deriving a simple trait that looks like this:

```rust
use tokio_postgres::Row;
use anyhow::Error;

pub trait FromRow {
    /// hydrate a struct from a database `Row`
    fn from_row(row: Row) -> Result<Self, Error>
    where
        Self: std::marker::Sized;

}
```

There are a few things I have made the decision not to handle to keep things simple:

* SQLx is already has some great macros for validating against a live schema. I don't want to reimplement this here.
* I don't want to pay the cost of column index lookups for named fields. Instead, we will use indices and expect the queries we write return fields in correct order.

So if I have a struct like this:

```rust
pub struct DiveMetric {
    pub time: i32,
    pub depth: f32,
    pub pressure: Option<f32>,
    pub temperature: Option<f32>,
}
```

It should automatically derive the trait like so:

```rust
impl FromRow for DiveMetric {
    fn from_row(row: Row) -> Result<Self, Error> {
        Ok(Self {
            time: row.try_get(0)?,
            depth: row.try_get(1)?,
            pressure: row.try_get(2)?,
            temperature: row.try_get(3)?,
        })
    }
}
```

## Crate Layout

You will need at minimum 2 crates to do this:

* One crate for the macro itself.  I.e, it has `proc-macro = true` in the Manifest, I called this `macro`
* One crate has the definitions of the trait. I have called this `core`.  This crate will include the macro as a dependency
* Any extra crates that require this can depend on `core`


## Procedural Macro Crate

A Procedural Macro Crate is built at compile time and can influence your source code, pretty much by injecting source code in the form of "tokens" in-line with the rest of your code. 

The [`TokenStream`](https://doc.rust-lang.org/proc_macro/struct.TokenStream.html) interface is very very bare minimum.  You get a list of these tokens and it's up to you to work out what they mean.  Luckily there is a crate out there, [`syn`](https://github.com/dtolnay/syn), for parsing a `TokenStream` into a higher level structure.  Think reflection but at compile time.

So firstly, let's set up the crate manifest:

```toml
[package]
name = "divedb_macro"
version = "0.1.0"
edition = "2021"

[lib]
proc-macro = true

[dependencies]
syn = { version = "2", features = ["full", "parsing"] }
```

This is a standard crate, with the only difference is in that `lib` section: we tell it this is a proc-macro crate.

Then, we are going to create a derive macro with the same name as the trait, i.e, `FromRow`.  We will just create this in `lib.rs` the entrypoint to our crate.

The scaffolding we have to lay out is a method that looks something like this:

```rust
use proc_macro::TokenStream;

#[proc_macro_derive(FromRow)]
pub fn derive_from_row(input: TokenStream) -> TokenStream {
    // do something with the token stream here
}
```

### Parsing the TokenStream

First step is to convert a `TokenStream` into a structure we can work with.  We'll use `syn` to do the heavy lifting here.  There are some helper methods to parse out as a [`DeriveInput`](https://docs.rs/syn/latest/syn/struct.DeriveInput.html) struct:

```rust
let input = parse_macro_input!(input as DeriveInput);
```

Looking at `DeriveInput`, there is the [`Data`](https://docs.rs/syn/latest/syn/enum.Data.html) enum, which is an enum of 3 different types, `Struct`, `Enum` or `Union`. 

We only care about structs, and don't want to derive it for anything else.

So what we can do is an `if let` match on the variant we want, and otherwise error out with some sort of message that will hopefully remind you when you derive it later on.  There are a few ways to error out & most require a `span`: the portion of the `TokenStream` that relates to the error.  I.e, where the red squiggly line will show up. You can also `panic!` if you are shortcutting massively, but I wouldn't recommend it.

I've elected to do a catchall in the case it's not in a structure that I want to support. 

Using [`syn::Error`](https://docs.rs/syn/latest/syn/struct.Error.html), we can provide it a span, and a message.

Since our method needs to return a `TokenStream`, not a `syn::Error`, we can convert the error by injecting a `compile_error!()` in the form of a `TokenStream` by using the [`to_compile_error()`](https://docs.rs/syn/latest/syn/struct.Error.html#method.to_compile_error) function:

```rust
#[proc_macro_derive(FromRow)]
pub fn derive_from_row(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);

    // Do something with the fields here

    // Catchall if we don't match on the structure we don't want
    TokenStream::from(
        syn::Error::new(
            input.ident.span(),
            "Only structs with named fields can derive `FromRow`"
        ).to_compile_error()
    )
}
```

A lot of the examples for proc macros I've seen introduce more methods and structure here, but for brevity I've kept it all in the one method.

With our high level structure out of the way, we can do an `if let` match on finding a struct:

```rust
if let syn::Data::Struct(ref data) = input.data {
    // we know it's a struct

}
```

You'll find that there are actually 3 variants of a struct: Named field structs, Unnamed (tuple) structs and unit structs:

```rust
struct Named {
    field: String
}

struct Unnamed(String);

struct Unit;
```

It might be important if you are fleshing out a derive macro to deal with those other two cases. For this macro we only care about the first variant, the name field struct, not the other two. So we match on that:

```rust
if let Fields::Named(ref fields) = data.fields {
    // Deal with a named-field struct

}
```

### Emitting our TokenStream

From here, we want to iterate over the fields, and for each field emit a bit of code like:

```rust
<field_name>: row.try_get(<field_idx>)?,
```

However: we can't just send a `String` back from our method, we need to provide a `TokenStream`.  This is where the second handy crate comes in: [`quote`](https://github.com/dtolnay/quote). We can use the `quote!` macro to generate code snippets in a mostly-readable fashion and output them as a `TokenStream`.  One thing to note: `quote!` will return a `proc_macro2` `TokenStream`, so you can either call `into()` or use `TokenStream::from`.

So let's iterate over our fields, and for each field, return a snippet, a `TokenStream`, which we can combine later:

```rust
let field_vals = fields.named.iter().enumerate().map(|(i, field)| {
    // grab the name of the field
    let name = &field.ident;
    quote!(#name: row.try_get(#i)?)
});
```

We're using the standard `iter().enumerate()` here to get out an index number of the field and the `field.ident` to reflect the name of the field.

As an alternative, if instead you wanted to use column names instead of column indices, you could adjust this easily:

```rust
let field_vals = fields.named.iter().map(|field| {
    let name = field.ident.as_ref().unwrap();
    let name_string = name.to_string();
    quote!(#name: row.try_get(#name_string)?)
});
```

Ok, now we have a list of fields, and for each field, we have a snippet of source code, the next is to combine them and implement the `FromRow` trait.  Once again we can use `quote!` to do this.

Another thing worth noting is that it's as if the proc macro literally injects a bit of source code in-line where you call derive. This means that if you are going to refer to any structs/crates/modules etc.. it makes things a lot easier to refer to them via their full path.  I.e, don't put `Row`, put `::tokio_postgres::Row` instead.

With that in mind, here's our simple `quote!` statement:

```rust
let name = input.ident;

quote!(
impl ::divedb_core::FromRow for #name {
    fn from_row(row: ::tokio_postgres::Row) -> Result<Self, ::anyhow::Error> {
        Ok(Self {
            #(#field_vals),*
        })
    }
})
```
Most of this is self-explanatory.  The only weird/interesting thing is this part:

```rust
#(#field_vals),*
```

This will just iterate over our field vals, and include each one.


Putting it all together, we get a pretty terse, but powerful proc macro:

```rust
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, DeriveInput, Fields};

/// This derives the `FromRow` trait for structs
/// Requires that the query is in field order, as it just uses row indices
#[proc_macro_derive(FromRow)]
pub fn derive_from_row(input: TokenStream) -> TokenStream {
    // Parse it as a proc macro
    let input = parse_macro_input!(input as DeriveInput);

    if let syn::Data::Struct(ref data) = input.data {
        if let Fields::Named(ref fields) = data.fields {
            let field_vals = fields.named.iter().enumerate().map(|(i, field)| {
                let name = &field.ident;
                quote!(#name: row.try_get(#i)?)
            });

            let name = input.ident;

            return TokenStream::from(quote!(
            impl divedb_core::FromRow for #name {
                fn from_row(row: tokio_postgres::Row) -> Result<Self, anyhow::Error> {
                    Ok(Self {
                        #(#field_vals),*
                    })
                }
            }));
        }
    }

    TokenStream::from(
        syn::Error::new(
            input.ident.span(),
            "Only structs with named fields can derive `FromRow`",
        )
        .to_compile_error(),
    )
}
```

## Core Crate

The core crate implements the trait itself & also exports the macro as well.  If the macro & trait are the same name, you can cheat and `pub use` the macro crate so you don't always have to import the macro & trait separately.

Otherwise the core `lib.rs` is rather simple:

```rust
use anyhow::Error;
// Export our derive macro here.
pub use divedb_macro::*;
use tokio_postgres::Row;

/// Really simple ORM for `tokio_postgres`
pub trait FromRow {
    /// hydrate a struct from a database `Row`
    fn from_row(row: Row) -> Result<Self, Error>
    where
        Self: std::marker::Sized;

    ...
}
```

## Standard Crate

Now we have the macro crate, the core crate, we can simply `use` this to derive on our own types:

```rust
use divedb_core::FromRow;

#[derive(FromRow)]
pub struct DiveMetric {
    pub time: i32,
    pub depth: f32,
    pub pressure: Option<f32>,
    pub temperature: Option<f32>,
}
```

### Expanding the Macro

Remember when I said that these macros pretty much inject more source code in line?  Well you can see that intermediate output rather easily by using [`cargo expand`](https://github.com/dtolnay/cargo-expand).  This will print out the results of the macro expansion and show what code is actually being generated & a *lot* more.  If you are writing something super tricky, it really helps to see this output & makes a great tool for debugging procedural macros.

I.e, for our above `DiveMetric` struct, using column indices:

```rust
impl divedb_core::FromRow for DiveMetric {
    fn from_row(row: tokio_postgres::Row) -> Result<Self, anyhow::Error> {
        Ok(Self {
            time: row.try_get(0usize)?,
            depth: row.try_get(1usize)?,
            pressure: row.try_get(2usize)?,
            temperature: row.try_get(3usize)?,
        })
    }
}
```

And for using the column name variant:

```rust
impl divedb_core::FromRow for DiveMetric {
    fn from_row(row: tokio_postgres::Row) -> Result<Self, anyhow::Error> {
        Ok(Self {
            time: row.try_get("time")?,
            depth: row.try_get("depth")?,
            pressure: row.try_get("pressure")?,
            temperature: row.try_get("temperature")?,
        })
    }
}
```

## Conclusion

As you see, it's not *too* onerous to make your own custom derives.  There is a bit of boilerplate that is needed, but you can write code to write code and keep things simple.

Without greater support for compile time reflection in rust, you can reach for macros to help you remove the repetitive tasks we face as programmers
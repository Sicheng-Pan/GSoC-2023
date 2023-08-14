---
title: Pola-rs in R
---

## Brief intro to the project

[*Polars*](https://www.pola.rs/) is a performant query engine written in *Rust*. It would be great if we could export it to other languages so that more people can use it in their favorite languages! The [*r-polars*](https://rpolars.github.io/) project, as its name suggests, aims to exports the functionalities of *polars* to *R*, but some *polars* features may not behave nicely with *R* codes, while many more remains missing on the *R* side. In short, my goal for this summer is to improve the existing *R* bindings in *r-polars*, export missing features from *polars*, and enhance user experiences in *R* with the help of underlying *Rust* codes.

## Better error handling

> Relevant PR: [#233](https://github.com/pola-rs/r-polars/pull/233)

Error handling could be tricky to done properly and efficiently. It would be especially hard for users to debug their codes if we do not provide informative error messages when errors occur, but it could also be hard for developers to manually guard against all possible errors in the code. Consider the following example:

```R
> f <- function() {
  # Internals not visible by users
  g <- function() {
    h
  }
  g()
}
> f()
Error in g() : object 'h' not found
```

The users could be completely confused by what the error message is saying, and this could be even worse for errors from the non-`R` codes in the library. Luckily, `Rust` has the `Result<T, E>` type that allows us to recover from errors and handle them properly in `Rust` codes, and it is not hard for us to export it to `R` with the help of the [*extendr*](https://github.com/extendr/extendr) crate. 

The most common error type was `E = String`, which suggests that the errors are `String` error messages. We could always manually compose `String` error messages like this:

```rust
fn i64_to_u64(x: i64) -> Result<u64, String> {
  u64::try_from(x).map_err(|e| format!("{e:?}"))
}
```

But this could be very tedious to do when the same error occurs in multiple places, or we want to properly handle a nested error like this:

```rust
fn r_task() -> Result<u64, String> {
  i64_to_u64(-1).map_err(|e| format!("When converting i64 to u64: {e}"))
}
```

To fix such issues, we would need a layered structure of errors, and we hope to make the construction of errors easy for developers and the appearance of errors nice for users. So I implemented our own error layer type `Rctx`, error type `RPolarsErr`, and result type `RResult<T> = Result<T, RPolarsErr>`:

```rust
#[derive(thiserror::Error)]
pub enum Rctx {
  #[error("Got value [{0}]")]
  BadVal(String),
  #[error("When {0}")]
  When(String),
  // And many more
}

pub struct RPolarsErr {
  contexts: VecDeque<Rctx>,
  // And other auxiliary fields
}

pub type RResult<T> = core::result::Result<T, RPolarsErr>;
```

Notice that I used the [*thiserror*](https://github.com/dtolnay/thiserror) crate to generate nice error messages. The functionalities of `RPolarsErr` are similar to the [*anyhow*](https://github.com/dtolnay/anyhow) crate, but I did not use it since I could not find a way to extract the contexts in `anyhow::Error`, and I could not find a way to directly implement the `anyhow::Context` trait as it is sealed within the crate. I also defined and implemented a `WithRctx<T>` trait to facilitate the construction of `RResult<T>`:

```rust
pub trait WithRctx<T> {
  fn bad_val(self, val: impl Into<String>) -> RResult<T>;
  fn when(self, env: impl Into<String>) -> RResult<T>;
  // And many more
}

impl<T, E: Into<RPolarsErr>> WithRctx<T> for core::result::Result<T, E> {
  // Implement the trait for all Results convertible to RResult
}
```

Later, I implemented the `std::fmt::Display` trait for `RPolarsErr`, and updated the `R` wrappers to handle `RResult`. Finally, we have a decent mechanism to handle errors for this project! The code for `i64_to_u64` and `r_task` in `Rust` can be as simple as:

```rust
fn i64_to_u64(x: i64) -> RResult<u64> {
  u64::try_from(x).bad_val(x.to_string()).when("converting from i64 to u64")
}

fn r_task() -> RResult<u64> {
  i64_to_u64(-1).when("deliberately making an error in Rust")
}
```

And on the `R` side:

```R
> r_task() |> unwrap("try unwrapping an error")
Error: Execution halted with the following contexts
   0: In R: try unwrapping an error
   0: During function call [unwrap(r_task(), "try unwrapping an error")]
   1: When deliberately making an error in Rust
   2: When converting from i64 to u64
   3: Got value [-1]
   4: TryFromIntError(())
```

Now it's the time to refactor the existing error handling codes for the project with the new mechanism! Although this is still an on-going process, many common functions have already been updated! For example:
```R
> polars::pl$scan_ipc(0)
Error: Execution halted with the following contexts
   0: In R: in pl$scan_ipc:
   0: During function call [polars::pl$scan_ipc(0)]
   1: The argument [path] caused an error
   2: Expected a value of type [alloc::string::String]
   3: Got value [Rvalue: 0.0, Rsexp: Doubles, Rclass: ["numeric"]]
```

## Background query execution

> Relevant PR: [#311](https://github.com/pola-rs/r-polars/pull/311)

## More *polars* features

### `LazyFrame` schema

> Relevant PR: [#250](https://github.com/pola-rs/r-polars/pull/250)

### `LazyFrame` profiling and optimizaion toggles

> Relevant PR: [#323](https://github.com/pola-rs/r-polars/pull/323)

### `LazyFrame` index column

> Relevant PR: [#329](https://github.com/pola-rs/r-polars/pull/329)

### `LazyFrame` sink stream

> Relevant PR: [#343](https://github.com/pola-rs/r-polars/pull/343)

### More statistic functions

> Relevant PR: [#351](https://github.com/pola-rs/r-polars/pull/351)

## Future works

## What I've learned

## Acknowledgement
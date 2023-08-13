---
title: Pola-rs in R
---

## Brief intro to the project

[*Polars*](https://www.pola.rs/) is a performant query engine written in *Rust*. It would be great if we could export it to other languages so that more people can use it in their favorite languages! The [*r-polars*](https://rpolars.github.io/) project, as its name suggests, aims to exports the functionalities of *polars* to *R*, but some *polars* features may not behave nicely with *R* codes, while many more remains missing on the *R* side. In short, my goal for this summer is to improve the existing *R* bindings in *r-polars*, export missing features from *polars*, and enhance user experiences in *R* with the help of underlying *Rust* codes.

## Better error handling

## Background query execution

## More *polars* features

## Future works

## What I've learned

## Acknowledgement

Some R code here:
```R
lf <- pl$LazyFrame(mtcars)
```

Some Rust code here:
```rust
fn what(x: i64) -> RResult<i64> {
  Ok(x)
}
```

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

The users could be completely confused by what the error message is saying, and this could be even worse for errors from the non-*R* codes in the library. Luckily, *Rust* has the `Result<T, E>` type that allows us to recover from errors and handle them properly in *Rust* codes, and it is not hard for us to export it to *R* with the help of the [*extendr*](https://github.com/extendr/extendr) crate. 

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

To fix such issues, we would need a layered structure of errors, and we hope to make the construction of errors easy for developers and the appearance of errors nice for users. So I implemented our own error layer type `Rctx`, error type `RPolarsErr`, and result type `RResult<T>`:

```rust
#[derive(thiserror::Error)]
pub enum Rctx {
    #[error("Got value [{0}]")]
    BadVal(String),
    #[error("When {0}")]
    When(String),
    // And more
}

pub struct RPolarsErr {
    contexts: std::collections::VecDeque<Rctx>,
    // And other auxiliary fields
}

pub type RResult<T> = Result<T, RPolarsErr>;
```

Notice that I used the [*thiserror*](https://github.com/dtolnay/thiserror) crate to generate nice error messages. The functionalities of `RPolarsErr` are similar to the [*anyhow*](https://github.com/dtolnay/anyhow) crate, but I did not use it since I could not find a way to extract the contexts in `anyhow::Error`, and I could not find a way to directly implement the `anyhow::Context` trait as it is sealed within the crate. I also defined and implemented a `WithRctx<T>` trait to facilitate the construction of `RResult<T>`:

```rust
pub trait WithRctx<T> {
    fn bad_val(self, val: impl Into<String>) -> RResult<T>;
    fn when(self, env: impl Into<String>) -> RResult<T>;
    // And more
}

impl<T, E: Into<RPolarsErr>> WithRctx<T> for Result<T, E> {
    // Implement the trait for all Results convertible to RResult
}
```

Later, I implemented the `std::fmt::Display` trait for `RPolarsErr`, and updated the *R* wrappers to handle `RResult`. Finally, we have a decent mechanism to handle errors for this project! The code for `i64_to_u64` and `r_task` in *Rust* can be as simple as:

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

It is common for *polars* users to query large datasets, and it is likely that such tasks will take a while. This could be a problem for users who run the code in an interactive fashion with interpreted languages like *R* and *Python*, as their console will stuck and the users have to wait until their large queries finish off. If we could offload the user tasks in the background and give back the control of the session, the users could continue to work on other independent routines, and only poll the results when necessary. The most intuitive solution from my perspective is to spawn a background thread to handle the user tasks in background, and return the corresponding thread handle to the user. This part is fairly straight forward to implement:

```rust
pub struct RThreadHandle<T> {
    handle: Option<std::thread::JoinHandle<T>>,
}  
```

I also wrapped a few methods from `thread::JoinHandle`, such as `join` and `is_finished`, and exported to the *R* side so that users could manipulate the join handle. I also added a additional toggle option `in_background` in the `collect` method for `LazyFrame`, so that users have the option to run a query in background like this:

```R
> lf <- polars::pl$LazyFrame(mtcars)$select(polars::pl$col("mpg") * 0.42)
> th <- lf$collect(collect_in_background = TRUE)
> th$join()
shape: (32, 1)
┌───────┐
│ mpg   │
│ ---   │
│ f64   │
╞═══════╡
│ 8.82  │
│ 8.82  │
│ 9.576 │
│ 8.988 │
│ ...   │
│ 6.636 │
│ 8.274 │
│ 6.3   │
│ 8.988 │
└───────┘
```

It seems that I have already implemented all necessary functionalities to have the queries executable in background, but there is one caveat: if the queries involves native *R* functions, such as in `map` and `apply`, we have to find an available *R* interpreter to evaluate the *R* expressions, and *R* do not have proper multi-threading support. Naively we could reuse the existing user *R* session in such scenarios, but this could defeat the purpose of running the queries in background, and the shared single-threaded *R* interpreter could become the bottleneck of query performance since *polars* always uses multiple threads for computing if possible. We need a better solution for this.

The only alternative option I have in mind is to spawn new *R* sessions, and use inter-process-communication (IPC) to transfer data across processes. I chose the [ipc-channel](https://github.com/servo/ipc-channel) crate for this purpose. To setup the connection between processes, I have to spawn a *R* process in *Rust* code, and invoke my custom *Rust* task handler in the child *R* interpreter:

```rust
use ipc_channel::ipc;

pub fn handle_background_request(server_name: String) -> RResult<()> {
    let rtx = ipc::IpcSender::connect(server_name)?;

    let (tx, rx) = ipc::channel::<RIPCJob>()?;
    rtx.send(tx)?;

    while let Ok(job) = rx.recv() {
        job.handle()?;
    }
    Ok(())
}
```

Notice that I first use a one-shot channel identified by its `String` name to setup the actual channel over which I can dispatch tasks. The `RIPCJob` is a enum defining all kinds of tasks that needs a background *R* interpreter to run and how to handle them:

```rust
pub enum RIPCJob {
    RMapSeries {
        raw_func: Vec<u8>,
        raw_series: ipc::IpcSharedMemory,
        collector: ipc::IpcSender<RResult<ipc::IpcSharedMemory>>,
    },
    // And more
}

impl RIPCJob {
    pub fn handle(self) -> RResult<()> {
        match self {
            Self::RMapSeries {
                raw_func,
                raw_series,
                collector,
            } => {
                // Handle the task to map a Polars series with a R function
            }
            // And more
        }
    }
}
```

Now we need to serialize and deserialize all relevant information across the IPC channel. For *R* functions and objects, I use the built-in *R* functions `serialize` and `deserialize` to them to bit vectors (i.e. `Vec<u8>`). For *polars* series, my initial implementation was first serializing it to bits and then transfering the bits across the channel, but the performance was not ideal when the series contain a lot of data. As suggested by the code above, eventually I chose to create shared memory across processes, and then send the information about the shared memory across the channel. In this way I can avoid transfering large amount of bits across the channel. But still, I have to serialize and deserialize the series, as I am unsure about the underlying structure of the series and could not directly create shared memory upon them, so there's still space for improvements.

With the functionalities implemented above, we could already offload the user tasks to background, even with *R* functions involved. A minor problem is that we're always creating *R* processes when needed, and throwing them away when finished. This could be improved with a pool of live *R* processes where we could directly fetch and store *R* processes, and this could eliminate the unecessary (and costly) construction and destruction of *R* processes. I also implements common routines to shrink/expand the pool, lease/return the *R* process, and directly dispatching a task to an available *R* process. Initially, the pool only had a soft cap: if the cap is reached, new *R* processes could still be created but will be destroyed on finish. My mentor suggests that this may lead to unexpected behavior in performance from the users' perspective, and could make it hard for the users to limit the usages of additional *R* processes by the library. He refactored the implementation of the pool so that it now has a hard cap, and threads have to wait if the cap is reached while all existing *R* processes are in-use. Thanks a lot for his help!

Finally, we have a way to execute large queries in the background! For example:

```R
> lf <- polars::pl$LazyFrame(mtcars)$select(polars::pl$col("mpg")$map(\(x) x * 0.42, in_background = TRUE)$alias("kpl"))
> th <- lf$collect(collect_in_background = TRUE)
> th$is_finished()
[1] TRUE
> th$join()
shape: (32, 1)
┌───────┐
│ kpl   │
│ ---   │
│ f64   │
╞═══════╡
│ 8.82  │
│ 8.82  │
│ 9.576 │
│ 8.988 │
│ ...   │
│ 6.636 │
│ 8.274 │
│ 6.3   │
│ 8.988 │
└───────┘
```

Running the queries in background could lead to significant performance boosts! You could run the [script](https://github.com/pola-rs/r-polars/blob/main/inst/misc/benchmark_rbackground.R) yourself to test it out!

And the error messages from the background also play nicely with our new error handling pipeline:

```R
> lf <- polars::pl$LazyFrame(mtcars)$select(polars::pl$col("mpg")$map(\(x) y, in_background = TRUE))
> th <- lf$collect(collect_in_background = TRUE)
> th$join()
Error: Execution halted with the following contexts
   0: During function call [th$join()]
   1: When trying to map a polars series with R function in the background R process
   2: EvalError(lang!(function (x) y, ExternalPtr.set_class(["Series"]))
```

## More *polars* features

I've also exported various *polars* features to *R* along the way. Here is a list of them:

### `LazyFrame` columns, dtypes, schema

> Relevant PR: [#250](https://github.com/pola-rs/r-polars/pull/250)

### `LazyFrame` profile, optimizaion toggles

> Relevant PR: [#323](https://github.com/pola-rs/r-polars/pull/323)

### `LazyFrame` index column

> Relevant PR: [#329](https://github.com/pola-rs/r-polars/pull/329)

### `LazyFrame` sink stream

> Relevant PR: [#343](https://github.com/pola-rs/r-polars/pull/343)

### More statistic functions: covariance, correlation, and their rolling versions

> Relevant PR: [#351](https://github.com/pola-rs/r-polars/pull/351)

## Status of the project

All of my works mentioned above have been merged to the main branch of the project, and they should be available in its next release ([0.8.0](https://github.com/pola-rs/r-polars/blob/main/NEWS.md)). Hopefully they could bring better experiences for the *rpolars* users!

In terms of future works, the *rpolars* project is not yet complete. Although I've implemented a few of the missing features from *polars*, many more still remains, such as `fold` and `reduce` for dataframes. Also more improvements could be made for this project (e.g. parallized I/O between *R* and *Rust*). I would definitely help contribute to this project if time permits. 

## Acknowledgement

Thanks a lot for the *rpolars* and *polars* community, especially my mentor [Søren Havelund Welling](https://github.com/sorhawell)! They provided tremendous help and support for my GSoC project, and I really appreciate their comments and suggestions for my project!
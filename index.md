---
title: Pola-rs in R
---

## Brief intro to the project

[*Polars*](https://www.pola.rs/) is a performant query engine written in *Rust*. It would be great if we could export it to other languages so that more people can use it in their favorite languages! The [*r-polars*](https://rpolars.github.io/) project, as its name suggests, aims to exports the functionalities of *polars* to *R*, but some *polars* features may not behave nicely with *R* codes, while many more remains missing on the *R* side. In short, my goal for this summer is to improve the existing *R* bindings in *r-polars*, export missing features from *polars*, and enhance user experiences in *R* with the help of underlying *Rust* codes.

## Better error handling

> Relevant PR: [#233](https://github.com/pola-rs/r-polars/pull/233)

Error handling can be tricky to done properly and efficiently. It would be especially hard for users to debug their codes if we do not provide informative error messages when errors occur, but it is hard for developers to manually guard against all possible errors in the code. We would like to have a mechanism that is informative enough for users and simple enough for developers.

### Motivting example

Consider the following:

```R
> f <- \() { g <- \(x) { x + 1 }; g("?") }
> f()
Error in x + 1 : non-numeric argument to binary operator
```

The users could be completely confused by what the error message is saying if the internals of the function `f` is not visible, and this could be even worse for errors from the non-*R* codes in the library. A better error message here could look like:

```R
Error encountered:
    When calling `f`:
        When calling `g`:
            When evaluating `x + 1`:
                Expected numeric argument `x`, but got string value "?"
```

Of course this could be achieved in plain *R* with the condition system (e.g. `tryCatch`), but this approach is not likely to be scallable. It would require extra efforts for developers to consider what could go wrong, as well as how to handle them properly, whenever they are writing new codes, and it is likely that some edge cases will be missed. Always using error handlers (e.g. `tryCatch`) to wrap around potentially errorneous codes could also break the overall sanity of the codebase.

Moreover, even if we have a better error message, the users may not have a easy way to recover from the error by its causes. For example, the users could tolerate certain causes of error because those are less severe, but it is hard for them to check this against pure error message strings. It would be better if the error could contain all relevant information in an accessible way.

### Requirements for a good error handling mechanism

Inspired by the example above, we hope that a good error handling system should be:

- Informative for users, so that they can easily know where things go wrong
- Structural for users, so that they can easily extract relevant information
- Explicit for developers, so that they know where to handle potential errors
- Simple for developers, so that error handling will not be costly during development

### `Result<T, E>` in *Rust*

To implement an error handling system satisfying the above criterias, we could use either *R* or *Rust* to achieve this. I choose *Rust* because it has the `Result<T, E>` type, which is designed for recoverable errors, and the related traits (i.e. interfaces) implemented for `Result<T, E>` can greatly facilitate us to handle errors easily. We can use it as the return type for our functions, so that the developers knows when to expect return values that contains potential errors, and it is not hard for us to export it to *R* with the help of the [*extendr*](https://github.com/extendr/extendr) crate.

In the following examples, I will show how we could use `Result<T, E>` to make error handling easier for developers. A value in `Result<T, E>` type is either a `Ok(t: T)`, which indicates a successful result with value `t` of type `T`, or an `Err(e: E)`, which indicates an errorneous result with error `e` of type `E`. For simplicity, let's set the error type to `E = String` for now, which suggests that the errors are `String` error messages. We will change this later. Let's first construct a function that should convert a 64-bit signed integer to a 64-bit unsigned integer, but actually always return an error message because it is not implemented yet:

```rust
fn i64_to_u64(x: i64) -> Result<u64, String> {
    Err(String::from("nobody implements this!"))
}
```

We just stated that its return type is `Result<u64, String>`, so the developers using this function will know that this function could produce errors, and the compiler will make sure that the developers have explicitly handled the potential error (e.g. ignoring it). Here's how this could be used:

```rust
fn ignore_error() -> u64 {
    i64_to_u64(0).unwrap() + 1
}
```

The `unwrap` method for `Result<T, E>` will assume that the result value is `Ok(t)` try to fetch the internal value `t`, but it will panic (i.e. stop the program with an unrecoverable error) if the result value is actually `Err(e)`, which is exactly the case in our example as `i64_to_u64` is guaranteed to generate an `Err(e)`. We can do better:

```rust
fn handle_error() -> Result<u64, String> {
    Ok(i64_to_u64(0)? + 1)
}
```

This function looks very similar to the previous one. Both `?` and `unwrap` tries to get the successful result from `i64_to_u64`, but with a major difference: the `?` will return the error as the return value, but `unwrap()` will panic immediately. In this way we can propagate the error to the next level.

We can also modify the error on the fly:

```rust
fn handle_error_escalate() -> Result<u64, String> {
    Ok(i64_to_u64(0).map_err(|e| format!("When converting signed 0 to unsigned integer: {e}"))? + 1)
}
```

The error message has been prepended with some contexts, as suggested by the format string `format!(...)`. 

### Classful, layered errors

The vanilla `Result<T, E = String>` can be used to construct decent error message and can already improve the error handling experiences for developers, but it's not enough. As suggested above, error messages alone is not very accessible for users, while manually formatting nested error message is bothering. It's time to invent our own error type `E`!

So I implemented the following:

- `Rctx`, which is the building block of our errors
- `RPolarsErr`, which is basically a stack of `Rctx`s, just like stack traces 
- `RResult<T>`, which is an alias of `Result<T, E = RPolarsErr>`, so we do not need to specify `E` every time

```rust
#[derive(thiserror::Error)]
pub enum Rctx {
    #[error("Possibly because {0}")]
    Hint(String),
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

I used the [*thiserror*](https://github.com/dtolnay/thiserror) crate to generate nice error messages. The functionalities of `RPolarsErr` are similar to the [*anyhow*](https://github.com/dtolnay/anyhow) crate. I did not use it because it is designed to merge errors of different types easily and then display them properly, but not maintaining these errors in their original forms. Then an `anyhow::Error` is equivalent to a properly constructed error message, as there is no other way to inspect its internals, and this will be hard for users to extract useful information from it.

I also defined and implemented a `WithRctx<T>` trait to facilitate the construction of `RResult<T>`:

```rust
pub trait WithRctx<T> {
    // Push a `Rctx::Hint` to the error if there is one
    fn hint(self, val: impl Into<String>) -> RResult<T>;
    // Push a `Rctx::When` to the error if there is one
    fn when(self, env: impl Into<String>) -> RResult<T>;
    // And more
}

impl<T, E: Into<RPolarsErr>> WithRctx<T> for Result<T, E> {
    // Implement the trait for all `Result<T, E>` convertible to `RResult<T>`
}

fn rerr<T>() -> RResult<T> {
    // Generate a `RResult<T>::Err` with an empty `RPolarsErr`
}
```

An important note is that the methods in `WithRctx<T>` have zero-costs: the `Rctx`s are created and appended only when necessary. If the result is `Ok(...)`, it will be directly returned as is!

With everything above, we have already addressed the concerns for developers, and we have a simple mechanism to construct and handle errors! Checkout the following example:

```rust
fn i64_to_u64_nice(x: i64) -> RResult<u64> {
    rerr().hint("nobody implements this!").when("converting from i64 to u64")
}

fn handle_error_nice() -> RResult<u64> {
    Ok(i64_to_u64_nice(0).when("trying out conversion in Rust")? + 1)
}
```

When `handle_error_nice` is called, we will construct a stack of errors. The only remaining tasks are displaying it properly to the users and provide a structural output so that the users could easily access the individual `Rctx`s.

Thus I implemented the `std::fmt::Display` trait for `RPolarsErr`, which can convert `RPolarsErr` to a pretty string. I also exposed an `RPolarsErr` method in *R* so that developers can directly access the `Rctx`s as a list, and updated the *R* wrappers to handle `RResult`. With all these implemented, we could test it out:

```R
> handle_error_nice() |> unwrap("try unwrapping an error")
> result <- handle_error_nice()
> result |> unwrap("try unwrapping an error in R")
Error: Execution halted with the following contexts
   0: In R: try unwrapping an error in R
   0: During function call [unwrap(result, "try unwrapping an error in R")]
   1: When trying out conversion in Rust
   2: When converting from i64 to u64
   3: Possibly because nobody implements this!
> result$err$contexts()
$When
[1] "trying out conversion in Rust"

$When
[1] "converting from i64 to u64"

$Hint
[1] "nobody implements this!"
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

It is common for *polars* users to query large datasets, and it is likely that such tasks will take a while. This could be a problem for users who run the code in an interactive fashion with interpreted languages like *R* and *Python*, as their console will block and the users have to wait until their large queries finish off. If we could offload the user queries in the background and give back the session to the users, they could continue to work on other independent routines, and only poll the results when necessary.


### Background threads for query offloading

The most intuitive solution from my perspective is to spawn a background thread to handle the user tasks in background, and return the corresponding thread handle to the user, similar to a future. This part is fairly straight forward to implement:

```rust
pub struct RThreadHandle<T> {
    handle: Option<std::thread::JoinHandle<T>>,
}  
```

I also wrapped a few methods from `thread::JoinHandle`, such as `join` and `is_finished`, and exported to the *R* side so that users could manipulate the join handle. I also added a additional toggle option `collect_in_background` in the `collect` method for `LazyFrame`, so that users have the option to run a query in background like this:

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

### Background *R* processes for background *R* evaluations

It seems that I have already implemented all necessary functionalities to have the queries executable in background, but there is one caveat: if the queries involves native *R* functions, such as in `map` and `apply`, we have to find an available *R* interpreter to evaluate the *R* expressions, and *R* do not have proper multi-threading support.

To solve this problem, naively we could reuse the existing user *R* session, but this could defeat the purpose of running the queries in background, as the user session will be interrupted and blocked by the hidden evaluations. Moreover, the shared single-threaded *R* interpreter could become the bottleneck of query performance since *polars* always uses multiple threads for computing if possible. We need a better solution for this.

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

Now we need to serialize and deserialize all relevant information across the IPC channel.

For *R* functions and objects, I use the built-in *R* functions `serialize` and `deserialize` to them to bit vectors (i.e. `Vec<u8>`).

For *polars* series, my initial implementation was first serializing it to bits and then transfering the bits across the channel, but the performance was not ideal when the series contain a lot of data. As suggested by the code above, eventually I chose to create shared memory across processes, and then send the information about the shared memory across the channel. In this way I can avoid transfering large amount of bits across the channel.

But still, I have to serialize and deserialize the *polars* series to and from bits, and pass it through a buffered channel in shared memory. This is because I am unsure about the underlying structure of the series, as a result of which I could not directly create shared memory upon them, so potentially there's still space for improvements.

### Background *R* process pool for better overheads

With the functionalities implemented above, we could already offload the user tasks to background, even with *R* functions involved, but we also have a minor problem: we're always creating *R* processes when needed, and throwing them away when finished.

This could be improved with a pool of live *R* processes where we could directly fetch and store *R* processes, and this could eliminate the unecessary (and costly) construction and destruction of *R* processes.

I also implements common routines to shrink/expand the pool, lease/return the *R* process, and directly dispatching a task to an available *R* process. Initially, the pool only had a soft cap: if the cap is reached, new *R* processes could still be created but will be destroyed on finish. My mentor suggests that this may lead to unexpected behavior in performance from the users' perspective, and could make it hard for the users to limit the usages of additional *R* processes by the library. He refactored the implementation of the pool so that it now has a hard cap, and threads have to wait if the cap is reached while all existing *R* processes are in-use. Thanks a lot for his help!

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

Running the queries in background could lead to significant performance boosts if *R* evaluations are the bottleneck of the queries. You could run the [script](https://github.com/pola-rs/r-polars/blob/main/inst/misc/benchmark_rbackground.R) yourself to see the performance of background execution under different scenarios.

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
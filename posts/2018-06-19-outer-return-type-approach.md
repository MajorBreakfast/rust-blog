# Why Rust's async functions should use the outer return type approach

Async Rust is currently experiencing an interesting time. The [pull request](https://github.com/rust-lang/rust/pull/51580) that adds initial support for async functions to the compiler is open, [pinning is maturing](https://github.com/rust-lang/rust/issues/49150), the essential parts of the task system needed to execute futures has been added into [libcore](https://github.com/rust-lang/rust/tree/master/src/libcore) and [version 0.3 of the futures crate](https://github.com/rust-lang-nursery/futures-rs/tree/0.3) which works with the new pinning-based approach is being actively worked on. Everything is coming together very nicely, and I firmly believe, once the dust settles, Rust will be one of the most compelling languages to write server applications in because it will be blazing fast and ergonomic to use.

The approach for async functions that is currently being implemented does, however, contain one design choice that I disagree with. It's about how the function signature of an async function looks like and I think that the chosen approach doesn't fit in well with the rest of the language. In this blog post I want to fully explain what it is that I'd like to do differently and more importantly why I think the other way is better.

Now, you might ask: *Why bring this up now and not during the RFC discussion?* The answer is simple: When I read the RFC I agreed with the reasons presented in it. Now, however, after seeing the various implications of the decision, I do not agree anymore and therefore I write this blog post to make my concerns known.

This post is mostly a summary of a [thread about async functions](https://internals.rust-lang.org/t/pre-pre-rfc-async-methods-bounding-async-fns/7656) on the Rust internals forum. I think this post should be a nice read even if you've read the thread.

I will present the reasons for and against both sides as clearly as possible. Feel free to contact me on the internals forum or on Discord if you think that I misrepresented something. However, keep in mind that this blog post is an opinion piece. Therefore, each point always includes my personal assessment of how important I feel it is and my final verdict is an opinion based on these assessments.

*Ah and yes, I know that this is just a GitHub repository and not a real blog. This is my first blog post ever. Until now I never felt the need to write a blog post. I'll probably set up a real website at some point. Until then, enjoy reading on GitHub! :-)*

## The two approaches

Here are the two approaches:

**Inner return type approach**
```rust
async fn foo() -> i32 { ... }
```

**Outer return type approach**
```rust
async fn foo() -> impl Future<Output = i32> { ... }
```

As you can see, the difference is in how the return type is defined. Keep in mind that both approaches define the same async function and only the notation used to define it is different.

- The "inner" return type approach leaves the `impl Future` part implicit and lists the `Output` associated type of the future as the return type in the function signature. This is the approach that was specified in the async/await RFC and is currently being implemented. It is called "inner" return type approach because the type "inside" the future is specified directly in the function signature.
- The "outer" return type approach on the other hand specifies `impl Future` explicitly and uses the usual notation to specify the `Output` associated type. This is the approach that I believe is the better choice.
- Both approaches have in common that neither specify any lifetimes. Instead, the returned future always captures all input lifetimes. (This is required because when an async function is called, it returns its future immediately without doing any work. Therefore, the returned future needs to hang on to all input values because they're needed later when the future is polled and consequently the async function's body is executed.)

## Learnability & Compactness

The signature of the outer return type approach is more complex. It makes use of `impl Trait` and an associated type (`Output`). The inner return type approach, on the other hand, is simpler because it makes use of less concepts and it is shorter.

```rust
// Inner: Simpler
async fn foo() -> i32 { ... }

// Outer: Usage of `impl Trait` and `Output` associated type
async fn foo() -> impl Future<Output = i32> { ... }
```

This round will go to the inner return type approach because I agree that it is less complex and therefore mildly easier to learn. However, I also believe that the outer return type approach is simple to learn as well:
- The notation makes the general meaning quite clear. It is not necessary to understand what exactly `impl Future` and `Output` are in order to use the notation.
- Both `impl Trait` and associated types are features that every Rust developer has to learn rather sooner than later. `impl Trait` is currently still new, but I think it will become frequently used soon because it was an eagerly awaited feature. Associated types are already used frequently because iterators make use of them.

And even if a user gets the return type wrong, the compiler can be helpful:

```
error[E????]: unexpected async function return type
  --> src/main.rs:42:38
   |
42 | async fn foo() {
   |               ^ expected `-> impl Future<Output = ()>`
   |
   = note: async functions must return `impl Future`
```

I think learnability is in the green for both approaches with the inner return type approach slightly in the lead. Learnability is always an important goal for all Rust features, but I think neither approach needs to worry about being too complicated.

**Winner:** Inner return type approach

## Conservativeness & Consistency

Next, I'd like to talk about how much the language is changed by introducing async functions with either syntax.

Putting `async` in front of a function has two effects:

- **Transforms function body:** Both approaches transform the function body into an anonymous type which implements `Future`. This body transformation not only happens for async functions, but also for async blocks and async closures. It is the central purpose of the `async` keyword.

- **Transforms function signature:** Since Rust makes explicit signatures mandatory for all functions, async functions, unlike async blocks and async closures, require a signature. Both approaches transform the signature in some way.

Let's see how the signature is transformed by each approach. First the outer return type approach:

```rust
// Outer return type approach
async fn foo(x: &str, y: &str) -> impl Future<Output = i32> { ... }

// Outer return type approach with explicit lifetime
async fn foo<'a>(x: &'a str, y: &'a str) -> impl Future<Output = i32> + 'a { ... }

// Outer return type approach with explicit lifetime ('in)
async fn foo(x: &str, y: &str) -> impl Future<Output = i32> + 'in { ... }
```

(**About `'in` lifetime:** `'in` is intended as a shorthand for the combination of all input lifetimes and meant to be available inside function signatures of all kinds, not just async functions. It was first proposed inside a [comment](https://internals.rust-lang.org/t/pre-pre-rfc-async-methods-bounding-async-fns/7656/47?u=majorbreakfast) by @aturon and has not yet been formally proposed via an RFC. It's however a nice idea and therefore I'm using it here.)

The signature produced by the outer return type approach is very similar to other function signatures in Rust. The only difference is that the `async` keyword makes it so that the lifetime elision rules apply differently. And there's a reason for this: The future needs to capture all input lifetimes because it needs all inputs later when it runs. There is only one possible choice for the lifetime parameters and therefore we might as well leave the lifetime params implicit. Explicit lifetime parameters should probably be allowed, but they should be entirely optional.

Now, the inner return type approach:
```rust
// Inner return type approach
async fn foo(a: &i32, b: &i32) -> i32 { ... }
```

The inner return type approach goes far beyond what the outer return type approach does. While the outer return type approach can be seen as a simple amendment to the lifetime elision rules, the inner return type approach applies a more invasive transformation to the signature. It transforms what the actual return type of the function is. Repercussions of this transformation are that it renders some of Rust's features unusable - more on that later. For now, let's focus on the transformation itself.

- **Unlike other keywords** Rust has already two keywords that can be put in front of a function: `const` and `unsafe`. But neither of them transforms the signature. Both keywords only affect what you can do in the body.

- **Implicitness:** The inner return type approach intentionally hides that the function returns a future. This is on the one hand beneficial because it enables a compelling mental model which I will present in more detail later in this post. On the other hand async functions are just normal functions that return a future and the signature should spell this out.

- **Inconsistency with other `impl Trait` uses:** The inner return type approach makes the async function signatures inconsistent with other uses of `impl Trait`. For example `impl Trait` can also be used in conjunction with `Iterator`s:
  ```rust
  fn foo() -> impl Iterator<Item = i32> { ... }
  ```
  Should this really be different for async functions? Should we introduce iter fns next?
  ```rust
  iter fn foo() -> i32 { ... }
  ```
  Note: Yes, I know that I'm trolling a bit ^^'

- **Inconsistency with the initialization pattern**: First a quick reminder what the "initialization pattern" is: It splits a function into a part that is executed synchronously and a part that runs asynchronously. The synchronous part is useful when dealing with data conversion and temporary borrows. Here's how it looks like:

  ```rust
  fn foo<'a>(...) -> impl Future<Output = i32> + 'a {
    // Synchronous initialization work

    async { ... } // Async block is executed asynchronously
  }
  ```

  Since the initialization pattern uses a normal function, explicit lifetimes are required. These explicit lifetimes give us precise control over what lifetimes are captured inside the future and which borrows end immediately when the function returns.

  ```rust
  // Inner return type approach
  async fn foo(...) -> i32 { ... }

  // Outer return type approach
  async fn foo(...) -> impl Future<Output = i32> { ... }

  // Equivalent function using initialization pattern
  fn foo<'a>(...) -> impl Future<Output = i32> + 'a {
    // No sync work

    async { ... } // Async block
  }
  ```

  The inner return type approach introduces an unnecessary notational split between how the initialization pattern looks and how async function signatures look like. It fails to indicate that these two notations are very closely related. The outer return type approach on the other hand clearly indicates that every async function can be written in the initialization pattern style by moving the function body into an async block.

I consider the consistency that the outer return type approach offers very valuable. This round goes to the outer return type approach.

**Winner:** Outer return type approach

## Mental model: Async as a modifier

The previous section was clearly geared against the inner return type approach. To balance it out again, this section presents the main argument for the inner return type approach: The mental model it offers.

With the inner return type approach, entering the async world is as easy as putting `async` in front of the function:
```rust
fn foo() -> i32 { ... } // Sync

async fn foo() -> i32 { ... } // Async
```

And that's it. It's immediately valid Rust. Now...
- instead of evaluating to `T`, the function evaluates to `impl Future<Output = T>`
- and `await` can be used inside the body

This mental model relies on the fact that the sync and async function signatures are almost identical.

This makes the bar of entry to the powerful world of async Rust very low. One single keyword is able to change the execution model from synchronous to delayed. When working within a framework, e.g. a web server framework, it is even possible to largely ignore that futures are involved under the hood. Instead, we can look at it from a simpler perspective and rely on an intuitive mental model to write code that is compact and easy to read.

```rust
// Inner return type approach
async fn foo() { ... }

// Outer return type approach
async fn foo() -> impl Future<Output = ()> { ... }
```

I consider the mental model that the inner return type approach offers valuable. It doesn't come for free because it, as already mentioned in the preceding section, has arguably some downsides as well. Also, the mental model isn't so complete because there are plenty of functions that return manual future implementations and there are scenarios that require the initialization pattern, so knowing about the future trait is more or less mandatory. It's however positioned in a less central way. And this, depending on how you look at it, can be seen as a good thing. If the story ended here, I'd be okay with Rust using either approach.

**Winner:** Inner return type approach

## Compatibility with abstract types

However, the story doesn't end here. There exist two Rust features that are not compatible with the inner return type approach. In this section and the next I will discuss these features.

One of those features is "abstract types" (defined in [RFC2071](https://github.com/rust-lang/rfcs/blob/master/text/2071-impl-trait-type-alias.md)) for which implementation work is currently in progress ([tracking issue](https://github.com/rust-lang/rust/issues/34511)). An abstract type makes it possible to give an existential type a name by using it in place of `impl Trait`:

```rust
// Abstract types & outer return type approach

abstract type Foo<'a>: Future<Output = i32> + 'a;
async fn foo() -> Foo<'in> { ... }
```

The inner return type approach is incompatible with abstract types because there is no `impl Trait` that an abstract type could replace. To better understand the situation, let's have a look at the alternatives:

First, it is of course possible to introduce a "special" syntax to make it possible:

```rust
// Abstract types & inner return type approach

abstract type Foo<'a>: Future<Output = i32> + 'a;
async fn foo() -> outer Foo<'in> { ... }
// `outer` tells the compiler that the abstract type is for the outer return type
```

Second, one might suggest that a different syntax which achieves the same thing could be introduced instead:
```rust
// ::Output syntax & inner return type approach

async fn foo() -> i32 { ... }
type Foo<'a> = foo::Output<'a>; // func::Output syntax
```

There are however multiple open questions/problems:
- How is the lifetime parameter wired up? Could the above code even work?
- Functions are currently not types. This means that `func::` is currently not possible
- Associated types like `Output` cannot current be accessed like this. Instead one would have to use the `<... as Trait>::AssocType` syntax:
  ```rust
  <foo as FnOnce(...) -> ...>::Output
  ```

And the final alternative is to avoid using the async function syntax:

```rust
abstract type Foo<'a>: Future<Output = i32> + 'a;
fn foo(...) -> Foo<'in> { // Initialization pattern
  async { ... }
}
```

The downside is of course that it means that we're already planning not to use the async function syntax we're about to put into the language. At least in this situation.

Abstract types are in my opinion a promising and interesting feature. I think async function should be fully compatible with it and not require any weird papercuts.

**Winner:** Outer return type approach

*Side note: There is currently a debate going on about how the abstract type syntax should look like. I just selected this notation because it was also used on the internals thread. I'm not attached to this notation :-)*

## Trait Bounds

The second feature that the inner return type approach isn't compatible with are trait bounds:

```rust
// Trait bounds & outer return type approach

async fn foo() -> impl Future<Output = i32> + Send { ... }
```

The code above throws a compilation error if the future returned by the function cannot be sent form one thread to another.

There two interesting things at play here:
- `impl Trait` cannot add new trait implementations. Bounding on a trait that isn't already implemented by the concrete type leads to a compilation error. However, getting a compilation error does of course mean that we can use trait bounds to ensure certain traits are implemented by failing to compile if they aren't.
- `impl Trait` leaks auto traits: `Send` and `Sync` are auto traits and leak automatically if the concrete type implements them. This means that even if we just write `impl Trait` instead of `impl Trait + Send`, we still return something that is `Send` as long as the underlying concrete type is `Send`. Adding `Send` is still beneficial, though, because, as mentioned in the previous point, it ensures that the concrete type has to be `Send` by failing to compile otherwise.

Trait bounds are especially handy inside trait definitions:

```rust
// Trait bounds inside trait def & outer return type approach

trait MyTrait {
  async fn foo() -> impl Future<Output = i32> + Send {
    // default implementation for foo()
  }
}

fn bar(x: impl MyTrait) { // Generic context -> Send bound matters!
  let _: &Send = &x.foo(); // Assert Send bound
}
```

What's different is that when declaring an async method inside a trait definition, it actually matters which bounds for auto traits are specified. When a bound is specified it means that *all* trait implementations need to fulfill it.

Note: Currently `impl Trait` is not yet allowed in traits. More about this [here](https://doc.rust-lang.org/stable/error-index.html#E0562).

The inner return type approach doesn't support trait bounds because the `impl Future` is implicit. Let's look at the workarounds:

Two of the previously presented workarounds for abstract types also work in this case:
- The `outer` syntax
- The initialization pattern

Additionally, the thread on the internals forum did present a third alternative: A special syntax for specifying bounds:

```rust
// Special bounds syntax & inner return type approach

async(Send) fn foo(&self) -> i32;
```

The problem with this syntax is that it looks confusing. By looking at it, it is not entirely clear to what the `Send` in parenthesis applies. Its placement at the front doesn't suggest that it applies to the return type.

And finally a fourth and last alternative was presented: Making async function `Send` by default. This avoids the confusing syntax from the previous alternative and builds on the idea that most async functions should return a future that is `Send`. However, there are some problems with this idea: Would this bound apply inconsistently? It is not needed in free functions and inherent methods. Can we disable it? And of course: Do we want this? It's a 180 on how the language does it elsewhere.

So, let's summarize this section. I think that being able to specify trait bounds is important. The inner return type approach can't do it in a clean way. The best workaround would be in my opinion to avoid the async function syntax altogether and use the initialization pattern instead.

**Winner:** Outer return type approach

## Conclusion

I think about `async` as a keyword that transforms a block of code into a future. Consequently I regard an async function as a function that has been transformed and now returns a future. Otherwise, however, I regard it as a normal function. I believe that its signature should reflect that it returns a future. After looking at it in great detail, I now believe that async functions should have `impl Future` in their signature.

With this blog post I tried to summarize all the angles one might look at this choice and I tried to gather all the pros and cons. If nothing else it should be clear now that the decision about whether Rust should adopt the inner or outer return type is not as clear-cut as the RFC made it out to be. There's a lot more to it than the points that were actually mentioned. If the choice was up to me I would pick the outer return type approach. I think it is the option that fits better.

Thanks for reading!

## Credits
Special thanks goes to all the people who took part in the conversation on [the thread](https://internals.rust-lang.org/t/pre-pre-rfc-async-methods-bounding-async-fns) in the Rust internals forum!

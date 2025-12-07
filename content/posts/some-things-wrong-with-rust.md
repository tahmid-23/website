---
title: Some Things Wrong With Rust
date: '2025-12-07T02:22:32-05:00'
draft: false
hidemeta: false
comments: false
disableHLJS: true
disableShare: false
hideSummary: false
searchHidden: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
---

I love using Rust, and it does lots of things right. But sometimes I wish it was better.

---

## `clone()` + `async move`

Asynchronous code lets you run tasks concurrently. Maybe that looks something like this:

```rs
let listener = TcpListener::bind("127.0.0.1:8080").await?;

tokio::spawn(async {
    loop {
        let (socket, _) = listener
            .accept()
            .await
            .expect("failed to accept connection");
        // handle the incoming TcpStream...
    }
});

// go do something else...
```

except:
```
error[E0373]: async block may outlive the current function, but it borrows `listener`, which is owned by the current function
 --> src/main.rs:7:18
  |
7 |     tokio::spawn(async {
  |                  ^^^^^ may outlive borrowed value `listener`
8 |         loop {
9 |             let (socket, _) = listener
  |                               -------- `listener` is borrowed here
  |
  = note: async blocks are not executed immediately and must either take a reference or ownership of outside variables they use
help: to force the async block to take ownership of `listener` (and any other referenced variables), use the `move` keyword
  |
7 |     tokio::spawn(async move {
  |                        ++++
```

Thanks, compiler. C++ could never.

The issue is that the async closure does not take ownership of `listener`. Hence, the enclosing scope may end, and `listener` may drop.

`async move` allows the async closure to take ownership of `listener`, which is really what we intended to do.

But what happens if we *don't* want the async closure to take ownership?

<br/>

Let's say we periodically want to print connection statistics. Let's handle TCP connections in the background task, and print the number of connections handled over time. You might come up with this:

```rs
let connections = Arc::new(AtomicU64::new(0)); // Arc since we might share the counter between threads
let listener = TcpListener::bind("127.0.0.1:8080").await?;

tokio::spawn(async move {
    loop {
        let (socket, _) = listener
            .accept()
            .await
            .expect("failed to accept connection");
        connections.fetch_add(1, Ordering::Relaxed);
        // handle the incoming TcpStream...
    }
});

loop {
    println!("{} connections handled", connections.load(Ordering::Relaxed));
    tokio::time::sleep(Duration::from_secs(1)).await;
}
```

except:

```
error[E0382]: borrow of moved value: `connections`
    --> src/main.rs:23:44
     |
   8 |     let connections = Arc::new(AtomicU64::new(0)); // Arc since we might share the counter between threads
     |         ----------- move occurs because `connections` has type `Arc<AtomicU64>`, which does not implement the `Copy` trait
...
  11 |     tokio::spawn(async move {
     |                  ---------- value moved here
  12 |         loop {
     |         ---- inside of this loop
...
  17 |             connections.fetch_add(1, Ordering::Relaxed);
     |             ----------- variable moved due to use in coroutine
...
  23 |         println!("{} connections handled", connections.load(Ordering::Relaxed));
     |                                            ^^^^^^^^^^^ value borrowed here after move
     |
     = note: borrow occurs due to deref coercion to `AtomicU64`
note: deref defined here
    --> /home/as-threadripper/.rustup/toolchains/stable-x86_64-unknown-linux-gnu/lib/rustlib/src/rust/library/alloc/src/sync.rs:2237:5
     |
2237 |     type Target = T;
     |     ^^^^^^^^^^^
help: consider cloning the value before moving it into the closure
     |
  11 ~     let value = connections.clone();
  12 ~     tokio::spawn(async move {
  13 |         loop {
 ...
  17 |                 .expect("failed to accept connection");
  18 ~             value.fetch_add(1, Ordering::Relaxed);
     |
```

The async closure takes full ownership of `connections`. But we want to share the counter between the two tasks!

As the compiler suggests, we need to `clone()` connections before passing it off. `value` isn't such a good name. `connections_clone`?

```diff
+let connections_clone = connections.clone();
 tokio::spawn(async move {
     loop {
         let (socket, _) = listener
             .accept()
             .await
             .expect("failed to accept connection");
-        connections.fetch_add(1, Ordering::Relaxed);
+        connections_clone.fetch_add(1, Ordering::Relaxed);
         // handle the incoming TcpStream...
     }
 });
```

It works, but it's a bit verbose. `connections` and `connections_clone` refer to the same underlying object--it'd be nice if they could have the same name. Otherwise, you might accidentally refer to `connections` in the async closure again. Let's shadow the original variable in a brace-enclosed scope:

```rs
let connections = Arc::new(AtomicU64::new(0));
let listener = TcpListener::bind("127.0.0.1:8080").await?;

tokio::spawn({
    let connections = connections.clone();

    async move {
        loop {
            let (socket, _) = listener
                .accept()
                .await
                .expect("failed to accept connection");
            connections.fetch_add(1, Ordering::Relaxed);
            // handle the incoming TcpStream...
        }
    }
});

loop {
    println!("{} connections handled", connections.load(Ordering::Relaxed));
    tokio::time::sleep(Duration::from_secs(1)).await;
}
```

Perfect! You can no longer accidentally reference an invalid variable from inside the async closure.

But now our code is more *verbose*. We had to introduce a brace-enclosed scope, indent our entire async closure, and `clone()` our variable.

This gets unwieldy when you need to share multiple variables:
```rs
let actually_owned = // ...;

tokio::spawn({
    let foo = foo.clone();
    let bar = bar.clone();
    let baz = baz.clone();

    async move {
        actually_owned.do_something(&foo, &bar, &baz).await;
    }
});
```

You need to perform this dance where we explicitly rebind all of our shared variables in a preamble to the actual asynchronous code (which requires another indent).

Compare this to what C++ might look like (barring the lack of an actual async runtime):
```cpp
tokio::spawn([actually_owned = std::move(actually_owned), foo, bar, baz]() mutable {
    co_await actually_owned.do_something(foo, bar, baz);
});
```

Of course, C++ benefits from copying variables in lambda captures. But this style is incredibly more ergonomic for such a common usecase.

---

## Non-Lexical Lifetimes

A common operation on a `HashMap` is to get a value by its key, or insert it if it doesn't exist. A simple use case and implementation might look like this:

```rs
struct User {
    pub id: u64
}

struct UserManager {
    id_to_users: HashMap<u64, User>,
}

impl UserManager {
    fn get_or_insert(&mut self, id: u64) -> &User {
        if !self.id_to_users.contains_key(&id) {
            self.id_to_users.insert(id, User { id });
        }

        self.id_to_users.get(&id).unwrap()
    }
}
```

However, we have to hash the `id` key twice in every successful lookup: once to check `contains_key`, and once to get the value. That's not terrible for `u64`, but for a more complex type, you would want to avoid repeated hashing. You might try this:
```rs
fn get_or_insert(&mut self, id: u64) -> &User {
    match self.id_to_users.get(&id) {
        Some(user) => user,
        None => {
            self.id_to_users.insert(id, User { id });
            self.id_to_users.get(&id).unwrap()
        }
    }
}
```

We still hash three times for insertion, but there's no API that enables us to do otherwise.

except:

```
error[E0502]: cannot borrow `self.id_to_users` as mutable because it is also borrowed as immutable
  --> src/main.rs:16:17
   |
12 |     fn get_or_insert(&mut self, id: u64) -> &User {
   |                      - let's call the lifetime of this reference `'1`
13 |         match self.id_to_users.get(&id) {
   |               ---------------- immutable borrow occurs here
14 |             Some(user) => user,
   |                           ---- returning this value requires that `self.id_to_users` is borrowed for `'1`
15 |             None => {
16 |                 self.id_to_users.insert(id, User { id });
   |                 ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ mutable borrow occurs here
```

...what? `user` isn't even bound in the `None` match arm. How is there a simultaneous borrow?

If you ask ChatGPT what the issue is, it'll blindly tell you that the error is correct and to use the `.entry()` API. That requires you to *always clone* the key--not ideal.

If you search up the error (in the big 2025?), you'll see examples of *actual* simultaneous immutable/mutable borrows. But we're not doing that!

The issue is that the Rust compiler does not *fully* support "non-lexical lifetimes".

Even though the return value of `self.id_to_users.get(&id)` is no longer referenced, the borrow checker treats its lifetime as if it still is valid. Hence, we can't take a mutable reference.

Fortunately, this is (eventually) being solved. [Polonius](https://blog.rust-lang.org/inside-rust/2023/10/06/polonius-update/#background-on-polonius) is Rust's next-generation proposal for a more lenient borrow checker that solves this exact problem. It's currently available on nightly Rust with `-Zpolonius`.

But it'd be nice if this error message weren't so opaque! :(

---

## Precise Capturing

[Precise Capturing](https://doc.rust-lang.org/stable/edition-guide/rust-2024/rpit-lifetime-capture.html) was introduced in Rust 2024 to specify captured lifetimes in return-position-impl-traits.

If your signature looked like:
```rs
fn make_bar(x: &u64) -> impl Bar
```
it's unclear whether the returned value depends on the lifetime of `x` or not.

If you're as afraid of lifetimes as the average user is, the likelihood is that the returned value does *not* depend on the lifetime of `x`. But, Rust 2024 decided to capture all lifetimes by default.

This is annoying with `&self`, the most common way to write a struct method.

Consider the following:
```rs
trait Bar {
    fn baz(&self);
}

// ...

struct Bootstrap;

impl Bootstrap {
    fn make_bar(&self) -> impl Bar {
        // ...
    }
}
```
where `Bootstrap` is a factory to create `Bar`s. What if we did something like try to move the created object to another thread?

```rs
let bootstrap = Bootstrap{};

let bar = bootstrap.make_bar();
tokio::spawn(async move {
    bar.baz();
});
```

except:
```
error[E0597]: `bootstrap` does not live long enough
  --> src/main.rs:25:11
   |
23 |       let bootstrap = Bootstrap{};
   |           --------- binding `bootstrap` declared here
24 |
25 |   let bar = bootstrap.make_bar();
   |             ^^^^^^^^^ borrowed value does not live long enough
26 | / tokio::spawn(async move {
27 | |     bar.baz();
28 | | });
   | |__- argument requires that `bootstrap` is borrowed for `'static`
29 |   }
   |   - `bootstrap` dropped here while still borrowed
   |
note: this call may capture more lifetimes than intended, because Rust 2024 has adjusted the `impl Trait` lifetime capture rules
  --> src/main.rs:25:11
   |
25 | let bar = bootstrap.make_bar();
   |           ^^^^^^^^^^^^^^^^^^^^
help: use the precise capturing `use<...>` syntax to make the captures explicit
   |
16 |     fn make_bar(&self) -> impl Bar + use<> {
   |                                    +++++++
```

And hence, we need to make the following change, with very little benefit to clarity:
```diff
-fn make_foo(&self) -> impl Foo {
+fn make_foo(&self) -> impl Foo + use<> {
```

The Rust blog post on this matter notes:
> Note that in Rust 2024, the examples [in the blog post] above will "just work" without needing use<..> syntax (or any tricks). This is because in the new edition, opaque types will automatically capture all lifetime parameters in scope. This is a better default, and we've seen a lot of evidence about how this cleans up code. In Rust 2024, use<..> syntax will serve as an important way of opting-out of that default.

Really.

<br/>

Return position impl traits also capture any generic parameters. In contrast, this is a *good* default. It's very likely that the return type uses the parameter as a dependency!

For example, if traits `Foo` and `Bar` were both defined, then we might write

```rs
fn make_bar(foo: &impl Foo) -> impl Bar
```

Here, we can avoid writing a generic for the type of `foo`, and the return type will capture `foo`'s actual type. Great!

But now let's add `&self` into the mix. Going back to the `Bootstrap` example, updating with the previous error:
```rs
impl Bootstrap {
    fn make_bar(&self, foo: &impl Foo) -> impl Bar + use<> {
        // assume that the returned implementation uses the type of foo
    } 
}
```

except...we're no longer capturing the type of `foo`. We're capturing nothing.

In fact, this generates a compile error:
```
error: `impl Trait` must mention all type parameters in scope in `use<...>`
  --> src/main.rs:22:43
   |
22 |     fn make_bar(&self, foo: &impl Foo) -> impl Bar + use<> {
   |                              --------     ^^^^^^^^^^^^^^^^
   |                              |
   |                              type parameter is implicitly captured by this `impl Trait`
   |
   = note: currently, all type parameters are required to be mentioned in the precise captures list
```

What is the type of `foo`? Well, it's `impl Foo`...that doesn't explicitly name a type. How can we add the value to the captures list?

The answer is that we can no longer use `impl Foo`. Instead, we *have to* introduce a generic.

As rustfmt intends:
```rs
impl Bootstrap {
    fn make_bar<T>(&self, foo: &T) -> impl Bar + use<T>
    where
        T: Foo,
    {
        // ...
    }
}
```

We've significantly obscured our function's signature using the least reasonable defaults possible.

---

Rust's design choices are usually ergonomic and feel great. But sometimes the state of affairs leaves more to be desired. No language is perfect...

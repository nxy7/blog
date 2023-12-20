---
title: "Golang vs Rust"
date: 2023-12-27T09:03:20-08:00
draft: false
---
Yes, it's another one of those 😈 My programming journey started few years ago with typescript (unsurprisingly), but sooner rather than later I've developed a view that typescript doesn't have any advantages on the backend. Seriously, does typescript have any advantages over alternatives in this space? It's interpreted, so we need to package runtime with our code to get it to work, even if it doesn't have benefits of compiled languages we need compile step, it doesn't have good native tooling (you need 3rd party projects for things like formatting), it's not very safe language (f.e. it doesn't protect you against accessing out of bounds index), it doesn't encourage good error handling (you don't even know which functions can throw errors) and so on. The only benefits I see TS have are big ecosystem, frontend <-> backend code sharing and many people already know TS so it's easy to hire for. If you ask me, the only worthwhile thing here is sharing code between your services, but this can also be done across languages (in my projects I'm often depending heavily on sharing types between Rust and TS).
Given those things I've started looking for alternatives and two stand out among the crowd - Golang and Rust. This post is my subjective comparison of two languages

# Ergonomics

## Error handling
Let's start with things that both languages have in common - they return errors as values. It's very nice approach that forces you to handle error path right away. I certainly prefer this approach to throwing and catching errors. Even if you want to ignore error handling (both languages provide you that option), with errors as values you know if function is fallible just by looking at its signature - great stuff.

Now let's see how it works in practice. We'll start with Golang.
```go
func fallible() (string, error) {
  return "", fmt.Errorf("this function failed")
}

func run() error {
  retValue, err := fallible()
  if err != nil {
    // handle error, it's common to handle errors by returning them (manual error bubbling)
    return err
  }
  // do something with retValue
  return nil
}
```

And before we make any judgements here's Rust approach
```rust
fn fallible() -> Result<String, String> {
  return Err("this function failed".into());
}

fn run_bubbling() -> Result<(), String> {
  // if fallible returns error, then error is automagically returned from run_bubbling
  // if it's Ok then we can use returned String from 'fallibleResult' variable
  let fallibleResult = fallible()?;
}

// match statements are a bit more verbose, but sometimes you don't want to return error up the call stack
// or you need to do something before bubbling
fn run_match() -> Result<(), String> {
  let fallibleResult = fallible();
  match fallibleResult {
    Ok(text) => {
      // handle Ok case
    }
    Err(e) => {
      // handle Err case
    }
  }
}

fn run_let_else -> Result<(), String> {
  // it's a bit similar to error bubbling but allows us to add some extra logic
  let Ok(fallibleResult) = fallible() else {
    // handle error case
    // you cannot access Error value though, so it's more usefull for
    // unwrapping Options, not Results
    // this code path could be used to return from the function or provide some
    // default value
  };
  // use fallibleResult
}
```
As you can see Rust provides you with much more choice in the way you want to handle errors. Is that a good thing? Depends on who you ask. Manually handling errors is fine, but I wish Golang had syntax that allowed error bubbling. Something as simple as '?' operator, that would return error and zero values for other arguments (if error is not nil). It'd make Go even more readable and I don't think it would lose anything in the process. One more thing I don't like about Go approach is the fact that in case of Error you need to return all values. It can sometimes lead to bad behavior as sometimes people treat zero values to indicate that something went wrong. One example of such thing I've encountered was in Kubernetes Cilium Gateway API. Stripping whole URL paths required you to replace `/some/path` with empty string, but proxy that Cilium depends on (Envoy) decided to return early if it encounters zero value string (empty string). This meant that there was no way to strip paths and I had to file PR with some less than ideal workarounds to get Gateway API to work correctly. This wouldn't happen with Option/Result types since they indicate intent clearly (I'd argue that zero values don't do that).
Rust on the other hand is very readable to me, allows faster coding as you don't need 3 additional lines with every fallible function, but it also allows some silly mistakes to creep in. Sometimes you should not bubble errors, but can do that as it can quickly become a habit. It's worth it trade off if you ask me, and I much prefer Rust error handling over Go approach.

## Sum types
Part of the reason why Rust error handling works so great is the fact it has sum types (i.e. types that can be A or B or C...). Thanks to this we can model things like Option (which is Some' type holding value or empty 'None' type) or Result (same as Option, but now it's 'Ok' or 'Err' both holding values). Without them '?' operator wouldn't be possible. Sum types are very helpful in modeling any problem and I wish Go had them. In Go we'd typically rely on interfaces, but it's not as clean as sum types (depending on the problem).

## Interfaces and Traits
Describing abstract behavior makes our architecture less coupled and more flexible. Both languages have ways to allow what: Golang uses interfaces while Rust uses Traits. They're very similar, both describe set of functions that concrete types have to implement, the biggest difference is the fact that Golang interfaces are implicit. If you have interface `stringer` and your type has function `String() string` then it automatically implements `stringer` interface.
In Rust we have to explicitly write `impl Stringer for MyType{...}`. Both approaches are fine if you ask me, explicitly implementing interfaces has one advantage - you know where specific methods are located. With Golang I sometimes find myself looking for methods implementing some interfaces as they are scattered among multiple types. Golang also has syntax allowing to make sure that type implements some interface `var _ SomeInterface = &MyType{}`, but it's quiet ugly. Since it does its job I won't complain about that too much :p

## Async model
One of the biggest advantages of Golang is its async system. Go is one of not so many languages that have built in support for concurrency. It's very different from how most languages approach this topic. All code is 'sync' and to run something in a background we spawn another goroutine (think lightweight threads). Want to run many operations and wait for their results? Create message passing channel, spawn X goroutines and wait until they all return. I think it's easier mental model, but is much more verbose (I see a trend here). This model has one huge advantage - all functions can be run this way. We don't have split between sync and async functions, which is a great benefit.
Rust on the other way has this sync/async split and doesn't handle async automatically, instead it has libraries that create async runtimes (like Tokio). Rust async ecosystem is not fully mature yet (async functions in traits are still problematic), but it's good enough to be used in production setting. Because Rust is as expressive as it is, it's totally possible to emulate Golang async model in Rust, although it doesn't look very idiomatic.

## Expressiveness
Expressiveness is a double-edged sword. It allows hiding complexity, but can also lead to awful code spaghetti. It's very hard to find the right balance here and Golang chose to be simple instead of expressive. Rust on the other hand gets new syntax with every release, to let you handle it's complexities a little bit more ergonomic, but there is such thing as too many features (C++ :P). Personally I don't think Rust crossed that point yet.

## Readability
There's nothing more subjective than readability. Gophers swear by Go simplicity and claim that simple language is readable language. Rustaceans say that the right combination of functional style programming and syntactic sugar make for the most maintainable code. My opinion here is that Go error handling can distract you from implementing business logic. I don't love the fact that every function call quickly turns into 4 lines (with `if err != nil` style error handling).
Maybe it's because I'm past the big initial learning curve investment that Rust requires, but Rust seems more readable to me. Go is not far off and with alternative error handling I'd give this point to Go. Maybe it'll happen in the future. 

## Tooling
Both languages have great tooling to deal with testing/formatting/code sharing. I think they are gold standard in the language industry and inspire other languages to come with similar tools. I'd say they are equal (equally great) in that department.

## Simplicity
Of course this point goes to Go. Go is simple by design. There's no magic, given enough time you can easily understand any codebase. I don't like all Go design choices, but I have to say they are consistent on their initial goal of making simple language.
Rust on the other hand is as complex as it gets. There are plenty ways to do one thing, it has ton of features, it requires you to understand how it works to work with its ecosystem. You could write Go after watching one tutorial, but Rust will usually take much more until you'll feel comfortable.
One more thing on that front - Rust requires you to manage your references lifetimes, which is absolute pain. I'd gladly take Rust with GC that allows specifying lifetimes when you need extra performance, but sadly there's no such thing and lifetimes tend to infest other code pretty quickly.

## Defer vs Drop
Both languages use different schemes for cleanup after some piece of code finished executing. Golang uses defer while Rust has Drop trait. There are times, that I'd like to use Defer in Rust (in fact it's possible with some macros that implement Drop on empty struct), but Drop is elegant in itself and works pretty well.
Golang's defer is also nice, but again - leads to verbose code. If we open a file, we're responsible for manually closing it (usually inside defer). With something like Drop it could be automatically closed when all references are GCd. It's just another convenience that Golang drops (pun intended) in favor of simplicity.

# Safety
Rust is touted as THE SAFE LANGUAGE, and it lives up to its promises. Rust successfully eliminates a whole class of bugs and programs made with it just do not crash. It's probably the most important characteristic of the language, and therefore it wins in safety match with Golang.
Go is 'safe' language too. It's garbage collected, so you don't need to worry about memory leaks (unless you have logic bug and something holds reference to some piece of the memory when it shouldn't), accessing out of bounds array indices will panic instead of giving you access to memory you should not access, but if you compare that to Rust it's pretty bleak. Additionally, there's no way to tell if some pointer is nil or not (Zig has concept of pointers that cannot be nil which is nice) and you either need to have a lot of guard code or make sure that there's no way nil pointer will be passed to a function. Pointers are often used instead as Rust Options, but because they're not single purpose using it as such can get quiet messy.
Overall Rust is much safer language.

# Performance
People often cite performance as Rust strength, but IMAO it's not something we should focus on. First - it's really hard to write efficient Rust code. It'll often require you to deeply understand packages you're using, it's async model, the best way to structure your code and a bunch of other things. I think what's really important is that even 'good enough' Rust is fast. So is Go. You can go an extra mile with Rust to make it the most performant solution, but if you're not building something really performance critical don't obsess over performance. I think it's pretty nice to know that if I really need performance I have a choice of optimizing my Rust code to the absolute limits. 

# Compilation speed
Go is ridiculously fast on that front. Go is in fact so fast, that you could use it instead of scripting languages and often not notice a difference. Rust on the other hand is known for it's slow compile times. Some people really care about that - others don't. I don't mind long compile times that much since 'if it compiles it works' factor that Rust has. If slow compile times weren't accompanied by Rust `no undefined behavior` guarantees then I'd most likely take more problems with it.
There's no denying though - this point goes to Golang. 

# Ecosystem
Both systems have big ecosystems with many libraries. Go has advantage of having great standard library. Even if there would be some better choices, using packages provided by the language itself is a great way to make sure that your code will be maintainable in the long run. Rust with its strong guarantees makes for a better 3rd party crates ecosystem thought. I feel that Rust APIs are often easier to use, and I generally trust them more than 3rd party Golang code. They're still both great and better than other languages so that's a tie for me.

# Summary
Both languages are great. Golang strength lies in its simplicity, which puts less mental strain on the person reading the code. Rust focuses on correctness and feature set that can satisfy everyone. I think you can't go wrong with either language and picking the best one for a project depends on few things.

1) Does your crew know one of them already? If so pick the one they're familiar with or pick Golang (it's super easy to learn along the way)
2) Is your app performance critical? You can't go wrong with Rust here.
3) Would your app suffer from GC? Most apps should not care about that, but if your does - pick Rust.
4) Is safety extra important to you? Should your app have 100% uptime? Pick Rust.
5) Do you have many people depending on some project and want them to PR changes they need quickly? Golang. 
6) Will you extensively use concurrency for IO bound tasks? That's Golang's niche!
7) Do you value elegant solutions? Rust! Do you just want to get stuff done? Go!
8) Would you like to learn new language? Pick the one you find more pleasant to look at :P

As you can see, I don't have decisive answer here - mainly because both languages are great choice. If you asked me to choose between C++ or Rust I'd be much more opinionated here :)
Thanks for reading this article, I'll come back here in a few years to see if I changed my mind on this topic and how both languages developed. Have a great day ^_^

---
title: "Perfect programming language"
date: 2023-12-12
draft: false
---

Hello, as someone that likes to try out new things I've used plenty of programming languages, some popular, some niche and not widely known.
This post sums up my experiences as wishlist of things I'd like my perfect language to have. This list is obviously subjective and there are some things that I might like, but you don't - if that's the case feel free to write comment with things you disagree with.
One last small disclaimer - I'll be writing about my 'perfect general purpose programming language', there are some applications where I'd be happy to give up on many things from this list - shell languages come to mind as one example that doesn't need many of features I'll list and instead they should allow you to quickly and conveniently interact with system. 

# Static typing

My first request is for the language to have static typing. Let's be honest, dynamically typed languages don't have many proponents these days for good reason. Being programmer is more about reading code, than writing it. Static typing allows you to quickly get a glimpse of part of the system to know what the arguments types are and what does the function output.
Good evidence of superiority of typed languages is the fact that even dynamically typed languages have typed layer on top of them now (f.e. TypeScript or optional types in Python).
Static typing gives more information about your program to compiler or interpreter, so it can make better decisions on how the code will be executing leading to potentially better performance and memory footprint. I personally don't see almost any advantages of dynamic typing. 

# Sum types

Sum types are great tool that I'm missing very much in Golang. If you don't know what they are, it's type that can be one of multiple other types. It'll be probably easier to show example :P

``` typescript
type Event = Donation | Patron | Raid;

```

This example comes straight from one of my codebases and showcases how sum types can help you model things using composition instead of inheritance. What we're basically saying here is that type 'Event' can come in one of three shapes - 'Donation', 'Patron' or 'Raid'. We could model that using inheritance instead (by creating class 'Donation' that would inherit 'Event' class), but first - composition is my preferred way of doing polymorphism and secondly - sometimes sum types are more versatile. Classes usually describe behavior, and sum types describe data. Now we can create array of Events, and it's very easy to look up what shape can Event have.

# Pattern matching

Sum types are the best when they're used with some other mechanism allowing to quickly assert type of variable. The best mechanism for that use case (and many more) is Rust pattern matching. Given the same example as before, here's how we'd easily use sum types with pattern matching in Rust.

```rust
enum Event {
    Donation,
    Raid,
    Patron,
}

fn DisplayEvent(event: Event) {
    match event {
        Event::Donation => println!("Donation"),
        Event::Raid => println!("Raid"),
        Event::Patron => {
            // do something special
            println!("Patron")
        },
    }
} 
```

Rust pattern matching is exhaustive meaning that you need to serve all possible branches which is great for predictability of your programs. It also has many other great features such as catch all and can actually match very nested types.
```rust
enum Event {
    Donation(Donation),
    Raid,
    Patron,
}
enum DonationType {
  Fiat(f64),
  Crypto(f64),
}
...
  match event {
      Event::Donation(DonationType::Crypto(amount)) => {
        // in this example Donation is actually another Sum type and in this branch we can directly access
        // amount of cryptocurrency donation and get less nested matches/if statements
        println!("{}", amount)
      },
      _ => {
          // catch all branch
          // if 'event' variable doesn't match any other branch then this will be executed
          // you can think of this as 'else' in 'if statement'
      },
  }
```
You get my point. Rust actually has more functionality related to pattern matching (like matching on ranges), but even simple naive exhaustive pattern matching would be greatly appreciated in any language I use.

# Error as values

Returning errors as values is great practice in my opinion and allows you to easily locate places where your program can fail. It makes your code more verbose, but verbose is better than unpredictable if you ask me :P 
Here's how Golang uses multiple returns as a way to do error handling
```go
func CanFail() (string, error) {
	return "", fmt.Errorf("This function failed")
}

func UsesCanFail() {
  str, err := CanFail()
  if err != nil {
    fmt.Println("This function failed: ", err)
  } else {
    fmt.Println(str)
  }
}
```
It's very nice that you can look at function signature and can immediately tell if function can fail. Furthermore, it also leads to much more robust code, as it forces you to handle all failure cases. There are things you might not like about Go, but returning errors as values should not be one of them.

# Options/Results types and some syntax around them

Using multiple returns is just one way of returning errors as values and not necessarily the best one. Don't quote me on that, but I think the language that first popularized concept of Optionals and Results was Haskell (again - don't quote me on that :P). Those are very nice use case of sum types to model operations that can return something/nothing or that can fail along the way. My favorite language with Options/Results is Rust so let's see how it uses them.
```rust
async fn get_active_stream(&self) -> Result<Option<LiveBroadcast>> {
    let mut broadcasts = self.list_active_broacasts().await?.into_iter().filter(|f| {
        f.status
            .as_ref()
            .is_some_and(|f| f.life_cycle_status.as_ref().is_some_and(|s| s == "live"))
    });
    let active = broadcasts.next();
    Ok(active)
}
```
To me Options and Result are such a clean way of expressing outcomes of operations. You only need to read the first line to know that function get_active_stream can fail and if it succeeds it returns some stream or nothing (if there isn't any stream ongoing). Pattern matching on every operation would be very exhausting though, so Rust has ? operator, that allows us to return Result if it's Error. In the first line you can see `self.list_active_broacasts().await?` which basically means "if list_active_broadcasts returned Result containing Error, then return whole function with that error. If it has some value unwrap it and go forward". This way we can chain Options/Results without much hassle and still get all the benefits of them. Perfect :-)

# Functions as first class objects

We want functions to be treated as first class objects. This allows for many interesting patterns such as passing function to another function. Great example of using such feature is Map function that accepts another function to change some set of elements to another set of elements. This kind of dependency injection can allow very flexible programs, that are easier to maintain.

# Compiled and not interpreted

Most languages have no reason to be interpreted languages. We write some code and then ship interpreter with few files that are executed over and over again. What's the point of interpreter if it always runs the same code? Why use interpreted language if you then have to ship interpreter with the code it will be running? If you ask me it's one of the worst parts of JS legacy. People were familiar with JS, so NodeJS was created, and now we have to ship NodeJS in every docker container. This whole thing is even worse now, as we're using Typescript, so we're compiling our code into other code that in the end is not compiled into binary and will be ran by interpreter instead. If you ask me, there's only one good place for interpreter - shell. Golang proved that compiled languages can compile SUPER FAST, seriously! Even bigger Golang projects compile so fast, that compilation+execution will be at least as fast as running similar code on interpreter.
To reiterate - interpreted languages are okay.. for shells and browsers. Most other code should be compiled unless you have really good reason for it not to be. 

# Implicit memory allocations (automatic memory management)

I'll tell you a secret - most of us are kind of dummies. It's easy to make mistakes with manual memory management and what's more important manual memory management takes your attention away from business logic. I'd like to avoid GC, but if that's the price for convenience - I'll take it! JS proved that most people don't care about memory footprint or performance of their programs, so we don't have to be purists and fight GC at all cost. Beside - there are examples of great GC languages like Go which is very lightweight and will make your programs fast and light without requiring you to allocate memory manually.
One other approach I really like is Rust memory management. It doesn't use GC, but it's strict borrowing rules allows compiler to deduce when the memory for each variable should be allocated and freed. It comes with a price though - writing reference lifetimes is no fun at all, and I'd happily turn on Rust GC mode if it allowed me to skip annotating them. I've heard that Ocaml allows the opposite - it uses GC, but will allow you to annotate lifetimes for the times you need the performance. That seems to be the best of the both worlds to me, but I haven't used Ocaml much yet.

# No cost abstractions

I mean, why would I want my abstractions to have cost. The ability to use functions like for_each()/map() without any additional cost (as they are compiled into simple for loops) is great thing to have in any language.  

# Some sort of clean compile time execution

Having a way to run some code during compilation can sometimes be very convenient and the best implementation of this concept I've seen is Zig with it's comptime. If you ever see this keyword it means that the code will be run during compilation, and it'll have no runtime cost. 
```zig
pub fn main() void {
    const len = comptime add(1, 3);
    const len_2 = comptime some_heavy_calculation();
    const arr: [len]u8 = undefined;
    const another_arr: [len_2]u8 = undefined;
}
```
If you run this program `comptime add(1,3)` will be swapped for raw value `4`, the same will happen with `len_2`. It can be really handy to calculate some values during compilation instead of calculating them on each program execution. 

# Package manager

I'm not even considering a language that doesn't have package manager. Call me lazy, but to me, it's essential feature of any new language. Code reusing is such a bit part of our work that some package manager endorsed by language creators (to avoid splitting the ecosystem) is a must. Maybe some people prefer cloning git repositories into their projects, but that's not me.

# LSP

One of the most important tools while writing code has to be good LSP that will catch all the errors and will give you hints regarding your code. I'd certainly consider it essential part of any language ecosystem.

### Ending notes

This list can change in the future (if I encounter new features that I really like), but right now these are all the features I'd really love to have in one language. There isn't any such language right now, but I think 2 that come really close are Rust and Golang. My perfect language probably sits somewhere in between which is not surprising considering the fact that they're both really well-thought-out and are gaining popularity as a result of that.


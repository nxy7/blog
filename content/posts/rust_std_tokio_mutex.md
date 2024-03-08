---
title: "Rust Std Mutex vs Tokio Mutex"
date: 2024-03-08
draft: false
tags: ['rust']
---
My last post was about benchmarking mutexes and actor model. I was wondering what's a faster way to synchronize state and conclusion of my
findings was that mutexes are 2-3 times faster depending on the exact Actor implementation. There was one underlying assumption that I didn't specify
explicitly in those benchmarks - I was using Tokio implementation of mutexes. Tokio mutexes are slower than their Std counterpart, so why would anyone pick them.

# Std Mutex vs Tokio Mutex
What's the difference between them? Std mutex is built for sharing values between OS threads. Tokio is async runtime and their Mutex implementation is
used to share values between tasks (green threads). That means that Std mutex is 'simpler' and more performant as Tokio has to account for switching context
not just between OS threads but also their internal abstraction (tasks).

One result of those architectural differences is that Std mutexes cannot be held across 'await', but even in async environment we often don't need to do that.
Very often mutexes are just guarding value for X amount of time. If you're locking some value just to access/modify it, then chances are that std mutex will
be more performant than tokio mutex.

# When to use Std Mutex and when to use Tokio Mutex
If Tokio mutex is slower, why would you use it? Huge advantage of Tokio mutexes is the fact that Tokio runtime can do another work while waiting for the lock.
Depending on your application and amount of OS threads your machine can offer to your program (tokio by default spawns one thread per cpu core),
if you're locking values behind std mutex, then nothing else will run on that thread until mutex can be acquired. 
That means this thread will not participate in other work and instead will sit idle. If your program is composed of many services, that could potentially mean
that one service locking mutex is introducing latency to unrelated service. Typical latency vs throughput problem.

Because of this despite lower performance Tokio mutexes are good default in async environments as they tend to scale with async software complexity a little bit better.

# Benchmark
I'm reusing the same benchmark that I've created to test Tokio mutex vs Actor model. Essentially I've struct with some value and 100_000 concurrent async futures try
to modify this value. One implementation is using Std mutex and one is using Tokio mutex. What we're effectively measuring here is overhead of tokio mutexes in environment where there's no other work in the background.

## Results 
```
std mutex vs tokio mutex/tokio mutex
                        time:   [16.158 ms 16.223 ms 16.303 ms]
                        change: [-49.342% -48.930% -48.440%] (p = 0.00 < 0.05)
std mutex vs tokio mutex/std mutex
                        time:   [7.2295 ms 7.2521 ms 7.2780 ms]
                        change: [+4.3129% +4.8652% +5.4262%] (p = 0.00 < 0.05)
```
Performance varies between runs, but generally std mutexes tend to be 2x faster on my machine. I think this just goes to show that tokio mutexes are good enough and
I'd personally go with them as my first choice until they become performance bottleneck (personally I've never encountered such situation).

### Additional notes
- tokio mutexes obviously poison functions with async, so they're only used in async environment
- std mutexes can be used in both sync and async environments, but they poison functions with Results<> (locking std mutex returns Result)
- if you're sure that lock is held only for very short amount of time std mutex will be faster even in async environment
- despite point above tokio mutex is still very performant and good enaugh

# Summary
In my last Actor vs Mutex benchmark mutexes were 2-3 times faster, std mutexes can increase the gap to 4-6 times the performance of Actor model. This just
goes to show how good synchronization mechanism they are. I really those quick benchmarks as they help me better understand alternatives I can pick between for
each task. I hope you've found results of this benchmark useful. Have a great day :-)

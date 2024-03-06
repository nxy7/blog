---
title: "Rust - Mutex vs Actor benchmark"
date: 2024-01-30
draft: false
---
Hello, this post is quick summary of my findings when I was benchmarking Actor model vs Mutex for concurrent access to value in async environment.
All code for this benchmark is available in (my github repo)[https://github.com/nxy7/rs-actor-mutex-benchmark], but I've decided to copy some parts of
implementation here.

# Actor implementation
```rs
  #[derive(Default)]
pub struct BenchActor {
    count: i64,
    /// optional channel to signal when implementation reached REACHED_COUNT_SIGNAL_AMOUNT
    tx: Option<oneshot::Sender<()>>,
}

pub enum Message {
    IncreaseBySync(i64, oneshot::Sender<()>),
    DecreaseBySync(i64, oneshot::Sender<()>),
    IncreaseBy(i64),
    DecreaseBy(i64),
    Get(oneshot::Sender<i64>),
}

impl BenchActor {
    pub fn new(m: oneshot::Sender<()>) -> Self {
        Self {
            tx: Some(m),
            ..Default::default()
        }
    }

    pub async fn start(mut self) -> Sender<Message> {
        let (tx, mut rx) = mpsc::channel(10000);
        tokio::spawn(async move {
            while let Some(m) = rx.recv().await {
                match m {
                    Message::IncreaseBySync(i, r) => {
                        self.count += i;
                        r.send(()).unwrap();
                    }
                    Message::DecreaseBySync(i, r) => {
                        self.count -= i;
                        r.send(()).unwrap();
                    }
                    Message::Get(r) => {
                        r.send(self.count).unwrap();
                    }
                    Message::IncreaseBy(i) => {
                        self.count += i;
                        if self.count == REACHED_COUNT_SIGNAL_AMOUNT {
                            if let Some(tx) = self.tx {
                                tx.send(()).unwrap();
                                break;
                            }
                        }
                    }
                    Message::DecreaseBy(i) => {
                        self.count -= i;
                        if self.count == REACHED_COUNT_SIGNAL_AMOUNT {
                            if let Some(tx) = self.tx {
                                tx.send(()).unwrap();
                                break;
                            }
                        }
                    }
                }
            }
        });
        tx
    }
}
```
This actor implementation is very simple, but does what Actor should do - it limits access to inner values to just one thread.
Beside that it doesn't do any work it doesn't have to, one inefficiency I've had to resort to was checking count in methods that
don't return any values, so I know when to finish benchmark iteration. The most time consuming thing in this implementation is
sending values over channels - something that you cannot avoid in Actor model.

# Mutex implementation
```rs
#[derive(Default)]
pub struct BenchMutex {
    count: Mutex<i64>,
    /// optional channel to signal when implementation reached REACHED_COUNT_SIGNAL_AMOUNT
    tx: Option<mpsc::Sender<()>>,
}

impl BenchMutex {
    pub fn new(m: mpsc::Sender<()>) -> Self {
        Self {
            tx: Some(m),
            ..Default::default()
        }
    }

    pub async fn increase_by(&self, i: i64) {
        (*self.count.lock().await) += i;
    }
    pub async fn decrease_by(&self, i: i64) {
        (*self.count.lock().await) -= i;
    }
    pub async fn increase_by_checked(&self, i: i64) {
        let mut v = self.count.lock().await;
        *v += i;
        if *v == REACHED_COUNT_SIGNAL_AMOUNT {
            if let Some(tx) = self.tx.clone() {
                tx.send(()).await;
            }
        }
    }
    pub async fn decrease_by_checked(&self, i: i64) {
        let mut v = self.count.lock().await;
        *v -= i;
        if *v == REACHED_COUNT_SIGNAL_AMOUNT {
            if let Some(tx) = self.tx.clone() {
                tx.send(()).await;
            }
        }
    }
    pub async fn get(&self) -> i64 {
        *self.count.lock().await
    }
}
```
Again - very simple struct. I've added `_checked` methods and made them do the same extra work that Actor had to do to know when
to stop bench iteration. I was a little worried that this small value check might in fact slow Mutex because I was holding the lock
longer than I had to, but that didn't turn out to be true.

# Mutex vs Actor benchmark
## Mutex vs Actor that returns values
### Results

```
mutex vs actor/mutex    time:   [1.4809 ms 1.4839 ms 1.4872 ms]
                        change: [+288.15% +290.30% +292.68%] (p = 0.00 < 0.05)
                        Performance has regressed.
Found 9 outliers among 100 measurements (9.00%)
  5 (5.00%) high mild
  4 (4.00%) high severe
mutex vs actor/actor    time:   [4.5462 ms 4.5587 ms 4.5713 ms]
                        change: [-96.680% -96.664% -96.648%] (p = 0.00 < 0.05)
                        Performance has improved.
Found 5 outliers among 100 measurements (5.00%)
  1 (1.00%) low mild
  2 (2.00%) high mild
  2 (2.00%) high severe
```

As you can see Mutex version turned out 3x faster. It doesn't really surprise me because this actor implementation was returning values
over oneshot channel, which is something that you might sometimes want, but I'm sure it slows it down significantly. I've mainly decided
to make actor return values because it made for easier benchmark setup. It's good to know though how much slower is returning values over channel
vs memory access. This benchmark results should apply to all actor methods that modify state and want to know new value instantly.

## Mutex vs Actor that don't have to return values
### Results

```
mutex vs actor 'async'/mutex
                        time:   [1.3995 ms 1.4029 ms 1.4067 ms]
                        change: [+17.433% +17.905% +18.354%] (p = 0.00 < 0.05)
mutex vs actor 'async'/actor
                        time:   [2.7879 ms 2.7944 ms 2.8008 ms]
                        change: [+48.814% +49.399% +49.947%] (p = 0.00 < 0.05)
```

I've added 'async' version of the benchmark. 'Async' here means that caller of actor doesn't want to get values back, so he's essentially just sending Message over
channel and leaves. This required additional check to know when Actor received all messages. To keep both implementations the same I've made Mutex also do
the same equality check thinking that it would significantly increase bench time for Mutex, but to my surprise the equality check was so fast, that it
didn't make any difference in Mutex time. There was a difference for Actor implementation - without the need to send return signal on each Message
Actors became ~40% faster. They're still 2x slower than Mutexes though. 


## Higher concurrency
In first set of benchmarks I was spawning 10000 futures per bench iteration. I've decided to see what happens if I increase it by x10. Is the time going to change
linearly for both implementations or are Mutexes going to suffer because they have to fight for each lock?

### Results
```
mutex vs actor 'sync'/mutex
                        time:   [16.629 ms 16.762 ms 16.908 ms]
                        change: [+11058% +11156% +11266%] (p = 0.00 < 0.05)
mutex vs actor 'sync'/actor
                        time:   [48.231 ms 48.551 ms 48.909 ms]
                        change: [+9403.5% +9498.0% +9591.7%] (p = 0.00 < 0.05)
mutex vs actor 'async'/mutex
                        time:   [16.613 ms 16.667 ms 16.730 ms]
                        change: [+1081.8% +1086.4% +1091.2%] (p = 0.00 < 0.05)
mutex vs actor 'async'/actor
                        time:   [32.053 ms 32.181 ms 32.353 ms]
                        change: [+1047.2% +1052.8% +1059.3%] (p = 0.00 < 0.05)
```

So that didn't really happen. Mutexes are still 2x faster than actors that don't return values and 3x faster than actors that return values.
I think that actors can make some designs simpler and mutexes can be hard to use, but when it comes to performance it seems that Mutexes
are clear winner.

This test was very simple - it was just 'dumb' value access and modification, but I still find the results interesting and hopefully one of you did as well.
 

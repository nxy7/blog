---
title: "Golang + Ethereum = ?"
date: 2024-01-06
draft: false
---
One of my recent jobs required me to interact with Ethereum network heavily. This was my first chance to develop actual working product on this network and this short post summarizes my thoughts about this experience. Does Golang play nicely with ETH network? Is it a good fit to develop services for such tasks? Let's see.

# Go-Ethereum
Fundamental part of Golang Ethereum story is Go-Ethereum package. It's... okay. In general Go fashion it provides you with wide array of all basic operations you need to do and nothing more. I don't really dislike that, and I'm okay building necessary abstractions myself. All basic functionality like calling contracts, sending transactions, accessing results of sent transactions is there, which are all the building blocks you'll need to build higher level of your client. Go-Ethereum also has pretty nice way to package payloads you'll be sending to smart contracts using abi.ABI objects. I didn't like it at first, but once I got accustomed to the Go-Ethereum way of doing things I don't mind it anymore. I wonder if there are any ready to use clients built on top of Go-Ethereum that support common operations like waiting for Tx to get mined and getting all data associated with them (like logs) all at once. It's not hard to build one yourself, but I imagine I'll need something like that for every app using Ethereum, so having small convenience methods like that out of the box would be super nice.

# Concurrency
Golang concurrency is great, we all know that, but it's even greater in working with ETH network. I really like that all operations in Go are sync. It makes tracking flow of the functions very easy. The other great thing about it is how easy it is to optimize code bottle necking your application. Very often I'll start writing some function iterating over list of IDs using `for _, _ := range x` when prototyping and fix pain points later once I know that my solution is correct. Changing sequential code into concurrent one is just too easy with Go, and it's probably one of the best reasons to use it for cryptocurrency projects - after all every single operation performed against the network introduces I/O latency, that Go concurrency can easily fix.

# Math
Probably the worst part of using Golang for Ethereum network was Golang big.Math packages. I really dislike their APIs and modeling any math problems with it is very unintuitive and unreadable. Depending on how you intend to use Ethereum network maybe you won't interact with big math too much (sometimes you just need to wrap some identifier in big.Int which is fine), but if your task involves any more complex math chances are you won't like the experience. Let me give you example of what I'm talking about. Let's say you have two integers, that you need to add and raise to the pow of 2. Readable solution should look as similar to `(x+y)^2` as possible. We don't necessarily need operator overloading here (doesn't exist in Golang) and I don't mind functions like `x.Add(y)`. So how would that problem look in Go?
```go
res := new(big.Int).Exp(new(big.Int).Add(big.NewInt(10), big.NewInt(20)), big.NewInt(2), nil)
```
Not exactly what I'd call readable. I think the main issue with big.Math is that methods don't act on pointers, which is why we need `new(big.Int)` with every operation. The alternative that would be much nicer in my opinion would be something like
```go
res := big.NewInt(10).Add(big.NewInt(20)).Exp(2, nil)
```
This would read left to right in the order that the operations are executed. I'm sure there are good reasons why big math packages are doing the things their doing, but to be honest - it doesn't make for a nice user experience.

# Summary
Because Golang doesn't really like to create new abstractions I think it doesn't work with cryptocurrency space that well - Golang just makes it difficult to hide all that abstraction behind nice interface. Additionally, math associated with crypto space (and there's lots of it) could really use some operator overloading. I know the reasoning behind not providing them, but personally I think they would be great addition to the language. All in all I'd rate my experience 4/7 - "can do again but would pick something else if I had a choice". 

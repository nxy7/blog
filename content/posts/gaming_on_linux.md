---
title: "Gaming on linux"
date: 2023-05-27T09:03:20-08:00
draft: true
---
Every time you ask someone what his favourite programming language is, chances are they'll respond "Typescript!". I think most people just don't have much contact with wide range of technologies and will usually pick language they use the most as their 'favourite'. It's even more obvious in case of Typescript which with all it's flaws is infinitely better than javascript, so in comparion it seems like the best thing in the world.

I'm fine using typescript when writing web applications, even with my dislike for some parts of the language it's still the best tool for the job (while webassembly ecosystem is still small). What I don't like is that javascript and typescript find themselves used for everything, even if there are much better alternatives.

# What's good about Typescript
Let's start with positives. Typescript certainly is improvement over *sad typeless javascript*. Maintaining even small JS repositories is cumbersome and pure JS gives up on tooling that typed languages can provide. 

## It's better than Javascript
Well that requires no explanation. Typescript was created just because writing Javascript was hard and it was hard in a bad way - it was hard to maintain. Some lanugages are challenging, but also rewarding at the same time. Writing Rust sometimes feels like solving a puzzle. You can have 100k puzzle pieces, so it's not easy, but solving it gives some instant dopamine. JS in comparison is like having small amount of puzzle pieces, but you have to have your eyes closed. Solving it is not hard because you don't know how to, it's hard because you literally cannot see anything. You don't know what the method returns, what arguments does it take and you have to rely on weak contracts like comments and naming conventions to solve that. 

## Typescript has really nice tooling and big ecosystem
There's no denying that, people try to build anything in js/ts and it shows. If you want to build something chances are it's already built and ready to use as npm package. Typescript LSP is also really good and you'll have tutorials for everything which makes it easy experience for beginners. 

## Sharing code between projects is easier
To me that's the biggest pro of TS. If you want to quickly build fullstack app, it'll be much easier without having to manually translate classes and types into another language. You pretty much have to use TS on the frontend side (webassembly is still young technology), so if you want to reuse code on the backend there's no other choice than using typescript there too. There are some tools translating classes/structs/types between languages, but setting those up and solving issues that come with them can sometimes take longer than not sharing code at all, so most people just go with TS.

Once WebAssembly becomes real contender for frontend apps it'll be significantly easier to reuse code, but for now that's definitely pro in favour of TS. 

# Why Typescript is bad
## Typescript is scripting language
While being scripting language alone is not really a negative thing, this fact combined with urge to put typescript into any kind of software leads to bad performance. I know: "performance doesn't matter", but I beg to differ on that. There's wide group of people that like their programs being snappy and sometimes just the fact that you're not using electron can be significant enough to make someone choose your software instead of competing solution. Slower performance is just a sign of a bigger problem, inefficient resource usage. If you're running JS/TS server you can probably scale your architecture horizontally to make up for worse performance, for client apps though there's finite amount of RAM you have available and shipping whole JS runtime instead of lean binary can make big difference here, especially if your app is supposed to run in the background. 'Foreground' apps have luxury of being resource intensive, when you're running VsCode, chances are that's the most resource intensive task your PC is doing. Now imagine what would happen if every single app you use in the background didn't care about resource usage. Quickly whole machine would slow down. Sad truth is that's the reality we kind of live in right now. There's plenty of slow bloated software, where even users can easily stop that the programs tend to consume too much ram and can get laggy.

My question is: what's the point of picking scripting language if you have to ship it's runtime with it? The advantage of scripting lanugages is that you can have one interpreter capable of receiving code and acting on it. JS 'hello world' file will be really small, so having interpreter built into browser and having servers send JS to it instead of whole runtimes makes sense. In web context application speed used to be less important than file size (although that's changing as web apps get more complex), so JS was perfect choice for websites. Do we really need it for web servers/offline apps/CLI tools/anything else? 

I like to think that we should pick the best tool for the job, if your job doesn't benefit from *scripting*, maybe there are better choices than *scripting lanugage*. 

## Typescript is confusing
implicit memory alocations lead to 
## Typescript can lie
## Typescript requires compile step

 

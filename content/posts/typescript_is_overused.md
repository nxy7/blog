---
title: "Typescript is overhyped"
date: 2023-05-30T09:03:20-08:00
draft: false
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
Let's say you have the following code
```typescript
let x = {
    value: 1
}

let y = 1

let z = {
    x: x,
    y: y,
    v: 1
}

x.value++
y++
z.v++

console.log(z)
```
The answer is `{ x: { value: 2 }, y: 1, v: 2 }`. I hate the fact that it's not obvious and even if you're familliar with the lanugage it's still necesarry to double check. If you're suprised at this answer, the reason for that is javascript implicit memory management. Some things get copied by reference others are copied by value and while you absolutely can learn all those rules, it'll still slow you down. Let's say for whatever reason we want to change `z` to 
```typescript
let z = {
    x: x.value,
    ...
}
```
It's not a big change, maybe you've decided that there's no need to keep whole object reference inside `z` as all you're really using is just this one value. I believe it's not a stretch to say that even though it's not hard to learn TS copying rules, it's still easy to make those mistakes while coding. If you didn't notice `z.x` value after `x.value++` is now still holding 1 as `x: x.value` was copied by value.

This is just something I dislike about implicit memory management, there's plenty of JS specific weird behaviors like
```typescript
true == ![]; // -> false
Number(true); // -> 1
Number([]); // -> 0
1 == 0; // -> false

// or

null == 0; // -> false
null > 0; // -> false
null >= 0; // -> true
```
while Typescript can catch some of this (not all), it's still like putting lipstick on a pig. Feel free to browse some JS/TS weird posts like [this one](https://javascript.plainenglish.io/here-are-7-weird-things-in-javascript-42da32e7b50) which showcases all those inconsistencies.

## Typescript can lie
I think that's big one. When you're writing Typescript what you're really doing is writing code describing your other code. TS isn't executed by the browser nor V8 and it's compiled to Javascript in the end. It's just some boilerplate that's can make your IDE give you more helpful hints about the program (and can enforce contracts on functions). As you're trying to manually describe what the code is doing and it's not tied to the real functionality TS have things like `!`, `any`, `as` which do not leak into runtime and are just ways to satisfy *Typescript compiler*. You can easily write TS code that's compiling but produces undefined behaviour. 

## Typescript requires compile step
It's 2023, do we really need to compile our code into other code? Setting up new projects should always be one command away and I personally think that dev enviroment simplicity is something valuable. Sure, setting up `tsc` is not difficult, but solving problems related to compile step instead of actual program is just waste of everyone's time. Imao having one context of code execution is enough and there's no valid reason to deal with additional compile step. I'm rooting for types in Javascript in the future as types integrated into JS runtime could also solve my other issues with the language.
 
# Summary
As you can probably tell I'm not the biggest fan of TS. I don't mind writing typescript, but when you actually spend some time seeing alternatives TS seems bleak in comparison. If it makes sense - sure, use typescript. All I'm asking for is not making it default choice for everything :-)
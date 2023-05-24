---
title: "The Best Kubernetes Development Workflow"
date: 2023-05-13T09:03:20-08:00
draft: false
---
At the start I want to point out that all opinions in this piece are just that - opinions. I've tried multiple workflows and all I want to do in this
article is sharing what I've landed at and what difficulties I've had using other methods of developing in kubernetes. My main motivation for writing it is 
sparing someone difficutlies related with setting up dev environment. There are many solutions out there claiming that they make working with k8s easier, but with greater ease of development comes another layer of abstraction that sometimes might actually make things worse.

Let's start with requirements I have for my dev environment:
- short feedback loop
- portability (I've windows and linux machine, but I also anyone to be able to easily contribute to my projects)
- easy to understand (dev environment should not introduce complexities that make development slower)
- share configuration with prod environment (ideally I don't want to maintain multiple configurations other than changing few secrets)
- share project caches across multiple runs and multiple services (I'm often developing rust projects that produce multiple binaries from one workspace and I want to
reuse cache from 'target' folder between those services)

# Dev environment solutions

I've encountered 4 approaches of solving k8s dev environment issues:
- building container image and swapping pod with new image
- mounting code inside kubernetes pod
- mounting binary inside kubernetes pod (only applicable to compiled languages like golang/rust/etc)
- routing traffic from kubernetes to local machine

Let's talk more about each approach and it's pros and cons

## Building new container image
Probably the easiest and most naive approach is rebuilding container image and recreating kubernetes pod with it. This works 'correctly' in a sense that it's the most
predictiable solution, sidesteps some issues with mounting code and building it inside container and is easy to reason about as basically we're just making new 'prod'
version of our app - dev and prod environments here are almost the same. I personally don't like this approach as it doesn't provide short feedback loop. Making new images
and deploying them can be automated (f.e. Tilt can watch your code and any time it detects any changes will automatically create new image and deploy it into k8s), but still
can require few seconds at best, minutes if you're not so lucky. You could have separete 'dev' Dockerfiles but to me it feels like putting lipstick on a pig. Fundamentally
you cannot have great dev experience this way.

## Mounting code inside pod
Much faster approach is mounting code inside pod and running it there. This is as fast as executing the code locally and the changes are nearly instant. There are multiple tools supporting this way of working (I've tried Tilt and Devspace) and it works okayish but there are some roadblocks. First of all syncing the code might take some time
when launching 'development pod' and while it's not necessary bad as it's one time cost it reveals bigger issue. File syncing with kubernetes works by literally copying the files. Doesn't sound bad until You want to share cache across multiple development pods. I'm developing rust backend with ~5 microservices. Those microservices rely on each other so when I build them locally first build will be slow, but the cache produced during this process can be reused when building other services. If I'd want to sync that cache into the pods it would be copied 5x times so I'd end up with 6 copies of the same cache. Yikes. Currently my 'target' folder weights around 15GB so we're looking at 90GB of files, 75GB transfered any time I start development. That's issue with compiled languages but scripting ones also have some problems. When creating frontend services you might notice differences between state of your development pod and local environment. Your code editor functions locally and looks for dependencies in local project so you need to 'npm install' both locally (for your IDE) and in development pod (so the program can actually run). These issues have some workarounds f.e. Devspace allows creating PVC for development pods so you can resuse cache between pods (not with your local environment though) and I guess if you're not using package manager with symbolic links (like pnpm) you could also just mount node_modules folder which would be created in the pod anyway, so overall this solution is not bad.

## Mounting binaries
This only works for compiled languages. I really like the idea of mounting binaries as all development happens locally and all you're really doing is watching produced output to sync it into the pod. This approach is supported by both Tilt and Devspace and when it works it works very well. The only problem with it is the fact that you're producing artifacts in different context than their deployment target so there might be some issues you'd have to deal with. These might vary case by case. I had to fight with missing linkers inside containers and different build targets. Probably it might not work if you're working on Windows (native, WSL works just fine) or MacOS so in big teams with multiple OS i'd avoid it

## Routing traffic from k8s to local machine
Now comes my favourite solution - rerouting traffic from k8s to local machine. Very versatile approach that allows for totally local development with all its benefits
(sharing cache, no files duplication, the shortest feedback loop) while still allowing access to k8s resources. The only tool I've tested for this approach is Telepresence and this definitely is my favourite. Networking issues might be hard to solve so initial setup could be hard (f.e. firewall could block requests) but it's well worth it. Telepresence proxies requests from k8s and makes all resources from it accessible at their k8s adresses. Because of this approach you don't need separate config files for
development and production while still retaining great developer experience.

# What setup I've landed on
If you've read everything up to this point it's probably not a shock that I've chosen using Telepresence. As telepresence commands can get repetitive rather quickly I've scripts that setup my project in 'prod' mode and then scripts that turn services in development mode (start telepresence -> run service -> stop telepresence when service is stopped). I might write followup with exact configuration I'm using in the future.

Thanks for reading this article, I hope that someone found it useful :) Have a great day everyone.
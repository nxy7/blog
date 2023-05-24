---
title: "Why I migrated from WSL2 and Windows to Linux"
date: 2023-05-03T09:03:20-08:00
draft: false
---
Hello everyone, this time I'll share why I stopped using Windows with WSL2 (Windows subsystem for Linux) as my development environment and instead opted for dual boot setup (I'm keeping Windows just for few games that don't work very well with Linux). 

To make it more clear I should probably share my dev experience with WSL2 first.

# My dev environment journey
I've started programming a few years ago (probably around 2017) with Windows as my primary system. As you might know most developer tools are supported on Windows, but it certainly is more cumbersome than on Linux. Windows is made for "users", not "creators" and I think it shows. Installing tools and fixing issues with missing dependencies is much harder on Windows in my experience. You certainly can do it, it's just not as smooth as on Linux side.

At some point I decided to give WSL a try and it was pretty pleasurable overall. When I was learning webdev basics it was sufficient to start some simple applications and display interfaces in Windows web browser. Once my applications became more complicated and I've started using docker compose, which later was replaced by kubernetes, having two contexts started being pretty irritating. Most tools assume you have access to localhost and while usually WSL2 localhost is forwarded to Windows it's not always the case. Now I've had to always keep in mind which devtools are exposed to Windows correctly and which are only accessible from whithin WSL2. Dev environment should be easy so you can focus on real problems, that's when I've decided to switch.

# More issues
## Broken Windows
For whatever reason Hyper-V breaks my PC. I'm not sure what causes this behavior, but with Hyper-V enabled 4/5 times I boot windows it loads without internet access and bluescreens after few minutes. Well, that's annoying. I was looking for solutions to this issue, but it doesn't seem common so I assume it's linked to my hardware somehow. If it boots correctly (1/5 chances :P) it would work without any issues.
## Worse PC performance
Hyper-V lowers PC performance. If you didn't know Hyper-V abstracts your windows system so it's basically Virtual Machine with some extra benefits ( ͡° ͜ʖ ͡°)
You cannot have that kind of abstraction with no performance impact and depending on your hardware it can range from not noticable to 20% performance hit.

## Networking problems
As I've written above, two application contexts got old real quick. I've had to keep in mind that windows localhost is not always the same as WSL2 localhost. Additionally some nice kubernetes tools like `telepresence` (which works by creating subnetworks so you can access services by URLs like `http://database.myapp` no matter if it's run locally or remotely) would not leak into windows userspace. 

# Linux is just better
Linux is real place for software innovation. People spend their time making it better and faster constantly, bad ideas can be filtered out before they make it into final release and there's plenty of experimentation. I'm not into configuring tools constantly so one piece of innovating software I find really valuable is Nix and NixOS. This allows me to spend some time wrting my PC configuration __ONCE__, so I don't have to waste my time on it ever again. Got new PC? Copy config files and run one command. Congratulations, you have the same system setup that you're used to. This kind of experimentation is just not possible in closed source Windows.


---
title: "One Year with NixOS"
date: 2023-05-27T09:03:20-08:00
draft: true
---
In this article I'll share with you pros and cons of NixOS system after using it for about a year. First things first - my NixOS configuration is publicly available at my [github](https://github.com/nxy7/dotfiles), feel free to lurk around and see how I'm configuring my system. If you think that my configuration could be improved, don't hesitate and leave comment. I'd like this repository to be something that others can use to bootstrap their configurations :-)

# How it all began

## My Linux journey 

I didn't like Linux at first. I'm kind of ashamed of that now, as it is my primary OS now and I didn't appreciate it for embarassingly long time. Throughout my whole university studies I avoided it and coded on Windows (even without WSL). Over time when I've started coding more I've grown more interested in this whole Linux business. Why are people using it? What's their deal? Sure - using `apt get` to download something quickly seems nice, but is that really so hard to download some installer and install things manually?

Yes! Yes it is. Windows really is a mess and despite belonging to great company (there are some nice things about MS), it feels like it lacks direction and thought. I think it speaks volumes about Linux, that despite infinitly smaller financial backing it is the place of innovation in OS space. Text is the best medium of communicating with PC, it allows to automate things in predictible manner, and in that regard Windows is years behind Linux. I think Windows has some built in app repository that you can use from CLI now, but nowhere as big as Linux. It's still much harder to automate things, you have two separate terminals, powershell is just awful to look at and there's nothing nice about it in terms of programming. It's OS that's not made for us.

Anyway as You can tell - I've started using Linux. At first as WSL2, but then tried to jump over to Linux as my daily driver and it was super easy experience. Even while I was using WSL2 I was already using Nix to package my software so NixOS felt like natural choice for me. I might be a bit unusual in that regard - NixOS is seen as niche distro and probably most people first use some of more mainstream ones, but to me it was easy choice. In my mind NixOS is to other distros what Linux is to Windows. It has rough edges - sure - but workflows it allows are as smooth as it gets. I've hopped to Linux to improve my dev environment and stopping half way there wouldn't seem right.

## So I've installed NixOS

I've had very short experience using arch (just few days/weeks), but it didn't cut it. Installing NixOS was super easy. More than you'd think. Even while using WSL2 I've had most of my tools managed by Nix Home-Manager, so clean install of NixOS quickly had my familiar tools. My system configuration is rather lean, I keep most things in user space (not system space), but NixOS allowed me to experiment with some kernel stuff. I've custom built kernel with all features required by `Cilium` which is kubernetes responsible for kubernetes network layer. Doing that on other system would be very scary, but with NixOS I've always knew that if things go wrong I can just roll back to previous generation of my system. NixOS is really great in that regard. Other than that my config has some udev and networking rules. I think it's great to have central place to check the state of the system. It's much more convinient than remembering paths of firewall configs, udev configs, commands repsonsible for listing packages from multiple repositories etc. etc. 

# Pros

## Reproducible is better than stable

# Cons

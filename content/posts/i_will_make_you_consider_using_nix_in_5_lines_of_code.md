---
title: "I will make you love Nix in 5 lines of code!!"
date: 2023-12-06T09:03:20-08:00
draft: true
tags: ['nix']
---
I'm not kidding. 5 lines of code (with some interactive input in between) and you'll appreciate what this crafty tool can do. If you haven't heard of Nix yet - it's package manager built with declarative management in mind. It allows reproducing systems of any size with just one command and if you've ever thought that managing conflicting versions of programs in dev environment isn't something that we should be doing in 2023, then chances are you'll like this short presentation.

# Let's do this
Those are the 5 lines of code. We'll install Nix with flakes enabled using installer from determinate systems, then we'll create temp directory (without it, we'd need just 3 lines of code :P), initiate it with template from my GitHub repository and see some magic happen. You can copy provided code all at once and wait until Nix installs dependencies listed in my flake template and while it's installing come back for further explanation on 2 actual Nix commands we're using here.

```bash
curl --proto '=https' --tlsv1.2 -sSf -L https://install.determinate.systems/nix | sh -s -- install
mkdir loveNix
cd loveNix
nix flake init -t github:nxy7/flake-templates#iLoveNix
nix develop .
```


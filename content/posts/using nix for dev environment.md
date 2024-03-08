---
title: "How to use Nix to make reproducible dev environments"
date: 2023-03-17T09:03:20-08:00
draft: false
tags: ['nix']
---
Perhaps just like me, you see value in making easily reproducible dev environments. Wouldn't it be great if you could have projects with different versions of dependencies that anyone could just jump right into, type command or two and have everything necessary for contributing?

Well, I think it's something that everyone should take into consideration. Reproducible dev environments have one more pro - there are self-documenting state of your project. You can look into things your dev env provides you to know which programming languages and tools the project is using.

# Leading dev environment solutions
I've personally tried 2 tools that are trying to achieve easily reproducible environments: *Nix* and *DevContainers*. Both come with some additional complexity when setting up and both are great tools. I prefer Nix over DevContainers as not every IDE supports DevContainers. IMAO the biggest things that DevContainers have going for them are their integrations with VsCode and online Dev Environments. If you specify *.devcontainer.json* file at the root of your project you could use something like github codespaces to load the project in browser version of vscode, with the project running on github server. It can be significantly faster than working locally if you're coding using cheaper laptop. Using Nix inside DevContainer also is a possibility - you would use Nix to produce coding environment and DevContainer to provide Nix, but that's out of the scope of this post.

# Getting started with Nix

## Installation
First things first - you have to install Nix on your system. I've personally tried it on Ubuntu (WSL2), Manjaro and Arch systems but Nix is supposed to work with MacOS too. Everything You need is on [nix page](https://nixos.org/download.html#download-nix), just remember to use 'single-user installation' instructions if you're running wsl2. After running a single script You're done. Well that was easy ^_^

## Enabling flakes
Nix has experimental feature that is pretty much the direction of the project - flakes. Flakes allow for declarative configuration of project dependencies and building process while being easily shareable. They integrate nicely with commonly used VCS - git. Nix has some baggage coming with it, that might be confusing for newcomers (there are multiple ways to achieve the same thing) but if you have option to use flakes I'd highly recommend it. To enable flake support add `experimental-features = nix-command flakes` to */etc/nix/nix.conf* (multi user installation) or *~/.config/nix/nix.conf* (single user installation).

## Explaining Nix flake structure
Open any project you're developing and run `nix flake init`. It should produce the following flake.nix file
```nix
{
  description = "A very basic flake";

  outputs = { self, nixpkgs }: {
    packages.x86_64-linux.hello = nixpkgs.legacyPackages.x86_64-linux.hello;
    packages.x86_64-linux.default = self.packages.x86_64-linux.hello;
  };
}
```
Default flake produced by `nix flake init` doesn't provide any developer shells and instead specifies packages that the project can build. In this case it's 'hello' package from nixpkgs (the biggest repository for nix packages). You can try it out by either running `nix run .` (which will build and run `hello` package) or by running `nix build .` (that will only build hello binary and add 'result' symlink in your project with binary in `result/bin/` folder).

Remember - flakes are declarative configuration of your project, so you need to know what options does flake.nix provide. As you can see at root level there's `description` and `outputs` properties. Description is pretty self-explanatory, outputs on the other hand specify things your flake can do for you. Our current flake has output that takes 2 arguments `self` and `nixpkgs` and provides 2 outputs `packages.x86_64-linux.hello` and `packages.x86_64-linux.default`. When we're running `nix run .` we're telling nix: "can you build and run default package for me?". You could also run specific output like `nix run .#hello`.

Now few words about arguments we provide to nix `outputs`. I'm not here to teach you about Nix language, and you'd be better off using official resources for that, but I'll try explaining structure of it. `outputs` field is in reality a nix function that takes some arguments and returns structured object (like JSON). Some fields in returned object are *special* and have meaning to other nix commands. `nix run` and `nix build` are looking for 'packages.{system}.{outputName}' property of `outputs`. Default flake has just 2 dependencies: "self" and "nixpkgs". Those 2 are special and provided to you by default. You could also provide `inputs` field at the root of flake file like:
```nix
{
  description = "A very basic flake";
  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixpkgs-unstable";
    flakeUtils.url = "github:numtide/flake-utils";
  };

  outputs = { self, nixpkgs, flakeUtils }: {
    packages.x86_64-linux.hello = nixpkgs.legacyPackages.x86_64-linux.hello;
    packages.x86_64-linux.default = self.packages.x86_64-linux.hello;
  };
}
```
We've provided 2 inputs for our flake here, one overrides nixpkgs, the other provides some useful Nix functions that we can use in our own flake. What those are really doing is just importing `flake.nix` files from other repositories, you can imagine how powerful is that concept. Let's modify flake.nix file, so it can provide developer shell for us.
```nix
{
  description = "A very basic flake";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixpkgs-unstable";
    flakeUtils.url = "github:numtide/flake-utils";
  };

  outputs = { self, nixpkgs, flakeUtils }:
    let pkgs = nixpkgs.legacyPackages.x86_64-linux.pkgs;
    in {
      packages.x86_64-linux.hello = pkgs.hello;
      packages.x86_64-linux.default = pkgs.hello;
      devShells.x86_64-linux.default =
        pkgs.mkShell { packages = with pkgs; [ hello nodejs-slim]; };

    };
}
```
As you can see, we've added pkgs variable that is just deeper field in nixpkgs flake. Additionally, we've created new output: `devShells.x86_64-linux.default` that uses *mkShell* function provided by our newly created `pkgs` variable (we could use whole path here directly, but making `pkgs` variable makes it more readable and is pretty common practise).
*mkShell* function takes object that can take have multiple fields, but we're using just 'packages' here. That's the place we specify our dependencies. You can find almost any open source software you might need at [nix search page](https://search.nixos.org). Are you using NodeJS in your project? Type `nodejs` in search fields and check what packages does `nixpkgs` provide. You can then just copy the package name like *nodejs-slim* (if you want NodeJS with fewer dependencies) into your project devshell like
```nix
devShells.x86_64-linux.default =
    pkgs.mkShell { packages = with pkgs; [ hello nodejs-slim]; };
```
Now run `nix develop .` ("nix can you run default devshell for me") and try `hello` and `nodejs` commands. Versions of software provided to you are pinned in flake.lock file so if someone copies your project he'll get the same `hello` and `nodejs` version. 

I think nix is great piece of technology. Sure it comes with another language you need to understand and has steep learning curve, but in the beginning even just using simple flakes like the one I've shown here is very useful and can greatly improve your productivity and developer experience. I hope you found this post useful. Have fun using nix and have a great day :-)

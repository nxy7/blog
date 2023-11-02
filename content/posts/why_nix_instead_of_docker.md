---
title: "Why You should be using Nix instead of Dockerfiles"
date: 2023-10-26T13:03:20-08:00
draft: false
---
Lately I've had to move building and pushing docker images process to CI. To my surprise it wasn't as straightforward as it should be.
Building images on GitHub CI runner worked just fine, but my image registry had no SSL and adding 'insecure' registry to docker can be a pain.
Sure - I could write script that changes docker `daemon.json` configuration and restarts it, but it felt like a hack. I've also tried using
podman and buildah, but both of them didn't seem to get the job done in easy manner. 

That's when I've decided to give Nix a try. I'm using Nix everywhere at this point - it handles my [PC configuration](https://github.com/nxy7/dotfiles),
it handles my project dependencies, and I'm sure it will be able to build my images!

In hindsight the easiest way to do it would be using Skopeo 
(which I've actually ended up using in my nix solution) to push images built by docker, but issues mentioned above gave me a push to explore other possibilities
and let me tell you - I don't regret it.

# Why would I use Nix instead of Docker?!

### Same dev and prod dependency versions

Nix is ecosystem that greatly benefits from You using it in many places. If You're using Nix to manage project dependencies, then chances are that some of them will be required in final docker image. With Nix you can be sure that dependencies You're using locally are the same dependencies that will be contained within your final docker image. 

### Smaller images

With Nix it's also really easy to make sure You don't end up putting too much junk inside your images. That's the advantage of declarative configuration - we specify what is needed and nix takes care of it for us. This way imperative commands like `apt update` will not accidentally pollute our containers. It's not a guarantee however and depends on the 'quality' of packages you decide to include. Some packages may still include things that are not necessary, but it's much easier to track them down - just look into .nix source of pkgs you import. 

### Reuse container image creation logic

Nix also makes reusing code for multiple docker images much easier. Let me show you one example. One of the projects I'm developing is made using rust. It produces few binaries
each from separate workspace member. My build phase looks like this `buildPhase = "cargo build --release --bin ${name}`, so the only variable in whole Dockerfile would be binary name.
To reuse logic responsible for building those images I've made separate `buildPackage.nix` file:

```nix
{ pkgs ? import <nixpkgs> { }, path ? ./. }:

let
  cargoFile = (builtins.fromTOML (builtins.readFile "${path}/Cargo.toml"));
  name = cargoFile.package.name;
  version = cargoFile.package.version;
  buildRustPackage = (pkgs.makeRustPlatform {
    cargo = pkgs.cargo;
    rustc = pkgs.rustc;
  }).buildRustPackage;
  dockerTools = pkgs.dockerTools;
in rec {
  app = buildRustPackage {
    pname = name;
    version = version;

    src = ./.;
    buildAndTestSubdir = name;

    nativeBuildInputs = with pkgs; [ pkg-config ];
    buildInputs = with pkgs; [ openssl ];
    PKG_CONFIG_PATH = "${pkgs.openssl.dev}/lib/pkgconfig";
    buildPhase = "cargo build --release --bin ${name}";
    installPhase = ''
      mkdir -p $out/bin
      cp target/release/${name} $out/bin/
    '';

    cargoLock = { lockFile = ./Cargo.lock; };
  };

  buildImage = dockerTools.buildImage {
    name = name;
    tag = "latest";

    copyToRoot = pkgs.buildEnv {
      name = "image-root";
      paths = [ app pkgs.busybox pkgs.libtree ];
      pathsToLink = [ "/bin" ];
    };

    config.Entrypoint = [ "/bin/${name}" ];
    config.Env = [ "SSL_CERT_FILE=${pkgs.cacert}/etc/ssl/certs/ca-bundle.crt" ];
  };
}
``` 

It's just a nix function! Now in my `backend/default.nix` I can easily build several rust binaries using the same 'dockerfile'.

```nix
let buildPackage = import ./buildPackage.nix;
in {
  serviceA = buildPackage {
    inherit pkgs;
    path = ./serviceA;
  };
  serviceB = buildPackage {
    inherit pkgs;
    path = ./serviceB;
  };
}
```

### Build and push images with one command

I admit, anything can be done in one command with shell scripts, but Nix gives us much stronger guarantees about correctness of our
logic and I personally feel much more confident about using Nix instead of equivalent shell scripts. Anyway I've shown how I build my rust images
in previous step, now let's make Nix push the image for us.

```nix
# reusable 'push' function that accepts docker image, name and optional tag
pushImage = { imageTar, name, tag ? "latest" }:
  pkgs.writeShellScriptBin "push-script" ''
    ${pkgs.skopeo}/bin/skopeo copy docker-archive:${imageTar} docker://${yourImageRegistry}/${projectName}/${name}:${tag} --dest-tls-verify=false
  '';

# in output of flake
packages = {
  serviceA = pushImage {
    imageTar = serviceA.buildImage;
    name = "serviceA";
  };
  serviceB = pushImage {
    imageTar = serviceB.buildImage;
    name = "serviceB";
  };
};
``` 

Now to build and push image we run `nix run .#serviceA`. As You can probably tell I'm actually using skopeo to push the final image and IMAO
one liner visible in pushImage function is great example of why Nix is so great. If I'd want to make shell script with equivalent functionality it would be
necessary to make script that validates arguments given to it from CLI, then you'd need to make sure `skopeo` is installed (or worse - install it imperatively)
and only then You'd be able to copy the image. Using Nix for this task it's trivial. There's no need to think about any of those things. You can just run anything
packaged for Nix as if it was already installed.

### Nix is the best abstraction for building environments

Think about your docker container. What's it purpose? It isolates network and runtime dependencies from your system. That's what allows you to select which version
of postgres you want to run easily. Even if they are used to making reproducible environments, it's not the best tool for the job - f.e. they lack version locking mechanism.
It can be used for this task, because as I mentioned it isolates runtime dependencies, but it's side effect and not something baked into containers design.
When we ship software we really need both - reproducible environment that can satisfy our runtime needs and runtime isolated from the underlying system.
Nix + Docker combination gives us both, which is why I think using Nix to build images is great idea.

# What next

Chances are You liked this short showcase of Nix, but still don't know how to use it Yourself. Nix learning curve is pretty hard after all. If you want to build images using Nix, but aren't very proficient in using the language I suggest the following learning path. First - package your software using Nix. In many cases it'll boil down to using appropriate builder (like buildRustPackage in examples above, but there are similar functions for other languages as well), but sometimes it can be much harder, especially if Your project depends on software not available in nixpkgs.
When your app is packaged correctly look into `dockerTools.buildImage` documentation. It's rather simple to use, but if You have any questions about it feel free to ask in the comments. I sincerely think that the best way to learn nix is by actively using it. No amount of reading helps as much as just porting some Dockerfiles to flake.nix files, so experiment and have fun :-) 

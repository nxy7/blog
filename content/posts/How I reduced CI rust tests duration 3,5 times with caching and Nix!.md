---
title: "How I've reduced CI rust tests duration 4x with caching and Nix!"
date: 2023-04-20T09:03:20-08:00
draft: false
---

One of my project consists of backend (rust) split into many services and frontend made with Svelte. Sooner rather than later I found out that rust builds that we're rather quick locally took ages in CI, resulting in poor CI feedback experience and test times exceeding 15minutes. This post was made to help people struggling with simmilar issues.

# Project Structure
The way I structure my projects will influence some of the futhurer solutions, so I thought it's worthwhile sharing
```yaml
    # in this case all backend services all written in rust
    backend:
        - serviceA
        - serviceB
        - serviceC
    frontend:
        - NodeJS Project
    # to make CI jobs testable locally I prefere to make scripts for them and run scripts from github workflows.
    # I'm using nushell as my script shell and not bash so you will have to 
    scripts:
        - ci_tests script
        - other scripts...
```

# Test script
As I've mentioned above I'm running tests from scripts locally and invoking scripts files in CI. Here's how my current ci_tests.nu script file looks like. You  might not be fammiliar with nushell (it's a shell but also a scripting language) but I'm sure you can get the general idea.

```nushell
#!/usr/bin/env nu
use ./utils.nu;

let services = [
  {folder: "frontend", lang: "node"},
  {folder: "backend/payouts", lang: "rust"},
  {folder: "backend/stream_chat", lang: "rust"},
  {folder: "backend/main", lang: "rust"},
]

# run tests for all services
export def main [] {
    let root = (utils project-root);
    cd $root;

    ci-setup
    build-rust-workspace


    $"(ansi green_bold)Starting tests(ansi reset)\n\n"
    let total_start_time = (date now);
    let testResults = ($services | par-each {|service|
      cd $service.folder

      mut test = null
      let start_time = (date now);
      if ($service.lang == "rust") {
        $test = (
          do { test-rust } | complete
        )
      } else if ($service.lang == "node") {
        $test = (
          do { test-node } | complete
        )
      }
      let end_time = (date now);
      let duration = ($end_time - $start_time);

      mut res = {"service": $service.folder, "test_result": $test, "duration": $duration}
      if ($test.exit_code != 0) {
        $res = ($res | insert status ❌)
        $"($service.folder) failed"
      } else {
        $res = ($res | insert status ✅)
        $"($service.folder) passed"
      }
      return $res
    })
    let total_end_time = (date now);
    let total_test_time = ($total_end_time - $total_start_time);

    $testResults | select service status duration

    mut allPassed = true
    for res in $testResults {
      if ($res.status != "✅") {
        $allPassed = false
        $res.test_result
      }
    }

    if ($allPassed == false) {
      error make {msg: $"(ansi red_bold)Some tests failed. Total test time: ($total_test_time)"}
    } else {
      $"(ansi green_bold)All tests passed in ($total_test_time)"
    }
}

def test-rust [] {
  cargo nextest r
}

def test-node [] {
  pnpm test
}

def ci-setup [] {
  if ((".env" | path exists) == false) {
    cp .env.example .env;
    "Copied .env.example to .env"
  }
}

def build-rust-workspace [] {
    $"(ansi mb)Starting to build rust workspace"
    let start_time = (date now);
    let build_backend = {|| cd backend; cargo build --all };
    let build_res = (do $build_backend | complete);
    let end_time = (date now)
    let total = ($end_time - $start_time)
    $"(ansi mb)Built rust workspace in ($total)"
}
```

This script gives me really nice output in my CI
![CI Pipeline output](https://i.imgur.com/NonjwGm.png)
# Solution
There are two main techniques that I've used to reduce CI times.
- Nix to manage project dependencies
- Cache rust build artifacts and project dependencies (managed by nix)

Arguably you can get away with just caching, but I think the two really go well together.

## Nix
Nix describes itself as `tool that takes a unique approach to package management and system configuration.`. You don't need to use it as system configuration tool to get rewards from it's ecosystem. In short: nix allows you to specify your projects dependencies and let anyone run your project from any machine. Imagine you clone your project onto machine that doesn't have rust or nodejs installed. With nix you can specify your project dependencies in flake.nix file (there are other ways but nix seems to head toward using flakes) and run one command `nix develop . -c bash` to get shell with all dependencies ready to use. If that doesn't sound awsome to you I don't know what will.

Without getting into internals of nix, when you run command above nix will download ALL dependencies needed by your project into 'nix store', that we can later cache in CI. Thanks to that if your project relies on rust, your CI will download rust only once and reuse it in futhurer CI runs.

Here's how my flake.nix file looks like
```nix
{
  description = "A very basic flake";
  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixpkgs-unstable";
    flakeUtils.url = "github:numtide/flake-utils";
  };

  outputs = { self, nixpkgs, flakeUtils }:
    flakeUtils.lib.eachSystem [ "x86_64-linux" ] (system:
      let pkgs = import nixpkgs { inherit system; };
      in {
        devShells.default = pkgs.mkShell { packages = with pkgs; [ ]; };
        devShells.ci = pkgs.mkShell {
          packages = with pkgs; [
            rustc
            nushell
            cargo
            bash
            cargo-nextest
            pkg-config
            openssl.dev
            clippy
            nodejs-slim
            nodePackages_latest.pnpm
          ];
        };
      });
}
```
There are many resources on nix flakes, but all You need to know for now is the fact that with this flake when i run `nix develop .#ci -c $SHELL` nix will make development shell for me looking for devShells.ci.packages list where all my dependencies are listed. I'm using $SHELL env variable as custom command, so I immiedietly start running my default shell - Nushell.

As you can see my dev dependencies include: rustc, nushell, cargo, nodejs and pnpm. I list all dependencies that my scripts or services might need here so it can be built both locally and in CI.

One additional benefit of using nix is that my test script can be run in CI and locally with the same effect (kind of). There's option to run `nix develop` command with `-i` flag to ignore environment. If you use this option shell that you get in CI and on your local machine should be pretty much identical and hopefully output of those tests should be the same. There are some caveats tho, I didn't find any good way to get docker into shell scope yet and my tests rely on testcontainers so for now I'm not using 'ignore environment option'.

To use nix in my CI use the following workflow action
```yml
      - uses: cachix/install-nix-action@v20
        with:
          nix_path: nixpkgs=channel:nixos-unstable
          extra_nix_config: |
            store = /home/runner/nix
            keep-outputs = true
            keep-derivations = true
```
Setting store inside /home/runner is actually pretty important here, if You don't do that You might run into some permissions issues when trying to save data from cache.

## Caching

Github has actions that make caching very easy. I use *cache/restore* and *cache/save* actions to manage my cache. Here's my workflows/tests.yml file, I'll later explain parts related to caching.
```yml
name: Tests
on:
  push:

# test runner environment variables
env:
  CARGO_TERM_COLOR: always
  SQLX_OFFLINE: true

jobs:
  tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Restore Nix Store
        id: restore-nix-cache
        uses: actions/cache/restore@v3
        with:
          path: |
            /home/runner/nix
          key: ${{ runner.os }}-${{ hashFiles('./flake.nix', './flake.lock') }}
      - name: Restore Incremental Builds
        id: restore-cargo-cache
        uses: actions/cache/restore@v3
        with:
          path: |
            ~/.cargo/bin/
            ~/.cargo/registry/index/
            ~/.cargo/registry/cache/
            ~/.cargo/git/db/
            ./backend/target
          key: ${{ runner.os }}-cargo-${{ hashFiles('./backend/Cargo.lock') }}
      - uses: cachix/install-nix-action@v20
        with:
          nix_path: nixpkgs=channel:nixos-unstable
          extra_nix_config: |
            # save space on disk and in cache
            # auto-optimise-store = true
            store = /home/runner/nix
            # keep all store paths necessary to build the outputs
            keep-outputs = true
            keep-derivations = true
      - name: Run Tests
        run: |
          nix develop .#ci -c nu ./scripts/ci_tests.nu
      - name: Save nix store
        uses: actions/cache/save@v3
        if: ${{ steps.restore-nix-cache.outputs.cache-hit == false }}
        with:
          key: ${{ runner.os }}-${{ hashFiles('./flake.nix', './flake.lock') }}
          path: |
            /home/runner/nix
      - name: Save incremental builds cache
        uses: actions/cache/save@v3
        if: ${{ steps.restore-cargo-cache.outputs.cache-hit == false }}
        with:
          key: ${{ runner.os }}-cargo-${{ hashFiles('./backend/Cargo.lock') }}
          path: |
            ~/.cargo/bin/
            ~/.cargo/registry/index/
            ~/.cargo/registry/cache/
            ~/.cargo/git/db/
            ./backend/target
```

As You can see, actually running tests is the shortest part of the whole thing:
```yml
      - name: Run Tests
        run: |
          nix develop .#ci -c nu ./scripts/ci_tests.nu
```
This line runs `nix develop .#ci` command, which brings developer dependencies from devshell named "ci" into scope and `-c nu ./scripts/ci_tests.nu` executes nushell script from the beggining of this post. I find it easier to hack around script file rather than executing bunch of "runs:" in workflow yaml  file.

Anyway let's see how we can reuse nix dependencies and rust build artifacts.
```yaml
      - name: Restore Nix Store
        id: restore-nix-cache
        uses: actions/cache/restore@v3
        with:
          path: |
            /home/runner/nix
          key: ${{ runner.os }}-${{ hashFiles('./flake.nix', './flake.lock') }}
      - name: Restore Incremental Builds
        id: restore-cargo-cache
        uses: actions/cache/restore@v3
        with:
          path: |
            ~/.cargo/bin/
            ~/.cargo/registry/index/
            ~/.cargo/registry/cache/
            ~/.cargo/git/db/
            ./backend/target
          key: ${{ runner.os }}-cargo-${{ hashFiles('./backend/Cargo.lock') }}
```
I split caching into two steps because otherwise we wouldn't be able to reuse nix/cargo cache if the other one changes. Let's read caching step line by line.
`name: Restore Nix Store` - Set step name displayed in CI
`id: restore-nix-cache` - set step ID, it's usefull so we can reference outputs of the step later on
`uses: actions/cache/restore@v3` - use actionts/cache/restore action
`with:` - run action with the following parameters
`path: |
/home/runner/nix` - we want to load cache into /home/runner/nix folder that we specified as nix store folder when setting up nix in CI
`key: ${{ runner.os }}-${{ hashFiles('./flake.nix', './flake.lock') }}` - every cache needs key associated with it. It's necesarry so Your CI knows which cache should it load. As all our project dependencies are listed in flake.nix i create my key as combinantion of runner system and hash of flake.nix+flake.lock. This way if I add dependency and drop it later on my flake.nix+flake.lock hash should be the same and I'll reuse older cache.

Restoring cache for rust build artifact works the same, we just specify more paths and use different hash in "key" propety.

Now, when You run this workflow for the first time there obviously will be no cache to load, we need to take care of that. The following part of workflow takes care of it.
```yml
      - name: Save nix store
        uses: actions/cache/save@v3
        if: ${{ steps.restore-nix-cache.outputs.cache-hit == false }}
        with:
          key: ${{ runner.os }}-${{ hashFiles('./flake.nix', './flake.lock') }}
          path: |
            /home/runner/nix

      - name: Save incremental builds cache
        uses: actions/cache/save@v3
        if: ${{ steps.restore-cargo-cache.outputs.cache-hit == false }}
        with:
          key: ${{ runner.os }}-cargo-${{ hashFiles('./backend/Cargo.lock') }}
          path: |
            ~/.cargo/bin/
            ~/.cargo/registry/index/
            ~/.cargo/registry/cache/
            ~/.cargo/git/db/
            ./backend/target
```
Let's again break it into lines (but I'll skip steps explained earlier):
`uses: actions/cache/save@v3` - this time we use cache/save action
`if: ${{ steps.restore-nix-cache.outputs.cache-hit == false }}` - Save action first compresses cache and then tries to save it. Github CI caches are immutable, so if no dependencies changed and build artifacts are the same we'll produce the same hash, which results in the same cache key. This won't fly and CI will waste ~30sec on compressing caches that it cannot save. Because of this I'm checking if *cache/restore* found cache in earlier steps and if it did we're not saving any cache. If we change 'flake.nix', 'flake.lock', or 'cargo.lock' files, we'll produce different cache and we'll be able to save it.
`path: | /home/runner/nix` - paths we want to save to cache

### Additional 'hacks'
- I'm running tests in parallel using nushell built in par-each function
- Rust tests are ran with cargo nextest, which supposedly is faster, but I didn't compare it to cargo test myself

# Results
Before:
![](https://i.imgur.com/HkBvqDN.png)
After:
![](https://i.imgur.com/V9JRWi4.png)

I think those are great results, especially considering that I run more tests now, so I think I might have gained more than 4x speed. Hope that someone finds this post helpful. Have a great day :-)

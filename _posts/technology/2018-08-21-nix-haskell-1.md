---
layout: post
title: "Understanding Nix & Haskell: Project Setup"
modified:
categories: technology
excerpt:
tags: []
image:
  feature:
date: 2018-08-08T00:00:00-00:00
modified: 2018-09-10T21:45:35
---

I've been working a lot with Haskell and Nix lately, and I thought now would be
a good time to pause and write about it before I either forget what I've learned
or forget why I found it difficult in the first place. This post will cover:
- pinning haskellPackages for your project 
- setting the GHC version you want
- overriding haskell packages
- getting documentation for your packages and enabling Hoogle 

I will assume basic knowledge of Nix (i.e. [this](https://learnxinyminutes.com/docs/nix/) makes sense to you)
and some basic knowledge of the Haskell ecosystem (but not much). Knowing 
information from project0 and project1 of https://github.com/Gabriel439/haskell-nix 
is also probably useful. I will try to provide full in-line examples wherever possible 
so you can follow along; if you get stuck, the full end code can be found [here](https://github.com/cah6/haskell-nix-skeleton-1). Note that any time
I change the `release.nix` derivation, I'll make a new file. 

## Setup
We're going to use a cabal file that's pretty basic but has a few extra dependencies
so we can see when things are getting downloaded / built from source:
```
name:                nix-test
version:             0.1.0.0
license:             BSD3
license-file:        LICENSE
build-type:          Simple
extra-source-files:  ChangeLog.md
cabal-version:       >=1.10

executable nix-test
  main-is:             Main.hs
  build-depends:      base >=4.9
                    , containers
                    , lens
                    , text
  hs-source-dirs:      src
  default-language:    Haskell2010
```
Then you'll want to do a `cabal2nix . > default.nix` to generate a derivation from that. 
Remember to run cabal2nix any time you change your cabal file!

## Pinning your haskell packages
The absolute minimum derivation, which we'll be building on (call it `release.nix`), is as follows:
```nix
let
  pkgs = import <nixpkgs> { };
in
  { project1 = pkgs.haskellPackages.callPackage ./default.nix { };
  }
```

The main thing I don't like about this is that it uses whatever "nixpkgs" is floating around
your system. Instead, let's pin to a specific commit of a stable channel using
nix-prefetch-git:
```bash
$ nix-prefetch-git https://github.com/nixos/nixpkgs-channels.git refs/heads/nixos-18.03 > nixos-18-03.json
```
which generates:
```json
{
  "url": "https://github.com/nixos/nixpkgs-channels.git",
  "rev": "8b92a4e600458c01ab0a72f2492eb7120e18f9bc",
  "date": "2018-09-02T16:06:16+02:00",
  "sha256": "1s28c24wydfiqkbz9x7bwhjnh2x4qr010p18si7xdnfdwrxn5mh1",
  "fetchSubmodules": true
}
```

Then to use it in your nix file, you'd use a combination of readFile, fromJSON, 
and fetchFromGitHub. In it's own file (call it `pkgs-from-json.nix`), this looks like:
```nix
{
  bootstrap ? import <nixpkgs> {}
, json
}:
let 
  nixpkgs = builtins.fromJSON (builtins.readFile json);
  src = bootstrap.fetchFromGitHub {
    owner = "NixOS";
    repo  = "nixpkgs-channels";
    inherit (nixpkgs) rev sha256;
  };
in 
  import src {} 
```
and is used in `release.nix` by changing that file to:
```nix
let
  pinnedPkgs = import ./pkgs-from-json.nix { json = ./nixos-18-03.json; };
in
  { project1 = pinnedPkgs.haskellPackages.callPackage ./default.nix { };
  }
```

If you want to see which haskell packages are pinned for that commit, you can
go to `hackage-packages.nix` for that "rev". So for the JSON above, that file would be [this link](https://raw.githubusercontent.com/NixOS/nixpkgs/8b92a4e600458c01ab0a72f2492eb7120e18f9bc/pkgs/development/haskell-modules/hackage-packages.nix).
Then you can either CTRL+F there or download that file and search locally to see
which version of a given dependency is pinned. 

## Setting GHC version

But which version of GHC are we using? One way would be to search for "base" and 
backtrack to the version of GHC from that.
My favorite way is via `nix repl`, made easier by the fact that we extracted our pinning function
to its own file. Since "haskellPackages" is an alias for "haskell.packages.<default-ghc-version>",
we can do the following:

```bash
$ nix repl
nix-repl> pkgs = import ./pkgs-from-json.nix { json = ./nixos-18-03.json; }
nix-repl> pkgs.haskellPackages.ghc
«derivation /nix/store/djy5y2x23cpzksxpwc1zb3df9kq4y3lw-ghc-8.2.2.drv»
```

So, this nixpkgs is defaulting to ghc-8.2.2. What if we want to use a different GHC version?
First let's see which versions we have available, this time using the auto-complete of 
`nix repl`:
```bash
nix-repl> pkgs.haskell.packages.ghc<type tab tab>
pkgs.haskell.packages.ghc7103        pkgs.haskell.packages.ghc822         pkgs.haskell.packages.ghcjs
pkgs.haskell.packages.ghc7103Binary  pkgs.haskell.packages.ghc841         pkgs.haskell.packages.ghcjsHEAD
pkgs.haskell.packages.ghc802         pkgs.haskell.packages.ghc843
pkgs.haskell.packages.ghc821Binary   pkgs.haskell.packages.ghcHEAD
```

If we want to use ghc843 instead, we would be tempted to replace `haskellPackages`
with `haskell.packages.ghc843`. Let's try that just to see what happens:
```bash
$ nix-build release3.nix --dry-run
these derivations will be built:
  /nix/store/l1yyi28qs37p64vr7s36rdz4fnv1gvcp-hscolour-1.24.2.drv
  /nix/store/5r1w8v0d8cfbm5cpp5z5z2k1n1hjzj5z-semigroups-0.18.4.drv
  /nix/store/6dpl86pga098dn0y5a7s74vl6wkzmq1p-transformers-compat-0.5.1.4.drv
  /nix/store/2ag3fwj1r6x9sg8h8raiadyzkvkmnn8d-primitive-0.6.3.0.drv
  /nix/store/qh092sigw95nazbaqna1g3flq4ll1r3p-random-1.1.drv
  /nix/store/jc2kzlndcx4xc5y6gmjjqgmzglaqw1qa-tf-random-0.5.drv
  /nix/store/d5a1b99r402jkmcq0cvqc1ghp23jps65-QuickCheck-2.10.1.drv
  /nix/store/7pxj00jqs1qaw2vdgbl5sz8i5hh3cldp-setenv-0.1.1.3.drv
  /nix/store/8i637r8ip2cc0f1sssgg01s1p86dvhy1-nanospec-0.2.2.drv
  ...
  many many more
  ...
  these paths will be fetched (115.61 MiB download, 1289.39 MiB unpacked):
  /nix/store/4wvwj5rqkj3kxwmbl10p3ridarfp1djl-ghc-8.4.3
  /nix/store/vmrgjlzdvrk2aii6m0fn3rwajckrpwpx-ghc-8.4.3-doc
```
Whoa whoa whoa, we don't want to BUILD all that. I'm not sure exactly why this happens, but 
my guess is that the only haskell packages that are cached for this commit are
the ones listed in the raw.githubusercontent.com page I linked above. Since a new
GHC version is using a new base and core packages, chances are that package versions
are going to be resolved slightly differently, and you'll need to build any that are
different. 

In my opinion, a better way is to pick a Nix channel that has whatever GHC you
want in the default `haskellPackages`, since it will come with a set of packages that will work
well together and not require that you build so much yourself. In this case, nixos-18.09
has GHC 8.4.3 as the default, so we could use it instead by running:
```bash
$ nix-prefetch-git https://github.com/nixos/nixpkgs-channels.git refs/heads/nixos-18.09 > nixos-18-09.json
```
and pointing to this new `nixos-18-09.json` in your release file (`release4.nix`).

## Overriding packages

There are a few ways to override haskell packages but most are, surprisingly,
not very composable. For instance, if you were to try:
```nix
let
  pkgs = import <nixpkgs> { };
  overriddenPackages1 = pkgs.haskellPackages.override {
    overrides = self: super: {
      project1 = self.callPackage ./default.nix { };
    };
  };
  
  overriddenPackages2 = overriddenPackages1.override {
    overrides = self: super: {
      # some more overrides
    };
  };

in
  { project1 = overriddenPackages2.project1;
  }
```
the execution would fail with `error: attribute 'project1' missing` at the final
line, since the second override cleared out what the first one set. 

The best override methodology I've found is with the (slightly arcane) syntax described in  
[this ticket](https://github.com/NixOS/nixpkgs/issues/26561). Applying this, your
release file becomes `release5.nix`:
```nix
let
  pinnedPkgs = import ./pkgs-from-json.nix { json = ./nixos-18-09.json; };

  customHaskellPackages = pinnedPkgs.haskellPackages.override (old: {
    overrides = pinnedPkgs.lib.composeExtensions (old.overrides or (_: _: {})) (self: super: {
      project1 = self.callPackage ./default.nix { };
      # additional overrides go here
    });
  });

in
  { project1 = customHaskellPackages.project1;
  }
```
This is using the built-in `composeExtensions` function to take the previous
overrides (or the empty set, {}) and merge it with whatever is inside the `self: super:` overlay.
In this change I also switched around where we define `project1`: now it is defined in our
haskellPackages with everything else and just returned at the end. I prefer
this approach since, from now on, our `project1` will be treated like any other haskell dependency.

## Enabling Hoogle

If you're working on Haskell locally, you really should be using a local Hoogle
instance so that the package's version you're viewing for documentation matches 
the version you're using in your code. Fortunately Nix makes this quite easy, 
especially once you set up your overrides as above. A standalone expression for 
this looks like the following, call it `toggle-hoogle.nix`:
```nix
{
  # Library of functions to use, for composeExtensions. 
  lib ? (import <nixpkgs> {}).pkgs.lib
  # Whether or not hoogle should be enabled.
, withHoogle
  # Input set of all haskell packages. A valid input would be:
  # (import <nixpkgs> {}).pkgs.haskellPackages
, input
}:
if withHoogle
  then  input.override (old: {
          overrides = lib.composeExtensions (old.overrides or (_: _: {})) (self: super: {
            ghc = super.ghc // { withPackages = super.ghc.withHoogle; };
            ghcWithPackages = self.ghc.withPackages;
          });
        })
  else  input
```
Then we can change our release file to call it with what we have so far, taking
an input to toggle this functionality. `release6.nix`:
```nix
{ withHoogle ? false
}:
let
  pinnedPkgs = import ./pinnedPkgs.nix { pinnedJsonFile = ./nixos-18-09.json; };

  customHaskellPackages = pinnedPkgs.haskellPackages.override (old: {
    overrides = pinnedPkgs.lib.composeExtensions (old.overrides or (_: _: {})) (self: super: {
      project1 = self.callPackage ./default.nix { };
      # addditional overrides go here
    });
  });

  hoogleAugmentedPackages = import ./toggle-hoogle.nix { withHoogle = withHoogle; input = customHaskellPackages; };

in
  { project1 = hoogleAugmentedPackages.project1;
  }
```
Then, if we wanted to enter a shell where we had GHC packages built with documentation, 
we could write a `shell.nix` with:
```nix
let 
  projectDrv = (import ./release6.nix { withHoogle = true; } ).project1;
in 
  projectDrv.env
```
And if we wanted to run Hoogle with these packages, it would just be a matter of
running the start command in this shell:
```bash
$ nix-shell --run 'hoogle server --port=8080 --local --haskell'
```

## Ending Notes

That's all for this post, thanks for reading along this far! Again, if you want to see the final setup as this post left
off, check out [this repository](https://github.com/cah6/haskell-nix-skeleton-1). Chances are I will expand on this setup in future
posts, with the end goal being a completely deterministic development environment 
with a great IDE experience and any tooling you would need. 

If you'd like to comment on any part of this post, please do so in the associated 
reddit post [here]()! 

---
layout: post
title: "Exploring Nix & Haskell Part 3: A Frontend Skeleton"
modified:
categories: technology
excerpt:
tags: []
image:
  feature: nix-haskell-banner.png
date: 2019-04-20T00:00:00
modified: 2019-04-20T00:00:00
published: true
---

In the last post, we ended off with a Nix-based environment with some basic workflow commands to build and test your Haskell code. I mentioned that my reason for investigating Nix environments in the first place was because it's the most supported way to build a frontend Haskell application with [Reflex](https://reflex-frp.org/) / [GHCJS](https://github.com/ghcjs/ghcjs).

With that being said, with this post we'll start with a project based off [reflex-project-skeleton](https://github.com/ElvishJerricco/reflex-project-skeleton) and remove/add bits and pieces to it until we have a deployable (frontend only, for now) application with a prescribed IDE experience and workflow.

##  A quick disclaimer on alternatives...

A lot of the information in this post will parallel what [Obelisk](https://github.com/obsidiansystems/obelisk) gives you. 
For my own project, I ended up using the setup described here rather than Obelisk for the following reasons:
- Most of Obelisk's commands are pass-throughs to nix-shell and nix-build operations. While it is a fairly thin wrapper on top of these commands, as a Nix noob it can be another layer of indirection that just adds complexity, especially when things go wrong. In its current state, you sometimes have to end up knowing what Obelisk is doing *as well as* the underlying Nix commands (which involves digging through a lot of source code). For this reason, I think reading through this post could help you even if you do decide that going through Obelisk is better for you.
- I mostly develop on Mac and lots of the commands would not work for a variety of reasons. Usually this was because the underlying Nix operation wasn't working, but for the above reason, this was slightly obscured from me. 
- Android and iOS builds would be theoretically nice but I didn't care about it that much. I'd rather just get some website/web-app up, and just make it so that it has a nice mobile format.
- I didn't care about server-side rendering. I wanted my backend to be a fairly simple Haskell CRUD server and able to be on a more modern version of GHC if needed.
- I didn't want to pay for the frontend aspect of my project. My solution to this (which I'll include in this post) is to use GitHub Pages, which makes makes some parts of Obelisk unnecessary. 
- Probably related to my lack of server-side rendering, the routing update broke my project. Since my backend was an independent REST service, I didn't gain any benefit from whatever routing things it introduced, and there wasn't much documentation on it so I didn't really understand it either.

That being said, a lot of these things have been fixed over the last few months, and you may want different things than me. The Obelisk folks have been very responsive to my questions, and I think the project has a good goal. Make sure to check it out to see if it'll work for you; if it does, hopefully the information in this post will still help you there too!

## Setting up the basic skeleton

The goal for this section is to get the skeleton to a spot where we can comfortably learn the workflow commands and make future changes. We'll start with the [reflex-project-skeleton](https://github.com/ElvishJerricco/reflex-project-skeleton), wiping history and re-committing:
```bash
git clone --recurse-submodules https://github.com/ElvishJerricco/reflex-project-skeleton my-haskell-reflex-project
cd my-haskell-reflex-project
rm -rf .git
git init
git add .
git commit -m "Initial commit after cloning skeleton"
```

You probably want to run `./reflex-platform/try-reflex` to set up and populate your caches. Some parts of this can take a while; if you've never tried reflex before, the `try-reflex` script will:
1. Download / install Nix (fairly quick, it's around 22MB)
2. Ask you to manually input whether to configure binary caches. Definitely say yes.
3. Download / build whatever is needed for Reflex / GHCJS environment. *This* part will take a while. To give you a general idea, I tried this on the default "hashicorp/precise64" Vagrant box with an internet connection of ~100 Mbit/s (fixed to match my usual speed with [this](https://alexbilbie.com/2014/11/vagrant-dns/)) running on my 2016 Mac and it took about 30 minutes. I'd recommend letting this run in the background while you follow along with the rest of the setup. Keep in mind that this is roughly the lead time to expect if you ever update `reflex-platform` in the future. 

One thing you should notice is that the `default.nix` file is re-using the Nix expression from the reflex-platform submodule. We're going to keep this, but know that this means that our entry point will be fundamentally different from the previous posts, where we built it up from scratch. Some settings are pre-filled in this `default.nix`; to see the full list (it's not too long) check the reflex-platform/project/default.nix [here](https://github.com/reflex-frp/reflex-platform/blob/7e002c573a3d7d3224eb2154ae55fc898e67d211/project/default.nix).

### Constraining to a frontend web-app 

First, I want to simplify the scope here and not consider a backend and android/ios builds. Those can be a topic for another time. We'll still keep the shells section though, so that this can be added in sometime later. So:
```bash
rm -rf backend
# edit default.nix to remove backend from packages and shells
# edit default.nix to remove android/ios sections
```
and you should end up with 
```nix
{ reflex-platform ? import ./reflex-platform {} }:
reflex-platform.project ({ pkgs, ... }: {
  packages = {
    common = ./common;
    frontend = ./frontend;
  };

  shells = {
    ghc = ["common" "frontend"];
    ghcjs = ["common" "frontend"];
  };
})
```

### Using Nix instead of git submodules

The source skeleton uses a git submodule to "import" the reflex-platform nix file. I had some issues getting this to work in my cloned repo (probably because I wiped the history), but that's fine -- by now we should be pros at importing Nix things! We know (from part 1 of this series) that we can use `fetchFromGitHub` to get a repository, and `nix-prefetch-git` to find the correct sha256 value. Looking at the [skeleton](https://github.com/ElvishJerricco/reflex-project-skeleton) again, we see that it's pinning a `reflex-platform` commin that starts with `7e002c5`, so we can run:
```bash
nix-prefetch-git https://github.com/reflex-frp/reflex-platform.git 7e002c5
```
to get the full `rev` and `sha256` fields, then put it into a `nix/reflex-platform.nix` file that binds all this together:
```nix
{ bootstrap ? import <nixpkgs> {} }:
let
  reflex-platform = bootstrap.fetchFromGitHub {
    owner = "reflex-frp";
    repo  = "reflex-platform";
    rev = "7e002c573a3d7d3224eb2154ae55fc898e67d211";
    sha256 = "1adhzvw32zahybwd6hn1fmqm0ky2x252mshscgq2g1qlks915436";
  };
in 
  import reflex-platform {}
```
Use this in our `default.nix` by changing line 1 to import this file instead of looking at a sub-folder:
```nix
{ reflex-platform ? import ./reflex-platform.nix {} }:
...
```
and remove the now unnecessary submodule pieces:
```bash
rm -rf reflex-platform
rm .gitmodules
```

### Adding Hoogle toggle

Let's also take as input whether hoogle is enabled, so that we don't have to wait for Hoogle to build when we don't need it: 
```nix
{ reflex-platform ? import ./reflex-platform.nix {}
, withHoogle ? false
}:
reflex-platform.project ({ pkgs, ... }: {
  
  inherit withHoogle;

  ...
})
```

### Wiring in Warp

Warp is what will provide us with a hot-reloading browser window *running GHC* when we save the source code. This will be very useful in our workflow, so let's enable that by default. To do that:
- Add jsaddle-warp as a dependency of `frontend.cabal`:

```yaml
...
executable frontend
  ...
  build-depends:       base >=4.9 && <4.12
                     , common
                     , jsaddle-warp
                     , reflex-dom
  ...
...
```
- Change our `Main.hs` to be started with Jsaddle.Warp instead:

```haskell
{-# LANGUAGE OverloadedStrings #-}

module Main where

import Reflex.Dom.Core
import Language.Javascript.JSaddle.Warp

main :: IO ()
main = run 3003 $ mainWidget $ text "Hello!"
```
- Set `useWarp = true;` in `default.nix`:

```nix
...
reflex-platform.project ({ pkgs, ... }: {
  withHoogle = false;
  useWarp = true;
...
```

### Miscellaneous cleanup and additions

Reflex-platform will give you cabal and other dev tools by default, so we don't need to touch `shellToolOverrides` in `default.nix`, which pretty much does what the `buildInputs` param did in the previous post. However, let's just add in empty sections for that and `overrides` in case we do need it sometime:
```nix
...
  shellToolOverrides = self: super: {
  };

  overrides = self: super: {
  };
...
```

As you'll see in the next section, I don't really use the `cabal` helper scripts, so I'm going to go ahead and remove them (if you find them helpful, feel free to keep them):
```bash
rm cabal
rm cabal-ghcjs
rm cabal-ghcjs.project # without a backend, we only have one "project"
```
and ignore intermediate things in the `.gitignore`:
```
dist*
frontend-result
**/*.hi
**/*.o
.ghc*
```

### Setting up Visual Studio Code

At this point, this guide is pretty committed to using Visual Studio Code so we'll make a settings override that will work with our project structure. The default settings set you up pretty well, but we'll also use `:set -i` (a ghci command) to include the `frontend/src` and `common/src` folders. The plugin is usually pretty good about figuring out your project type but I have had it incorrectly default to Stack, so we'll specify that directly. Note that, for now, this is non-new-style cabal. All together your `.vscode/settings.json` file should look like:
```json
{
  "ghcSimple.startupCommands.custom": [
    ":set -fno-diagnostics-show-caret -fdiagnostics-color=never -ferror-spans",
    ":set -fdefer-type-errors -fdefer-typed-holes -fdefer-out-of-scope-variables",
    ":set -ifrontend/src:common/src"
  ],
  "ghcSimple.workspaceType": "cabal"
}
```

*Now* we have a skeleton that will work with everything in the next section. To see the code tagged now, see [here](https://github.com/cah6/haskell-nix-skeleton-2/tree/barebones_webapp). 

## Workflow commands

I mentioned some of these in the past, but let's get a complete list of commands you need to know while you're working in this environment. Most of these need to be ran in the GHC nix-shell, which you launch with:
```bash
nix-shell -A shells.ghc
```
and then run the command in it. If you want to run it in one single-fire command, enter it in the form:
```bash
nix-shell -A shells.ghc --run "<command>"
```

### Start up IDE

As an example of above: since we want to primarily develop on GHC (it's faster), we start up the GHC shell:
```bash
nix-shell -A shells.ghc
```
and launch Visual Studio Code from there:
```bash
code .
```
Or as one command:
```bash
nix-shell -A shells.ghc --run "code ."
```

### Fire up ghcid

Note: this is basically what Obelisk's `ob run` does.

The main benefit of ghcid here is that we can hot-reload our app a few seconds after saving our source file. Another benefit is that you can see the full compile error message (sometimes it gets formatted weird on the IDE), and you can have a "sanity test" for your IDE if it freezes or something. We start ghcid like this by running (in the GHC shell):
```bash
ghcid -W -c "cabal new-repl frontend" -T Main.main
```
To break that down:
- -W lets you run tests even if there's warnings
- -T runs that method whenever a re-compile is done
- -c specifies the shell/repl to run that command in. We want new-repl since we have a `cabal.project` file in our repo.

Since this is something you want to fire once and keep running, you probably want the single command to copy-paste:
```bash
nix-shell -A shells.ghc --run 'ghcid -W -c "cabal new-repl frontend" -T Main.main'
```
And then you can go to `localhost:3003` (or whatever port you have set in your entry point) to see your application.


### Fire up a repl

Note: this is basically what Obelisk's `ob repl` does. 

You shouldn't be using this for compile errors, but sometimes you want a repl to test out small sections of code. Do this by firing up the same repl ghcid does, i.e:
```bash
nix-shell -A shells.ghc --run "cabal new-repl frontend"
```

### Start Hoogle

```bash
nix-shell -A shells.ghc --run --arg withHoogle true "hoogle server --local --port=8080"
```
Note that we could have also done what we did in previous posts where we define a nix-shell in a new file that passes in `withHoogle = true` and run the hoogle command in there. Either works.

### Build frontend application

You probably don't want to do this very often, since ghcjs will compile much slower than ghc. However, I'd recommend doing this whenever you edit your cabal files, since your ghcjs dependencies could require some updating too, and it can be frustrating to do this in one push when you want to deploy. So to build the final thing, we'll use nix-build with:
```bash
nix-build -o frontend-result -A ghcjs.frontend
```

### "Deploy" your application

In your github project settings, make sure you have GitHub Pages enabled with the option to build from the `/docs` folder. With this, all you need to do is copy the stuff in `frontend-result` to docs:
```bash
rm -rf docs; cp -r frontend-result/bin/frontend.jsexe docs
```
and push the updated docs folder to github.

Before pushing, you may want to test out the final ghcjs app by opening it in your browser:
```bash
open docs/index.html
```

I know this method isn't for everybody, but if you have strong counter-opinions to doing this you probably have enough knowledge or desire to host the docs folder elsewhere.

### All together now

All this means that I usually have a number of terminals (in my case iTerm split windows) open:
1. One terminal for a consistent Hoogle db running in foreground
2. One terminal for a consistent ghcid running in foreground
3. One terminal sitting in `nix-shell -A shells.ghc` to re-launch the IDE or pop into a repl.
4. One terminal sitting in `nix-shell -A shells.ghcjs` to rebuild the final frontend.

Organizing them in the foreground like this means that it's easier to restart all 4 when you change a package in your cabal file / Nix expression.

## Making some changes

Now that we have these core commands out of the way, let's make some changes to the project to illustrate common types of environment changes you'll need to make. This is pretty similar to the info in [https://github.com/Gabriel439/haskell-nix](https://github.com/Gabriel439/haskell-nix), but this can be really confusing for newcomers, so I think it's worth re-showing.

### dontCheck to skip tests

Let's say our frontend needed the package [http-media](http://hackage.haskell.org/package/http-media). First, add that package to the `frontend.cabal` file. Re-entering our `nix-shell -A shells.ghc` works quickly but re-entering `nix-shell -A shells.ghcjs` makes it build and test the package from scratch, since it's not in the cached set of packages for reflex-platform.

If we don't care about running the tests for this package, whether because it takes too long, can't compile, or can't pass with the package combination we have, we can override it with the `dontCheck` function in our `default.nix`:
```nix
...
  overrides = self: super: {
    http-media = pkgs.haskell.lib.dontCheck super.http-media;
  };
...
```

Another reason to use `dontCheck` is if the test suites depend on incompatible packages. The biggest culprit here is the `doctest` package, which [can't build on GHCJS](https://github.com/NixOS/nixpkgs/issues/47437#issuecomment-429657590). For example, the [lens-aeson](https://github.com/lens/lens-aeson/blob/master/lens-aeson.cabal) library has a doctests test-suite, so you'll need to run it through `dontCheck` in order to use it.

### Pinning a library that has a Nix expression

Let's say we need the `servant-reflex` package now. Adding this to our frontend cabal file and re-entering the GHC shell will fail with:
```
Setup: Encountered missing dependencies:
base >=4.8 && <4.10,
exceptions ==0.8.*,
http-media ==0.6.*,
servant >=0.8 && <0.13
```
because this package uses strict bounds on dependencies and the version it decided to use (v0.3.3) doesn't mesh with what we have. The latest commit (v0.3.4) does though, and this library has a Nix derivation in the repository, so we can grab the full info with:
```bash
nix-prefetch-git https://github.com/imalsogreg/servant-reflex 4459563
```
and put the rev/sha256 into a `nix/servant-reflex.nix` file. Note that `callPackage` expects the `fetchFromGitHub` return value, without the `import` like before:
```nix
{ bootstrap ? import <nixpkgs> {} }:
bootstrap.fetchFromGitHub {
  owner = "imalsogreg";
  repo  = "servant-reflex";
  rev = "44595630e2d1597911ecb204e792d17db7e4a4ee";
  sha256 = "009d8vr6mxfm9czywhb8haq8pwvnl9ha2cdmaagk1hp6q4yhfq1n";
}
```
Then put it in your `default.nix` overrides:
```nix
...
  overrides = self: super: {
    servant-reflex = pkgs.haskellPackages.callPackage ./nix/servant-reflex.nix { };
    ...
  };
...
```

### Pinning through cabal

If you need library xyz (you put it in your cabal file and entering the shell fails) and it doesn't have a Nix file, you can also pin it by running `cabal2nix` directly on the GitHub repository. 
For example, to pin the lastest commit:
```
cabal2nix https://github.com/owner/xyz > nix/xyz.nix
```
or for a specific commit:
```
cabal2nix https://github.com/owner/xyz --revision abcd1234 > nix/xyz.nix
```

Then edit your overrides:
```
...
  overrides = self: super: {
    xyz = pkgs.haskellPackages.callPackage ./nix/xyz.nix { };
    ...
  };
...
```

## Ending Notes

If you want to play around with the barebones skeleton in your own repository, just do something similar to the initial skeleton clone:
```bash
git clone https://github.com/cah6/haskell-nix-skeleton-2/tree/barebones_webapp my-haskell-widgets
cd my-haskell-reflex-project
rm -rf .git
git init
git add .
git commit -m "Initial commit after cloning skeleton"
```

Hopefully this post will help you get started in making widgets and Haskell and providing an easy way to show them off to your friends. Friend send friends widgets made in Haskell, right?


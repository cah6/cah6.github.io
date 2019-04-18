---
layout: post
title: "Exploring Nix & Haskell Part 2: Dev Tools & IDE Integration"
modified:
categories: technology
excerpt:
tags: []
image:
  feature: nix-haskell-banner.png
date: 2019-04-18T00:00:00
modified: 2019-04-18T00:00:00
published: true
---

In the [previous post from a loong time ago](https://cah6.github.io/technology/nix-haskell-1/), 
I left off with a simple `shell.nix` file that you could use to run a local hoogle server. 

In this post, I'll build on that to show how you can add dependencies to your 
dev environment, then use that to bring up a no-fuss Haskell environment with a solid IDE
experience. Be warned that the IDE section is fairly opinionated -- if you have one
that works for you, great! However, I think a lot of people don't know where to start,
and hopefully this will help in establishing a starting point.

## Adding Dev Tools

Last time, we entered a nix shell with the env provided by our `releaseX.nix` by default. 
This contained ghc/ghci, but not much else. To add more, we'll need to "override 
attributes" in this env. Notably, you'll probably want to edit `buildInputs`
(which will load programs into our `nix-shell`) and `shellHook` (which will run
whatever code block we provide when the shell is entered). You should *definitely*
install cabal this way (even if you have a globally installed cabal), so that your GHC and
cabal versions play nicely together. Note that your `releaseX.nix` file only exposes
your project, so you'll need to re-import your pinned nixpkgs in this shell file to get the other
haskell packages you want. In practice, putting this all together with a few more
dev tools looks `shell2.nix`:
```
let 
  pinnedPkgs = import ./pkgs-from-json.nix { json = ./nixos-18-09.json; };
  myPackages = (import ./release6.nix { withHoogle = true; } );

  projectDrvEnv = myPackages.project1.env.overrideAttrs (oldAttrs: rec {
    buildInputs = oldAttrs.buildInputs ++ [ 
      # required!
      pinnedPkgs.haskellPackages.cabal-install
      # optional
      pinnedPkgs.haskellPackages.hlint
      pinnedPkgs.haskellPackages.hsimport
      ];
    shellHook = ''
      export USERNAME="christian.henry"
    '';
  });
in 
  projectDrvEnv
```

The best thing about this is that your shell inputs don't have to be haskell packages!
If you have a project that needs elasticsearch, you could add `pinnedPkgs.elasticsearch5`
to the `buildInputs`, add startup commands as needed, and fire it up! The nice
thing about doing tools this way is:
1. It reduces the number of manual steps for anybody that wants to develop on your
  project.
2. You ensure everybody is using the same version of these dependencies.
3. There's no conflict between other projects with similar dependencies, but needing
  different versions -- each project has its own sandbox.

As another example: I recently wanted to try out elm as a frontend for a haskell backend. I could
have installed it globally, but I needed elm 0.18 which is not very easy to find anymore.
I did, however, find that elm 0.18 is the default version on the Nix 18.03 channel, so all I 
had to do to get the correct version in my shell was import that channel and
load up the right package:
```
let 
  elmPkgs = import ./pkgs-from-json.nix { json = ./nixos-18-03.json; };
  pinnedPkgs = import ./pkgs-from-json.nix { json = ./nixos-18-09.json; };
  myPackages = (import ./release6.nix { withHoogle = true; } );

  projectDrvEnv = myPackages.project1.env.overrideAttrs (oldAttrs: rec {
    buildInputs = oldAttrs.buildInputs ++ [ 
      ...
      elmPkgs.elmPackages.elm
      ...
    shellHook = ''
      ...
    '';
  });
in 
  projectDrvEnv
```
Note that in this case it's fine to mix and match Nix channels because the packages
don't rely on each other at all.

## Quick IDE Survey

Now that we have whatever dev tools we need, let's focus on setting up the actual
IDE experience. If you want to have a Haskell IDE with a Nix project, there are a couple options.
I've tried:
- Dante for emacs
- Haskell IDE Engine (HIE) with [Atom, Sublime, Visual Studio Code, Emacs, etc]
- ghc-simple for Visual Studio Code

I also want to quickly mention intellij-haskell. Coming from Java, I'm used to
intellij and find that this plugin provides an excellent overall dev experience. However,
it's tied to Stack projects and isn't suitable, as far as I can tell, for projects built with Nix. If your
project is using Stack, check it out too! For what it's worth, my main reason
for using Nix over Stack was to be able to work on projects using [Reflex](https://reflex-frp.org/),
which is more supported through the Nix ecosystem. 

Here's my experience with each:

Dante is pretty good, but ultimately I couldn't be bothered to learn emacs along
with everything else. If you're already comfortable there, try it out!

HIE consistently looks promising but can be a frustrating end user experience. While it has wide adoption
and looks to be the converging point for a lot of other tools, I've had lots of issues getting
it fully working. I would get through cache issues, installing the version
corresponding to my GHC version, linking that to my project, whatever other errors
sprang up, and it might work in some way for some time but would eventually hit something else. Note that this evaluation was around 6 months ago; it could be better now but it doesn't really matter to me because...

I decided to try out [vscode-ghc-simple](https://github.com/dramforever/vscode-ghc-simple)
and love it. It "just works" in a way that lets you focus on actually writing
haskell code, and works with any project that has GHC 8.0+. There is almost zero
project setup and it works with most environment setups. 

## Setting up ghc-simple

To set this up, all you need to do is:
1. [Install vscode](https://code.visualstudio.com/). 
2. Install the `code` shell command by opening the command palette (cmd+shift+p), type
  "shell command", select. 
3. Install the "Simple GHC (Haskell) Integration" extension. It should automatically
  install "Haskell Syntax Highlighting" which detects Haskell files and styles them.

Then from your project root directory, run
```
$ nix-shell shell.nix
$ code .
```
to launch vscode with your project's GHC loaded up. Go to some Haskell files, edit
them, you should see compile errors. That's about it!

If for some reason you don't see compile errors or the plugin doesn't seem to be
working, it can be helpful to check the plugin logs. The default location for these
are:
* NixOS: ~/.config/Code/logs
* Mac: ~/Library/Application\ Support/Code/

and you can find the most recent logs by digging into this general area:
```
logs/{last folder (ordered by date)}/extHost#/output_logging_XXX/#-GHC.log
```

## My workflow

That being said, since your packages are tied to your shell, you need to do a little
reloading if you add a haskell dependency to your project. 

But first, let's make a little quality of life change to our Nix release file. I
routinely forget to run "cabal2nix" when updating my cabal file, so let's eliminate
that manual step by using `callCabal2nix` instead of `callPackage`. The only
change necessary for this is to replace
```
project1 = self.callPackage ./default.nix { };
```
with
```
project1 = self.callCabal2nix "project1" ./project1.cabal { };
```

With this change, my process for adding a new dependency is:
1. List it in your cabal file.
2. `exit` your `nix-shell`. 
3. Reload it with `nix-shell shell.nix`. This will need to re-build your Hoogle database, 
  which takes around 2-3 minutes. Note that [lorri](https://github.com/target/lorri) looks to be a good tool for automatically doing this in the background for you, though I haven't tested it out yet.  
4. Restart Hoogle with your new shell.
5. Exit vscode and relaunch with `code .` in your new shell. It should take ~5-10 seconds
  for errors and such to show up in the editor as it connects to ghci. 

At this point, you should have all the tools for a good workflow:
- editor for quick feedback on compile errors
- pinned Hoogle tab for searching through library documentation
- `cabal new-repl` for one-off tests of low-scope functions
- `cabal run` for running the entire program

Some assorted things I've found while using this workflow:
- If you put `default-extensions` in your cabal file instead of at the top of 
  every haskell file (lots easier), you should load your REPL with `cabal new-repl`
  instead of `ghci`, as that will go through your cabal file.
- Sometimes entering nix shell will fail with something like `package … cannot be satisfied`.
  If that happens, it can usually be fixed by manually deleting your `.ghc.environment*`
  directory that cabal new-style commands produce.

## Ending Notes

That's it for this post -- my next one will show how to set up a
Reflex project using Nix. That will hopefully come out faster than this one did (sorry)! It hasn't
changed much, but the code associated with this can be found [here](https://github.com/cah6/haskell-nix-skeleton-1).

If you’d like to comment on any part of this post, please do so in the associated 
reddit post [here](https://www.reddit.com/r/haskell/comments/beq792/exploring_nix_haskell_part_2_dev_tools_ide/)! Feel free to also message me directly.
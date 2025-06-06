---
layout: post
title: An early look at Nix Dynamic Derivations
date: 2025-03-10 16:34 -0700
---

I normally like to write about concepts from first principles and wait for much of the dust to have settled on the implementation details, but let me take you on a small tour of an upcoming feature instead.

One of the talks I attended at [PlanetNix2025](https://planetnix.com/) was from _the legendary_ John Ericson ([@ericson2314](https://github.com/ericson2314)), who is a core contributor to [NixOS/nix](https://github.com/NixOS/nix) about **dynamic derivations** [RFC#92](https://github.com/NixOS/rfcs/blob/master/rfcs/0092-plan-dynamism.md).

> The talk was a demo on [sandstone](https://github.com/obsidiansystems/sandstone), which is an example of the benefits of dynamic-derivations for Haskell.

The talk had me so energized and excited that I wanted to peek at the current state of the implementation and see if I could contribute. ⚡

John had left us all with a _call to arms_ to try and adopt dynamic derivations for cases where it made sense. I'm writing this to spread the word and get the community similarly energized.

So...

What are dynamic-derivations? 🫠

_Dynamic derivations_ is the ability to create additional derivations at _build time_ to expand the graph.

At the moment, this is _sort of possible_ in Nix through [import from derivations]({% post_url 2020-10-20-nix-parallelism-import-from-derivation %}) (IFD) but it comes with the downside that this can pause the evaluation phase, which is why it's often banned in codebases such as [nixpkgs](https://github.com/NixOS/nixpkgs).

Let's revisit what the problem with IFD is.

```nix
let
  pkgs =
    import (fetchTarball "https://github.com/NixOS/nixpkgs/archive/5ef6c425980847c78a80d759abc476e941a9bf42.tar.gz") {
    };
  inner = pkgs.runCommand "inner" {} ''
    sleep 10;
    echo "Hello from inner." > $out
  '';
  ifd_inner = builtins.readFile inner;
in
  pkgs.runCommand "outer" {} ''
    echo ${ifd_inner} > $out
    echo "Hello from outer" >> $out
  ''
```

The following derivation, when only evaluated, **still takes 10 seconds**, even though I have not done a build yet.

```console
> time nix-instantiate ifd.nix
building '/nix/store/i4m9gschkcr8g8lzzg8a30dw4gpjv393-inner.drv'...
/nix/store/n67w30lgdjzn12fzqranbr9g1v7149bx-outer.drv

________________________________________________________
Executed in   12.15 secs      fish           external
   usr time  333.79 millis    0.28 millis  333.50 millis
   sys time  226.14 millis    1.89 millis  224.25 millis
```

This is the reason all the `lang2nix` tools exist, as nixpkgs has banned IFD. At the moment, the alternate approach is to have a separate tool create all the Nix derivation files you need in a _preprocessor step_.

How can dynamic-derivations make this better? 🤔

⚠️ The state of dynamic-derivations is changing and is _somewhat brittle_. At the moment, if you want to play with it, it's important you use [nix@d904921](https://github.com/NixOS/nix/commit/d904921eecbc17662fef67e8162bd3c7d1a54ce0). Additionally, you need to enable `experimental-features = ["nix-command" "dynamic-derivations" "ca-derivations" "recursive-nix"]`. Here, there be dragons 🐲.

First, we can now create derivations whose output is a file that ends in **.drv** -- meaning the output of a derivation is a derivation itself!

> 😲 I never bothered to create a derivation whose name ended in _drv_, so I was surprised this even was a restriction previously.

```nix
let
  pkgs =
    import (fetchTarball "https://github.com/NixOS/nixpkgs/archive/5ef6c425980847c78a80d759abc476e941a9bf42.tar.gz") {
    };
in
  pkgs.runCommand "world.drv" {
    outputHashMode = "text";
    outputHashAlgo = "sha256";
  } ''
    cat > $out <<END
    Derive([("out","/nix/store/ixzl30c15sg9q0q35dx8z0wbap59pq2w-world","","")],[],[],"mysystem","mybuilder",[],[("out","/nix/store/ixzl30c15sg9q0q35dx8z0wbap59pq2w-world")])
    END
  ''
```

The `outputHashMode` and `outputHashAlgo` are important, as those are the hashing modes traditionally done for derivation files.

We can now build this file, and it will be the output `$out`.

```console
> nix-instantiate end-drv.nix 
/nix/store/hm1d9ihxsws8pcdlqyn32qkfaxcjmblr-world.drv.drv

> nix build -f end-drv.nix --print-out-paths -L
/nix/store/2r65y379iga77g8z42gfibn0bn0w7kgd-world.drv
```

Second, there is a new `builtin.outputOf` that, as best as I can tell, instructs Nix that there is a chain of derivations to follow.

Let's rework our _slow_ IFD example from before, but now leverage _dynamic-derivations_.

```nix
let
  pkgs =
    import (fetchTarball "https://github.com/NixOS/nixpkgs/archive/5ef6c425980847c78a80d759abc476e941a9bf42.tar.gz") {
    };
  inner =
    pkgs.runCommand "inner" {
    } ''
      sleep 10;
      echo "Hello from inner!" > $out
    '';
  producing =
    pkgs.runCommand "inner.drv" {
      outputHashMode = "text";
    } ''
      # we need the unsafe to break deep dependency source drvs
      cp ${builtins.unsafeDiscardOutputDependency inner.drvPath} $out
    '';
in (builtins.outputOf producing.outPath "out")
```

> I wish I had a _succinct_ explanation for why we need `unsafeDiscardOutputDependency` right now. Without it though, building the derivation will try and build all `nativeBuildInputs` of the derivation which is a large set as it includes `stdenv`.

We can now `eval` this Nix expression (_dynamic-derivations_ needs the new Nix commands and does not work with `nix-instantiate`). The evaluation is near instant.

```console
time > nix eval -f ifd_dyn_drv.nix --store /tmp/dyn-drvs
"/1qzln6f3acpj6y443v3j3hcbb8bp3kh1hbzd8qyjazgv1cmnsii0"

________________________________________________________
Executed in  278.95 millis    fish           external
   usr time  161.36 millis    0.17 millis  161.19 millis
   sys time  115.78 millis    1.05 millis  114.73 millis
```

We can now build this derivation, and what gets built is in fact the **inner** derivation, which of course takes ~10 seconds! 🤯

```nix
> time nix build -f ifd_dyn_drv.nix --store /tmp/dyn-drvs --print-out-paths -L
/nix/store/fii3k1jsv95qhgwi3jvb687lpl4p0856-inner

________________________________________________________
Executed in   11.01 secs      fish           external
   usr time  233.14 millis    1.14 millis  232.00 millis
   sys time  179.96 millis    2.03 millis  177.93 millis
```

Ok, so the "dynamic-derivation" was still a Nix expression in the same file. Big whoop... 🙃 

It doesn't have to be, thanks to _recursive Nix_. 🫨

Let's now do this example _again_, but craft our Nix expression dynamically from within another Nix derivation.

> I am writing this in bash so the quoting is _very ugly_ as it's all in a single file for demonstration purposes. In practice, you would probably do it programmatically with `libstore`, or at least with separate Nix files.

```nix
let
  pkgs =
    import (fetchTarball "https://github.com/NixOS/nixpkgs/archive/5ef6c425980847c78a80d759abc476e941a9bf42.tar.gz") {
    };
  producing =
    pkgs.runCommand "inner.drv" {
      outputHashMode = "text";
      requiredSystemFeatures = ["recursive-nix"];
    } ''
      echo "let pkgs = import \"${pkgs.path}\" {};
            in
            pkgs.runCommand \"inner\" {} '''
              sleep 10;
              echo \"Hello from inner!\" > \$out
            '''
            " > inner.nix
      cp $(${pkgs.nix}/bin/nix-instantiate inner.nix) $out
    '';
in (builtins.outputOf producing.outPath "out")
```

We can now build our derivation, and it will in fact build the `inner.nix` recipe we crafted within it.

```console
> time nix build -f simple-raw.nix --store /tmp/dyn-drvs --print-out-paths -L
/nix/store/fii3k1jsv95qhgwi3jvb687lpl4p0856-inner
________________________________________________________
Executed in    8.54 secs    fish           external
   usr time    1.80 secs    1.93 millis    1.79 secs
   sys time    5.83 secs    0.81 millis    5.83 secs

> cat /tmp/dyn-drvs/nix/store/fii3k1jsv95qhgwi3jvb687lpl4p0856-inner
Hello from inner!
```

Cool! Wait, what was the point of all this again? 🫠

We can now dynamically construct a graph of Nix expressions at build time and link them to a top-level derivation.

Imagine any tool that has knowledge of the code graph such as CMake, Bazel or even `-MD` for `gcc`.

We can leverage these tools at the top-level derivation to construct a series of additional derivations for each "module" -- giving us the hermetic seal of Nix, but all the incremental builds of these language toolchains!

No more `lang2nix`. Derivations can now parse lockfiles and generate derivations for all the packages without incurring the cost of IFD.

The work on _dynamic-derivations_ is still somewhat new, but I agree with [@ericson2314](https://github.com/ericson2314) that this will unlock a whole range of new, simpler UX for Nix users.

What can you come up with? 💪

Many thanks to my good friend Mark Williams ([@markrwilliams](https://github.com/markrwilliams)), who hacked on this stuff late into the night after PlanetNix, and John Ericson ([@ericson2314](https://github.com/ericson2314)), who put up with me asking a ton of questions as I wandered around this new feature. 🙇

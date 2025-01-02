---
title: Flakes
---

Flakes are one of the most complex and confusing concepts in nix, especially for beginners. However, understanding flakes is key for working with nix systems effectively, and is a lot easier than often thought.

## What are flakes?

Flakes are an experimental feature of nix that adds two new important files, the `flake.nix` and `flake.lock`. While flakes are an experimental feature, they are widely used among the nix community. Flakes allow you to extend your nix projects, including development shells, NixOS systems, home-manager installations, and more, with nix code from other projects and repositories.

Flakes are split into two major parts, the inputs and outputs. The inputs include a list of all code included in the project. This list commonly includes nixpkgs and home-manager, but can also include any git repository which has a flake.nix file. The outputs include everything which is created by the nix project. This can include dev shells, applications, entire NixOS installations, and more.

## Why use flakes?

The primary benefit flakes give is making nix truly declarative. Without flakes, there is no way to include specific versions of git repos like nixpkgs in a nix project. This can cause issues when sharing a project with others, as users could be using different versions of nixpkgs, causing bugs and conflicts. Flakes solve this with the flake.lock file. This file is automatically generated and pins all inputs to a specific git commit. This ensures that everyone using the nix project is running the same version of nixpkgs and any other inputs. This setup is commonly found in other programming languages, especially Node.JS, which uses a package.json and package.lock file.

## How can I use flakes?

To begin with using flakes, you need to enable flakes on your system. This can be done by editing `/etc/nix/nix.conf` or your NixOS configuration.

For non-NixOS systems, add the following to `/etc/nix/nix.conf` or `~/.config/nix/nix.conf`:

```
experimental-features = nix-command flakes
```

For NixOS systems, add the following to your NixOS configuration:

```nix
# configuration.nix
nix.settings.experimental-features = [ "nix-command" "flakes" ];
```

Next, initialise a nix flake using the `nix flake init` command. This will create a basic flake.nix and flake.lock, which should look like this:

```nix
# flake.nix
{
  description = "A very basic flake";

  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs?ref=nixos-unstable";
  };

  outputs = { self, nixpkgs }: {

    packages.x86_64-linux.hello = nixpkgs.legacyPackages.x86_64-linux.hello;

    packages.x86_64-linux.default = self.packages.x86_64-linux.hello;

  };
}
```

The default flake template includes a description, a list of inputs including nixpkgs, and a set of packages as outputs. There are many different possible types of outputs, which are all used by different nix commands and projects. In this instance, the `packages.x86_64-linux.default` output is used by the `nix build .` command, which builds the package and adds it to the nix store. The most common output type, and the most important one for this guide, is `nixosConfiguration."<hostname>"`. This outputs a NixOS configuration which is used by `nixos-rebuild switch`. Here is an example of a basic flake for a NixOS configuration:

```nix
# flake.nix
{
  description = "A very basic flake";

  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs?ref=nixos-unstable";
  };

  outputs = { self, nixpkgs, ... } @ inputs: {
    nixosConfiguration.nixos-system = {
      specialArgs = {inherit inputs;};
      modules = [
        ./configuration.nix
      ];
    };
  };
}
```

This file includes a few new parts. First, the `{ self, nixpkgs }` has been changed to `{ self, nixpkgs, ... } @ inputs`. This allows for the flake to recognise any inputs passed in, rather than only being able to use nixpkgs. Next, the packages have been replaced with a `nixosConfiguration.nixos-system`. This exposes a new nixos configuration with the hostname nixos-system to `nixos-rebuild`. When using this in a real system, make sure to change nixos-system to whatever your system hostname is. Thirdly, the `specialArgs = {inherit inputs;};` allows other files to access the flake inputs. Finally, the modules list includes every file imported into the NixOS config. This file can be placed alongside an existing NixOS configuration to add flakes to it. Using `nixos-rebuild switch` will create a flake.lock file and begin using flakes.

## Adding extra inputs to Flakes

One of the key benefits of flakes is the ability to use code from other projects. For example, the Hyprland WM has a flake, allowing you to use the latest builds rather than the build included in nixpkgs. This can be done using the following example:

```nix
# flake.nix
{
  description = "A very basic flake";

  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs?ref=nixos-unstable";
    hyprland.url = "github:hyprwm/Hyprland";
  };

  outputs = { self, nixpkgs, ... } @ inputs: {
    nixosConfiguration.nixos-system = {
      specialArgs = {inherit inputs;};
      modules = [
        ./configuration.nix
      ];
    };
  };
}
```

```nix
# configuration.nix
{inputs, pkgs, ...}: {
  programs.hyprland = {
    enable = true;
    # set the flake package
    package = inputs.hyprland.packages.${pkgs.stdenv.hostPlatform.system}.hyprland;
    portalPackage = inputs.hyprland.packages.${pkgs.stdenv.hostPlatform.system}.xdg-desktop-portal-hyprland;
  };
}
```

The change made to the flake.nix adds a new input, which pulls code from Hyprland's git repo. The configuration.nix can then be updated to use the package pulled from the git repo, instead of using the nixpkgs package.

## Further reading

If flakes still confuse you, [this video by Vimjoyer](https://www.youtube.com/watch?v=JCeYq72Sko0) is a great source which teaches everything you need to know about flakes. All of Vimjoyer's content is very useful and is highly recommended when learning Nix and NixOS.
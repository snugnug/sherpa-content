A nix module is a file that contains an expression with a given structure. A module declares a bunch of options for other modules to define, think of it as an "easier" way to handle configurations in the long run, this is one of the main advantages of the nix ecosystem.

The biggest example of a module is in `etc/nix/configuration/nix`, others can be found (here)[https://github.com/NixOS/nixpkgs/tree/master/nixos/modules/programs].

## Structure

a module should have the following structure:

```{
    imports = [
        # Path from which you import other modules.
        # Helps splitting modules in even smaller ones?
    ];

    options = {
        # where you should define the module options.
        # Declares what options the module has.
        # it usually has an enable option, which defines if the module will be used.
    };

    config = {
        # where the actually magic happens, and the modules options declared before will be used to define the options of the module? .
        # if you import form another module that already has an options set you can just define the options here.
    };
}
```

Modules can also be used as functions accepting an attribute set:
```
{ config, pkgs, ...}:
    let

    in {
        imports = [];
        options = {};
        config = {};
    }
```
the following module has:
    
    `config`
    this means the configuration of the entire system.
    
    `pkgs`
    attribute set extracted from nix packages collection and enhanced with `nixpkgs.config` option.

    `...`
    means that any extra parameter passed shall be either ignored or assigned to a keyword which you can use later on.

    `imports`
    as the name states it imports from other files .
    
    `options`
    options declarations with all the definitions and their references.

    `config`
    options definition for the already declared module previously.



here is an example of nix module:

```
{
  pkgs,
  config,
  lib,
  ...
}: let
  cfg = config.gui.obs;
  inherit (lib) mkEnableOption mkIf;
in {
  options.gui.obs.enable = mkEnableOption "obs";

  config = mkIf cfg.enable {
      programs.obs-studio = {
        enable = true;
        plugins = with pkgs; [
          obs-studio-plugins.wlrobs
          obs-studio-plugins.obs-vkcapture
          obs-studio-plugins.obs-vaapi
        ];
      };
  };
}
```
At first this module might look very hard to understand if you aren't familiar with nix language bt this module is pretty straight forward, lets discuss the main points in it:

first we define the attribute set, in that case we need : `pkgs` , `config` and `lib`:
 we need packages because we will some plugins for the program(OBS)  that is in the nix repository and to access to it we need  `pkgs`,
`config` because as we described earlier it means or entire system configuration and last but not least we have `lib` the latter one is used in basically every module you will see in nix as it is a helper, the lib function has tons of uses but here we are just going to see the modules and options handling functions that helps us define, evaluate and merge modules, we will discuss this deeper later on the guide.

we then have a let-in with `cfg = config.gui.obs;` and `inherit(lib) ...(look up)`, the first one is for good looking code you could write the whole line everywhere when needed but at the end of the day it can get very long so just get it inside of some variable of your preference, cfg from what i seen is the most used one,
now lets discuss about the second line, as you might know the inherit keyword is basically for inheriting as the name says duh, here we are inheriting from `lib` the `mkEnableOption` and `mkIf` to use later on without the need of using `lib.thing`.

know we have `options.gui.obs.enable = mkEnableOption "obs"; ` so with this we are defining an option named gui.obs and and making it an enable option(something that can be on or off) with mkEnableOption and the `"obs"` at the end is an argument that most people i see use for an brief explanation of the use case of the module.

down we have the `config = mkIf cfg.enable`, basically: if the option is enabled the configuration inside it will be enabled/used for the need generation, take into account that to enable it you instead of using `options.the_thing.enable = true;` you need to use what you earlier defined inside the cfg `config.the_thing.enable = true;`

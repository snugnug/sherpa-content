---
title: NVIDIA
---

NVIDIA has come a long way in the past year regarding support on Linux, particularly with Wayland. This page will cover how to setup and configure your drivers for NVIDIA on NixOS, as well as additional information you may wish to know.

## Installing Drivers

There are a couple options NVIDIA users have for drivers:

1. The propriety, official drivers (now with an open source kernel module)
2. The reverse-engineered, open source drivers, Nouveau
    - These drivers struggle with performance issues.

The official drivers are strongly recommended. To enable them, you can set the following options:

```nix
# configuration.nix
nixpkgs.config.allowUnfree = true; # Allows for installation of proprietary packages

hardware.graphics = {
    enable = true;
    enable32Bit  = true; # Primarily for gaming
};

services.xserver.videoDrivers = ["nvidia"]; # Enables the proprietary drivers

hardware.nvidia = {
    open = true; # Enables the use of the open source kernel module
};
```

> **Note:** On [Turing](https://en.wikipedia.org/wiki/Turing_(microarchitecture)) and newer cards, it is recommended by NVIDIA that you use their open source driver on versions 560 and up. See [Suspend Issues](#Suspend) for info on whether or not you should actually listen to them.

If you wish instead to use the Nouveau drivers, only set the options under `hardware.graphics`.

### Driver Version Configuration

On NixOS, it is quite easy to customize your driver version. In your NVIDIA module, you can select various packages:

```nix
# configuration.nix
hardware.nvidia = {
    package = config.boot.kernelPackages.nvidiaPackages.<version>;
};
```

As of writing, the following options correspond to the following versions (bolded is default):

| `<version>` | Driver Version |
| --------- | -------------- |
| **production** | 550 |
| latest | 565 |
| beta | 565 |
| vulkan_beta | 550 |

This list is non-exhaustive. See the full list in the source code in [nixpkgs](https://github.com/NixOS/nixpkgs/blob/nixos-unstable/pkgs/os-specific/linux/nvidia-x11/default.nix) for more information.

If you wish to choose a specific version, note that you can define the driver version without referencing a packaged option. For example, if you wanted to use the 555 driver:

```nix
# configuration.nix
hardware.nvidia = {
    package = config.boot.kernelPackages.nvidiaPackages.mkDriver {
        version = "555.58.02";
        sha256_64bit = "sha256-xctt4TPRlOJ6r5S54h5W6PT6/3Zy2R4ASNFPu8TSHKM=";
        sha256_aarch64 = "sha256-wb20isMrRg8PeQBU96lWJzBMkjfySAUaqt4EgZnhyF8=";
        openSha256 = "sha256-8hyRiGB+m2hL3c9MDA/Pon+Xl6E788MZ50WrrAGUVuY=";
        settingsSha256 = "sha256-ZpuVZybW6CFN/gz9rx+UJvQ715FZnAOYfHn5jt5Z2C8=";
        persistencedSha256 = "sha256-a1D7ZZmcKFWfPjjH1REqPM5j/YLWKnbkP9qfRyIyxAw="; 
    };
};
```

This utilizes the mkDriver function to select a specific version (verified by the hashes) and build it on your system. This may be useful if you are on a stable build but are in dire need of a bugfix or feature from a newer driver (for example, when 555 was released). 

To obtain the hashes, it is recommended to obtain them from the [source code](https://github.com/NixOS/nixpkgs/blob/nixos-unstable/pkgs/os-specific/linux/nvidia-x11/default.nix) if it is available there, or possibly from a PR.

### Legacy Cards

You may be using a card that now only has legacy support from NVIDIA. In those cases, you should use the legacy drivers as your driver package. Please reference [this page](https://www.nvidia.com/en-us/drivers/unix/legacy-gpu/) for the newest driver version your GPU supports, and select the legacy package accordingly:

```nix
# configuration.nix
# Last one supporting Kepler architecture
hardware.nvidia.package = config.boot.kernelPackages.nvidiaPackages.legacy_470;
```

Depending on your card, you may wish to change the last three numbers. For example, if you have a GTX 480, it should look like this:

```nix
# configuration.nix
# Last one supporting x86
hardware.nvidia.package = config.boot.kernelPackages.nvidiaPackages.legacy_390;
```

You may also reference the source code in [nixpkgs](https://github.com/NixOS/nixpkgs/blob/nixos-unstable/pkgs/os-specific/linux/nvidia-x11/default.nix) for NVIDIA driver version options.

## Driver Configuration

The NVIDIA module also comes with some additional configuration and options you should be aware of.

### NVIDIA Prime

For those of you with a laptop, NVIDIA has support for hybrid graphics optimization through NVIDIA Prime. This means that you can tell your system when to use your integrated graphics (on your CPU) or your dedicated graphics (on your graphics card). NVIDIA Prime has two main modes:

1. Sync: Always use the dedicated graphics to optimize performance
2. Offload: Disable the dedicated graphics except when performing a heavy task (e.g. gaming)

To enable "sync" mode:

```nix
# configuration.nix
hardware.nvidia = {
    prime = {
        sync.enable = true;
    
        # Internal graphics from an Intel CPU
        intelBusId = "PCI:0:2:0";

        # Or, if you have an AMD CPU:
        # amdgpuBusId = "PCI:4:0:0";

        # Dedicated Nvidia graphics card
        nvidiaBusId = "PCI:1:0:0";
    };

    # This is set to true by default on driver versions above 535
    modesetting.enable = true; # Prevents tearing with NVIDIA Prime set to sync
};
```

To instead use "offload" mode:

```nix
# configuration.nix
hardware.nvidia = {
    prime = {
        offload.enable = true;

        # See note below
        offload.enableOffloadCmd = true;

        # Internal graphics from an Intel CPU
        intelBusId = "PCI:0:2:0";

        # Or, if you have an AMD CPU:
        # amdgpuBusId = "PCI:4:0:0";

        # Dedicated Nvidia graphics card
        nvidiaBusId = "PCI:1:0:0";
    };
};
```

> **Note:** You can also enable `hardware.nvidia.prime.offload.enableOffloadCmd` to enable an `nvidia-offload` command that automatically offloads a program to the GPU. You can use it by prefixing the running command with `nvidia-offload`, for example with Steam, setting the launch options to `nvidia-offload %command%`.

To find your respective bus IDs, simply run `lspci` from a terminal. On the left, a bus ID such as "01:00.0" will be shown, and to the right, a description of the hardware the ID corresponds to will be shown. You should be able to identify the bus ID for both your CPU and GPU.

In the case of "01:00.0," the bus ID should be written as `PCI:1:0:0`. For "00:02.0," you would use `PCI:0:2:0`.

### nvidia-settings

You can also use NVIDIA's GUI config tool, nvidia-settings. To enable it:

```nix
# configuration.nix
hardware.nvidia = {
   nvidiaSettings = true; 
};
```

This is enabled by default.

## Issues

### Suspend

NVIDIA currently has issues with suspend resuming to a black screen when using the open source kernel module. To fix this, set the following settings:

```nix
# configuration.nix
hardware.nvidia = {
    powerManagement.enable = true;
    open = false; # Be sure this option is not enabled elsewhere
};

boot.kernelParams = ["nvidia.NVreg_PreserveVideoMemoryAllocations=1"];
```

Suspend should work properly now.


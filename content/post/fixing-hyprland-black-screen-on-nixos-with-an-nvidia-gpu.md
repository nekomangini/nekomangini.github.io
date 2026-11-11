+++
date = '2026-11-10T13:27:11+08:00'
draft = false
title = 'Fixing Hyprland Black Screen on NixOS With an NVIDIA GPU'
tags = ["nixos", "hyprland", "nvidia", "wayland", "home-manager", "noctalia"]
description = "How I went from a black screen to a working Hyprland + Noctalia setup: Home Manager symlink conflicts, misleading logs, and a firmware framebuffer that stole the primary GPU slot."
+++

I wanted to move my NixOS desktop to Hyprland with a Lua config and the Noctalia shell. Logging in gave me a black screen and nothing else. This post is the path I took to fix it, including the dead ends, because the dead ends are where most of the learning happened.

**Setup:** NixOS 26.05, Hyprland 0.55.2, Home Manager as a NixOS module, GTX 1050 Ti, three monitors, and a flake-based dotfiles repo.

## Part 1: Getting the config to build

Before the black screen even mattered, my config had a few problems of its own.

### Two ways to "install" Hyprland

NixOS has `programs.hyprland.enable`, and Home Manager has `wayland.windowManager.hyprland`. They do different jobs. The NixOS module installs the package and registers the session. The Home Manager module generates `~/.config/hypr/` from Nix options.

I wanted to write my config in Lua and symlink it from my repo, so I dropped the Home Manager Hyprland module entirely and kept only the NixOS one:

```nix
programs.hyprland = {
  enable = true;
  xwayland.enable = true;
};
```

If both modules try to produce a config, they fight over the same files. My old `extraConfig` blocks were also Hyprlang text, which is not valid once Hyprland expects `hyprland.lua`.

### Symlinking the Lua files

I used `mkOutOfStoreSymlink` so I can edit the Lua files in my repo and reload without a rebuild. My first attempt symlinked the whole `~/.config/hypr` directory, and the build failed:

```
Error installing file '.config/hypr/hypridle.conf' outside $HOME
```

Other Home Manager modules (hypridle, hyprlock, hyprpaper) also write into that directory, so it can't be one big symlink. The fix was to symlink each `.lua` file individually:

```nix
{ config, lib, ... }:
let
  hyprDir = "${config.home.homeDirectory}/nix-server/modules/home-manager/hyprland/config";
  luaFiles = lib.filter (lib.hasSuffix ".lua") (builtins.attrNames (builtins.readDir ./config));
in
{
  xdg.configFile = lib.listToAttrs (
    map (f: {
      name = "hypr/${f}";
      value = {
        source = config.lib.file.mkOutOfStoreSymlink "${hyprDir}/${f}";
        force = true;
      };
    }) luaFiles
  );
}
```

Two things to remember: the path must be an absolute string (a Nix path gets copied into the store, which defeats the point), and a *new* `.lua` file needs a `git add` and a rebuild, because the list comes from `readDir`.

### Two small flake mistakes

- I wrote `program.noctalia.enable` instead of `programs.noctalia.enable`. NixOS even suggested the right name.
- I imported a Home Manager module from `configuration.nix`, which gave `attribute 'inputs' missing`. NixOS modules only get `inputs` through `specialArgs`, while Home Manager gets it through `extraSpecialArgs`. A module that uses `inputs.noctalia.homeModules.default` belongs in the Home Manager imports.

## Part 2: The black screen

Everything built, I logged in, and I saw nothing.

### Step 1: Read the right log

My first log from `start-hyprland` looked clean, with one harmless warning. That was misleading. Near the top it prints:

```
Disabling stdout logs! Check the log for further logs.
```

After that line, everything useful goes to a separate file in `/run/user/1000/hypr/<instance>/hyprland.log`, and crashes also write a report to `~/.cache/hyprland/`.

### Step 2: Don't lose the log

When I went to read the log, `/run/user/1000/hypr` didn't exist. `/run/user/<uid>` is created at login and removed when your last session ends, so logging out and back in wiped it. The fix is to keep a second login open while you test. I SSH'd in from my laptop and left the session running.

### Step 3: A red herring

From that SSH session I launched `start-hyprland` to see what happened, and it crashed with:

```
Could not connect to socket /run/seatd.sock
Backend 'logind' failed to open seat, skipping
libseat: Couldn't open device at /dev/dri/card1
CRIT: Cannot open backend: no allocator available
```

This looked like a GPU failure, but it wasn't. An SSH login has no active seat on the physical display, so Hyprland can't get permission to open the GPU or the input devices. It was only a symptom of the way I tested. Hyprland has to be started from a local TTY, and you can check the session first:

```bash
loginctl show-session $XDG_SESSION_ID -p Type -p Active -p Seat
# want: Type=tty, Active=yes, Seat=seat0
```

### Step 4: The real log

I switched to a real TTY with an SSH session still open, started Hyprland, and read the log. Now it had a seat (`Seat opened with backend 'logind'`), found my real monitors and my keyboard and mice, and showed this:

```
drm: Starting backend for /dev/dri/card0, with driver simpledrm
drm: gpu /dev/dri/card0 becomes primary drm
CDRMRenderer(drm): Can't create renderer, no matching devices found
drm: Failed to initialize renderer state for /dev/dri/card0
```

There were two GPU devices:

```
pci-0000:01:00.0-card -> ../card1                                   (the real NVIDIA GPU)
pci-0000:01:00.0-platform-simple-framebuffer.0-card -> ../card0     (firmware framebuffer)
```

`card0` is a simple framebuffer from the firmware, living on the same PCI device. It can't do OpenGL, but Hyprland picked it as the primary device, so rendering failed.

### The fix

I told Hyprland which GPU to use:

```bash
AQ_DRM_DEVICES=/dev/dri/card1 start-hyprland
```

The screen came up immediately. To make it permanent, I put it in my Hyprland Lua environment file (`env.lua`):

```lua
hl.env("AQ_DRM_DEVICES", "/dev/dri/card1")
```

Card numbers aren't guaranteed to be stable between boots, so if the black screen ever comes back, check `ls -l /dev/dri/by-path/` first. Another option I considered is removing the fake GPU entirely with `boot.kernelParams = [ "initcall_blacklist=simpledrm_platform_driver_init" ];`.

## A leftover annoyance

My editor flagged `Undefined global 'hl'` on every Lua file. It's cosmetic, because Hyprland injects `hl` at runtime. Adding `---@diagnostic disable: undefined-global` at the top of each lua file silences it:

```lua
---@diagnostic disable: undefined-global

require("env")
require("settings")
require("autostart")
require("binds")
require("rules")
```

## What I'd tell myself next time

1. **Test from a local TTY.** SSH-launched Hyprland has no seat and will fail in confusing ways.
2. **Find the second log.** Stdout stops early, and the real log is in `/run/user/<uid>/hypr/`.
3. **Keep a second session open** so the runtime directory isn't wiped.
4. **Look for GPU ordering problems** if you see `no matching devices found`. Firmware framebuffers like simpledrm can hijack the primary slot.
5. **Bisect before reinstalling.** I almost wiped Hyprland from my config for a clean reinstall. The cause turned out to be one environment variable.

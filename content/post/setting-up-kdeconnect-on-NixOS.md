+++
date = '2026-06-02T17:55:03+08:00'
draft = false
title = 'Setting Up Kdeconnect on NixOS'
+++

No matter what desktop environment or window manager I use, KDE Connect is an absolute necessity. Clipboard sharing, receiving phone notifications on my desktop, and using my phone as an emergency trackpad are features I simply cannot live without.

But as with most things, setting up network-dependent services on NixOS requires a bit of planning. You can't just install the package and expect device discovery to work out of the box—our system firewall has other plans.

Here is how to set up KDE Connect the "NixOS" way.

## The Core Concept: The Split-Module Architecture

On traditional distributions, installing KDE Connect handles the network policies automatically. On NixOS, we must respect the boundary between system-level permissions and user-level desktop environments:

1. System-Level (NixOS): Must open the official KDE Connect port ranges (`1714-1764` for TCP and UDP) in the system-wide firewall. Home Manager cannot do this.

2. User-Level (Home Manager): Handles running the actual client daemon, autostarting it, and enabling the system tray indicator.

By splitting our configuration, we keep our declarative design clean and secure.

## Step 1: Opening the System Firewall (NixOS)

First, we create a system-level configuration specifically for network discovery. Without these specific port ranges open, your phone and desktop will remain completely invisible to each other.

```nix
# modules/nixos/services/kdeconnect.nix
{ ... }:

{
  # KDE Connect - System Firewall Configuration
  # Opens the required TCP/UDP ports for device discovery and communication.
  networking.firewall = rec {
    allowedTCPPortRanges = [ { from = 1714; to = 1764; } ];
    allowedUDPPortRanges = allowedTCPPortRanges; # Same range for UDP discovery
  };
}
```

Now, import this file into your system-wide `configuration.nix` under your services imports:

```nix
# hosts/desktop/configuration.nix
{ pkgs, ... }:

{
  imports = [
    # === OTHER IMPORTS ===
    # ...
    ../../modules/nixos/services/syncthing.nix
    ../../modules/nixos/services/openssh.nix

    # Enable the system firewall exception for KDE Connect
    ../../modules/nixos/services/kdeconnect.nix
  ];
}
```

## Step 2: Enabling the Service (Home Manager)

With the network pathways cleared, we can configure the user-level background service and the system tray applet.

Since I'm modernizing my configurations, I opted to use the Qt6-based version `kdePackages.kdeconnect-kde`.

```nix
# modules/home-manager/kdeconnect.nix
{ pkgs, ... }:

{
  # KDE Connect - User Service Configuration
  # Enables the background daemon and system tray indicator for the current user.
  services.kdeconnect = {
    enable = true;
    package = pkgs.kdePackages.kdeconnect-kde; # The modern Qt6-based version
    indicator = true;                          # Show icon in the system tray
  };
}
```

Import this module into your user profile (such as `modules/home-manager/profiles/desktop.nix`), run `sudo nixos-rebuild switch`, and your environment is completely configured!

## Lessons Learned

- Discovery Pitfall: If your devices still cannot see each other after a rebuild, double-check that your phone isn't on a different Wi-Fi band (like a 2.4GHz network vs a 5GHz guest network).

- Tray Indicator: On minimal compositors and standalone window managers (like Hyprland, Niri, Qtile, or i3), ensure you have an active system tray running — for Wayland, something like `waybar`; for X11, `trayer` or `stalonetray` — otherwise the indicator may silently fail to appear even if the daemon is running. You can verify the background service via `systemctl --user status kdeconnect`.

## Final Thoughts

Once again, NixOS proves that while the initial configuration takes a couple of extra steps, the payoff of having a modular, reproducible setup is unmatched. No matter how many times I reinstall my OS or rebuild my flake, my phone and desktop will always find each other seamlessly.

Check out my full implementation here in my [neko-dotfiles](https://github.com/nekomangini/neko-dotfiles)!

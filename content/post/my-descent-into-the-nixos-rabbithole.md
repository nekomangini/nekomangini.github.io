+++
date = '2026-04-24T23:14:26+08:00'
draft = false
title = 'My Descent Into the Nixos Rabbithole: A Retrospective'
+++

I’ve been using NixOS since October 2025. I’ll be honest: I’m still very much a beginner, so if you see me doing something "the wrong way," I am more than happy to be corrected! To keep my head above water, I’ve been living by the [NixOS & Flakes Book](https://nixos-and-flakes.thiscute.world/nixos-with-flakes/get-started-with-nixos).

## The Config Migration

At first, I kept my NixOS config in the standard `/etc/nixos/` directory. But as a developer, I wanted to keep it in a GitHub repository, and managing a Git repo inside `/etc` is... a bit of a headache.

After some YouTube rabbit holes, I learned the secret: you can put your config anywhere as long as you use the right command. I moved everything to my [neko-dotfiles](https://github.com/nekomangini/neko-dotfiles) repository. Now, I run my updates like this:

```nix
sudo nixos-rebuild switch --flake .#neko-desktop --show-trace
```

### Where does `neko-desktop` come from?

For a while, I thought this command was just looking at the hostname I set in my `configuration.nix`. While I do have `networking.hostName = "neko-desktop";` defined there, the `#neko-desktop` in the command actually refers to the **output name** defined in my `flake.nix`.

Looking at my `flake.nix`, you can see how this is structured:

```nix
outputs = { nixpkgs, home-manager, agenix, ... }@inputs:
let
  system = "x86_64-linux";
in
{
  nixosConfigurations = {
    # This is what the '#' in the command refers to!
    neko-desktop = nixpkgs.lib.nixosSystem {
      modules = [
        ./hosts/desktop/configuration.nix
        home-manager.nixosModules.home-manager
        # ... other modules
      ];
    };

    # laptop
    # TODO
    neko-laptop = nixpkgs.lib.nixosSystem {
      modules = [ ./hosts/laptop/configuration.nix ];
    };
  };
};
```

The name you choose there is what `nixos-rebuild` looks for. It's small but important distinction when you start managing multiple machines (like a desktop and a laptop) from the same flake!

### The 2GB RAM Struggle: NixOS vs. My Laptop

You might notice a `# TODO` in my laptop configuration. Here’s the reality: my laptop only has **2GB of RAM**.

In my experience, NixOS really needs about 4GB of RAM to rebuild successfully, especially when evaluating complex flakes and Home Manager modules. Whenever I try to build on the laptop, it hits a wall. Because of this, my laptop setup is currently on hold. Until I can upgrade to a better machine, I’m sticking with **Void Linux** on that device (but that’s a story for a different blog post!).

From there, I dived into [Home Manager](https://nixos-and-flakes.thiscute.world/nixos-with-flakes/start-using-home-manager) and used a mix of Google, YouTube, the NixOS Discourse, Wikis, and AI tools to find my way. My configs started out messy (okay, they are still messy), but it’s a functional mess!

## The Organization Breakthrough: Abstracting with `default.nix`

As my configuration grew, my files became massive. At first, my **Hyprland** config was one giant, intimidating file. While browsing other people's GitHub repositories, I noticed a much cleaner pattern: using `default.nix` to abstract the code.

I converted my `hyprland` into a directory structure like this:

- `hyprland/default.nix`
- `hyprland/autostart.nix`
- `hyprland/environment.nix`
- `hyprland/keybinds.nix`
- `hyprland/rules.nix`
- `hyprland/settings.nix`

My `default.nix` now acts as a "traffic controller," importing all the specific pieces. Now instead of scrolling through 300 lines of code to find a single shortcut, I just open `keybinds.nix`. It makes the whole system much easier to maintain and reason about.

```nix
{ ... }:

{
  imports = [
    ./environment.nix
    ./settings.nix
    ./autostart.nix
    ./keybinds.nix
    ./rules.nix
    ./group.nix
    ../shell/scripts.nix
  ];

  wayland.windowManager.hyprland = {
    enable = true;
    xwayland.enable = true;
  };
}
```

## The Editor Shift: Helix vs. Neovim

Since moving to NixOS, I've found myself switching to **Helix** as my daily driver. The reason is simple: **Helix is much easier to configure the "NixOS way"**. Since Helix comes with most features built-in, configuring it via Home Manager is clean and straightforward.

However, I haven't abandoned Neovim. I've transitioned to **AstroNvim**, and I use a clever Nix trick to manage it. Instead of a traditional alias, I use `writeShellApplication` to create a custom wrappers that set the `NVIM_APPNAME` environment variable.
This allows me to keep my AstroNvim config in `~/.config` while launching it easily through Nix-defined commands like `av` (for terminal) or `ad` (for Neovide)

```nix
# modules/home-manager/neovim/astronvim.nix
{ pkgs, ... }:
{
  home.packages = [
    (pkgs.writeShellApplication {
      name = "av";
      runtimeInputs = [ pkgs.neovim ];
      text = ''
        export NVIM_APPNAME="astronvim-v5"
        exec ${pkgs.neovim}/bin/nvim "$@"
      '';
    })
    (pkgs.writeShellApplication {
      name = "ad";
      runtimeInputs = [ pkgs.neovide ];
      text = ''
        export NVIM_APPNAME="astronvim-v5"
        exec ${pkgs.neovide}/bin/neovide "$@"
      '';
    })
  ];
}
```

My Home Manager profile now imports everything neatly, keeping my terminal, shell, and multiple editors (I even have Emacs, Kakoune, and Vim in there!) organized:

```nix
# modules/home-manager/profiles/desktop.nix
{ pkgs, ... }:
{
  imports = [
    ../helix
    ../neovim/astronvim.nix
    ../emacs.nix
    ../shell/fish
    # ... and many more
  ];

  home.sessionVariables = {
    EDITOR = "${pkgs.helix}/bin/hx";
    VISUAL = "${pkgs.helix}/bin/hx";
  };
}
```

## The Wall: IntelliJ and Android Studio

NixOS is great until you try to install a heavy IDE. Installing **IntelliJ Community Edition** was a struggle because it has many dependencies that conflict with the "Nix way" of doing things. I tried the Flatpak version, but ultimately, I decided to just dual-boot my PC to Windows when I need to use IntelliJ properly.

**Flutter** was another challenge. I learned that setting up a mobile dev environment on NixOS is quite a quest. I found out about `nix-ld`, which is necessary to get Android Studio and its binaries running correctly. My setup is currently "held together with duct tape"—if it works, I’m definitely not touching it again because I don't want to break it!

I’ve also started "nixifying" my environment further. I created some `Raku` scripts for my work, and I eventually learned I could put Raku code directly inside a Nix file. It's a weird but powerful way to live.

## Productivity Tools: TickTick vs. Super Productivity

I’ve used **TickTick** for a long time, but I recently discovered an open-source app called **Super-Productivity**. I really wanted to switch, but the sync between the app and my phone was unreliable. I tried the AppImage and it worked, but it didn't feel like the "NixOS way." For now, I’m sticking with TickTick.

## My Android Studio & Flutter Setup

If you're struggling to get Android Studio and Flutter running on NixOS, here is the setup that finally worked for me.

### 1. Enabling ADB

In my `configuration.nix` (or a dedicated Android module), I enable ADB and use an activation script to symlink the Nix-provided ADB into the Android SDK folder. This prevents Android Studio from complaining about missing or incompatible ADB binaries.

```nix
# modules/nixos/development/android.nix
{ ... }:

{
  nixpkgs.config.android_sdk.accept_license = true;
  programs.adb.enable = true;

  system.userActivationScripts = {
    stdio = {
      text = ''
        rm -f ~/Android/Sdk/platform-tools/adb
        ln -s /run/current-system/sw/bin/adb ~/Android/Sdk/platform-tools/adb
      '';
    };
  };
}
```

### 2. The Magic of `nix-ld`

To run the various non-Nix binaries that Flutter and Android Studio depend on, I enabled nix-ld with the necessary libraries:

```nix
# modules/nixos/development/nix-ld.nix
{ pkgs, ... }:

{
  programs.nix-ld.enable = true;
  programs.nix-ld.libraries = with pkgs; [
    zlib
    stdenv.cc.cc.lib
    ncurses5
    libGL
    libpulseaudio
  ];
}

```

### 3. Home Manager Configuration

Then, I define my packages and session variables in Home Manager:

```nix

{ pkgs, ... }:

{
  home.packages = with pkgs; [
    android-studio
    android-tools
    gradle
    jdk17
    flutter
    ninja
    pkg-config
    mesa-demos
  ];

  home.sessionVariables = {
    CHROME_EXECUTABLE = "${pkgs.vivaldi}/bin/vivaldi";
    ANDROID_HOME = "$HOME/Android/Sdk";
    ANDROID_SDK_ROOT = "$HOME/Android/Sdk";
    JAVA_HOME = "${pkgs.jdk17.home}";
  };
}

```

### 4. Pro-Tip: Finding the Flutter SDK Path

On NixOS, the Flutter SDK is wrapped and buried in the /nix/store. If you need the root directory for your IDE or scripts, use this command:

```nix
dirname $(dirname $(readlink -f $(which flutter)))
```

**What it does**: It finds the flutter binary, resolves all symlinks to the actual store path, and strips away /bin/flutter to give you the SDK root.

## **Lessons Learned**

After a few months in the rabbit hole, here are my main takeaways for anyone starting out:

1. Don't fight the `/etc/nixos` default: Move your config to a user folder and use Flakes as soon as possible. It makes managing your GitHub repo much easier.

2. **AI is a great tour guide, but check the map**: AI tools are amazing for finding Nix syntax, but always cross-reference with the NixOS Wiki or Discourse, as Nix syntax changes fast.

3. `nix-ld` **is your best friend**: If you are a dev using pre-compiled binaries (like VS Code extensions or Android tools), you almost certainly need nix-ld enabled to save your sanity.

4. **Accept the "Functional Mess"**: Your first Nix config won't be elegant. It will be full of "stolen" code from other people's dotfiles. That's okay! Get it working first, then optimize later.

5. **Dual-booting isn't "failing"**: If an IDE like IntelliJ is giving you a 5-hour headache on Nix, it's okay to just use Windows/Standard Linux for that specific task while you learn.

## Final Thoughts

NixOS is a steep learning curve, but the ability to reproduce my entire system from a few files is worth the struggle. If you have any tips on how to clean up a messy flake, let me know! Check out my full configuration here: [neko-dotfiles](https://github.com/nekomangini/neko-dotfiles)

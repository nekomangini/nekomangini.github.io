+++
date = '2026-10-27T11:17:24+08:00'
draft = false
title = 'Void Linux Setup and Why I Moved On'
+++

My laptop is a Lenovo IdeaPad 120S with only 2GB of RAM — not exactly a powerhouse.
NixOS was out of the question, so I turned to Void Linux, a minimal and fast
distribution that felt right at home on constrained hardware.

Here's how I set it up from scratch, and why I eventually moved on.

## Installation

Installation was straightforward — I just followed the official
[Void Linux guide](https://docs.voidlinux.org/installation/live-images/guide.html).
After booting in, the first thing to do is a full system update:

```sh
sudo xbps-install -Su
```

## Window Manager

I went with i3 as my tiling window manager, along with Rofi as a launcher
and i3lock for the lock screen:

```sh
sudo xbps-install -Sy i3 i3lock rofi
```

## Editors

```sh
sudo xbps-install -Sy neovim kakoune helix neovide
```

## Shell

I use Fish as my default shell:

```sh
sudo xbps-install -Sy fish-shell

# Find the fish binary location
which fish

# Check available shells
cat /etc/shells

# Set fish as the default
chsh -s /usr/bin/fish
```

## Git and SSH

```sh
git config --global user.name "Nekomangini"
git config --global user.email "nekomangini.dev@gmail.com"
git config --list
```

Setting up SSH for GitHub:

```sh
# Check for an existing key
ls -al ~/.ssh/id_ed25519.pub

# Generate a new one if needed
ssh-keygen -t ed25519 -C "nekomangini.dev@gmail.com"

# Start the SSH agent
eval (ssh-agent -c)

# Add the key
ssh-add ~/.ssh/id_ed25519
```

Then copy the output of `cat ~/.ssh/id_ed25519.pub` and add it under
**GitHub Settings → SSH and GPG Keys → New SSH Key**.

## Terminal Utilities

```sh
sudo xbps-install -Sy fzf zoxide eza fastfetch htop jq curl wget yazi lazygit cmake
```

**zoxide** — add this to `~/.config/fish/config.fish`:

```fish
zoxide init fish | source
```

**yazi** — install its dependencies first:

```sh
sudo xbps-install -Sy ffmpeg 7zip jq poppler fd ripgrep fzf zoxide resvg ImageMagick xclip
```

Then create `~/.config/fish/functions/y.fish` for the directory-switching wrapper:

```fish
function y
    set tmp (mktemp -t "yazi-cwd.XXXXXX")
    command yazi $argv --cwd-file="$tmp"
    if read -z cwd < "$tmp"; and [ "$cwd" != "$PWD" ]; and test -d "$cwd"
        builtin cd -- "$cwd"
    end
    rm -f -- "$tmp"
end
```

## Emacs (Doom)

Install the dependencies:

```sh
sudo xbps-install -Sy ripgrep fd
```

Install the Symbola font (required by Doom):

```sh
sudo xbps-install -Sy unzip
mkdir -p ~/.local/share/fonts
cp ~/Downloads/symbola.zip ~/.local/share/fonts
cd ~/.local/share/fonts && unzip symbola.zip
fc-cache -fv
```

For Markdown and Bash support:

```sh
sudo xbps-install -Sy shellcheck pandoc shfmt bash-language-server
```

Clone and install Doom:

```sh
git clone --depth 1 https://github.com/doomemacs/doomemacs ~/.config/emacs
~/.config/emacs/bin/doom install
```

Add Doom to your Fish path in `~/.config/fish/config.fish`:

```fish
fish_add_path ~/.config/emacs/bin
```

**Running Emacs as a daemon via runit:**

```sh
sudo mkdir -p /etc/sv/emacs-daemon
sudo nano /etc/sv/emacs-daemon/run
```

Paste this into the run script:

```sh
#!/bin/sh
export HOME=/home/void-nekomangini
cd $HOME
exec chpst -u void-nekomangini:void-nekomangini /usr/bin/emacs --fg-daemon=void-nekomangini-emacsd 2>&1
```

```sh
sudo chmod +x /etc/sv/emacs-daemon/run

# Prevent auto-start before testing
sudo touch /etc/sv/emacs-daemon/down
sudo ln -s /etc/sv/emacs-daemon /var/service/

# Test it
sudo sv once emacs-daemon
sudo sv status emacs-daemon

# If it works, enable permanently
sudo rm /etc/sv/emacs-daemon/down
```

Add to your i3 keybinds:
```config
bindsym $mod+e exec emacsclient --socket-name=void-nekomangini-emacsd -c -n
```
## Programming Languages

```sh
sudo xbps-install -Sy go nodejs-lts lua rakudo
npm i -g @vue/language-server typescript
```

**Go tools:**

```sh
go install golang.org/x/tools/gopls@latest
go install github.com/go-delve/delve/cmd/dlv@latest
go install golang.org/x/tools/cmd/goimports@latest
go install mvdan.cc/gofumpt@latest
go install github.com/fatih/gomodifytags@latest
go install github.com/cweill/gotests/gotests@latest
go install github.com/x-motemen/gore/cmd/gore@latest
go install github.com/mdempsky/gocode@latest

fish_add_path ~/go/bin
```

**Web LSP and tooling:**

```sh
npm i -g typescript typescript-language-server @vue/language-server \
  vscode-langservers-extracted @astrojs/language-server \
  prettier js-beautify eslint stylelint stylelint-config-standard emmet-ls

sudo xbps-install -Sy tidy5
```

## Why I Moved On

Void Linux worked well, but maintaining two separate sets of dotfiles — one for Void Linux and one for my NixOS desktop — was becoming a chore.
That changed when I finally learned how to run Home Manager standalone on a non-NixOS system.
Once I knew that was possible, I switched to [Omarchy](https://omarchy.org), an opinionated Arch-based distribution, and moved my laptop config entirely under Home Manager.

Now a single Home Manager configuration covers both my desktop (`sylphiette`,
running NixOS) and my laptop (`roxy`, running Omarchy). No more drift,
no more duplication.

In a future post, I'll walk through exactly how I set up standalone Home
Manager on Omarchy — including how I structure the config so it stays in sync
with my NixOS flake.

---

Check out my full configuration in my
[nix-server](https://github.com/nekomangini/nix-server) dotfiles repo.

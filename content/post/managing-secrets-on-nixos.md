+++
date = '2026-05-08T12:07:22+08:00'
draft = false
title = 'Managing Secrets on Nixos: My Journey with Agenix'
+++

As my NixOS configuration grows, so does the need to handle sensitive data securely. I’ve recently started using **Agenix** to manage secrets, specifically to hide project folder paths.

The transition hasn't been without its hurdles, particularly setting up Agenix as a noob.

## The Challenge: Installing Agenix

As I continue to "rice" my system and automate my workflows, I’ve hit a common roadblock: how to handle sensitive data without leaking it to my public GitHub repository. My latest challenge involves using **Agenix** to hide project folder paths for testing purposes.

### Flake Configuration & CLI Installation

Here is how I structured my `flake.nix` to ensure Agenix is available across my desktop and my other configurations:

```nix
{
  description = "NixOS configuration with flakes";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-25.11";
    agenix.url = "github:ryantm/agenix";
    home-manager = {
      url = "github:nix-community/home-manager/release-25.11";
      inputs.nixpkgs.follows = "nixpkgs";
    };
  };

  outputs = { nixpkgs, home-manager, agenix, ... }@inputs:
    let
      system = "x86_64-linux";
    in {
      homeConfiguration.neko-desktop = home-manager.lib.homeManagerConfiguration { };
      nixosConfigurations = {
        neko-desktop = nixpkgs.lib.nixosSystem {
        modules = [
          ./hosts/desktop/configuration.nix
          agenix.nixosModules.default # System-level Agenix
          { environment.systemPackages = [ agenix.packages.${system}.default ]; } # Agenix cli

          home-manager.nixosModules.home-manager {
            home-manager.useGlobalPkgs = true;
            home-manager.useUserPackages = true;
            home-manager.sharedModules = [ agenix.homeManagerModules.default ]; # User-level Agenix
            home-manager.users.nekomangini = import ./modules/home-manager/profiles/desktop.nix;
            home-manager.extraSpecialArgs = { inherit inputs; };
          }
        ];
      };
    };
  };
}
```

My goal was to pass an age-encrypted secret (containing a project path) into a **Raku** script I use for managing **tmux** sessions.

### Why Nix Can't See Your Secrets Folder

Initially, I kept my `secrets/` folder outside my dotfiles repository --- sitting somewhere in my home directory. When I tried referencing it with a relative path like `../../../secrets/` in my Nix configuration, the build simply failed. The folder wasn't found

The reason is straightforward: **Nix Flakes only see files that are tracked by git**. If a file isn't added to the repository, it doesn't exist from the flakes perspective --- no matter where it actually lives on disk. This has nothing to do with `--impure` mode;it's simply how the Nix sandbox works with flakes.

The fix was simple: move the `secrets/` folder directly into my dotfiles repository and run `git add secrets/`. Since the `.age` files are encrypted, committing them to a public repo is completely safe -- they are useless to anyone without the corresponding private key.

## The Workflow

Here are the steps I took to get the environment ready:

1.  **SSH Key**: I ensure that I have a **SSH** key. You can follow this [tutorial](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent) if you don't have one.
2.  **Secrets Definition**: I created a directory named _secrets/_ to store **secrets.nix** and mapped my SSH IDs and host keys within `secrets.nix` to authorize decryption for my specific machine.
3.  **Creating secrets**: Create a secret file `tmux-manager-paths.age`.

    ```nix
    cd secrets/
    agenix -e tmux-manager-paths.age
    ```

4.  **Add the secret to NixOS module config**

    ```nix
    {
      age = {
        # identityPaths = [ "/home/nekomangini/.ssh/id_ed25519" ]; # requires passphrase on boot
        identityPaths = [ "/etc/ssh/ssh_host_ed25519_key" ]; # no passphrase on boot
        secrets = {
          tmux-manager-paths = {
            file = ../../../secrets/tmux-manager-paths.age;
            owner = "nekomangini";
          };
        };
      };
    }
    ```

5.  **Reference the mount path**

    ```nix
      home.sessionVariables = {
        TMUX_PATHS_FILE = "/run/agenix/tmux-manager-paths";
      };

    ```

    You can also check the contents tmux-manager.age

    ```shell

    cat /run/agenix/tmux-manager-paths
    ```

6.  **Use it on my tmux-manager.nix script**:

    ```nix

    { pkgs, ... }:
    let
      tmux-manager = pkgs.writeScriptBin "tmx" ''
        #!${pkgs.rakudo}/bin/raku
        use v6.d;

        our %env;

        sub BUILD-ENV {
            my $env-file = %*ENV<TMUX_PATHS_FILE> // "/run/agenix/tmux-manager-paths";
            my $file = $env-file.IO;

            unless $file.e {
                note "⚠️ Secret not found at: $env-file";
                return;
            }

            %env = $file.lines
                .grep({ .contains('=') && !.starts-with('#') })
                .map({
                    my ($key, $val) = .split('=', 2);
                    $val = $val.trim;
                    $val ~~ s/^ <['"]> //;
                    $val ~~ s/ <['"]> $ //;
                    $val = $val.subst(/^ '~'/, $*HOME.Str) if $val.starts-with('~');
                    $key.trim => $val;
                })
                .hash;
        }

        sub MAIN(Str $choice?) {
            BUILD-ENV();

            my @sessions = <vue flutter notes dotfiles ruby main>;

            my $selected = $choice // do {
                my $proc   = run '${pkgs.fzf}/bin/fzf',
                                 '--prompt=Tmux session: ',
                                 '--height=~50%',
                                 '--layout=reverse',
                                 '--border',
                                 :in, :out;
                $proc.in.say: @sessions.join("\n");
                $proc.in.close;
                $proc.out.slurp(:close).trim;
            };

            exit 0 unless $selected;

            unless $selected ∈ @sessions {
                note "❌ '$selected' is not a predefined session.";
                exit 1;
            }

            my $check = run '${pkgs.tmux}/bin/tmux', 'has-session', '-t', $selected, :out, :err;
            create-session($selected) if $check.exitcode != 0;

            if %*ENV<TMUX> {
                shell '${pkgs.tmux}/bin/tmux switch-client -t ' ~ $selected;
            } else {
                shell '${pkgs.tmux}/bin/tmux attach-session -t ' ~ $selected;
            }
        }

        sub create-session(Str $name) {
            my $path = do given $name {
                when 'flutter'  { %env<FLUTTER_PATH>  }
                when 'vue'      { %env<VUE_PATH>      }
                when 'notes'    { %env<NOTES_PATH>    }
                when 'dotfiles' { %env<DOTFILES_PATH> }
                when 'ruby'     { %env<RUBY_PATH>     }
                when 'main'     { $*HOME.Str          }
                default         { $*HOME.Str          }
            } // $*HOME.Str;

            unless $path.IO.e && $path.IO.d {
                note "📂 Path not found: '$path'. Falling back to \$HOME.";
                $path = $*HOME.Str;
            }

            say "🚀 Creating '$name' in: $path";
            run '${pkgs.tmux}/bin/tmux', 'new-session', '-d', '-s', $name, '-c', $path;
            sleep 0.1;

            given $name {
                when 'vue' {
                    run '${pkgs.tmux}/bin/tmux', 'rename-window', '-t', "{$name}:0", 'editor';
                    run '${pkgs.tmux}/bin/tmux', 'send-keys', '-t', "{$name}:0", 'hx', 'C-m';
                }
                when 'flutter' {
                    run '${pkgs.tmux}/bin/tmux', 'rename-window', '-t', "{$name}:0", 'editor';
                    run '${pkgs.tmux}/bin/tmux', 'send-keys', '-t', $name, 'hx', 'C-m';
                }
                when 'dotfiles' {
                    run '${pkgs.tmux}/bin/tmux', 'rename-window', '-t', "{$name}:0", 'editor';
                    run '${pkgs.tmux}/bin/tmux', 'send-keys', '-t', "{$name}:0", 'hx', 'C-m';
                    run '${pkgs.tmux}/bin/tmux', 'new-window', '-t', $name, '-n', 'console', '-c', $path;
                    run '${pkgs.tmux}/bin/tmux', 'select-window', '-t', "{$name}:editor";
                }
                when 'notes' {
                    run '${pkgs.tmux}/bin/tmux', 'rename-window', '-t', "{$name}:0", 'editor';
                    run '${pkgs.tmux}/bin/tmux', 'send-keys', '-t', "{$name}:0", '${pkgs.emacs}/bin/emacsclient -nw -a "" "' ~ $path ~ '"', 'C-m';
                }
                when 'ruby' {
                    run '${pkgs.tmux}/bin/tmux', 'rename-window', '-t', "{$name}:0", 'editor';
                    run '${pkgs.tmux}/bin/tmux', 'send-keys', '-t', "{$name}:0", 'hx', 'C-m';
                    run '${pkgs.tmux}/bin/tmux', 'new-window', '-t', $name, '-n', 'server', '-c', $path;
                    run '${pkgs.tmux}/bin/tmux', 'send-keys', '-t', "{$name}:server", 'hx .', 'C-m';
                    run '${pkgs.tmux}/bin/tmux', 'new-window', '-t', $name, '-n', 'console', '-c', $path;
                    run '${pkgs.tmux}/bin/tmux', 'select-window', '-t', "{$name}:server";
                }
                default {
                    run '${pkgs.tmux}/bin/tmux', 'send-keys', '-t', $name, 'y', 'C-m';
                }
            }
        }
      '';
    in
    {
      home.packages = [ tmux-manager ];
    }

    ```

7.  **Rebuild NixOS**

    ```shell
     sudo nixos-rebuild switch --flake .#neko-desktop
    ```

## Current Status

I am currently fine-tuning my [neko-dotfiles](https://github.com/nekomangini/neko-dotfiles) to integrate these changes. While it has been a challenge to use Agenix as a NixOS newbie—especially when it comes to understanding how secrets are symlinked and accessed across different modules—the learning curve has been rewarding.

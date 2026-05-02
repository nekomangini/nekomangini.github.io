+++
date = '2026-04-30T22:44:29+08:00'
draft = false
title = 'Setting Up Helix for Vuejs Development on Nixos'
+++

Following my move to NixOS and my shift toward **Helix** as my primary editor, I recently tackled the challenge of setting up a robust Vue.js development environment. While the [Helix Wiki](https://github.com/helix-editor/helix/wiki/Language-Server-Configurations#vue) provides a great starting point, NixOS adds a unique layer of complexity when it comes to locating plugins in the `/nix/store`.

Here is how I got it working.

## The Toolchain

First, we need to ensure the necessary language servers and formatters are available in the environment. I use Home Manager to manage these packages:

```nix
# home-manager/modules/dev-tools.nix
{ pkgs, ... }:

{
  home.packages = with pkgs; [
    vue-language-server          # The core Vue LSP
    typescript-language-server   # Required for TS support in Vue
    typescript                   # Global TS package
    prettier                     # For auto-formatting
    emmet-ls                     # For those sweet HTML snippets
  ];
}
```

## The "Hard Part": Locating the TypeScript Plugin

Helix needs to know exactly where the `@vue/typescript-plugin` is located to provide intellisense inside `.vue` files. On a traditional OS, this is usually tucked away in a global `node_modules`. On NixOS, it's buried deep within a unique, read-only path in the `/nix/store`.

To find it, I had to do a bit of "store-spelunking":

1. Find the base path: `ls /nix/store | grep vue-language-server`

2. Locate the specific plugin folder:

```bash
find /nix/store/<your-hash>-vue-language-server-3.x.x -wholename "*/@vue/typescript-plugin"
```

### The Solution (Nix Configuration)

Instead of hardcoding a long, brittle path that changes every time the package updates, we can use Nix's string interpolation `${pkgs.vue-language-server}` to point directly to the store path dynamically.
Here is the resulting `languages.toml` configuration:

I split my configuration into servers.nix for the LSP logic and languages.nix for the language-specific settings.

### 1. Defining the Language Server (`servers.nix`)

In the server definition, we tell the `typescript-language-server` where to find the Vue plugin:

```nix
{ pkgs, ... }:

{
  programs.helix.languages.language-server = {
    "typescript-language-server" = {
      command = "typescript-language-server";
      args = [ "--stdio" ];
      config = {
        plugins = [
          {
            name = "@vue/typescript-plugin";
            location = "${pkgs.vue-language-server}/lib/language-tools/packages/language-server/node_modules/@vue/typescript-plugin";
            languages = [ "vue" ];
          }
        ];
      };
    };

    "emmet-ls" = {
      command = "emmet-ls";
      args = [ "--stdio" ];
    };
  };
}
```

### 2. Configuring the Vue Language (`languages.nix`)

Next, we define the `vue` language settings, including the auto-formatter and the comment token:

```nix
{ ... }:

{
  programs.helix.languages = {
    language = [
      {
        name = "vue";
        scope = "source.vue";
        injection-regex = "vue";
        file-types = [ "vue" ];
        comment-token = "//"; # My temporary fix for script/style comments
        roots = [
          "package.json"
          "nuxt.config.ts"
          "nuxt.config.js"
          "vite.config.ts"
        ];
        language-servers = [
          "typescript-language-server"
          "emmet-ls"
        ];
        formatter = {
          command = "prettier";
          args = [ "--parser" "vue" ];
        };
        auto-format = true;
      }
    ];
  };
}
```

## The Commenting Quirk

One minor hurdle remained: Helix's default behavior for Vue is to use HTML-style comments `<!-- -->`. This is great for the `<template>`, but a bit of a headache when you're inside a `<script>` or `<style>` block.

Adding `comment-token = "//"` to the language config forces the use of double-slashes everywhere. It’s not a perfect context-aware solution yet (since it uses `//` in the HTML part too), but for now, it's a trade-off I can live with!

## Final Thoughts

The steep learning curve of NixOS pays off once you have a configuration that is entirely reproducible. Now, no matter what machine I’m on, my Vue environment is just a `nixos-rebuild switch` away.

### Support my Journey

If you enjoyed this post and want to support my descent into Linux madness, you can find me here:

- [Buy Me a Coffee](https://buymeacoffee.com/nekomangini)
- [Support on Ko-fi](https://ko-fi.com/nekomangini)

Every bit of support helps keep the nix-builds running!

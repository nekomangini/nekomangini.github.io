+++
date = '2026-03-03T01:42:10+08:00'
draft = false
title = 'From openSUSE to NixOS: A Migration Story'
+++

The last time I updated this blog was back in September 2024. If you’ve been wondering where I disappeared to, the answer is simple: I fell deep down the Neovim and Emacs rabbit holes. What started as "just a few tweaks" to my dotfiles turned into a month-long obsession with perfecting my development environment. Since then, my setup has evolved even further—I've actually switched from Neovim to **Helix** for my main editing, while still keeping Emacs around specifically for the power of **Org-mode**.

## The Leap to NixOS

In October 2025, I finally made the leap from openSUSE to **NixOS**. I’ve always had a soft spot for openSUSE—it’s incredibly stable and served me well for a long time. However, I eventually hit a breaking point with my hardware configuration.

As an Nvidia user, I dealt with broken drivers after kernel updates one too many times. If you’ve ever tried to maintain Nvidia drivers on a traditional Linux distro, you know exactly what a pain it can be. NixOS offered a way out through its declarative configuration; being able to roll back to a working "generation" if a driver update fails is a total game-changer for my peace of mind. To be fair, openSUSE also has rollback features (via Snapper), but I never quite learned how to use them properly—NixOS just made the process feel more intuitive for my workflow.

## A Change in Workflow: From Jekyll to Hugo

The migration to NixOS didn't just change my OS; it changed how I blog. On my previous setup, I used **Jekyll**, which relies heavily on Ruby and various Gems. Trying to manage those Ruby dependencies and environments within the Nix ecosystem became a massive headache. It felt like I was fighting my tools just to write a simple post.

That frustration led me to officially migrate the site to **Hugo**. While I briefly considered Zola (mostly because it’s written in Rust and I love the speed), I ultimately stuck with Hugo. Its build times are nearly instantaneous, and the community support is massive, making it much easier to customize my "Neko" theme exactly how I want it.

## What’s Next?

Since the switch, I’ve also spent quite a bit of time curating my own personal wallpaper repository, [nekopaper], which has been a fun side project to keep my desktop aesthetic in sync with my new NixOS setup. You can also check out the [nekopaper repository] directly.

I’m currently drafting a much more detailed "Deep Dive" post about my experience as a NixOS "noob"—from the initial confusion of Nix Flakes to the satisfaction of a perfectly working `configuration.nix`. Stay tuned for that!

[nekopaper]: https://nekopaper.netlify.app
[nekopaper repository]: https://github.com/nekomangini/nekopaper

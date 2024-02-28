---
title: 'Some dotfile updates'
description: A few updates and changes to my Nix dotfiles.
tags: ['nix']
date: 2023-02-27
---

I use Nix to manage the user environment for all my machines, and I'm constantly finding new ways to use it. Here are some updates to my [Nix dotfiles](https://github.com/x0ba/dotfiles) that I've made over the past month.

## Sops-Nix to Agenix

I've been using Sops-Nix for a long time to manage secrets in my dotfiles. There were a number of problems with this. For one, I store my personal PGP key, which is used to encrypt and decrypt the sops secrets, on a Yubikey 5C. A weird side-effect (feature?) of sops-nix is that the secrets are _continually_ decrypted by a daemon. As soon as I unplugged my Yubikey, the secrets disappeared. I didn't want my Yubikey to be held hostage by my MacBook's USB port, so I decided to switch to Agenix. With Agenix, the secrets are decrypted once on rebuild, and they stay in my home folder. This also has another advantage; I don't need to physically plug in my Yubikey to _decrypt_ secrets, only to _encrypt_ them with [`age-plugin-yubikey`](https://github.com/str4d/age-plugin-yubikey). This is especially helpful with servers, where I can just use the machine's `ssh` key to decrypt the age-encrypted secrets.

## Nvfetcher to fetch non-flake sources

For things like `zsh` plugins that are not in `nixpkgs`, [`yabai`](https://github.com/koekeishiya/yabai) releases not yet updated in `nixpkgs`, and [`wezterm`](https://wezfurlong.org/wezterm/index.html) plugins, I've been using Nix functions like `fetchZip` and `fetchFromGitHub` to fetch them using Nix. This was clunky and unintuitive, and it was tedious to go through and update URLs and SHAs whenever a new update to these sources was released. Nvfetcher makes this a lot easier. I can simply define sources in a `toml` file like so:

```toml
[yabai]
src.github = "koekeishiya/yabai"
fetch.url = "https://github.com/koekeishiya/yabai/releases/download/$ver/yabai-$ver.tar.gz"

```

and have Nvfetcher automatically generate a Nix expression to fetch it, complete with `url` and `sha`, that I can then reference in my Nix configurations:

```nix
  yabai = {
    pname = "yabai";
    version = "v6.0.15";
    src = fetchurl {
      url = "https://github.com/koekeishiya/yabai/releases/download/v6.0.15/yabai-v6.0.15.tar.gz";
      sha256 = "sha256-8+jAdwF7Yganvv1NsbtMIBWv0rh9JmHuwLWmwiFmDu4=";
    };
  };
```

Very handy.

## Configurations for light-dark mode sync

Before this Nix madness, I used to configure my machines to automatically switch between dark and light mode based on the time of day. I still firmly believe that's a good thing to do; nobody wants to read light text on a dark screen when sunlight is shining through the window. However, some tiny problems, such as discrepancies in `zsh-syntax-highlighting` themes, `bat` themes and `starship` themes kept me from doing so. Themes that were made for light mode were being applied in dark mode, and vice versa. They looked bad.

{% eleventyImage "./src/assets/images/blog/eww.png", "An example of text-background mismatch", "An example of text-background mismatch" %}

I fixed this with a little Nix code that I half borrowed half wrote. It performs a check that returns whether the system is in light mode or dark mode, and adds a startup hook to my shell that changes the `starship` and `bat` colorschemes accordingly[^first]. It looks something like this:

```nix
{
  config,
  lib,
  pkgs,
  ...
}: let
  inherit (pkgs.stdenv) isDarwin isLinux;
in {
  config = lib.mkIf config.isGraphical {
    home.packages = [
      (pkgs.writeShellApplication {
        name = "dark-mode-ternary";
        runtimeInputs = [pkgs.gnugrep];
        text = let
          queryCommand =
            if isLinux
            then "dbus-send --session --print-reply=literal --reply-timeout=5 --dest=org.freedesktop.portal.Desktop /org/freedesktop/portal/desktop org.freedesktop.portal.Settings.Read string:'org.freedesktop.appearance' string:'color-scheme' | grep -q 'uint32 1'"
            else if isDarwin
            then "defaults read -g AppleInterfaceStyle &>/dev/null"
            else throw "Unsupported platform";
        in ''
          [[ -z "''${1-}" ]] && [[ -z "''${2-}" ]] && echo "Usage: $0 <dark> <light>" && exit 1

          if ${queryCommand}; then
            echo "$1"
          else
            echo "$2"
          fi
        '';
      })
    ];

    programs.zsh = {
      shellAliases.cat = "bat --theme=$(dark-mode-ternary 'Catppuccin-frappe' 'Catppuccin-latte')";
      initExtra = ''
        zadm_sync() {
          export STARSHIP_CONFIG__PALETTE="catppuccin_$(dark-mode-ternary frappe latte)"
          fast-theme "XDG:catppuccin-$(dark-mode-ternary frappe latte)" >/dev/null
        }
        add-zsh-hook precmd zadm_sync
      '';
    };
  };
}
```

If you'd like to see more, feel free to browse (and steal from) my full dotfiles repo [here](https://github.com/x0ba/dotfiles).

[^first]: Note: this code only works for Starship if [this](https://patch-diff.githubusercontent.com/raw/starship/starship/pull/4439.patch) patch is applied to Starship

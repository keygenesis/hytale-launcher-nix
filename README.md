# Hytale Launcher for Nix

[![Build Status](https://github.com/JPyke3/hytale-launcher-nix/actions/workflows/build.yml/badge.svg)](https://github.com/JPyke3/hytale-launcher-nix/actions/workflows/build.yml)
[![Update Check](https://github.com/JPyke3/hytale-launcher-nix/actions/workflows/update-hytale.yml/badge.svg)](https://github.com/JPyke3/hytale-launcher-nix/actions/workflows/update-hytale.yml)

A Nix flake that packages the official [Hytale Launcher](https://hytale.com) with automatic updates. New upstream releases are detected hourly and packaged automatically.

## Why This Exists

Hypixel Studios distributes the Hytale Launcher exclusively as a Flatpak. While Flatpak works, it comes with trade-offs for NixOS users:

| Concern | Flatpak | This Flake |
|---------|---------|------------|
| **Integration** | Separate runtime, sandbox overhead | Native system integration |
| **Declarative config** | Requires extra setup | Works with standard Nix patterns |
| **Reproducibility** | Flatpak-managed updates | Pinnable via flake lock |
| **Disk usage** | Flatpak runtime + app | Just the launcher (~50MB) |

This flake extracts the launcher from the official Flatpak and repackages it as a native Nix derivation, giving you the best of both worlds: official binaries with Nix's declarative package management.

## Quick Start

**Try it without installing:**
```bash
nix run github:JPyke3/hytale-launcher-nix
```

**Install to your profile:**
```bash
nix profile install github:JPyke3/hytale-launcher-nix
```

## Installation

### NixOS Configuration

```nix
{
  inputs.hytale-launcher.url = "github:JPyke3/hytale-launcher-nix";

  outputs = { nixpkgs, hytale-launcher, ... }: {
    nixosConfigurations.yourhost = nixpkgs.lib.nixosSystem {
      modules = [{
        environment.systemPackages = [
          hytale-launcher.packages.x86_64-linux.default
        ];
      }];
    };
  };
}
```

### Home Manager

```nix
{ inputs, pkgs, ... }:
{
  home.packages = [
    inputs.hytale-launcher.packages.${pkgs.system}.default
  ];
}
```

## Available Packages

| Package | Description |
|---------|-------------|
| `hytale-launcher` | FHS-wrapped launcher (default) - supports self-updates |
| `hytale-launcher-unwrapped` | Raw binary without FHS wrapper |

The default package uses an FHS environment, allowing the launcher's built-in update mechanism to function normally. When Hytale pushes an update, the launcher can update itself just like it would on a traditional Linux system.

## How It Works

### Automatic Updates

This repository checks for new Hytale Launcher versions every hour:

1. **Detection**: GitHub Actions fetches the latest Flatpak and computes its SHA256 hash
2. **Comparison**: If the hash differs from the current package, an update is available
3. **PR Creation**: A pull request is automatically created with the new hash
4. **Auto-merge**: After CI verifies the build succeeds, the PR merges automatically

Since Hytale doesn't publish semantic versions, we use date-based versioning (`YYYY.MM.DD`). Multiple same-day releases get a suffix like `2025.01.14.2`.

### Technical Details

The build process:

1. **Fetch** the official Flatpak from `launcher.hytale.com`
2. **Extract** using `ostree` to unpack the Flatpak's OSTree repository
3. **Patch** ELF binaries with `autoPatchelfHook` to use Nix store libraries
4. **Wrap** in an FHS environment with all required dependencies (GTK, WebKit, graphics drivers, audio)

The FHS wrapper ensures the launcher can:
- Write to `~/.local/share/Hytale` for game data
- Self-update its binary when Hytale pushes updates
- Access system graphics (OpenGL, Vulkan) and audio (PipeWire, PulseAudio)

## Requirements

- NixOS or Nix with flakes enabled
- x86_64-linux (the only platform Hytale supports)
- Graphics drivers configured (Mesa/NVIDIA)

## Troubleshooting

**Launcher crashes immediately**
- Ensure you have graphics drivers installed (`hardware.opengl.enable = true` on NixOS)
- Check if Vulkan is available: `vulkaninfo`

**No audio**
- Verify PipeWire or PulseAudio is running
- The FHS environment includes both; your system just needs one configured

**Self-update fails**
- This is expected with `hytale-launcher-unwrapped`
- Use the default `hytale-launcher` package for self-update support

## Contributing

Issues and PRs welcome. The update mechanism is fully automated, but improvements to the packaging or documentation are appreciated.

## Credits

- [Hypixel Studios](https://hypixelstudios.com/) for Hytale
- Inspired by [claude-code-nix](https://github.com/sadjow/claude-code-nix)'s auto-update approach

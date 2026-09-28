# Scrcpy Wireless Manager

Cross-platform desktop GUI built with Tauri v2 and Vue 3 for wireless Android device control via ADB and Scrcpy.

## Key Features

- Daily wireless connection management with local storage persistence (`localStorage`).
- 6-digit PIN device pairing support for initial Android wireless pairing.
- Integrated active device connection scanner.
- Startup dependency check with a guided installer for missing binaries.
- One-click Scrcpy engine update that tracks upstream Genymobile releases.
- Modern dark mode interface designed with pure inline SVG iconography.

## Prerequisites

The application checks for `adb` and `scrcpy` on startup. When a binary is missing, the setup wizard offers to install it for the current platform. The manual commands below are only needed if you prefer to manage these binaries yourself, or if an automatic installation fails.

### Linux

Install `adb` and `scrcpy` using your distribution package manager:

```bash
sudo apt install scrcpy adb
```

Note: For devices running Android 16 or newer, Scrcpy v2.7+ is required.

### macOS

Use Homebrew to install the dependencies:

```bash
brew install scrcpy android-platform-tools
```

### Windows

1. Download the official Scrcpy release from Genymobile's GitHub repository.
2. Extract the downloaded archive.
3. Add the extracted folder (which contains `scrcpy.exe` and `adb.exe`) to your Windows system PATH environment variable.

## Updating the Scrcpy Engine

The dashboard shows the detected engine version and a **Check for updates** button. Pressing it queries the GitHub Releases API for `Genymobile/scrcpy` and compares the upstream tag against the locally detected version.

| Platform | Update method |
| --- | --- |
| Linux | Downloads the upstream release archive into `~/.local/share/scrcpy-gui/portable/<version>/` and runs the engine from that directory. No root access required. |
| Windows | Downloads the upstream release archive into `%LOCALAPPDATA%\scrcpy-gui\bin`. |
| macOS | Runs `brew upgrade scrcpy`. |

Every downloaded archive is verified against the SHA-256 digest published by the GitHub Releases API before it is executed. An archive whose digest is missing or does not match is discarded and never run.

Portable mode keeps the `scrcpy` binary and its matching `scrcpy-server` in the same directory, so the client and the server pushed to the device cannot drift out of version sync. Previous portable versions are removed during an update.

If upstream publishes no verified build for your platform, or the portable installation fails, the application falls back to the system package manager: `apt-get install --only-upgrade scrcpy adb` on Linux, `brew upgrade scrcpy` on macOS.

## Development & Build

To start the development mode:

```bash
npm run tauri dev
```

To compile the application and build a local installer:

```bash
npm run tauri build
```

## Acknowledgements & Credits

- **Scrcpy**: Developed by Genymobile / Romain Vimont (@rom1v) and released under the Apache License 2.0.
- **ADB**: Part of the Android Open Source Project (AOSP).

## Disclaimer

This application is an independent graphical user interface (GUI) and is not an official product of Genymobile.

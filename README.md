# Scrcpy Wireless Manager

Cross-platform desktop GUI built with Tauri v2 and Vue 3 for wireless Android device control via ADB and Scrcpy.

## Key Features

- Daily wireless connection management with local storage persistence (`localStorage`).
- 6-digit PIN device pairing support for initial Android wireless pairing.
- Integrated active device connection scanner.
- Modern dark mode interface designed with pure inline SVG iconography.

## Prerequisites

This application requires `adb` and `scrcpy` binaries available on your system and included in the system PATH environment variable.

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

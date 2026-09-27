# Scrcpy Wireless Manager

GUI lintas platform berbasis Tauri v2 dan Vue 3 untuk kontrol nirkabel Android (ADB & Scrcpy).

## Fitur Utama

- Tab koneksi harian dengan penyimpanan preferensi lokal (localStorage).
- Tab pairing 6 digit kode.
- Pemindai status koneksi perangkat terintegrasi.
- Desain antarmuka modern dark mode berbasis ikon SVG murni.

## Prerequisites (Dependensi Sistem)

Aplikasi ini membutuhkan `adb` dan `scrcpy` tersedia di sistem Anda dan terdaftar pada Environment Variable (PATH).

### Linux

Pastikan menginstal `adb` dan `scrcpy` menggunakan manajer paket distribusi Anda (misalnya `apt`, `pacman`, atau `dnf`).
Catatan: Untuk perangkat dengan Android 16, Anda membutuhkan Scrcpy v2.7+.

### macOS

Gunakan Homebrew untuk menginstal dependensi:

```bash
brew install scrcpy android-platform-tools
```

### Windows

1. Unduh rilis resmi Scrcpy dari repositori GitHub Genymobile.
2. Ekstrak arsip yang diunduh.
3. Tambahkan folder hasil ekstraksi (yang berisi `scrcpy.exe` dan `adb.exe`) ke dalam Environment Variable PATH sistem Windows Anda.

## Development & Build

Untuk memulai mode pengembangan:

```bash
npm run tauri dev
```

Untuk melakukan kompilasi (build) dan membuat installer lokal:

```bash
npm run tauri build
```

## Acknowledgements & Credits

- **Scrcpy**: Dikembangkan oleh Genymobile / Romain Vimont (@rom1v) dan didistribusikan di bawah lisensi Apache License 2.0.
- **ADB**: Merupakan bagian dari Android Open Source Project (AOSP).

## Disclaimer

Aplikasi ini adalah antarmuka (GUI) independen dan bukan merupakan produk resmi dari Genymobile.

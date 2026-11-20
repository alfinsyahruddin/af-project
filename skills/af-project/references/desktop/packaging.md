# Desktop Packaging & Distribution

This reference provides production-ready packaging workflows for distributing native Rust + GPUI desktop applications across macOS, Windows, and Linux.

---

## 1. Multi-Platform Support Matrix

| Platform | Primary Output | Packaging Method | Prerequisites |
| :--- | :--- | :--- | :--- |
| **macOS** | `.app` bundle, `.dmg` | Shell scripts (`archive.sh`, `package.sh`), `hdiutil`, codesign | Xcode CLI tools, Apple Developer ID certificate (for notarization) |
| **Windows** | Setup `.exe` | Inno Setup (`installer.iss`), `build.rs` embedding `winres` resources | Inno Setup 6+, MSVC compiler toolchain |
| **Linux** | `.deb` package | Shell script (`package.sh`), `dpkg-deb`, freedesktop `.desktop` | Ubuntu/Debian environment, build-essential |

---

## 2. macOS Packaging Pipeline

Desktop apps on macOS require a valid `.app` bundle containing the binary, icons, `Info.plist`, and proper codesign entitlements.

### Directory Structure
```text
packaging/macos/
├── Info.plist.in       # Bundle identifier, version, executable name template
├── entitlements.plist  # Hardened runtime entitlements
├── archive.sh          # Universal/target-specific xcarchive builder
└── package.sh          # DMG generator
```

### Key Packaging Steps (`archive.sh`)
```bash
#!/bin/bash
set -euo pipefail

TARGET="${1:-aarch64-apple-darwin}"
APP_NAME="MyApp"
VERSION="0.1.0"
BUNDLE_ID="com.alfin.myapp"

echo "==> Building release binary for $TARGET..."
cargo build --release --locked --target "$TARGET" -p "$APP_NAME-desktop"

APP_BUNDLE="dist/$APP_NAME.app"
rm -rf "$APP_BUNDLE"
mkdir -p "$APP_BUNDLE/Contents/MacOS"
mkdir -p "$APP_BUNDLE/Contents/Resources"

# 1. Copy binary
cp "target/$TARGET/release/$APP_NAME-desktop" "$APP_BUNDLE/Contents/MacOS/$APP_NAME"
chmod +x "$APP_BUNDLE/Contents/MacOS/$APP_NAME"

# 2. Copy App Icons & Assets
cp "assets/icons/app.icns" "$APP_BUNDLE/Contents/Resources/app.icns"

# 3. Generate Info.plist
sed -e "s/@APP_NAME@/$APP_NAME/g" \
    -e "s/@VERSION@/$VERSION/g" \
    -e "s/@BUNDLE_ID@/$BUNDLE_ID/g" \
    packaging/macos/Info.plist.in > "$APP_BUNDLE/Contents/Info.plist"

# 4. Ad-hoc or Developer ID Codesign
if [ -n "${MACOS_SIGNING_IDENTITY:-}" ]; then
    codesign --force --options runtime --sign "$MACOS_SIGNING_IDENTITY" \
        --entitlements packaging/macos/entitlements.plist "$APP_BUNDLE"
else
    codesign --force --deep --sign - "$APP_BUNDLE"
fi
```

### Packaging DMG (`package.sh`)
```bash
hdiutil create -volname "$APP_NAME" \
    -srcfolder "$APP_BUNDLE" \
    -ov -format UDZO \
    "dist/$APP_NAME-$VERSION-$TARGET.dmg"
```

---

## 3. Windows Packaging Pipeline

Windows applications require embedded icons, version metadata in the executable PE header, and a professional Inno Setup installer.

### Embedding Windows Resources (`build.rs`)
In `crates/<app>-desktop/build.rs`:
```rust
fn main() {
    if std::env::var("CARGO_CFG_TARGET_OS").unwrap_or_default() == "windows" {
        let mut res = winres::WindowsResource::new();
        res.set_icon("assets/icons/app.ico");
        res.set("ProductName", "MyApp");
        res.set("FileDescription", "MyApp Desktop Application");
        res.set("LegalCopyright", "Copyright (c) 2026 Alfin Syahruddin");
        if let Err(e) = res.compile() {
            eprintln!("Failed to compile Windows resources: {e}");
        }
    }
}
```

### Inno Setup Script (`packaging/windows/installer.iss`)
```iss
[Setup]
AppName=MyApp
AppVersion=0.1.0
DefaultDirName={autopf}\MyApp
DefaultGroupName=MyApp
OutputBaseFilename=MyApp-0.1.0-windows-setup
Compression=lzma2/max
SolidCompression=yes
SetupIconFile=..\..\assets\icons\app.ico
ArchitecturesInstallIn64BitMode=x64compatible

[Files]
Source: "..\..\target\release\myapp-desktop.exe"; DestDir: "{app}"; DestName: "MyApp.exe"; Flags: ignoreversion
Source: "..\..\assets\*"; DestDir: "{app}\assets"; Flags: ignoreversion recursesubdirs createallsubdirs

[Icons]
Name: "{group}\MyApp"; Filename: "{app}\MyApp.exe"
Name: "{autodesktop}\MyApp"; Filename: "{app}\MyApp.exe"

[Run]
Filename: "{app}\MyApp.exe"; Description: "{cm:LaunchProgram,MyApp}"; Flags: nowait postinstall skipifsilent
```

---

## 4. Linux Packaging Pipeline

Linux distribution packages standard `.deb` bundles complying with XDG Base Directory and Freedesktop standards.

### Freedesktop Desktop Entry (`packaging/linux/myapp.desktop`)
```ini
[Desktop Entry]
Name=MyApp
Comment=Native Desktop Application
Exec=myapp
Icon=myapp
Terminal=false
Type=Application
Categories=Utility;Finance;
StartupNotify=true
```

### Debian Package Assembly (`packaging/linux/package.sh`)
```bash
#!/bin/bash
set -euo pipefail

APP_NAME="myapp"
VERSION="0.1.0"
ARCH="amd64"
PKG_DIR="dist/${APP_NAME}_${VERSION}_${ARCH}"

rm -rf "$PKG_DIR"
mkdir -p "$PKG_DIR/DEBIAN"
mkdir -p "$PKG_DIR/usr/bin"
mkdir -p "$PKG_DIR/usr/share/applications"
mkdir -p "$PKG_DIR/usr/share/icons/hicolor/256x256/apps"

# 1. Copy binary and desktop spec
cp "target/release/$APP_NAME-desktop" "$PKG_DIR/usr/bin/$APP_NAME"
chmod 755 "$PKG_DIR/usr/bin/$APP_NAME"
cp "packaging/linux/$APP_NAME.desktop" "$PKG_DIR/usr/share/applications/"
cp "assets/icons/app-256.png" "$PKG_DIR/usr/share/icons/hicolor/256x256/apps/$APP_NAME.png"

# 2. Generate Debian Control File
cat <<EOF > "$PKG_DIR/DEBIAN/control"
Package: $APP_NAME
Version: $VERSION
Section: utils
Priority: optional
Architecture: $ARCH
Depends: libfontconfig1, libxkbcommon0, libssl3
Maintainer: Alfin Syahruddin <alfin@example.com>
Description: High performance native desktop application.
EOF

# 3. Build .deb package
dpkg-deb --build --root-owner-group "$PKG_DIR"
```

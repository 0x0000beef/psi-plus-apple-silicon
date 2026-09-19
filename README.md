
## Native macOS build on Apple Silicon (arm64): Build guide, plugins packaging, and [BONUS] macOS Now Playing integration

> **Disclaimer:** Based on official documentation, upgraded and polished with assistance from Google Gemini. Provided **as-is**: may contain bugs, rough edges, and plot holes — I'm just a hobbyist, not a professional programmer.


Official macOS builds for Psi / Psi+ haven't been updated in a while and were built strictly for the legacy `x86_64` architecture. Modern Macs running on Apple Silicon (M1/M2/M3/M4) either execute them through Rosetta 2 translation or fail to launch entirely since 27 release.

I have successfully compiled, packaged, and verified a native **`arm64` (Apple Silicon)** application bundle of **Psi+** on modern macOS (tested on Mac mini M4 and macOS 26.6.1). The client runs smoothly, plugins (only used by me, not all of them) load correctly, translations function as expected, and system-wide macOS media playback syncs seamlessly via XMPP PEP User Tune.

Below is the complete walkthrough to reproduce the build from source, package the application bundle, and configure the Now Playing bridge.

---

## 1. Precompiled Binary Download

* **Download link:** [Psi+ v1.5.2182 (Apple Silicon DMG)](https://github.com/0x0000beef/psi-plus-apple-silicon/releases/tag/v1.5.2182)
* **Architecture:** `Mach-O 64-bit arm64-apple-darwin` (Native Apple Silicon)
* **Base repository:** `psi-plus-snapshots (master)`

> **Note for macOS users:** If Gatekeeper restricts the app on first launch, open it once via right-click -> "Open", or strip the quarantine flag using Terminal:
> ```bash
> xattr -cr /Applications/Psi+.app
> ```

---

## 2. Building from Source

### Prerequisites
Install native ARM64 build tools and libraries via Homebrew (`/opt/homebrew`). Qt 5 is used because Qt 6 requires significant codebase refactoring (`QtMacExtras`, `QRegExp`, `QTextCodec`, etc.):

```bash
xcode-select --install
brew install cmake ninja pkg-config qt@5 openssl@3 libidn qca

```

### Clone Repositories & Fetch Translations

In `psi-plus-snapshots`, localization files reside in the standalone `psi-plus-l10n` repository. Clone the source tree and pull the translations directly into the root folder before configuring:

```bash
# Clone the main snapshot repository
git clone --recursive [https://github.com/psi-plus/psi-plus-snapshots.git](https://github.com/psi-plus/psi-plus-snapshots.git)
cd psi-plus-snapshots

# Pull official translations from the dedicated l10n repo
git clone --depth=1 [https://github.com/psi-plus/psi-plus-l10n.git](https://github.com/psi-plus/psi-plus-l10n.git) /tmp/psi-l10n
cp -r /tmp/psi-l10n/translations ./translations
rm -rf /tmp/psi-l10n

```

### CMake Configuration

Direct CMake to Homebrew's keg-only Qt 5 prefix, enable plugins, and disable obsolete frameworks (`Growl`, `Sparkle`) and heavyweight `QtWebEngine`:

```bash
mkdir build && cd build

cmake .. -G Ninja \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_OSX_ARCHITECTURES="arm64" \
  -DCMAKE_PREFIX_PATH="/opt/homebrew/opt/qt@5;/opt/homebrew/opt/openssl@3;/opt/homebrew/opt/qca" \
  -DENABLE_PLUGINS=ON \
  -DENABLE_GROWL=OFF \
  -DENABLE_SPARKLE=OFF \
  -DUSE_WEBENGINE=OFF

```

### Compilation

```bash
ninja

```

---

## 3. Packaging, Rpath Relinking & Code Signing

By default, CMake and `macdeployqt` leave absolute dylib references pointing directly to `/opt/homebrew`. If distributed as-is, the application will crash on machines lacking Homebrew with dyld errors (`Library not loaded: /opt/homebrew/...`).

To ensure the bundle is completely standalone and portable, follow these bundling and relinking steps:

1. **Deploy Qt frameworks:**
```bash
/opt/homebrew/opt/qt@5/bin/macdeployqt psi/Psi+.app
```

2. **Copy plugins and compiled translations:**
```bash
mkdir -p "psi/Psi+.app/Contents/Resources/plugins"
find psi/plugins -name "*.dylib" -exec cp -v {} "psi/Psi+.app/Contents/Resources/plugins/" \;

mkdir -p "psi/Psi+.app/Contents/Resources/translations"
cp -v psi/translations/*.qm "psi/Psi+.app/Contents/Resources/translations/"

```


3. **Copy Homebrew runtime dependencies into Frameworks:**
Copy the required non-Qt dylibs (`openssl@3`, `qca`, `libidn`, etc.) straight into `Contents/Frameworks/`:
```bash
mkdir -p "psi/Psi+.app/Contents/Frameworks"

cp -v /opt/homebrew/opt/openssl@3/lib/libcrypto.3.dylib "psi/Psi+.app/Contents/Frameworks/"
cp -v /opt/homebrew/opt/openssl@3/lib/libssl.3.dylib "psi/Psi+.app/Contents/Frameworks/"
cp -v /opt/homebrew/opt/libidn/lib/libidn.12.dylib "psi/Psi+.app/Contents/Frameworks/"
cp -v /opt/homebrew/opt/qca/lib/libqca-qt5.2.dylib "psi/Psi+.app/Contents/Frameworks/"

```


4. **Relink dependencies (Fix Homebrew hardcoded paths):**
Point the main binary and bundled libraries to `@executable_path/../Frameworks` using `install_name_tool`:
```bash
MAIN_BIN="psi/Psi+.app/Contents/MacOS/psi"

# Fix paths in the main executable
otool -L "$MAIN_BIN" | grep "/opt/homebrew" | awk '{print $1}' | while read -r dylib; do
    filename=$(basename "$dylib")
    install_name_tool -change "$dylib" "@executable_path/../Frameworks/$filename" "$MAIN_BIN"
done

# Fix cross-references inside the Frameworks directory
for dylib in psi/Psi+.app/Contents/Frameworks/*.dylib; do
    otool -L "$dylib" | grep "/opt/homebrew" | awk '{print $1}' | while read -r dep; do
        dep_name=$(basename "$dep")
        install_name_tool -change "$dep" "@loader_path/$dep_name" "$dylib"
    done
    install_name_tool -id "@loader_path/$(basename "$dylib")" "$dylib"
done

# Fix plugin dependencies
for plugin in psi/Psi+.app/Contents/Resources/plugins/*.dylib; do
    otool -L "$plugin" | grep "/opt/homebrew" | awk '{print $1}' | while read -r dep; do
        dep_name=$(basename "$dep")
        install_name_tool -change "$dep" "@loader_path/../../Frameworks/$dep_name" "$plugin"
    done
done

```


5. **Verify that zero Homebrew paths remain:**
```bash
otool -L "psi/Psi+.app/Contents/MacOS/psi" | grep "/opt/homebrew"
# Should return nothing!

```


6. **Re-sign the entire bundle (Ad-Hoc):**
```bash
codesign --force --deep --sign - "psi/Psi+.app"
```








---

## 4. [BONUS] macOS System-Wide Now Playing Integration (PEP User Tune)

Legacy AppleScript and iTunes hooks no longer function with modern third-party media players (Cog, IINA, Safari, Spotify, Music.app, etc.).

Psi+ ships with an integrated **`FileTuneController`** that tracks local file modifications via `QCA::FileWatch`. On macOS, this file is located at `~/Library/Caches/Psi+/tune`.

### File Format Specification

The file must be encoded in UTF-8 and contain strictly 5 lines:

* **Line 1:** Track Title (`title`)
* **Line 2:** Artist (`artist`)
* **Line 3:** Album (`album`)
* **Line 4:** Track number (optional, can be empty)
* **Line 5:** Duration in integer seconds (`duration`)

Deleting the file automatically clears the tune status in the roster.

### Setup Instructions

1. Install `nowplaying-cli` to query the native macOS MediaRemote framework:
```bash
brew install nowplaying-cli

```


2. Create the bridge script `~/bin/psi-tune.sh`:
```bash
#!/usr/bin/env bash

CACHE_DIR="$HOME/Library/Caches/Psi+"
TUNE_FILE="$CACHE_DIR/tune"
LAST_TRACK=""

mkdir -p "$CACHE_DIR"

while true; do
    RATE=$(nowplaying-cli get playbackRate 2>/dev/null)

    if [ "$RATE" = "1" ]; then
        TITLE=$(nowplaying-cli get title 2>/dev/null)
        ARTIST=$(nowplaying-cli get artist 2>/dev/null)
        ALBUM=$(nowplaying-cli get album 2>/dev/null)
        DURATION=$(nowplaying-cli get duration 2>/dev/null | cut -d. -f1)

        [ -z "$DURATION" ] && DURATION=0
        CURRENT_TRACK="${TITLE}///${ARTIST}///${ALBUM}"

        if [ -n "$TITLE" ] && [ "$CURRENT_TRACK" != "$LAST_TRACK" ]; then
            printf "%s\n%s\n%s\n%s\n%s\n" \
                "$TITLE" "$ARTIST" "$ALBUM" "" "$DURATION" > "${TUNE_FILE}.tmp"
            mv "${TUNE_FILE}.tmp" "$TUNE_FILE"
            LAST_TRACK="$CURRENT_TRACK"
        fi
    else
        if [ -n "$LAST_TRACK" ]; then
            rm -f "$TUNE_FILE"
            LAST_TRACK=""
        fi
    fi
    sleep 2
done

```


3. Make the script executable:
```bash
chmod +x ~/bin/psi-tune.sh

```


4. *(Optional)* Run the script automatically in the background via LaunchAgent (`~/Library/LaunchAgents/com.user.psitune.plist`):
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "[http://www.apple.com/DTDs/PropertyList-1.0.dtd](http://www.apple.com/DTDs/PropertyList-1.0.dtd)">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.user.psitune</string>
    <key>ProgramArguments</key>
    <array>
        <string>/bin/bash</string>
        <string>-c</string>
        <string>$HOME/bin/psi-tune.sh</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>EnvironmentVariables</key>
    <dict>
        <key>PATH</key>
        <string>/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin</string>
    </dict>
</dict>
</plist>

```


Load the agent:
```bash
launchctl load ~/Library/LaunchAgents/com.user.psitune.plist

```


5. In Psi+ (**Options -> PEP -> Tune**), enable **Psi File controller** and turn on Tune publishing.

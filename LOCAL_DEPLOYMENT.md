# Ghostty Local Deployment Guide

This guide covers building, installing, and configuring Ghostty locally on Fedora Linux.

## 1. Prerequisites

Ghostty requires Zig version 0.15.2 and several system development libraries.

### Installing Zig via ZVM
It's highly recommended to manage Zig versions using Zig Version Manager (zvm) to avoid clashing with your system's default version:

```bash
zvm install 0.15.2
zvm use 0.15.2
```

### Installing C Dependencies
Install the required GTK4 and shell dependencies:

```bash
sudo dnf install gtk4-devel libadwaita-devel gtk4-layer-shell-devel blueprint-compiler
```

## 2. Building and Installing

Ghostty can be built in release mode and installed directly to your local user directory. This ensures the binary, themes, terminfo, and desktop icons are available without requiring root permissions.

From the root of the Ghostty repository, run:

```bash
zig build -p ~/.local -Doptimize=ReleaseFast
```

The compiled executable will be located at `~/.local/bin/ghostty`.

## 3. Desktop Environment Integration

If you have previously installed Ghostty globally via your package manager (like DNF COPR or Flatpak), ensure it is removed to prevent conflicts:

```bash
sudo dnf remove ghostty
```

### Fixing Application Launcher Issues (KDE / GNOME)

Sometimes, the local `.desktop` file installed by the build process tries to use D-Bus activation. This can fail with a *"The name is not activatable"* error if the local D-Bus daemon hasn't synced the newly installed `com.mitchellh.ghostty.service` user service.

To fix this reliably for local builds:

1. Open `~/.local/share/applications/com.mitchellh.ghostty.desktop` in a text editor.
2. Change the line `DBusActivatable=true` to `DBusActivatable=false`.
3. Force a refresh of your desktop database and launcher cache:

```bash
update-desktop-database ~/.local/share/applications/ && kbuildsycoca6
```
*(Note: Use `kbuildsycoca5` instead if you are on an older KDE Plasma 5 setup).*

Ghostty will now be fully integrated into your `Alt + Space` desktop launcher and seamlessly inherit your `~/.config/ghostty/config` configurations!

### Restoring Native KDE Wayland Background Blur

Ghostty's `main` branch recently (March 2026) replaced the legacy KDE-specific blur protocol (`org.kde.kwin.blur`) with the generic `ext-background-effect-v1` protocol. However, KDE Plasma 6 does not currently support this generic protocol natively, meaning **background blur will not work out-of-the-box on native Wayland** when built from `main`.

To restore the background blur on KDE Wayland, you have two options:

**Option A: Force XWayland (Easiest)**
Force Ghostty to run under X11 by editing your `.desktop` file:
```ini
Exec=env GDK_BACKEND=x11 /home/scual/.local/bin/ghostty --gtk-single-instance=true
```
Under X11, Ghostty uses a different mechanism (`_KDE_NET_WM_BLUR_BEHIND_REGION`) which is fully supported by KDE.

**Option B: Revert the Wayland Protocol Change (Native Wayland)**
If you want the crisp rendering of native Wayland and still keep the blur, you must reverse the commit `9e2e99c55fb5f2c0709938eb590b62112f3d7446` which removed the `org.kde.kwin.blur` implementation:

```bash
# Generate a reverse patch for only the Wayland source files
git diff 9e2e99c55fb5f2c0709938eb590b62112f3d7446^..9e2e99c55fb5f2c0709938eb590b62112f3d7446 src/apprt/gtk/winproto/wayland.zig src/apprt/gtk/winproto/wayland/Globals.zig src/build/SharedDeps.zig > /tmp/kde-blur.patch

# Apply the patch in reverse
git apply -R --3way /tmp/kde-blur.patch
```
*(Note: You will encounter minor merge conflicts in `src/apprt/gtk/winproto/wayland.zig`. Manually resolve them by keeping the restored `blur_token` logic, then rebuild Ghostty).*

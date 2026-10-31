# MacDoom2016
With the help from Claude, I was able to create a working wrapper for Doom 2016 on Mac. 

The app will install steam
With steam installed, install Doom 2016
Then in preference apply: +r_fullscreen 0 +r_windowWidth 1920+r_windowHeight 1080

# DOOM (2016) on Apple Silicon: what it takes

DOOM (2016) is a Windows game on id Tech 6. On a Mac it runs through **Wine**, using its
**Vulkan** renderer, translated to Metal by **MoltenVK**. Status: boots to the main menu and
plays levels (tested on an M3 Pro, macOS 26).

```
DOOMx64vk.exe → Wine (win32u / winevulkan) → MoltenVK → Metal
```

CrossOver's DirectX translators (D3DMetal, DXMT) don't help here, because DOOM uses Vulkan, or
OpenGL 4.5, which macOS doesn't support beyond 4.1.

## 1. Wine changes (`win32u.so`, rebuilt from WineHQ 11.16)

| Patch | Why DOOM needs it |
|---|---|
| `wine-win32u-portability-enumeration.patch` | MoltenVK is a "portability" Vulkan driver. Without this, the Vulkan loader hides it and `vkCreateInstance` fails. |
| `wine-win32u-mask-unsupported-features.patch` | DOOM asks for `depthBounds` and `shaderCullDistance`, which Apple GPUs lack, and fails to create a device. With `WINE_VK_MASK_UNSUPPORTED_FEATURES=1`, Wine drops them from the request; the game doesn't need them. |
| `wine-win32u-foreign-dc-proxy.patch` + `wine-win32u-foreign-dc-full-copy.patch` | Steam's window stays black on macOS: Steam's browser draws in one process into a window owned by another, which Wine's Mac driver doesn't support. These forward that drawing (`WINE_FOREIGN_DC_PROXY=1`) and repaint the whole window when it's resized. |

Only `win32u.so` is replaced, in a copy of Gcenx's Wine 11.16 build.

## 2. Game settings

| Setting | Why |
|---|---|
| `r_renderAPI "1"` in `Saved Games/id Software/DOOM/base/DOOMConfig.local` | Steam's Play starts `DOOMx64.exe` (OpenGL). It switches to `DOOMx64vk.exe` (Vulkan) only when this local setting is set; the file isn't synced by Steam Cloud, so a new install doesn't have it. |
| Launch Options `+r_fullscreen 0 +r_windowWidth 1280 +r_windowHeight 720` | In exclusive full screen, Steam's window takes focus right after the game's window appears. id Tech then minimizes itself, the Vulkan surface size changes, and the renderer fails ("Too many frame buffers"). A window avoids this. |

## 3. Steam inside Wine

- **64-bit Steam client.** Valve's `SteamSetup.exe` installs a 32-bit bootstrapper whose first
  update hangs or crashes under Wine on Rosetta 2. Instead, the 64-bit client is installed straight
  from Valve's update servers: read the `steam_client_win64` manifest, download each package, check
  its SHA-256 and unzip it. The 64-bit client keeps itself updated after that.
- **Start Steam with `-cef-disable-gpu`**, and with `WINE_FOREIGN_DC_PROXY=1` (see the Wine
  patches above).
- **Game started from Steam.** DOOM exits if Steam isn't running, and needs Steam's app ID
  (`SteamAppId=379720`) when started outside Steam's Play button.

## 4. Controller

A DualSense over Bluetooth works as an Xbox-style controller after setting
`HKLM\System\CurrentControlSet\Services\WineBus\Devices\054c/0ce6` → `Hidraw = 0` (DWORD) in the
Wine prefix's registry.

## 5. Packaging as a Mac app (DOOM.app)

The app holds its own Wine, a Wine prefix with Steam and the game, and a launcher script.

- **First open** creates the Wine prefix, installs Steam from Valve (above), and applies the
  controller and `r_renderAPI` settings.
- **Opening the app** starts this app's Steam on DOOM's page.
- **Closing Steam's window quits Steam.** Windows Steam only hides itself and, after a game, its
  restarted web process reopens the window under Wine. A small helper watches for the main window
  being closed (not minimized) and shuts Steam down.
- **macOS privacy checks.** The app isn't signed, so macOS identifies it by reading the whole bundle
  (tens of GB with the game) the first time Wine triggers a privacy check. Steam stalls meanwhile.
  Wine is started as its own "responsible process", and rebuilds leave the app's `Info.plist` and
  `Contents/MacOS` untouched unless they change, so macOS keeps its cached identity.

## Known limits

- Exclusive full screen doesn't work (see the Launch Options above). Use a window.
- Two shaders log `in_TexCoord1 is missing from the vertex descriptor` (MoltenVK); no visible effect
  found so far.

## Source

The patches are in `patches/`, built with `scripts/build-wine.sh`. The app is built with
`scripts/make-wrapper.sh doom-2016` (`--distro` for a shareable copy without Steam or the game).

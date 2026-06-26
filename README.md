# ubuntu-quick-tile

A one-shot bulk tiler for the
[Tiling Shell](https://extensions.gnome.org/extension/7065/tiling-shell/) GNOME
extension. Press `Super+T` and every window on your monitor snaps into the zones
of your active layout.

Tiling Shell only tiles one window at a time. `tile-all` reads its active layout
and places every open window into the zones in one go. You design the layouts in
Tiling Shell; this applies the active one to everything on screen.

## Requires Tiling Shell

`tile-all` reads Tiling Shell's `gsettings` schema, so the extension has to be
installed and enabled. Get it from
https://extensions.gnome.org/extension/7065/tiling-shell/ and check:

```shell
gnome-extensions list --enabled | grep tilingshell
```

If that prints nothing, install and enable it first.

## How It Works

It reads the active layout from Tiling Shell's settings, works out each zone's
pixel position on your monitor, and moves every window into a zone with
`xdotool` and `wmctrl`.

It only touches windows on the target monitor, so other screens are left alone.
Tile a different one by name (`tile-all HDMI-1`).

## Installation

```shell
sudo apt install wmctrl xdotool
./install
```

This copies `tile-all` to `~/.local/bin/`, makes it executable, and binds
`Super+T` to it. Re-running is safe.

`~/.local/bin` has to be on your `PATH` for the bare `tile-all` command (it
is usually on Ubuntu). If you get "command not found", use the full path or the
`Super+T` hotkey, which ignores `PATH`.

## Usage

Set the layout you want active, then tile.

| Action                            | Shortcut                            |
| --------------------------------- | ----------------------------------- |
| Tile all windows                  | `Super+T`                           |
| Cycle Tiling Shell layouts        | `Ctrl+Shift+L`                      |
| Move focused window between zones | `Super+Left` / `Super+Right`        |
| Snap one window (drag overlay)    | hold the activation key, drag, drop |

From a terminal:

```shell
# primary monitor
tile-all

# a specific monitor by connector name
tile-all HDMI-1
```

It always uses the currently active layout, so switch with `Ctrl+Shift+L` first.

### Spawn Five Terminals and Tile Them

```shell
for i in $(seq 5); do tilix & done; sleep 1; tile-all
```

Each `tilix` opens its own window, then `tile-all` arranges them. Switch to a
five-zone layout first so each gets a zone. The `sleep 1` is there because
spawning and tiling are async — without it `tile-all` runs before the windows
exist and finds nothing.

## Limitations

### X11 Only

It uses `wmctrl` and `xdotool`, which need the X server. Wayland blocks one app
from moving another's windows, so it won't work there. Check with
`echo $XDG_SESSION_TYPE` (must be `x11`); pick "Ubuntu on Xorg" from the gear
menu at login if needed.

### More Windows Than Zones Overlap

Eight windows into five zones wraps around — windows six through eight stack
onto the first three. Use a layout with more tiles for one window per zone.

### Active Layout Only

No per-zone targeting and no tiling into a layout you're not on. Select the
layout, then tile.

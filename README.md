# ubuntu-quick-tile

An auto-tile helper for Ubuntu running the **Tiling Shell** GNOME extension. It
adds the one thing Tiling Shell deliberately leaves out: a single command, bound
to `Super+T`, that grabs every window on your monitor and snaps them all into
the zones of your currently-active layout at once.

## Introduction

Tiling Shell is a manual tiler. You move windows into zones one at a time, with
the mouse or with `Super`+arrow keys. That is great for precision but tedious
when you have a screen full of windows and just want them laid out _now_. This
project fills that gap with a single script, `tile-all`, that reads your active
Tiling Shell layout and distributes all open windows across its zones in one go.

It is intentionally thin. It does not replace Tiling Shell, it leans on it. The
layouts you design in Tiling Shell's preferences are the source of truth and
this script just bulk-applies the active one.

## Dependency: Tiling Shell

This tool does nothing on its own. It requires the **Tiling Shell** GNOME Shell
extension to be installed and enabled, because it reads that extension's
`gsettings` schema to discover your layouts and which one is active.

Install it from the official GNOME Extensions page:

https://extensions.gnome.org/extension/7065/tiling-shell/

The extension id is `tilingshell@ferrarodomenico.com`. You can confirm it is
enabled with:

```shell
gnome-extensions list --enabled | grep tilingshell
```

If that prints nothing, install and enable the extension first. The script reads
its schema from
`~/.local/share/gnome-shell/extensions/tilingshell@ferrarodomenico.com/schemas`
and refuses to run if it is not present.

### Why it depends on Tiling Shell specifically

The script does not invent its own layouts. It reads two keys from Tiling
Shell's settings: `selected-layouts`, which records the active layout per
monitor, and `layouts-json`, the full set of layouts where each is a list of
fractional tile rectangles. It then converts the active layout's fractional
rectangles into real pixel coordinates for your monitor and applies them. That
tight coupling is the whole point. You design layouts visually in Tiling Shell
and this just bulk-applies the one you have selected.

## How it works

The flow is straightforward once you know the moving parts.

**Read the active layout**

Tiling Shell stores each layout as a set of tiles, where every tile is a
fractional rectangle (`x`, `y`, `width`, `height`) in the range 0 to 1. A
three-column "thirds" layout, for example, is three tiles each a third wide and
full height. The script reads `selected-layouts` to find which layout is active,
then pulls that layout's tiles from `layouts-json`.

**Resolve the monitor rectangle**

It asks `xrandr --listmonitors` for the target monitor's pixel rectangle and
parses only the single geometry token (`W/mmxH/mm+X+Y`) rather than scraping the
whole line. By default it picks the primary monitor, the one xrandr marks with
`*`. It then subtracts the GNOME top panel height, read from `wmctrl`'s reported
work area, so the top row of tiles starts below the panel rather than under it.

**Place the windows**

It lists every window on the current workspace via `wmctrl -lG`, keeps only
those whose centre point falls inside the target monitor, and walks through them
assigning each to the next tile. For every window it first removes the maximized
state, because the window manager ignores move and resize requests on a
maximized window, then uses `xdotool` to move and size it to the computed pixel
rectangle.

## Multi-monitor behaviour

This is the part that needed the most care, so it is worth being explicit. The
window manager addresses all monitors with a single global coordinate space. A
naive tiler that computes positions against the full desktop will happily place
a "full height" window so that it spans from your top monitor down into a second
monitor below it.

`tile-all` avoids that by confining all of its maths to one monitor's rectangle
and by only touching windows whose centre already sits on that monitor. Windows
on your other screens are left exactly where they are. To tile a different
monitor, pass its connector name as an argument (`tile-all HDMI-1`); run it once
per monitor if you want to tile several.

## Why a single Python script

The runtime work, reading gsettings, parsing monitors, computing pixel
rectangles and driving `wmctrl`/`xdotool`, all lives in one self-contained
Python file. An earlier version was a bash script that shelled out to Python
repeatedly, which meant fragile text passing across every boundary and a fresh
interpreter spawned for even simple arithmetic. Consolidating the logic into one
Python file removes those seams.

Python is not an extra dependency here. It ships with every Ubuntu install, the
same as bash, and the script uses only the standard library plus the same system
binaries the task needs anyway. There is nothing to isolate, so there is no need
for a virtualenv or a packaging tool like pipx. The script is just a file with a
shebang that goes on your `PATH`. Bash is kept only for the installer, where its
single job, wiring up a GNOME keybinding, is genuinely bash-friendly.

## Installation

Install the runtime dependencies, then run the installer:

```shell
sudo apt install wmctrl xdotool
./install
```

The installer copies `tile-all` to `~/.local/bin/tile-all`, makes it executable,
and registers a GNOME custom keyboard shortcut binding `Super+T` to it. It is
safe to re-run; it overwrites the previous copy and re-points the shortcut.

For `tile-all` to work as a bare command in your terminal, `~/.local/bin` needs
to be on your `PATH`. On Ubuntu it usually is by default. If `tile-all` reports
"command not found", call it by full path (`~/.local/bin/tile-all`) or use the
`Super+T` hotkey, which does not depend on `PATH`.

## Usage

Once installed, the everyday flow is two keystrokes. First make sure the layout
you want is the active one, then tile.

| Action                            | Shortcut                            |
| --------------------------------- | ----------------------------------- |
| Tile all windows (this tool)      | `Super+T`                           |
| Cycle Tiling Shell layouts        | `Ctrl+Shift+L`                      |
| Move focused window between zones | `Super+Left` / `Super+Right`        |
| Snap one window (drag overlay)    | hold the activation key, drag, drop |

From a terminal you can also run it directly:

```shell
# Tile the primary monitor
tile-all

# Tile a specific monitor by connector name
tile-all HDMI-1
```

The script tiles into whatever Tiling Shell layout is **currently active**, so
cycle to the layout you want with `Ctrl+Shift+L` before pressing `Super+T`.

### Example: spawn five terminals and tile them

A handy one-liner for spinning up a grid of terminals. Each `tilix` call opens
its own window, so five calls give five windows, then `tile-all` lays them out.
Switch to a five-zone layout first (`Ctrl+Shift+L`) so each terminal lands in
its own zone:

```shell
for i in $(seq 5); do tilix & done; sleep 1; tile-all
```

The `sleep 1` matters. Spawning a window and tiling are asynchronous, so without
a brief pause `tile-all` runs before the windows have been mapped and finds
nothing to place. One second is usually enough; bump it up if your machine is
slow to open the terminals.

## Limitations

A few honest caveats so nothing surprises you.

**X11 only**

It relies on `wmctrl` and `xdotool`, which talk to the X server. It will not
work on a Wayland session, because Wayland deliberately forbids one application
from moving another application's windows. Check your session with
`echo $XDG_SESSION_TYPE`; it must say `x11`. On the GNOME login screen you can
pick "Ubuntu on Xorg" from the gear menu.

**More windows than zones means overlap**

If you have eight windows and a five-zone layout, the script wraps around: the
sixth, seventh and eighth windows stack on top of the first three in the same
zones. That overlap is unavoidable without more zones. Switch to a layout with
more tiles if you want one window per zone.

**It tiles the active layout only**

There is no per-zone targeting and no way to spread into a layout you are not
currently using. Select the layout first, then tile. This keeps the tool
predictable and in step with what Tiling Shell shows you on screen.

## Conclusion

`ubuntu-quick-tile` is a small bridge over a deliberate gap in Tiling Shell.
Manual tilers do not bulk-arrange, and sometimes you just want every window
placed at once. By reading Tiling Shell's own layout definitions it stays in
sync with the layouts you have already designed, while adding monitor-aware bulk
placement and a single hotkey to trigger it. Design your layouts in Tiling
Shell, press `Super+T`, done.

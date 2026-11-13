# native-hidpi

Find out whether your monitor is being fed the wrong resolution, and fix it.

On GNOME + Wayland. One file, no dependencies beyond `python3-gi`.

## The problem this looks for

"Ubuntu fonts look blurry", "everything looks a bit soft", "why doesn't Linux look
like a Mac" is very often **not** a font problem. A depressingly common setup is a 4K
monitor driven at 2560x1440, because that was the mode someone picked years ago, or
because a docking station negotiated it once and it stuck. The panel then upscales a
1440p image by 1.5x in its own scaler, so *every pixel on screen* is resampled before
you see it.

Nothing tells you this. The resolution list doesn't warn you, and no amount of
fontconfig, hinting, antialiasing, `text-scaling-factor`, or Infinality-style tuning
will recover detail that was never rendered.

```
$ native-hidpi status

DP-1  scale 1.0  primary=True
  active : 2560x1440@59.951
  native : 3840x2160@59.997
  scales supported on the active mode: 1.0, 1.25, 1.5, 2.0

WARNING  DP-1: 2560x1440 signal on a 3840x2160 panel, monitor upscales x1.50.
Everything on screen is soft and no font setting will fix it.
```

## Fixing it

```
$ native-hidpi fix
verified 3840x2160@59.997 scale 2.0 on DP-1
applied 3840x2160@59.997 scale 2.0 on DP-1 (persistent)
wrote ~/.config/fontconfig/fonts.conf and set antialiasing=grayscale, hinting=slight
```

`fix` does two things:

1. Switches the output to its **native mode** (from the EDID `is-preferred` entry) with
   an **integer framebuffer scale**, via `org.gnome.Mutter.DisplayConfig`. This is
   verified (`METHOD_VERIFY`) before it is applied.
2. Sets a sane antialiasing profile: GSettings for GNOME/GTK/Qt/snap-Firefox, plus
   `~/.config/fontconfig/fonts.conf` for Chrome, Blender, Java and everything else that
   reads fontconfig instead.

The previous mode, scale, GSettings values and `fonts.conf` are recorded in
`~/.local/state/native-hidpi/state.json`, so `native-hidpi revert` puts everything back.

Browsers read fontconfig once at startup, so restart them.

## Scale: why 2x and not 1.5x

Density is `mode / scale`. Only an **integer** scale maps one logical pixel onto one
physical pixel.

| mode | scale | logical space | pixel-exact |
|---|---|---|---|
| 2560x1440 on a 4K panel | 1x | 2560x1440 | no - monitor scaler, this is the fog |
| 3840x2160 | 2x | 1920x1080 | **yes** |
| 3840x2160 | 1.5x | 2560x1440 | no - resampled in the GPU |

Fractional scale has a second cost that is easy to miss: Mutter keeps `Xft.dpi` at
`96 * next integer` (192 at 1.5x), so XWayland and Electron windows render at 2x and get
resampled down, and apps that ignore fractional scaling end up a different size from
their neighbours. `status` prints a note when it sees this.

If 2x makes everything too big, shrink the contents rather than the pixel grid:

```
gsettings set org.gnome.desktop.interface font-name "Ubuntu Sans 11"
gsettings set org.gnome.shell.extensions.dash-to-dock dash-max-icon-size 32
```

## Antialiasing profiles

| profile | fontconfig | GSettings | notes |
|---|---|---|---|
| `mac` (default) | `rgba=none`, `hintslight` | grayscale, slight | what macOS does; correct at 2x |
| `sharp` | `rgba=none`, `hintfull` | grayscale, full | maximum edge definition at 1x |
| `subpixel` | `rgba=rgb`, `hintslight` | rgba, slight | only worth trying on a sub-110 PPI 1x panel |

`native-hidpi fix --profile sharp`, etc.

## Other flags

```
--scale 2                 force a framebuffer scale
--want-logical-width 1920 pick the largest integer scale that still fits this width
--no-fonts                display config only
--dry-run                 print what would change
selftest                  exercise the detection logic, no display changes
```

## Requirements and limits

- GNOME on Wayland. Needs `org.gnome.Mutter.DisplayConfig`; it exits with a clear
  message on X11 or other desktops.
- Multi-monitor: `fix` changes one mismatched output per run. Run it again for the next.
- Applied configuration is persistent (`METHOD_PERSIST`), so it survives reboot.
- No root needed.
- Written and tested on Ubuntu 26.04, GNOME Shell / Mutter 50.1, single Dell U2720Q,
  with Google Chrome 157 and snap Firefox. Other vendors' EDID quirks are untested; if
  `is-preferred` is missing, `status` reports `native : unknown` and `fix` does nothing.

## Why this is a script and not a patch to Ubuntu

The mode is per-machine state, and Ubuntu's `font-antialiasing=rgba` /
`font-hinting=none` pair is GNOME's deliberate `automatic` policy for low-DPI displays,
so neither is a bug anyone can merge. The gap that *is* worth closing is that the
Displays panel will happily let you select a resolution far below the panel's native one
with no warning - see [`UPSTREAM-ISSUE.md`](UPSTREAM-ISSUE.md) for that request.

## License

MIT

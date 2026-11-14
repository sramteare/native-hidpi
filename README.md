# native-hidpi

Find out whether your monitor is being fed the wrong resolution, and fix it
**permanently**.

On GNOME (Mutter) with Wayland. One file; needs `python3-gi`, plus `gsettings` and
`fc-match` for the font half.

## The problem this looks for

"Ubuntu fonts look blurry", "everything looks a bit soft", "why doesn't Linux look like a
Mac" is very often **not** a font problem. A common setup is a 4K monitor driven at
2560x1440, because that mode was picked once, or a docking station negotiated it and it
stuck. The panel then upscales a 1440p image by 1.5x in its own scaler, so *every pixel on
screen* is resampled before you see it.

Nothing warns you. And no amount of fontconfig, hinting, `text-scaling-factor`, RGB order
or Infinality-style tuning recovers detail that was never rasterized.

```
$ native-hidpi status

DP-1  scale 1.0  primary=True
  active : 2560x1440@59.951
  native : 3840x2160@59.997
  scales supported on the active mode: 1.0, 1.25, 1.5, 2.0

WARNING  DP-1: 2560x1440 signal on a 3840x2160 panel, monitor upscales x1.50.
Everything on screen is soft and no font setting will fix it.
```

## Why your change keeps disappearing

This is the part that costs people hours. Applying a mode over DBus does not reliably
rewrite `~/.config/monitors.xml`, and that file is what Mutter restores after a **hotplug
or a login**. Observed on GNOME Shell 50.1:

```
$ native-hidpi fix            # 1.5 -> 2.0, confirmed by reading Mutter's state back
$ stat -c %y ~/.config/monitors.xml     # unchanged: still the old timestamp
... unplug and replug the monitor ...
$ native-hidpi status         # scale 1.5 again
```

So a live change looks applied, and then quietly isn't. `fix` therefore writes
`monitors.xml` itself, matching the stored block by connector **and serial** - connector
names are reused between monitors, and a real config file on the author's machine has two
different panels both called `DP-1`. Blocks for other monitors or other layouts are left
alone, and the original file is backed up.

A `<configuration>` block is the *whole* set of connected monitors (one `<logicalmonitor>`
each, or several `<monitor>` elements in one for a mirrored pair), so a block is only a
candidate when its full connector+serial set is what is running: with two panels attached
`fix` updates the two-monitor block and leaves the one for the single-monitor layout as it
was, not the other way round.

`status` reports the mismatch when it appears:

```
NOTE     DP-1: running 3840x2160 scale 2.0 but ~/.config/monitors.xml stores
         3840x2160 scale 1.5 - the next hotplug or login will switch it back
```

## Fixing it

```
$ native-hidpi fix
planned DP-1: 2560x1440@59.951 scale 1.0 -> 3840x2160@59.997 scale 2.0
(untouched outputs: 0)
applied DP-1: 3840x2160@59.997 scale 2.0 (read back and confirmed)
persisted to ~/.config/monitors.xml (backup at ~/.config/monitors.xml.native-hidpi.bak)
wrote ~/.config/fontconfig/fonts.conf and set antialiasing=grayscale, hinting=slight
```

1. Applies native mode + integer framebuffer scale over `org.gnome.Mutter.DisplayConfig`,
   sending **every** connected output in one configuration - the array is the complete
   desired state, so anything omitted gets disabled.
2. Reads the result back and, if it did not take, re-applies the previous configuration.
   Mutter documents the `method` flags inconsistently between versions, so this tool does
   not rely on a "verify" flag: it applies once and checks the outcome.
3. Persists to `monitors.xml` - including the case where nothing live needed changing but
   the stored file disagrees with what is running, which is what `status` warns about.
4. Installs an antialiasing profile in GSettings *and* `~/.config/fontconfig/fonts.conf`.

Browsers read fontconfig once at startup, so restart them.

## Scale: why 2x and not 1.5x

Density is `mode / scale`. Only an **integer** scale maps one logical pixel onto one
physical pixel.

| mode | scale | logical space | pixel-exact |
|---|---|---|---|
| 2560x1440 on a 4K panel | 1x | 2560x1440 | no - monitor scaler, this is the fog |
| 3840x2160 | 2x | 1920x1080 | **yes** |
| 3840x2160 | 1.5x | 2560x1440 | no - resampled in the GPU |

At a fractional monitor scale, Mutter reports `Xft.dpi` as `96 * next integer` (192 at
1.5x), so clients that do not implement `fractional-scale-v1` render at 2x and get
resampled down, and apps that ignore scaling end up a different size from their
neighbours. Measured on a single scaled monitor; with several outputs `Xft.dpi` is global,
so read it with care.

Fractional scales are float32 values, not decimal ones: between 1.5 and 2.0 on a 4K panel
Mutter offers `1.6666666269302368` (5/3), because 3840/1.75 is not an integer. So `--scale`
snaps to the closest scale the mode actually offers (`--scale 1.67` → 5/3, `--scale 1.33`
→ 4/3). The same value goes into `monitors.xml`: writing `1.66667` there is a *different*
scale, and `status` would report drift that is not real.

## Want 1.8x? Keep the pixel grid, scale the text

An in-between request is not a framebuffer step at all, so `fix` builds it out of the two
things that are: the **pixel-exact integer framebuffer scale**, and
`text-scaling-factor` to bring the text back down to the size you asked for.

```
$ native-hidpi fix --scale 1.8
planned DP-1: 3840x2160@59.997 scale 1.6666666269302368 -> 3840x2160@59.997 scale 2.0
note: scale 1.8 is not a framebuffer step this mode offers; using pixel-exact scale 2
      + text-scaling-factor 0.9 (effective 1.8x)
```

| request | framebuffer | text-scaling-factor | what happens |
|---|---|---|---|
| `--scale 2` | 2x | 1.0 | one logical pixel = four device pixels, exact |
| `--scale 1.8` | 2x | 0.9 | exact grid, text 10% smaller |
| `--scale 1.75` | 2x | 0.875 | exact grid, text 12.5% smaller |
| `--scale 1.67` | 5/3 | 1.0 | resampled in the GPU - the soft option |

`status` prints both halves:

```
UI font  'Ubuntu Sans 11'  -> 26.4 device px per em at scale 2.0
         x text-scaling-factor 0.9 (effective 1.8x)
```

Set the halves yourself with `native-hidpi fix --scale 2 --text-scale 0.9`, or only the text
with `--text-scale`. The honest caveat: the layout grid stays at the integer scale (1920x1080
logical at 2x on a 4K panel) and `text-scaling-factor` only resizes **text** - icons and
widgets sized in pixels keep their 2x size. It is still the better trade, because nothing is
interpolated. A request no integer scale in the mode can carry (`--scale 5`, or `--scale
1.75` on a panel whose steps stop at 5/3) is refused rather than quietly rounded, and
`revert` puts the old `text-scaling-factor` back.

Chrome is the client people notice. The Chrome 157 build tested here binds `wp_viewport`
and contains **no** `zwp_fractional_scale` symbols, i.e. it does not appear to implement
`fractional-scale-v1`, so fractional scaling looked soft in Chrome even though Chrome was
running as a native Wayland client (it held a wayland socket, no X11). Check your own
build - presence of the symbols is a compile-time indication, not proof of runtime
negotiation:

```bash
strings -a /opt/google/chrome/chrome | grep -oE "zwp_fractional_scale[a-z_]*|wp_viewport" | sort -u
```

If 2x makes everything too big, shrink the contents rather than the pixel grid and keep
the crispness:

```bash
gsettings set org.gnome.desktop.interface font-name "Ubuntu Sans 11"
gsettings set org.gnome.shell.extensions.dash-to-dock dash-max-icon-size 32
```

## Antialiasing profiles

| profile | fontconfig | GSettings | notes |
|---|---|---|---|
| `mac` (default) | `rgba=none`, `hintslight` | grayscale, slight | approximates macOS text: grayscale AA with light hinting (macOS adds its own stem darkening, which FreeType/fontconfig here do not expose) |
| `sharp` | `rgba=none`, `hintfull` | grayscale, full | maximum edge definition at 1x |
| `subpixel` | `rgba=rgb`, `hintslight` | rgba, slight | worth trying only on a sub-110 PPI panel at 1x |

Choose with `native-hidpi fix --profile sharp`.

Which layer reaches which client, as far as I have verified:

- **GSettings** (`org.gnome.desktop.interface`) reaches GNOME Shell, GTK3/GTK4, and Qt
  when it uses the GTK platform theme (Ubuntu ships `qt6-gtk-platformtheme`), and snap
  Firefox, which has the `gsettings` interface connected but cannot read dotfiles.
- **fontconfig** (`~/.config/fontconfig/fonts.conf`) reaches Chrome/Chromium/Electron and
  is how font discovery works for everything else. Blender and Java enumerate fonts
  through fontconfig but apply antialiasing with their own internal flags, so fontconfig
  may not change their UI text - neither is a reason to skip the file.
- Neither reaches the Linux virtual console (no antialiasing at all; that needs a PSF
  console font) or Wine/Proton titles (Wine's own font stack).

Note on Ubuntu's defaults: Ubuntu overrides `font-antialiasing` to `rgba`, but *not*
`font-hinting`, whose schema default is `slight`, and `font-rendering` stays `automatic`,
where the toolkit applies its own heuristics. A machine showing `hinting=none` has it from
local configuration, not from a distro default - `status` prints what you actually have.

## Flags

All of these belong to `fix`:

```
native-hidpi fix --scale 2                  force a framebuffer scale (snapped to the
                                            closest scale the mode actually offers)
native-hidpi fix --scale 1.8                pixel-exact integer scale + text-scaling-factor
native-hidpi fix --text-scale 0.9           set text-scaling-factor (e.g. 0.9 at 2x)
native-hidpi fix --want-logical-width 1920  keep at least this many logical px wide
native-hidpi fix --profile sharp            antialiasing profile
native-hidpi fix --no-fonts                 display config only
native-hidpi fix --dry-run                  print what would change
```

Other subcommands: `status`, `revert`, `selftest` (57 assertions on the decision logic,
including a scratch `monitors.xml`; touches no display and no real config).

## Safety

- Every connected output is included in each apply, so a second monitor is never disabled.
- Cloned (mirrored) logical monitors keep all of their connectors.
- One apply, then the result is read back; on mismatch the previous configuration is
  re-applied and the tool exits non-zero. Read-back cannot prove the panel is actually
  displaying an image - if it goes black, the previous configuration is what the tool
  restored, and Settings -> Displays remains the way out.
- `monitors.xml` is matched by connector **and serial**, backed up once, and is never
  edited if it does not parse.
- Scales are compared with a float32-sized tolerance (1e-5) and written back at full
  precision, so a fractional scale neither looks like drift nor gets truncated.
- An output that needs no change keeps the mode it is on, refresh rate included: fixing
  the scale on a 4K panel already running 4K@30 does not silently move it to 4K@60.
- What `revert` restores is recorded per connector **and serial**, and is read back to
  confirm it took; if a step fails the failure is named and `state.json` is kept.
- `fonts.conf` is parsed before writing and fontconfig is then asked to load it; errors
  are attributed only to our own path, and the backup is restored if it is rejected.
  fontconfig drops the **entire** file on one syntax error and every client silently falls
  back to distro defaults - that failure is invisible unless you read the journal.
- `text-scaling-factor` is recorded in `state.json` before it is touched (including the
  value synthesized for an in-between `--scale`) and put back by `revert`.
- `state.json` and `monitors.xml`/`fonts.conf` writes are atomic.
- `revert` keeps `state.json` when any step fails, so you can retry.
- Refuses to run when Mutter reports no outputs (powered-off monitor, headless session)
  rather than claiming success.
- No root. Writes: `~/.config/monitors.xml` (+ `.native-hidpi.bak`),
  `org.gnome.desktop.interface` GSettings, `~/.config/fontconfig/fonts.conf`
  (+ `.native-hidpi.bak`), `~/.local/state/native-hidpi/state.json`.

## Requirements and limits

- A GNOME session running Mutter; it needs `org.gnome.Mutter.DisplayConfig`. On non-GNOME
  desktops it exits with a clear message. It will also *run* on GNOME on Xorg, but the
  framebuffer-scale reasoning here is Wayland-specific, and on Xorg the mode is better set
  with `xrandr`.
- Multi-monitor: every output is included in the applied configuration; each run changes
  the outputs that need it.
- If a panel exposes no `is-preferred` mode, it is reported as `native : unknown` and left
  alone.
- Built-in (laptop) panels are labelled `[built-in]`. A built-in panel driven below its
  native eDP mode is resampled too, so the warning applies there as well - it is just much
  rarer than on external displays.
- Written and tested on Ubuntu 26.04, GNOME Shell / Mutter 50.1, Dell U2720Q (single
  output), Google Chrome 157, snap Firefox 160. Other vendors' EDID quirks are untested.

## Why this is a script and not a patch to Ubuntu

The mode is per-machine state, and Ubuntu's `font-antialiasing=rgba` default is a
deliberate distribution choice, so neither is a bug anyone can merge. The gaps that *are*
worth closing upstream are the missing warning in the Displays panel and the fact that a
DBus persistent apply does not reliably land in `monitors.xml` - see
[`UPSTREAM-ISSUE.md`](UPSTREAM-ISSUE.md).

## License

MIT

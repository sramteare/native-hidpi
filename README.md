# native-hidpi

Find out whether your monitor is being fed the wrong resolution, and fix it
**permanently**.

On GNOME (Mutter) with Wayland, or on Hyprland. One file; needs `python3-gi` on GNOME
(Hyprland goes through `hyprctl`), plus `gsettings` and `fc-match` for the font half.

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

## Hyprland

The same commands work in a Hyprland session (detected from
`HYPRLAND_INSTANCE_SIGNATURE`); only the display half changes:

| | GNOME | Hyprland |
|---|---|---|
| live state | `org.gnome.Mutter.DisplayConfig` | `hyprctl -j monitors all` |
| apply | `ApplyMonitorsConfig`, every output | one `hyprctl --batch` of `keyword monitor` rules |
| persisted in | `~/.config/monitors.xml` | the `monitor=` lines in `hyprland.conf` and every file it `source`s |
| offered scales | what Mutter lists per mode | 1/120 steps that divide the mode into whole pixels |

What changes in practice:

- **`--scale`** works the same way: `native-hidpi fix --scale 1.5` keeps the native mode
  at 1.5 on every output, and a scale that does not divide the mode into whole pixels
  (1.75, 1.8 on 4K) is refused with the closest step. Hyprland's 1/120 grid has more exact
  steps than Mutter offers, so `1.6` here is exactly 1.6 (3840/1.6 = 2400), not
  shorthand for 5/3; type `1.67` for that.
- **The native mode** is the largest one in `availableModes`, first in kernel order, since
  Hyprland marks no preferred mode. An output already at native resolution keeps its
  refresh rate.
- **Layout.** A scale change changes an output's logical width while Hyprland keeps its
  old position, so side-by-side screens would overlap or leave a gap. A row of outputs
  whose vertical extents touch is re-packed left to right in its current order, and an
  existing overlap gets fixed on the way. Stacked or otherwise arranged outputs keep their
  positions.
- **Persisting** rewrites the rules that name each output (by connector or `desc:`) in
  place: their spelling, trailing comment and anything after the scale (`transform`,
  `bitdepth`, `vrr`, ...) stay. An output with no rule of its own gets a `desc:` rule,
  which carries the serial, next to the other named rules. Every file edited is backed up
  once as `<file>.native-hidpi.bak`. Then `hyprctl reload`, `configerrors`, and a read-back:
  if another rule overrides the new one, `fix` says so instead of reporting success.
- **What `status` reports**, the Hyprland versions of "your change will be undone":

```
NOTE     ~/.config/hypr/monitors.conf has monitor rules for DP-2, DP-3 but nothing
         sources it from ~/.config/hypr/hyprland.conf, so they have no effect
NOTE     DP-3: only the catch-all `monitor = ,preferred,auto,auto` covers it, so Hyprland
         picks the scale itself on every reload or hotplug (running 1 now)
```

  The first is what `nwg-displays` leaves behind until you uncomment its `source =` line.
  The second is how two identical 4K panels end up at 1.5 and 1.0: `auto` decides per
  panel from its DPI.
- **XWayland, the usual reason text stays soft.** At any scale above 1, Hyprland draws an
  X11 client at 1x and stretches it, so its text is resampled whatever the scale or
  fontconfig says. `status` lists every XWayland window on a scaled output as a WARNING
  (and exits non-zero):

  ```
  WARNING  com.anthropic.Claude on DP-2 (scale 1.5), Google-chrome on DP-3 (scale 1.5)
           run under XWayland: Hyprland draws them at 1x and stretches them ...
  ```

  The usual cause is not the app but how it was started. A launcher (Caelestia, rofi via
  systemd, D-Bus activation) starts apps under `systemd --user`, not as children of
  Hyprland, so they never see the `env =` lines in `hyprland.conf`; without
  `XDG_SESSION_TYPE=wayland` and `ELECTRON_OZONE_PLATFORM_HINT` there, Chrome and
  Electron apps fall back to X11. `status` checks `systemctl --user show-environment` for
  those and says which are missing. The fix is one line at login, then restarting the
  apps:

  ```
  exec-once = dbus-update-activation-environment --systemd WAYLAND_DISPLAY XDG_CURRENT_DESKTOP XDG_SESSION_TYPE HYPRLAND_INSTANCE_SIGNATURE ELECTRON_OZONE_PLATFORM_HINT MOZ_ENABLE_WAYLAND QT_QPA_PLATFORM QT_QPA_PLATFORMTHEME
  ```

  Chrome does not switch on the environment alone: Chrome 157 restarted with
  `XDG_SESSION_TYPE=wayland` in its environment and still chose `--ozone-platform=x11`.
  Set `chrome://flags/#ozone-platform-hint` to **Wayland** and relaunch; the setting lives
  in the profile, so it holds however Chrome is started.
  `xwayland:force_zero_scaling` is left alone: it stops the stretching, but without
  `GDK_SCALE`/`QT_SCALE_FACTOR` the apps it affects come out tiny.
- `monitorv2 { }` blocks are not read or edited; only `monitor =` lines are.

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
Mutter offers `1.6666666269302368` (5/3), because 3840/1.75 is not a whole number of
pixels. So `--scale` accepts the shorthand you would naturally type for those steps -
1.6, 1.66 and 1.67 all mean 5/3, 1.3 and 1.33 mean 4/3, 2.6/2.66/2.67 mean 8/3 - and
resolves it to the exact value the mode reports. The same value goes into `monitors.xml`:
writing `1.66667` there is a *different* scale, and `status` would report drift that is
not real.

## `--scale` is a framebuffer scale, nothing else

Mutter accepts a scale only when it divides the mode into whole pixels; anything else comes
back as `InvalidArgs`. A request it cannot do is therefore **refused**, with the closest
step and the full list of supported ones:

```
$ native-hidpi fix --scale 1.8
scale 1.8 is not a framebuffer scale this mode offers: the compositor only accepts
scales that divide the mode into whole pixels (3840/1.8 = 2133.33 is one such rejection;
Mutter answers InvalidArgs, Hyprland silently picks a different one). Closest available: 1.6666666269302368; supported: 1.0, 1.25,
1.3333333730697632, 1.5, 1.6666666269302368, 2.0, 2.5, 2.6666667461395264, 3.0,
3.3333332538604736, 3.75, 4.0. It is not faked with text-scaling-factor, which only
resizes GNOME text and leaves windows, Chrome and Electron at the old scale - pass
--text-scale yourself if that is what you want.
```

The temptation this refuses is real: `2.0 + text-scaling-factor 0.9` *looks* like the 1.8x
that was asked for, and is not. `text-scaling-factor` only changes the size of GNOME and
GTK text. Windows, their layouts, Chrome, Electron, and anything that reads the monitor
scale keep the framebuffer scale, so the desktop stays at 2x (1920x1080 logical) with
slightly smaller labels - which is not the request, and is worse than an honest refusal.

`--text-scale` remains available as the separate, explicit knob it is, and `status` shows
both halves when they differ:

```
UI font  'Ubuntu Sans 11'  -> 26.4 device px per em at scale 2.0
         x text-scaling-factor 0.9 (effective 1.8x)
```

| request | framebuffer scale | text-scaling-factor | result |
|---|---|---|---|
| `--scale 2` | 2x | untouched | pixel-exact |
| `--scale 1.6` / `1.66` / `1.67` | 5/3 | untouched | real 2560x1440 logical, resampled in the GPU |
| `--scale 1.8`, `--scale 1.75` | - | - | refused: no such step |
| `--text-scale 0.9` | untouched | 0.9 | only GNOME text changes |

A `text-scaling-factor` left over from an earlier experiment is reset to 1.0 by `fix`
(unless you pass `--text-scale`), so a stale 0.9 does not sit there shrinking your labels
while the display runs at 2x; the value it replaced is recorded and `revert` puts it back.

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
| `mac` (default) | `rgba=none`, `hintslight` | grayscale, slight | approximates macOS text: grayscale AA, light hinting, and FreeType stem darkening (below) |
| `sharp` | `rgba=none`, `hintfull` | grayscale, full | maximum edge definition at 1x |
| `subpixel` | `rgba=rgb`, `hintslight` | rgba, slight | worth trying only on a sub-110 PPI panel at 1x |

Choose with `native-hidpi fix --profile sharp`.

### Stem darkening

Grayscale AA and light hinting get the shapes right, but macOS text also looks *heavier*:
it thickens thin strokes on screen, most at small sizes. FreeType can do the same - stem
darkening - but it is not a fontconfig setting. It is a FreeType property read from the
environment of each app:

```
FREETYPE_PROPERTIES="cff:no-stem-darkening=0 autofitter:no-stem-darkening=0"
```

Measured on Ubuntu 26.04 by rendering one line of text through cairo/FreeType with and
without it (sum of coverage, i.e. how much ink the strokes put down):

| font | 16.5 px | 22 px |
|---|---|---|
| Ubuntu Sans | +13.5% | +4.2% |
| Noto Sans | +17.3% | +11.8% |

The `mac` profile turns it on. Since it has to reach every app, `fix` puts it in three
places:

- `~/.config/environment.d/90-native-hidpi.conf`, which systemd reads at login, so
  launcher-started and D-Bus-activated apps (and GNOME's session) get it;
- the running session (`systemctl --user set-environment` and D-Bus activation), so
  apps started from now on get it without logging out;
- on Hyprland, an `env =` line in `hyprland.conf` plus `hyprctl keyword env`, for apps
  Hyprland starts itself (keybinds, `exec`).

Any other properties already in `FREETYPE_PROPERTIES` are kept. It works through
FreeType's autofitter, which is what `hintslight` uses, so `sharp` (full bytecode hinting)
would get no darkening; `fix --profile sharp` or `subpixel` takes the keys back out.
Apps read it once at startup, so restart them. Chrome bundles its own FreeType and may
ignore it.

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
native-hidpi fix --scale 2                  force a framebuffer scale; 1.6 / 1.66 / 1.67
                                            are shorthand for the mode's 5/3 step, and a
                                            step the mode does not have is refused
native-hidpi fix --text-scale 0.9           set text-scaling-factor: resizes GNOME text
                                            only, never the framebuffer scale
native-hidpi fix --want-logical-width 1920  keep at least this many logical px wide
native-hidpi fix --profile sharp            antialiasing profile
native-hidpi fix --no-fonts                 display config only
native-hidpi fix --no-display               fonts only (fontconfig, GSettings, stem
                                            darkening): modes, scales, layout untouched
native-hidpi fix --dry-run                  print what would change
```

Other subcommands: `status`, `revert`, `selftest` (116 assertions on the decision logic,
including a scratch `monitors.xml` and scratch Hyprland config files; touches no display and no real config).

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
- `--scale` only ever sets a real framebuffer scale; `text-scaling-factor` is set solely by
  `--text-scale`, and a stale one is reset to 1.0 (recorded in `state.json` first, and put
  back by `revert`).
- `state.json` and `monitors.xml`/`fonts.conf` writes are atomic.
- `revert` keeps `state.json` when any step fails, so you can retry.
- Refuses to run when Mutter reports no outputs (powered-off monitor, headless session)
  rather than claiming success.
- No root. Writes: `~/.config/monitors.xml` (+ `.native-hidpi.bak`) on GNOME, the
  `monitor=` lines of the sourced Hyprland config files (+ `.native-hidpi.bak` each) on
  Hyprland (plus one `env =` line in `hyprland.conf` for stem darkening),
  `~/.config/environment.d/90-native-hidpi.conf`, the systemd user environment's
  `FREETYPE_PROPERTIES`, `org.gnome.desktop.interface` GSettings, `~/.config/fontconfig/fonts.conf`
  (+ `.native-hidpi.bak`), `~/.local/state/native-hidpi/state.json`.

## Requirements and limits

- A GNOME session running Mutter (it needs `org.gnome.Mutter.DisplayConfig`), or a
  Hyprland session with `hyprctl` on the path. Anywhere else it exits with a clear message. It will also *run* on GNOME on Xorg, but the
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
  output), Google Chrome 157, snap Firefox 160. On Hyprland 0.53.3 with a Dell U2720Q and
  U3225QE side by side, `fix` took both from 1.5 to 2x live, re-packed them to
  0x0 / 1920x0 and persisted the rules, confirmed by reload and read-back. Other vendors' EDID quirks are untested.

## Why this is a script and not a patch to Ubuntu

The mode is per-machine state, and Ubuntu's `font-antialiasing=rgba` default is a
deliberate distribution choice, so neither is a bug anyone can merge. The gaps that *are*
worth closing upstream are the missing warning in the Displays panel and the fact that a
DBus persistent apply does not reliably land in `monitors.xml` - see
[`UPSTREAM-ISSUE.md`](UPSTREAM-ISSUE.md).

## License

MIT

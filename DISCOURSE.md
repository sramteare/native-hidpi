# Draft post for Ubuntu Discourse / Ask Ubuntu

Discourse: <https://discourse.ubuntu.com/c/support> - Ask Ubuntu: <https://askubuntu.com/questions/ask>

Suggested title: **Blurry fonts on Ubuntu? Check that your monitor isn't running below its native resolution**

---

If text looks faint, soft or "foggy" on Ubuntu - especially compared to a Mac - then before
you touch fontconfig, hinting, `text-scaling-factor` or any Infinality-style patch set,
check what resolution the panel is actually being fed.

A common state, especially with docking stations, is a 4K monitor driven at 2560x1440. The
monitor's own scaler upscales every pixel by 1.5x and the whole desktop looks soft. Nothing
warns you, and it reads exactly like a font rendering defect, so people spend hours tuning
antialiasing for something font rendering cannot fix.

## Check it

Settings → Displays: if the resolution is lower than the panel's native one (say
2560x1440 on a 4K screen), that is your blur.

Or ask Mutter - the state it reports contains both the `is-current` and `is-preferred`
mode flags:

```bash
gdbus call --session --dest org.gnome.Mutter.DisplayConfig \
  --object-path /org/gnome/Mutter/DisplayConfig \
  --method org.gnome.Mutter.DisplayConfig.GetCurrentState | head -c 600
```

I wrote a small tool for this, mostly because it also fixes the persistence problem below:
<https://github.com/sramteare/native-hidpi>

```bash
native-hidpi status   # warns when the active mode is below the panel's native mode
native-hidpi fix      # native mode + integer scale, persisted, plus antialiasing
native-hidpi revert   # back to whatever it was
```

It also works on **Hyprland** now. There the usual culprit is different: Chrome and
Electron apps started from a launcher run under `systemd --user`, never see the `env =`
lines in `hyprland.conf`, and fall back to XWayland, where Hyprland draws them at 1x and
stretches them by the monitor scale. `native-hidpi status` lists those windows and says
which variables the session is missing.

The default `mac` profile now also turns on FreeType *stem darkening*, which thickens
thin strokes the way macOS does (about +4% to +17% stroke weight at UI sizes, most at the
smallest).

## Why the fix keeps disappearing (the part nobody mentions)

Applying a mode through the display API does not reliably rewrite
`~/.config/monitors.xml`, and that file is what gets restored after you **unplug the
monitor or log back in**. So the change appears to work, and later silently doesn't. Check
whether your stored config matches what is running:

```bash
cat ~/.config/monitors.xml
```

If the `<scale>` or `<mode>` in there differs from your live setup, that is what you will
get back at the next login. Selecting the mode in Settings → Displays writes that file;
that is the reliable route. The tool above edits it directly (matching the stored block by
connector *and serial*, since connector names get reused between monitors), after backing
it up.

## Why integer scale matters

Density is `mode / scale`. Only integer scale maps one logical pixel to one physical pixel.

| mode | scale | logical space | pixel-exact |
|---|---|---|---|
| 2560x1440 on a 4K panel | 1x | 2560x1440 | no - monitor scaler, this is the fog |
| 3840x2160 | 2x | 1920x1080 | yes |
| 3840x2160 | 1.5x | 2560x1440 | no - resampled in the GPU |

Fractional scale has a second cost: Mutter reports `Xft.dpi` as `96 * next integer` (192 at
1.5x), so clients that don't implement `fractional-scale-v1` render at 2x and get resampled
down, and apps that ignore scaling end up a different size from their neighbours. Chrome is
the one people notice: the Chrome 157 build I checked binds `wp_viewport` but has no
`zwp_fractional_scale` symbols at all, so it never looked right at fractional scaling even
though it was running as a native Wayland client.

## If 200% makes everything huge

Don't go back to a downscaled resolution - shrink the contents, and you keep the crispness:

```bash
gsettings set org.gnome.desktop.interface font-name "Ubuntu Sans 11"
gsettings set org.gnome.shell.extensions.dash-to-dock dash-max-icon-size 32
```

## Antialiasing, once you are at native resolution

Check what you actually have:

```bash
gsettings get org.gnome.desktop.interface font-antialiasing
gsettings get org.gnome.desktop.interface font-hinting
gsettings get org.gnome.desktop.interface font-rendering
```

Ubuntu sets `font-antialiasing` to `rgba`; `font-hinting` is not overridden, so its default
is `slight`, and `font-rendering` stays `automatic`, where GNOME applies its own
heuristics. If your `font-hinting` reads `none`, that came from local configuration, not a
distro default.

Either way, subpixel antialiasing combined with *no* hinting is the worst pair on a
non-Retina panel: unhinted stems land on fractional pixels and get smeared across R/G/B,
which reads as blur with coloured edges. Grayscale with slight hinting is the closer match
to how macOS renders text once you are at native resolution with integer 2x (macOS dropped
subpixel text in Mojave, citing high-DPI displays):

```bash
gsettings set org.gnome.desktop.interface font-rendering manual
gsettings set org.gnome.desktop.interface font-antialiasing grayscale
gsettings set org.gnome.desktop.interface font-hinting slight
```

Two gotchas about reach:

- The GNOME Fonts panel (GSettings) does not reach Chrome or Electron - those take
  antialiasing from fontconfig, so they need `~/.config/fontconfig/fonts.conf`. Snap
  Firefox is the opposite: it can read GSettings but cannot see your dotfiles.
- A **single syntax error** in `~/.config/fontconfig/fonts.conf` makes fontconfig discard
  the entire file, and every client silently reverts to distro defaults. Nothing appears
  on screen; the only sign is a `Fontconfig error` line in the journal. Verify with
  `fc-match "Sans"` and look at stderr.

Browsers must be restarted either way: fontconfig is read once at startup.

# Draft post for Ubuntu Discourse / Ask Ubuntu

Discourse: <https://discourse.ubuntu.com/c/support> - Ask Ubuntu: <https://askubuntu.com/questions/ask>

Suggested title: **Blurry fonts on Ubuntu? Check that your monitor isn't running below its native resolution**

---

If text looks faint, soft or "foggy" on Ubuntu - especially compared to a Mac - before you
touch fontconfig, hinting, `text-scaling-factor` or any Infinality-style patch set, check
what resolution the panel is actually being fed.

A very common state, especially with docking stations, is a 4K monitor driven at
2560x1440. The monitor's own scaler then upscales every pixel by 1.5x and the whole
desktop looks soft. Nothing in the display stack warns you about it, and it reads exactly
like a font rendering bug, so people spend hours tuning antialiasing for a problem that
font rendering cannot fix.

## Check it in five seconds

Settings → Displays. If the resolution shown is lower than the panel's native one
(e.g. 2560x1440 on a 4K screen), that is your blur.

Or ask Mutter directly - the `is-current` and `is-preferred` mode flags are both in the
state it reports:

```bash
gdbus call --session --dest org.gnome.Mutter.DisplayConfig \
  --object-path /org/gnome/Mutter/DisplayConfig \
  --method org.gnome.Mutter.DisplayConfig.GetCurrentState | head -c 600
```

I put the check and the fix into a small script, since the DBus call is awkward to type
and the warning is easy to misread: <https://github.com/sramteare/native-hidpi>

```bash
native-hidpi status          # warns if the active mode is below the panel's native mode
native-hidpi fix             # native mode + integer scale + sane antialiasing
native-hidpi revert          # back to whatever it was
```

## Why integer scale matters

Density is `mode / scale`. Only integer scale maps one logical pixel to one physical
pixel.

| mode | scale | logical space | pixel-exact |
|---|---|---|---|
| 2560x1440 on a 4K panel | 1x | 2560x1440 | no - monitor scaler, this is the fog |
| 3840x2160 | 2x | 1920x1080 | yes |
| 3840x2160 | 1.5x | 2560x1440 | no - resampled in the GPU |

Fractional scale has a second cost: `Xft.dpi` stays at `96 * next integer` (192 at 1.5x),
so XWayland and Electron windows render at 2x and get resampled down, and apps that ignore
fractional scaling end up a different size from their neighbours.

## If 200% makes everything huge

Don't go back to a scaled-down resolution - shrink the contents instead, and you keep the
crispness:

```bash
gsettings set org.gnome.desktop.interface font-name "Ubuntu Sans 11"
gsettings set org.gnome.shell.extensions.dash-to-dock dash-max-icon-size 32
```

## Antialiasing, once you are at native

Ubuntu's `automatic` policy can leave you with `font-antialiasing=rgba` and
`font-hinting=none` at once. Subpixel antialiasing with no hinting is the worst
combination: unhinted stems land on fractional pixels and get smeared across R/G/B, which
looks like blur with coloured edges. macOS stopped using subpixel antialiasing in Mojave
for this reason. Grayscale + slight hinting is the closer match once you are at native
resolution with integer 2x:

```bash
gsettings set org.gnome.desktop.interface font-rendering manual
gsettings set org.gnome.desktop.interface font-antialiasing grayscale
gsettings set org.gnome.desktop.interface font-hinting slight
```

Note that GNOME's Fonts panel (GSettings) does not reach Chrome, Blender or Java - those
read fontconfig, so they need `~/.config/fontconfig/fonts.conf`. Snap Firefox can read
GSettings but cannot see your dotfiles, so it needs the GSettings side. Either way
browsers must be restarted: fontconfig is read once at startup.

# Draft issue for GNOME Control Center

Target: <https://gitlab.gnome.org/GNOME/gnome-control-center/-/issues>
Component: Displays panel.

---

## Warn when the selected resolution is not the panel's native mode

### The problem

In Settings → Displays you can select any mode the EDID advertises, including modes far
below the panel's native resolution, with no indication that the result will be
resampled by the monitor's scaler. A 3840x2160 panel driven at 2560x1440 upscales every
pixel by a non-integer factor and the entire desktop looks soft.

This is a very common state on Linux desktops - a mode selected once, or negotiated by a
dock and left in `~/.config/monitors.xml` - and it reads exactly like a font rendering
defect. Users therefore spend hours on fontconfig, hinting, `text-scaling-factor`, RGB
order and Infinality-style patch sets, none of which can restore detail that was never
rasterized. The actual fix is one dropdown, but nothing points at it.

### Proposed change

`Meta.Monitor` already exposes the preferred mode, and `GetCurrentState` marks it with
`is-preferred` in the mode properties dict. When the mode a user selects is not the
preferred/native mode, the Displays panel could:

- show the resolution row as "Not recommended - monitor will resample", and
- label the native mode as "Recommended (native)", and
- optionally show an inline banner: "This display's native resolution is 3840x2160.
  Running at 2560x1440 makes everything look blurry."

Laptop panels can be excluded, since the compositor scales there rather than an external
scaler.

### Detection logic

```
active = mode with is-current
native = mode with is-preferred
if active.id != native.id: resampled -> warn
```

### Notes on related knobs

- Fractional scale has a similar silent cost: with a 1.5x monitor scale, `Xft.dpi` stays
  at `96 * next integer` (192), so XWayland/Electron clients render at 2x and get
  resampled. A note next to non-integer scales would help too.
- A standalone script implementing exactly this check and switching to native + integer
  scale over the `org.gnome.Mutter.DisplayConfig` DBus API is at
  <https://github.com/sramteare/native-hidpi>, if a reference implementation is useful.

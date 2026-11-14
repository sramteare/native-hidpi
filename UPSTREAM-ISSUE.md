# Upstream reports

Two separate things, against two different projects. Both are drafted from observations
made on Ubuntu 26.04 with GNOME Shell / Mutter 50.1, a Dell U2720Q (native 3840x2160) and
a Dell U3225QE (native 3840x2160).

---

## 1. gnome-control-center: warn when the selected resolution is not the panel's native mode

Target: <https://gitlab.gnome.org/GNOME/gnome-control-center/-/issues> - component: Displays.

### Problem

Settings → Displays lets a user select any EDID-advertised mode, including modes far below
the panel's native resolution, with no indication that an external scaler will resample the
image. A 3840x2160 panel driven at 2560x1440 upscales every pixel by 1.5x and the entire
desktop looks soft.

This state is easy to end up in and hard to notice: a mode chosen once, or negotiated by a
dock, left in `~/.config/monitors.xml`. It presents as "fonts are blurry on Linux", so
users burn time on fontconfig, hinting, `text-scaling-factor`, RGB order and Infinality
patch sets, none of which can restore detail that was never rasterized. The fix is one
dropdown, but nothing points at it.

### Proposal

`org.gnome.Mutter.DisplayConfig.GetCurrentState` already marks the modes: exactly one entry
per monitor carries `is-current` and one carries `is-preferred`. In the Displays panel, when
the selected mode is not the preferred one:

- label the native mode "Recommended (native)",
- mark the selected row "Not recommended - this display will resample the image",
- and show an inline note naming the native resolution.

Built-in panels usually already run their native eDP mode with a compositor scale factor, so
they will rarely trigger this; but a built-in panel driven below native is resampled too, so
the warning should apply there as well rather than be suppressed by an is-builtin check.

### Detection

```
active = the mode whose properties contain is-current
native = the mode whose properties contain is-preferred
warn if (active.width, active.height) != (native.width, native.height)
```

Comparing full mode identity including refresh rate is wrong: running 4K@30 on a 4K@60
preferred panel is a refresh-rate choice, not resampling.

---

## 2. mutter: a persistent ApplyMonitorsConfig does not land in ~/.config/monitors.xml

Target: <https://gitlab.gnome.org/GNOME/mutter/-/issues>

### Problem

`ApplyMonitorsConfig` with the persistent method succeeds, and the new configuration is
immediately reported by `GetCurrentState`, but `~/.config/monitors.xml` is **not** rewritten
(`mtime` unchanged, contents unchanged). That file is what Mutter restores on monitor hotplug
and at login, so the change is silently undone.

Observed sequence:

```
GetCurrentState                  -> DP-1 3840x2160@59.997, scale 1.5
ApplyMonitorsConfig(persistent)  -> DP-1 3840x2160@59.997, scale 2.0
GetCurrentState                  -> DP-1 3840x2160@59.997, scale 2.0   (looks applied)
stat -c %y ~/.config/monitors.xml-> unchanged
[unplug and replug the monitor]
GetCurrentState                  -> DP-1 3840x2160@59.997, scale 1.5   (reverted)
```

The user has no way to distinguish "applied and saved" from "applied but about to be lost",
which is a much worse failure than an error message: the display visibly changes, so it is
the Settings UI's own behaviour that appears broken afterwards.

### Expectation

Either the persistent method writes `monitors.xml` before returning success, or the API
reports that the configuration is live-but-unstored, so callers can say so.

### Notes

- `org.gnome.Mutter.DisplayConfig.xml` is not installed on Ubuntu 26.04
  (`/usr/share/dbus-1/interfaces/` has no copy), so clients cannot check the `method` flag
  values locally, and the meaning of those flags has not been stable across versions.
  Shipping the interface XML would let callers get this right.
- A `ConfirmNewConfiguration`-style step is what gnome-control-center uses; direct DBus
  callers have no equivalent, which may be why the write is skipped.
- A reference implementation that works around this by writing the file itself - matching
  stored blocks by connector **and serial**, because connector names are reused between
  monitors, and leaving non-matching blocks alone - is at
  <https://github.com/sramteare/native-hidpi>.

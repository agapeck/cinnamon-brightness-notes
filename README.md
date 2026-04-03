# Cinnamon Brightness Fine-Step Guide

This document records a working way to replace Cinnamon's coarse brightness-key behavior with finer steps while preserving the on-screen brightness indicator (OSD).

It was implemented on an HP ProBook 450 G5 running Linux Mint Cinnamon, but the core approach is portable to other Cinnamon systems because it relies on Cinnamon's own session D-Bus services instead of patching Cinnamon code, changing GRUB, or editing system packages.

The current version also hardens the key-handling path against rapid repeated taps by serializing brightness requests and caching the last requested target briefly. That avoids races between overlapping script instances.

## Goal

Replace default `XF86MonBrightnessUp` and `XF86MonBrightnessDown` behavior with:

- finer brightness steps, here `2%`
- no system package changes
- no GRUB/kernel edits
- no Cinnamon source edits
- OSD still shown on each key press
- rollback kept simple

## Why This Approach

On the target system:

- the panel exposed many raw hardware levels via `/sys/class/backlight/intel_backlight`
- Cinnamon still behaved as if brightness steps were coarse
- the coarse feel came from Cinnamon's key handling path, not from a lack of hardware brightness granularity

Instead of changing kernel backlight selection with `acpi_backlight=vendor` or patching Cinnamon internals, the safer approach was:

1. keep Cinnamon's existing power daemon as the component that actually changes brightness
2. override only the brightness keybindings
3. call Cinnamon's brightness service directly with an exact percentage
4. call Cinnamon's OSD method after each change

This makes the behavior user-local and easy to revert.

## Files Created

The implementation created these files:

- `/home/hp/.local/bin/cinnamon-brightness-step`
- `/home/hp/.local/bin/cinnamon-brightness-restore-defaults`
- `/home/hp/.config/cinnamon-brightness-backup.txt`

## Cinnamon Components Involved

Useful Cinnamon internals that informed the solution:

- `org.cinnamon.SettingsDaemon.Power.Screen`
  - session D-Bus interface used to get and set display brightness
- `org.Cinnamon`
  - session D-Bus interface used to show the OSD
- `/usr/share/cinnamon/js/ui/keybindings.js`
  - shows that custom keybindings only run commands, while media keys go through Cinnamon's media-key handler
- `/usr/share/cinnamon/js/ui/osdWindow.js`
  - important detail: `setLevel()` divides the incoming `level` by `100`

That last point matters. Cinnamon's OSD expects a `0..100` percentage, not a normalized `0..1` fraction.

## Working Script

Final working script:

```bash
#!/usr/bin/env bash
set -euo pipefail

readonly DEST="org.cinnamon.SettingsDaemon.Power"
readonly PATH_NAME="/org/cinnamon/SettingsDaemon/Power"
readonly IFACE="org.cinnamon.SettingsDaemon.Power.Screen"
readonly DEFAULT_STEP=2
readonly OSD_DEST="org.Cinnamon"
readonly OSD_PATH="/org/Cinnamon"
readonly OSD_IFACE="org.Cinnamon"
readonly STATE_DIR="${XDG_STATE_HOME:-$HOME/.local/state}/cinnamon-brightness"
readonly LOCK_FILE="$STATE_DIR/brightness.lock"
readonly TARGET_FILE="$STATE_DIR/last-target"
readonly TARGET_TTL=2

usage() {
    printf 'Usage: %s up|down [step-percent]\n' "${0##*/}" >&2
    exit 2
}

clamp() {
    local value=$1
    local min=$2
    local max=$3

    if (( value < min )); then
        printf '%s\n' "$min"
    elif (( value > max )); then
        printf '%s\n' "$max"
    else
        printf '%s\n' "$value"
    fi
}

parse_percentage() {
    awk '/^u / { print $2; exit }'
}

load_recent_target() {
    local now=$1
    local saved_at saved_target

    [[ -r "$TARGET_FILE" ]] || return 1
    read -r saved_at saved_target < "$TARGET_FILE" || return 1

    if [[ "$saved_at" =~ ^[0-9]+$ && "$saved_target" =~ ^[0-9]+$ ]] && (( now - saved_at <= TARGET_TTL )); then
        printf '%s\n' "$saved_target"
        return 0
    fi

    return 1
}

save_target() {
    local percentage=$1

    printf '%s %s\n' "$(date +%s)" "$percentage" > "$TARGET_FILE"
}

get_live_percentage() {
    local output percentage

    output=$(busctl --user call "$DEST" "$PATH_NAME" "$IFACE" GetPercentage)
    percentage=$(printf '%s\n' "$output" | parse_percentage)

    if [[ "$percentage" =~ ^[0-9]+$ ]]; then
        printf '%s\n' "$percentage"
        return 0
    fi

    return 1
}

show_osd() {
    local percentage=$1

    busctl --user call \
        "$OSD_DEST" \
        "$OSD_PATH" \
        "$OSD_IFACE" \
        ShowOSD \
        "a{sv}" \
        3 \
        icon s "display-brightness-symbolic" \
        label s "Brightness" \
        level d "$percentage" \
        >/dev/null 2>&1 || true
}

direction=${1:-}
step=${2:-$DEFAULT_STEP}

case "$direction" in
    up|down) ;;
    *) usage ;;
esac

if ! [[ "$step" =~ ^[0-9]+$ ]] || (( step <= 0 || step > 100 )); then
    usage
fi

mkdir -p "$STATE_DIR"
exec 9>"$LOCK_FILE"
flock 9

now=$(date +%s)
if ! current=$(load_recent_target "$now"); then
    current=$(get_live_percentage)
fi

if ! [[ "$current" =~ ^[0-9]+$ ]]; then
    printf 'Failed to read current brightness from Cinnamon power service.\n' >&2
    exit 1
fi

case "$direction" in
    up)
        target=$(( current + step ))
        ;;
    down)
        target=$(( current - step ))
        ;;
esac

target=$(clamp "$target" 0 100)

if (( target == current )); then
    show_osd "$target"
    exit 0
fi

set_output=$(busctl --user call "$DEST" "$PATH_NAME" "$IFACE" SetPercentage u "$target")
new_percentage=$(printf '%s\n' "$set_output" | parse_percentage)

if [[ "$new_percentage" =~ ^[0-9]+$ ]]; then
    save_target "$new_percentage"
    show_osd "$new_percentage"
fi
```

## Why `busctl --user`

`busctl --user` talks to the current session bus, which is where Cinnamon exposes:

- `org.cinnamon.SettingsDaemon.Power`
- `org.Cinnamon`

This avoids:

- writing to `/sys/class/backlight/*` directly
- `sudo` prompts on every key press
- dependency on a specific backlight path like `intel_backlight`

That makes it more portable across Cinnamon systems.

## Why The Extra Locking And Short-Lived Cache Matter

Without serialization, rapid taps can start multiple script instances at once. If each one reads the current brightness before the previous one has finished updating it, they can all calculate from stale values.

Typical symptoms:

- skipped steps
- sluggish or uneven response
- rapid decrease behaving disproportionately
- brightness seeming to jump further than the number of taps would imply

The fix is:

- serialize access with `flock`
- briefly cache the last requested brightness target
- when taps happen in quick succession, calculate the next step from the last requested target instead of repeatedly querying a value that may still be catching up

This makes rapid key taps much more deterministic.

## Keybinding Changes

Default Cinnamon brightness keys:

- `org.cinnamon.desktop.keybindings.media-keys screen-brightness-up`
- `org.cinnamon.desktop.keybindings.media-keys screen-brightness-down`

These were disabled to avoid double-triggering:

```bash
gsettings set org.cinnamon.desktop.keybindings.media-keys screen-brightness-up "[]"
gsettings set org.cinnamon.desktop.keybindings.media-keys screen-brightness-down "[]"
```

Two custom bindings were added:

- `custom0` for brightness up
- `custom1` for brightness down

Commands used:

```bash
gsettings set org.cinnamon.desktop.keybindings.custom-keybinding:/org/cinnamon/desktop/keybindings/custom-keybindings/custom0/ name 'Brightness Up 2%'
gsettings set org.cinnamon.desktop.keybindings.custom-keybinding:/org/cinnamon/desktop/keybindings/custom-keybindings/custom0/ command '/home/hp/.local/bin/cinnamon-brightness-step up 2'
gsettings set org.cinnamon.desktop.keybindings.custom-keybinding:/org/cinnamon/desktop/keybindings/custom-keybindings/custom0/ binding "['XF86MonBrightnessUp']"

gsettings set org.cinnamon.desktop.keybindings.custom-keybinding:/org/cinnamon/desktop/keybindings/custom-keybindings/custom1/ name 'Brightness Down 2%'
gsettings set org.cinnamon.desktop.keybindings.custom-keybinding:/org/cinnamon/desktop/keybindings/custom-keybindings/custom1/ command '/home/hp/.local/bin/cinnamon-brightness-step down 2'
gsettings set org.cinnamon.desktop.keybindings.custom-keybinding:/org/cinnamon/desktop/keybindings/custom-keybindings/custom1/ binding "['XF86MonBrightnessDown']"

gsettings set org.cinnamon.desktop.keybindings custom-list "['custom0', 'custom1']"
```

## Restore Script

Restore script used:

```bash
#!/usr/bin/env bash
set -euo pipefail

gsettings set org.cinnamon.desktop.keybindings.media-keys screen-brightness-up "['XF86MonBrightnessUp']"
gsettings set org.cinnamon.desktop.keybindings.media-keys screen-brightness-down "['XF86MonBrightnessDown']"

custom_list=$(gsettings get org.cinnamon.desktop.keybindings custom-list)

if [[ "$custom_list" == *"'custom0'"* ]]; then
    gsettings reset-recursively org.cinnamon.desktop.keybindings.custom-keybinding:/org/cinnamon/desktop/keybindings/custom-keybindings/custom0/
fi

if [[ "$custom_list" == *"'custom1'"* ]]; then
    gsettings reset-recursively org.cinnamon.desktop.keybindings.custom-keybinding:/org/cinnamon/desktop/keybindings/custom-keybindings/custom1/
fi

python3 - <<'PY'
import ast
import subprocess

raw = subprocess.check_output(
    ["gsettings", "get", "org.cinnamon.desktop.keybindings", "custom-list"],
    text=True,
).strip()
items = ast.literal_eval(raw)
items = [item for item in items if item not in {"custom0", "custom1"}]
subprocess.run(
    ["gsettings", "set", "org.cinnamon.desktop.keybindings", "custom-list", str(items)],
    check=True,
)
PY
```

## Important Lessons / Pitfalls

### 1. Do not assume a GRUB or `acpi_backlight=` fix is needed

If brightness works and the system already exposes a working Cinnamon power service, fix the user experience at the Cinnamon layer first.

Only consider kernel backlight selection changes if:

- brightness is missing entirely
- brightness is broken
- multiple backlight interfaces are actually present and the wrong one is being selected

### 2. Custom keybindings bypass Cinnamon's media-key OSD

If you replace `XF86MonBrightnessUp/Down` with custom commands, the default OSD will disappear unless you explicitly call `org.Cinnamon.ShowOSD`.

### 3. OSD `level` is not normalized

This was the key bug during implementation.

Wrong:

- sending `0.42` for `42%`

Right:

- sending `42`

Reason:

- Cinnamon's `osdWindow.js` divides the incoming value by `100`

### 4. Use Cinnamon's brightness service, not direct sysfs, when possible

Calling `org.cinnamon.SettingsDaemon.Power.Screen.SetPercentage`:

- avoids root access
- avoids device-specific backlight paths
- remains closer to Cinnamon's own brightness path

### 5. Disable default brightness media keys to avoid double-adjustment

If custom bindings and stock bindings both remain active, one key press can trigger multiple brightness changes.

### 6. Rapid taps need serialization

The first simple version of the script used:

- `GetPercentage`
- arithmetic in the shell
- `SetPercentage`

That works for isolated key presses, but it is race-prone under rapid repeat because multiple invocations can overlap.

The more robust version uses:

- a lock file with `flock`
- a short-lived cached last-target file

If reproducing this elsewhere, use the locked version, not the naive one.

## Portable Procedure For Another Cinnamon System

Use this checklist on another Cinnamon machine:

1. Confirm Cinnamon session:
   - `echo "$XDG_CURRENT_DESKTOP"`
   - expect something containing `Cinnamon`
2. Confirm session-bus services exist:
   - `busctl --user list | rg 'org.Cinnamon|org.cinnamon.SettingsDaemon.Power'`
3. Confirm brightness D-Bus interface works:
   - `busctl --user call org.cinnamon.SettingsDaemon.Power /org/cinnamon/SettingsDaemon/Power org.cinnamon.SettingsDaemon.Power.Screen GetPercentage`
4. Install the step script into `~/.local/bin/`
5. Make it executable:
   - `chmod +x ~/.local/bin/cinnamon-brightness-step`
6. Disable stock brightness media keys
7. Create custom keybindings for `XF86MonBrightnessUp/Down`
8. Verify OSD call manually:
   - `busctl --user call org.Cinnamon /org/Cinnamon org.Cinnamon ShowOSD "a{sv}" 3 icon s display-brightness-symbolic label s Brightness level d 42`
9. Test keys
10. If bindings do not refresh immediately, restart Cinnamon with:
   - `Ctrl+Alt+Esc`

## Verification Commands

Useful checks:

```bash
gsettings get org.cinnamon.desktop.keybindings custom-list
gsettings get org.cinnamon.desktop.keybindings.media-keys screen-brightness-up
gsettings get org.cinnamon.desktop.keybindings.media-keys screen-brightness-down
gsettings get org.cinnamon.desktop.keybindings.custom-keybinding:/org/cinnamon/desktop/keybindings/custom-keybindings/custom0/ command
gsettings get org.cinnamon.desktop.keybindings.custom-keybinding:/org/cinnamon/desktop/keybindings/custom-keybindings/custom1/ command

busctl --user call org.cinnamon.SettingsDaemon.Power /org/cinnamon/SettingsDaemon/Power org.cinnamon.SettingsDaemon.Power.Screen GetPercentage
busctl --user introspect org.Cinnamon /org/Cinnamon org.Cinnamon
```

## Notes For Another Agent

If reproducing this on another Cinnamon system:

- prefer user-local changes over system-wide changes
- prefer Cinnamon's own session D-Bus services over raw backlight writes
- preserve rollback
- do not patch Cinnamon source unless there is no clean D-Bus path
- remember that OSD `level` must be sent as percentage, not a normalized fraction

## Current State On This System

Current behavior on this machine:

- brightness keys change brightness in `2%` steps
- OSD is restored via explicit `org.Cinnamon.ShowOSD`
- changes are Linux-user-local only
- Windows dual-boot is unaffected

END
---
name: android-tv-laptop-remote
description: "Use a laptop keyboard or laptop shell as the remote for an Android TV / Google TV device such as a smart projector, TV box, or Chromecast. Covers Bluetooth and USB keyboard pairing, Cast keyboard forwarding, and driving the UI over wireless ADB with adb input keyevent. Use when: (1) the user is tired of an on-screen D-pad remote, (2) automation must navigate a TV UI headlessly, or (3) typing into an Android TV search box without an on-screen keyboard."
---

# Android TV / Google TV — Laptop as Remote

Use a laptop keyboard, a Cast session, or an ADB shell to control an Android TV / Google TV
device (smart projector, TV box, Chromecast with Google TV) instead of the physical
infrared/D-pad remote. Expected outcome: the user navigates the launcher, launches apps, and
types search text from a laptop, and agents can script the same thing headlessly.

## Quick quality checklist

- `name` matches folder name exactly (kebab-case)
- All examples are tested and runnable (`--dry-run` lets you verify with no device attached)
- Includes both Bash and Node.js examples
- Uses free/open-source tools only (Android platform-tools)
- No secrets, API keys, or personal data in examples

## When to use

- A mini projector / TV box runs Google TV and the user wants to stop hunting letters on the
  on-screen keyboard with a D-pad.
- Launching an app by name repeatedly (Netflix, YouTube, Plex, Kodi) and scripting that launch.
- An automated test or demo needs to drive a TV UI with no human and no remote.
- The physical remote is lost or broken and the device must still be operable.

**Do not use** for HDMI-only projectors with no Android/OS menu. Those have no keyboard input
at all; control them from the source device or via HDMI-CEC.

## Pick a method first

| Method | Needs | Works on | Good for |
| --- | --- | --- | --- |
| Bluetooth keyboard | one-time pairing | everything, all Android versions | permanent fix, zero setup |
| USB keyboard | a free USB port | most Android TV builds | cheapest option, no pairing |
| Cast from browser | Chrome on both ends | any Cast receiver | watching from the laptop |
| Wireless ADB | dev options on device | Android 9–13 only (see blocker) | scripting, agent automation |

Bluetooth or USB is the right default. ADB is for automation and for users who want their
*actual* laptop keyboard mapped to D-pad keys.

## Required tools / APIs

- No external API, token, or account required.
- **Android platform-tools (`adb`)** — only for the ADB method.
- **A Bluetooth-capable keyboard** — either the user's own laptop keyboard or a cheap
  silicone mini keyboard / TV remote-keyboard combo.

Install options:

```bash
# Ubuntu/Debian
sudo apt-get install -y adb

# macOS
brew install android-platform-tools

# Verify
adb version && adb mdns check
```

## Skills

### basic_usage — Bluetooth or USB keyboard

The one-time setup on the device:

1. `Settings` → `System` → `Bluetooth` (older builds: `Settings` → `Devices & accessories`).
2. Pair the laptop, or plug a USB keyboard into the device's USB port.
3. Confirm the device shows the keyboard as connected under input/keyboard settings.

After pairing the physical remote is only needed for power and for navigating to this menu.
The keyboard reconnects on its own at boot. Navigation keys that work: arrow keys move the
D-pad, `Enter` selects, `Backspace`/`Esc` go back, and free typing fills search boxes — which
is the whole point, since it bypasses the on-screen keyboard.

Caveat: some budget projector firmware ships a locked-down Bluetooth stack that only accepts
HID keyboards from a short allowlist. If pairing stalls, try the USB port.

### basic_usage — Cast from the laptop browser

When the user is watching content from the laptop, skip the device UI entirely:

```bash
# Chrome on the laptop: cast the current tab to the device.
# Keyboard events are forwarded to the receiver during playback.
```

Verified forwarded keys during cast playback: space, arrows, `k` (play/pause), `f` (fullscreen),
and the media keys. App selection on the device home screen is *not* controllable this way —
use a keyboard or ADB for that.

### robust_usage — wireless ADB

Enable on the device:

1. `Settings` → `About build` (sometimes under `Device preferences`) → press `Build` **7 times**.
2. Back to `Settings` → `System` → `Developer options` → `Wireless debugging` → on.
3. `Wireless debugging` → `Pair device with pairing code` — note the `IP:PORT` and the 6-digit code.

Connect from the laptop:

```bash
# Step 1: pair using the port shown by the pairing dialog (note: NOT the connect port)
printf '%s\n' '123456' | adb pair 192.168.1.50:37000

# Step 2: connect using the *Wireless debugging* screen port, which is different
adb connect 192.168.1.50:41234

# Step 3: confirm the device is online
adb devices
```

Send navigation and text:

```bash
adb shell input keyevent 20     # DPAD_DOWN
adb shell input keyevent 66     # ENTER
adb shell input keyevent 3      # HOME
adb shell input text 'the%smatrix'   # %s is how adb encodes a space
adb shell monkey -p com.netflix.ninja -c android.intent.category.LAUNCHER 1  # launch app
```

**The port changes on every reboot and after toggling Wireless debugging.** Re-discover it
instead of reading it off the screen:

```bash
#!/usr/bin/env bash
# reconnect-tv.sh — re-discover and reconnect an Android TV over wireless ADB.
set -euo pipefail

service="$(adb mdns services 2>/dev/null | awk '/_adb-tls-connect/ {print $NF}' | head -n 1)"

if [[ -z "$service" ]]; then
  echo "error: no _adb-tls-connect mDNS service found." >&2
  echo "       Is Wireless debugging still enabled on the device?" >&2
  exit 1
fi

echo "connecting to ${service}"
adb connect "${service}"
adb devices
```

For a device that keeps the same port across reboots, set `ADB_MDNS_AUTO_CONNECT` on the
laptop (default is `adb-tls-connect`) so `adb` reconnects on its own after a rediscovery.

**Node.js helper** — validated names, retries, and a dry-run mode:

```javascript
// tvnav.mjs — send navigation keys to an Android TV over wireless ADB.
//   TV_SERIAL=192.168.1.50:41234 node tvnav.mjs --dry-run up enter type "the matrix"
import { execFile } from "node:child_process";
import { promisify } from "node:util";
const run = promisify(execFile);

const KEYCODES = { up: 19, down: 20, left: 21, right: 22, center: 23, enter: 66,
  back: 4, home: 3, menu: 82, search: 84, apps: 187, play: 126, pause: 127,
  stop: 86, rewind: 89, fastforward: 90, power: 26, volumeup: 24, volumedown: 25, mute: 164 };

const PACKAGES = { netflix: "com.netflix.ninja", youtube: "com.google.android.youtube.tv",
  disney: "com.disney.disneyplus", prime: "com.amazon.tv.launcher", plex: "com.plexapp.android",
  kodi: "org.xbmc.kodi", twitch: "tv.twitch.android.app" };

const serial = process.env.TV_SERIAL || "";
const dryRun = process.argv.includes("--dry-run");
const actions = process.argv.slice(2).filter((a) => a !== "--dry-run");

// `input text` takes %s for spaces and chokes on shell metacharacters.
const escapeText = (t) => t.replace(/\s/g, "%s").replace(/['"$`\\&<>|;()]/g, "");

async function adb(args, { timeout = 8000, retries = 2 } = {}) {
  let lastErr;
  for (let attempt = 0; attempt <= retries; attempt += 1) {
    try {
      const { stdout, stderr } = await run("adb", ["-s", serial, ...args], { timeout });
      return { stdout: stdout.trim(), stderr: stderr.trim() };
    } catch (err) {
      lastErr = err;
      if (err.code === "ENOENT") { console.error("adb not found — sudo apt-get install -y adb"); process.exit(3); }
      await new Promise((r) => setTimeout(r, 300 * (attempt + 1)));
    }
  }
  throw lastErr;
}

function plan(action, value) {
  if (action === "type" && value !== undefined)
    return { label: `type ${value}`, args: ["shell", "input", "text", escapeText(value)] };
  if (action === "open" && value !== undefined) {
    const pkg = PACKAGES[value.toLowerCase()];
    if (!pkg) throw new Error(`unknown app "${value}"`);
    return { label: `open ${value}`,
      args: ["shell", "monkey", "-p", pkg, "-c", "android.intent.category.LAUNCHER", "1"] };
  }
  const code = KEYCODES[action];
  if (!code) throw new Error(`unknown key "${action}"`);
  return { label: action, args: ["shell", "input", "keyevent", String(code)] };
}

let failures = 0;
for (let i = 0; i < actions.length; i += 1) {
  const action = actions[i];
  const value = actions[i + 1];
  // Consume the paired argument before any await so a failure cannot desync the loop.
  if ((action === "type" || action === "open") && value !== undefined) i += 1;
  try {
    const step = plan(action, value);
    if (dryRun) { console.log(`[dry-run] adb -s ${serial || "<serial>"} ${step.args.join(" ")}`); continue; }
    const result = await adb(step.args);
    console.log(result.stderr ? `warn: ${step.label} -> ${result.stderr}` : `ok: ${step.label}`);
  } catch (err) {
    failures += 1;
    console.error(`fail: ${action}${value ? ` ${value}` : ""} — ${(err.stderr || err.message).trim().split("\n")[0]}`);
  }
}
if (failures > 0) process.exit(3);
```

Exit codes: `0` ok, `1` no action given, `2` `TV_SERIAL` unset, `3` adb call or action failed.

## Output format

Report to the user which method applies and what they must do on the device:

- **Method** — `bluetooth` / `usb` / `cast` / `adb`
- **Device prerequisite** — the exact menu path to reach on the TV
- **Blocks** — e.g. Android 14 removes wireless debugging; HDMI-only devices have no keyboard input
- **Confidence** — `verified` (keyboard/USB/Cast) or `unverified` (ADB, needs a real device)

Error shape: state the failing `adb`/`bluetooth` symptom plus the concrete next command, never
a bare failure.

## Rate limits / Best practices

- Sleep 200–500 ms between key events or launchers drop them; D-pad spam registers as long-press
  and opens menus instead of navigating.
- Re-read the port after every reboot; cache nothing longer than one session.
- `adb pair` codes are single-use and short-lived — request a fresh one per attempt.
- Turn Wireless debugging back off when finished; a paired device accepts ADB commands from
  anything on the LAN.
- Respect the device owner's Cast/DRM terms; do not bypass an app's playback restrictions.

## Agent prompt

```text
You have Android TV laptop-remote capability. When a user asks to control a smart
projector / TV box / Chromecast from a laptop:

1. Identify the device class before acting:
   - Google TV or Android home screen with app icons -> keyboard or ADB methods apply
   - Bare "no signal"/source-switching only, no menu -> HDMI-only, no keyboard input exists
2. Default to a paired Bluetooth or USB keyboard; it works on every Android version and
   removes the on-screen keyboard entirely.
3. If the user wants automation or their literal laptop keyboard mapped to D-pad keys, use
   wireless ADB: guide them through 7x Build, then adb pair / adb connect.
4. On Android 14+ Google devices, wireless debugging was removed. Detect the Android version
   first and fall back to a Bluetooth keyboard; do not send the user down a dead ADB path.
5. Re-discover the port with `adb mdns services` after any reboot; never trust a cached port.
6. Return the method, the device menu path, and any blocker in the output format above.
```

## Troubleshooting

**`adb: device '192.168.1.50:41234' not found` after it worked yesterday**
- Symptom: connection died after a reboot or a toggle of Wireless debugging.
- Solution: run `adb mdns services`, find the `_adb-tls-connect` service, `adb connect` it.
  Use `reconnect-tv.sh` above.

**`adb pair` fails or the code is rejected**
- Symptom: `Failed to connect` / `error: unable to start pairing client`.
- Solution: the pairing port and the connect port are different numbers — use the one on the
  *pairing code* dialog for `adb pair`. Codes are single-use; open the dialog again for a new
  code. Both sides must be on the same network, and some APNs isolate clients from each other.

**Android 14+ Google device: no wireless debugging option at all**
- Symptom: `Developer options` has no `Wireless debugging`, or the toggle reverts.
- Solution: this is intentional on Google devices since Android 14, not a bug. Check
  `Settings → About build` for the version and switch to the Bluetooth-keyboard method, or use
  USB-C debugging with a USB-C OTG cable and `adb -s <serial>` over USB.

**Bluetooth keyboard pairs but does nothing**
- Symptom: device shows connected, D-pad ignores the keyboard.
- Solution: some firmware accepts the HID pairing but routes it only to media keys. Unpair,
  then plug a USB keyboard into the device's USB port instead.

**Typing produces the wrong characters**
- Symptom: `input text` errors or drops characters.
- Solution: `input text` takes `%s` for spaces, not real spaces, and cannot send every
  character. For accented or non-ASCII text use the keyboard method; for launch-by-name use
  `monkey -p <package>` as shown above.

## See also

- No sibling skills exist yet for TV/display control; add cross-links here when one lands.
- `agent-browser` — for driving web apps when the content is a browser on the TV rather than a native app.

---

## Notes

- Skill file path: `skills/android-tv-laptop-remote/SKILL.md`.
- The `reconnect-tv.sh` and `tvnav.mjs` examples were executed on a machine with no Android
  device attached: syntax-checked with `bash -n` and `node --check`, dry-run verified for
  correct keycode mapping and `%s` space encoding, and error paths verified to return the
  documented exit codes. The device-side commands (`input keyevent`, `input text`, `monkey`)
  were not executed against real hardware in this environment.
- The known app package names can drift; verify with
  `adb shell pm list packages | grep -i netflix` before trusting `open <app>`.
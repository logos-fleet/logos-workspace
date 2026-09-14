This is the Mac venue: macOS with Xcode, iOS simulators, `xcrun simctl`, and adb. Run `source .sandcastle/devices.env` first; it exports `FLEET_ANDROID_SERIALS`, `FLEET_IOS_UDIDS`, and `ANDROID_SERIAL`.

Rules:
- **Claim a device before you use it; never assume the first entry.** Several issue
  pipelines run at once and `FLEET_IOS_UDIDS`/`FLEET_ANDROID_SERIALS` are shared, so two
  agents on one device install the same bundle id over each other and read each other's
  app. Do NOT derive your device from your issue number: two issues can share a remainder
  (17 and 11 both give 2 of 3). Claim one atomically instead — `mkdir` succeeds for exactly
  one caller:

  ```bash
  claim() {  # claim <space-separated ids>; echoes the id it got, or nothing
    mkdir -p /tmp/fleet-devices
    for id in $1; do
      if mkdir "/tmp/fleet-devices/$id" 2>/dev/null; then
        echo "$ISSUE_NUMBER" > "/tmp/fleet-devices/$id/owner"; echo "$id"; return 0
      fi
    done
    return 1
  }
  source .sandcastle/devices.env
  SIM=$(claim "$FLEET_IOS_UDIDS") || { echo "no iOS device free"; }
  PHONE=$(claim "$FLEET_ANDROID_SERIALS") || { echo "no Android device free"; }
  ```

  Claim Android the same way — there are **two** Android devices now, so an unqualified
  `adb shell` fails with "more than one device". Pass `-s "$PHONE"` on every adb call
  rather than relying on `ANDROID_SERIAL`, which names only the first entry. The second
  entry (`HA26QKR0`, Lenovo TB520FU, 1840x2944) is the venue's only Android TABLET: claim
  it by serial for anything layout-shaped, and leave the phone for everything else.

  The helper is **bash** — `for id in $1` relies on word splitting, which zsh does not do.

  **Boot what you claimed, and shut down what you booted.** Nothing in the runner boots a
  simulator, so a run against a shut-down one dies at install — and a booted simulator
  nobody is using costs GBs of RAM. Four of them idling put this Mac into swap: load 865,
  1.3 GB free, and a 3-minute build took ten. So a simulator is booted for the length of
  one run and shut down at the end of it:

  ```bash
  BOOTED_IT=no
  xcrun simctl list devices booted | grep -q "$SIM" || {
    xcrun simctl boot "$SIM" 2>/dev/null; BOOTED_IT=yes; }
  xcrun simctl bootstatus "$SIM" -b            # wait for it either way
  # ... your run ...
  [ "$BOOTED_IT" = yes ] && xcrun simctl shutdown "$SIM"   # only what YOU booted
  ```

  The `BOOTED_IT` guard matters: a simulator that was ALREADY booted when you claimed it
  may be one a person is looking at, and shutting it down takes their app away mid-test.
  Leave those running and shut down only the one you started.

  An Android device that **locks its screen** backgrounds your app and its console goes
  silent with the process still alive — a stall that reads exactly like a hang (it cost
  issue 110 twenty minutes). Both devices are set to stay awake while charging; if one
  locks anyway, `adb -s <serial> shell input keyevent KEYCODE_WAKEUP` then swipe up, and
  the SAME run resumes. Screencap before concluding anything about a silent Android run.

  Release it when you are done (`rm -rf /tmp/fleet-devices/<id>`, after the shutdown above), and release it before
  you finish even if your run failed. If nothing is free, wait and retry a few times; if it
  stays busy, build only and say so in the issue rather than sharing a device. State in the
  issue which device the evidence came from.
- Only touch devices in those allowlists (`adb -s $ANDROID_SERIAL`, simulators by listed UDID). If a list is empty, that platform has no device available to you: build only, and say so in the issue.
- Never `adb reboot/root/sideload`, `fastboot`, `simctl delete/erase`, `sudo`, keychain (`security`), `defaults write`, `launchctl`. A guard hook blocks them.
- Simulators: `xcrun simctl boot <UDID>`, `xcrun simctl install/launch`, `xcrun simctl io <UDID> screenshot out.png`; shut down what you booted.
- `simctl launch --console-pty` (and `devicectl device process launch --console`) returns only when the app exits. For an app that stays alive after printing its result (smoke hosts, Shell preview), never wrap it in a long `timeout`: launch it with its output redirected to a file and in the background, wait for the result line with a bounded loop (`for i in $(seq 1 60); do grep -q "<result marker>" out.log && break; sleep 2; done`), then `xcrun simctl terminate <UDID> <bundle-id>` (or `devicectl device process terminate`).
- The repo launchers (`nix run .#run-*-ios-sim`, `.#run-*-ios-device`) end in exactly such a console-attached launch, so wrapping them in `nohup … &` does not help: the wrapper still only exits when the app does. Use them to build and install, then either terminate the app yourself once its result line has appeared, or launch it directly without a console: `xcrun simctl launch --stdout=/tmp/app.out --stderr=/tmp/app.err <UDID> <bundle-id>`, poll `/tmp/app.out` with a bounded loop, then terminate.
- In a pipeline that writes to a file, use `grep --line-buffered`; plain `grep` holds its output until the input ends, so the file stays empty while the app runs.
- Builds and tests still go through `ws` (`ws build`, `ws test --auto-local`); nix works on darwin.

# NOOP Android: fixing "some days it doesn't pull any data"

Notes from an analysis of `NOOP-android-v9.2.1.apk`, compared against the upstream source
(github.com/ryanbr/noop, `main` at `2a1de760`, latest tag `v12.0.0`).

## 1. Update. You are about 250 strap-sync commits behind

Version 9.2.1 is from July 2026. These fixes in later versions match "works some days, not others":

| Fix | What went wrong in 9.2.1 |
|---|---|
| #1544 (`2efd75fe`) | After a pairing hiccup, NOOP **paused auto-reconnect**. The only thing that undid the pause was opening the app. With the phone in a pocket overnight, nothing was collected (one report: 6 hours). |
| #1734 (`add5191f`) | If heart rate arrived before motion during a partial sync, the night was scored with **no sleep**. That empty result then stuck for up to 15 minutes, or until the next sync. |
| #1066 (`99b5627c`) | Android ran service discovery before the MTU exchange finished, so connections **failed some of the time**. |
| #1063 (`ca4112a3`) | A race in the history-drain handoff could **strand backfill data** in one session. |
| #1196 (`8777202f`) | A re-score after an offload could make history **show as empty** ("0 days") even though nothing had been deleted. |
| #2002 / #2003 | Better detection of a stale Bluetooth pairing, with a re-pair guide when one is found. |

Download the latest APK from https://github.com/ryanbr/noop/releases/latest and install it over your
current version. Make a backup in NOOP first (Settings → Backup). Don't uninstall first, because
that deletes your local data. If Android says "App not installed" because the signature doesn't match,
restore from that backup after a clean install.

## 2. Phone settings (these matter even after you update)

NOOP collects data through a foreground service. If the phone kills that service, the strap keeps
recording but nothing reaches the app until you open it.

1. **Battery → Unrestricted** for NOOP (Settings → Apps → NOOP → Battery).
2. **Samsung:** Settings → Battery → Background usage limits. Make sure NOOP is *not* in "Sleeping apps"
   or "Deep sleeping apps", and add it to **"Never sleeping apps"**. NOOP never prompts for this on
   Samsung (see patch below), so you have to do it yourself.
3. Xiaomi/Oppo/Vivo/Huawei/OnePlus: also turn on **Auto-start** for NOOP.
4. In NOOP Settings, keep **"Keep connected in the background"** on.
5. Keep Bluetooth and Location (scanning) on, and grant "Nearby devices".
6. Only one app can hold the strap. If the **official WHOOP app** is installed and signed in, it can take
   the connection. Force-stop or uninstall it.
7. If the strap's battery ran flat, its clock can reset (data dated 1970). Charge it, then reconnect
   in NOOP so the clock gets set again.
8. Still failing every time? Unpair the strap in Android Bluetooth settings and pair it again in NOOP.
   A stale pairing is a common cause (#1997).

**WHOOP 5.0 / MG owners:** upstream says history backfill doesn't work on these straps yet. The strap
refuses the pairing handshake and doesn't store data offline. Live heart rate works only while the
phone is connected. Data for times the phone wasn't connected is *never* recovered. This is a known,
unsolved limitation, not a bug in your setup.

If it still fails after updating, export the strap log (Settings → Test Centre / debug export) the
morning after a missed night, and attach it to an issue at github.com/ryanbr/noop/issues.

## 3. Code improvement: `patches/0001-android-prompt-samsung-battery-exemption.patch`

`BackgroundHealth.AGGRESSIVE_VENDORS` decides which phones see the "Keep NOOP alive overnight"
battery-exemption prompt and get the Test Centre warning. It lists Xiaomi, Oppo, Vivo, Huawei,
OnePlus, Realme and Meizu. It leaves out Samsung, and a test pins that, even though dontkillmyapp.com
ranks Samsung's One UI ("Sleeping apps"/"Deep sleeping apps") among the worst at killing background apps.
A Samsung user is never prompted, so collection randomly stops on days when One UI puts NOOP to sleep.

The patch adds `"samsung"` to the list and updates the two unit tests that pin it. It applies cleanly
to upstream `main`:

```
git clone https://github.com/ryanbr/noop && cd noop
git apply /path/to/patches/0001-android-prompt-samsung-battery-exemption.patch
cd android && ./gradlew testFullDebugUnitTest --tests '*BackgroundHealthTest' --tests '*AndroidDiagnosticsTest'
```

It's best sent upstream as a PR (the project is PolyForm Noncommercial, and a self-built APK has a
different signature, so installing it would mean uninstalling and losing local data).

# Install with a free Apple ID (External JIT)

Written by [@Morv](https://github.com/Morv)

> **Current as of 2026-11-09.** Experimental. Read [what has been measured](#what-has-been-measured)
> before you spend an evening on it.
>
> This page lives on the `duo-free-route` branch of the fork
> [Morv/wolkaraforever-fork](https://github.com/Morv/wolkaraforever-fork). The two fixes it
> depends on are proposed upstream in
> [pull request 37](https://github.com/jklock/wolkaraforever/pull/37) and are already on this
> branch.

The [supported installation](INSTALLATION.md) needs a paid Apple Developer membership. This
page is the other route: **External JIT**, where a sideloading tool signs the app with a free
Apple ID and a JIT enabler prepares executable memory each time the app opens. Nothing here
needs a paid account.

It has been run on exactly one phone, an iPhone Duo. There the game starts on every launch
and its owner signed in, entered the world and moved around. No session length, frame rate or
memory figure has been recorded on this route.

## What a free Apple ID costs you

- **The app expires every 7 days.** It has to be refreshed from a Mac on the same network.
- **Three sideloaded apps at most.** This route uses all three: AltStore, StikDebug and
  WolkaraForever (it is listed on the phone as **Tolkara**).
- **Less memory.** Free signing cannot request the increased memory limit or extended virtual
  addressing. The game ran at the login screen and for a short session in the world; nobody
  has measured how much headroom is left there.
- **A Mac stays part of the routine**: for the weekly refresh, and once after every phone
  restart (see [Keeping it working](#keeping-it-working)).

## What you need

- An Apple-silicon Mac with Xcode, XcodeGen and Python 3. **Xcode must be new enough to know
  your phone and its iOS version**; an older Xcode builds the app but cannot give the phone
  the developer disk image that the JIT enabler depends on. The measured pair was Xcode 27.2
  with iOS 27.2.
- A free Apple ID. You type its password into AltServer and AltStore yourself; nothing in this
  repository asks for it or stores it.
- On the Mac: [AltServer](https://altstore.io).
- On the phone: AltStore (installed by AltServer),
  [StikDebug](https://github.com/StikDebug/StikDebug) (sideloaded with AltStore), and
  **LocalDevVPN** from the App Store.
- **Developer Mode** on the phone: Settings → Privacy & Security → Developer Mode. The switch
  appears once the first sideloaded app is installed.
- Storage for the game, and your own eligible World of Warcraft account. This repository
  contains no game files and no account access.

## 1. Build the app

```sh
git clone -b duo-free-route https://github.com/Morv/wolkaraforever-fork.git
cd wolkaraforever-fork
cp local.env.example local.env
wolkaraforever/tools/install_prerequisites.sh --check
```

Build from this branch, or from any revision that contains both
`wolkaraforever/patches/0048-external-jit-clear-enabler-arena.patch` and
`wolkaraforever/patches/0049-external-jit-unprepared-high-region.patch`. Upstream `main` does
not have them until its pull request 37 is merged. Without the first the game closes about 35
seconds after it starts on this route; without the second about one launch in three closes the
moment you press Play.

In `local.env`, set `TOLKARA_BUNDLE_ID` to something of your own, for example
`local.wolkaraforever.yourname`. Leave `DEVELOPMENT_TEAM` empty: you have no team, and the
sideloading tool does the signing. Then:

```sh
wolkaraforever/tools/package_ipa.sh
```

The result is `build/platform/Tolkara-unsigned.ipa`. It contains this project's code and no
game files. Keep the default `TOLKARA_SYSTEM_ROOTS=YES` for a build you install yourself: it
adds your Mac's public root certificates so the game can verify the servers it connects to.
See [System certificates](IPHONE.md#system-certificates).

## 2. Install it with AltStore

1. Start AltServer on the Mac and use it to install AltStore on the phone. Trust your Apple ID
   under Settings → General → VPN & Device Management, then turn on Developer Mode.
2. Put `Tolkara-unsigned.ipa` where the phone's Files app can see it. iCloud Drive worked;
   AirDrop into Files did not deliver the file.
3. In AltStore, open **My Apps**, tap **+**, and choose the file. AltServer must be running on
   the Mac, with the phone on the same Wi-Fi or connected by cable.

Install StikDebug the same way from its own release file.

## 3. Set up the JIT enabler, once

1. Connect **LocalDevVPN** and leave other VPNs off.
2. In StikDebug, use its own on-device pairing (Settings → Pair a Device). A pairing file
   made on the Mac is not needed.
3. Make sure the phone has its developer disk image. Connect it to the Mac by cable, unlock
   it, and run:

   ```sh
   xcrun devicectl device info apps --device "Name of your iPhone"
   ```

   Listing the apps is enough to make Xcode's device support mount the image. Until this has
   happened StikDebug reports `Failed to connect debug proxy: ServiceNotFound`.
4. In StikDebug's app list, press and hold **Tolkara**, choose **Assign Script**, and pick
   **universal.js**. StikDebug assigns that script by itself only to apps on its own list, and
   this one is not on it. Without the script the app closes the moment you press Play.

## 4. Start the game

1. LocalDevVPN connected.
2. Open **StikDebug** and tap **Tolkara** in its list. Do not open the app from its home-screen
   icon: it will say *JIT Not Enabled*.
3. First time only: choose **Forever**, the region **your account belongs to**, and a language,
   then **Install** and keep the app open while the game downloads. Set the region before you
   press Install. The game's sign-in region is written once at install, and changing the app's
   setting afterwards does not change it.
4. Press **Play**. The screen can stay black for a while before the login screen appears.

## Keeping it working

- **Every 7 days:** start AltServer on the Mac, then AltStore → My Apps → Refresh All.
- **After restarting the phone:** repeat step 3.3 once, or StikDebug has no developer service
  to talk to.
- **After reinstalling the app:** check that `universal.js` is still assigned.
- **Never delete the app** to fix something. Installing a new build over it keeps the
  downloaded game; deleting it does not.

## Troubleshooting

| What you see | Why | What to do |
| --- | --- | --- |
| StikDebug: `Failed to connect debug proxy: ServiceNotFound` | No developer disk image on the phone, usually after a restart or with an Xcode older than the phone's iOS | Update Xcode if needed, then step 3.3 |
| The app closes the moment you press Play | No script is answering the app's request for executable memory | Assign `universal.js` (step 3.4) |
| *JIT Not Enabled* | The app was opened from its own icon | Open it from StikDebug's list |
| About one launch in three closes the moment you press Play, with the script assigned | The build predates layer patch 0049. StikDebug cannot prepare memory that iOS placed above 64 GB, and reports success anyway | Rebuild from a revision that has the patch and install over the app. Until then, open it again from StikDebug |
| The game window appears, then the app closes about 35 seconds later, every time. `Documents/World of Warcraft/_forever_/Errors/` has a report with `ERROR #132` at address `0x71` | The build predates layer patch 0048, which clears the enabler's memory before the game is placed in it | Rebuild from a revision that has the patch and install over the app |
| AltStore cannot install or refresh | AltServer is not running on the Mac, or the phone cannot reach it | Start AltServer; same Wi-Fi or a cable |
| Sign-in says your game time has expired, or stays at *Connecting*, and the account is active | The game is signing in to another region. The app's region setting only chooses where the game is downloaded from; the login region is the `SET portal` line in `World of Warcraft/_forever_/WTF/Config.wtf`, written once at install | With the game closed, copy that file to the Mac, change the line to your region (`US`, `EU`, `KR` or `TW`) and copy it back |

The app's own logs are in its Documents folder, visible in the Files app under **On My
iPhone**: `wolkaraforever.log` for the launcher and `native-guest.log` for the runtime. The
launcher's lines tagged `[jit]` say what the enabler was asked for and what it granted. When
you report a problem, send those lines and the model, iOS version and game build. Leave out
credentials, pairing records and device identifiers.

## What has been measured

One device, on 2026-11-09:

| | |
| --- | --- |
| Phone | iPhone Duo, iOS 27.2 |
| Mac | macOS 27.2, Xcode 27.2 |
| Signing | AltStore 2.3.2 with AltServer 1.8.3, free Apple ID |
| JIT enabler | StikDebug 3.2.2 with `universal.js`, LocalDevVPN 1.3.0 |
| Game | World of Warcraft Forever 1.60.1 (70862), downloaded in the app |
| Result | With both patches: eight launches, eight starts. In two of them the enabler's first grant was unusable and the retry succeeded. The owner signed in with an Americas account, entered the world and moved around |

Not measured on this route: play of any length, frame rate, memory headroom in the world, a
controller, add-ons, or any other phone or iOS version. By
[this project's own levels](README.md#how-much-a-statement-here-proves) that is one short
*human full-game* run, and the iPhone Duo is in the test matrix as a simulator only.

# Install Fender Studio Pro X in 3 steps

> **Heads up: this is vibe-coded, so expect bugs.** I'm a producer, not a programmer. I build these mods with AI coding tools and test each release on my own setup, on one Studio Pro build. They change how Studio Pro looks and works, so things can still break, especially after a Studio Pro update. Back up your songs first, use it at your own risk, and if anything acts up, run the uninstaller to go back to stock.

Version 2.0.0 comes in **12 packages**. Take only the ones you want; the installer adds anything a package needs. Unofficial fan project, not made or supported by Fender or PreSonus.

## What you need

- Windows 10 or 11.
- Your own licensed **Fender Studio Pro 8.2, build 8.2.0.114903**. This version is made for that exact build only. If your Studio Pro is a different build, the installer says so. Still on Studio Pro 8.1.2? Use release 1.4.1.
- Administrator rights once, because the small loader file (`version.dll`) goes next to `Studio Pro.exe`.
- Themes need nothing else: they come ready-made, with no building on your PC. Python and ImageMagick are no longer needed to get a theme.
- Optional, only for the packages that use them: Python 3.11 or newer (the look options), Python 3.12 or newer and **Claude Code** (the chat assistant and its mixing tools), yt-dlp and ffmpeg (YouTube panel), a DirectX 12 graphics card (faster Sound search; the processor works too), an NVIDIA graphics card (Better stems).

## The 12 packages

| # | Package | What it does | Switches inside (default) | Ticked at first |
|---|---|---|---|---|
| 01 | **Skins** | The Appearance window and ready-made themes (Liquid Glass in seven accent colours, and Original), with options for faders, M/S/R, meters, track header text and the transport bar (Console, Island, Classic). | Live colours (on) | yes |
| 02 | **Home page** | A calm Start page: your sessions, a Quick start card with your templates, music news, your artist card and your audio device. | Session library (on), News (on), Spotify panel (off) | yes |
| 03 | **Name and icon** | "Fender Studio Pro X" in the title bar and the X app icon. | Hide About and Check for Updates (off) | yes |
| 04 | **Track List** | A redesigned Track List with saved views, search and quick show/hide. | none | yes |
| 05 | **Plug-in browser** | Instruments and Effects as a modern panel: filters, favourites, folders and search. | none | yes |
| 06 | **Sound search** | Find samples by name, type, key, tempo, tags and sound; save searches; watch folders; paste at the playhead; play MIDI files on the selected track; preview loops in time with your song. | none | yes |
| 07 | **YouTube panel** | Search YouTube audio in Browser > Cloud, preview it (with a volume slider) and bring it into your song; optional account tabs for your own YouTube lists (needs yt-dlp and ffmpeg). | none | no |
| 08 | **Look extras** | Smooth motion, glass pop-up menus, rounded ruler markers and loop. | Smooth motion (on), Glass pop-ups (on), Ruler markers and loop (on) | yes |
| 09 | **Pitch curves** | Expression mode: draw, edit and shape a pitch line for each note in the piano roll. | none | yes |
| 10 | **AI tools** | A chat assistant that drives Studio Pro through your own Claude Code subscription (any MCP program can use the bridge too), and optional better stems. | Chat assistant & bridge (on), Better stems (off) | no |
| 11 | **Chords panel** | The Chords column beside the piano roll: see and swap chords, ideas that fit your key, build into empty bars, ghost notes before you commit, and a play style. | none | yes |
| 12 | **Small extras** | A free-Hz arpeggiator rate and a fix for some docked plug-in windows. | Free-Hz arpeggiator rate (on), Docked plug-in editor fix (on) | yes |

**Added for you:**

- **Core** is always installed and never listed. It is the mod loader and the web-panel crash guard.
- **Skins** comes along with Home page, Track List, Plug-in browser, Sound search, YouTube panel and Chords panel, because they use its look. The installer says what it added and why.
- **Chords panel** also brings Pitch curves, Sound search and the **bridge** part of AI tools. If you install Chords without ticking AI tools, the bridge runs alone: there is no chat panel, nothing for other programs to connect to, and the chat never starts Claude Code. Tick AI tools and run Install.bat again to get the chat.

## 1. Close and unzip

Close Studio Pro. Unzip the pack anywhere. Every package has its own folder under `packages` with a `README.md` that says what it does, how to use it and how to turn it off.

## 2. Run Install

Double-click **Install.bat**. A window lists the packages, with the switches of each one indented underneath. The recommended ones are ticked already.

- Tick what you want. If a package needs another one, that one is ticked for you and the window tells you why.
- **Before anything changes**, you get a summary: your packages and switches, what was added for you, and what goes into the Studio Pro folder. Press **Install**, or **Cancel** to change nothing.
- It asks for administrator rights once. Everything else goes to `%LOCALAPPDATA%\StudioMods\studio-pro-x`.

**Only one package?** Open its folder (for example `packages\05-plugin-browser\`) and double-click the `Install.bat` inside. Core and anything it needs are added automatically.

To look without changing anything: run `Install.bat -DryRun` from a terminal.

**Upgrading from 1.4.1:** close Studio Pro, make sure it is on 8.2, and run the new Install.bat. Your settings are kept. The old Options > Skins page is replaced by the Appearance window, and your old built skins are not reused: pick a theme again.

## 3. Pick a theme

Start Studio Pro and open the **Appearance** window. Pick a theme (Liquid Glass, with seven accent colours, or Original), press **Apply**, and your song is saved and reopens with the new look. The same window has the look options: the track volume fader, mixer faders, meters, M/S/R buttons, track header text and the transport bar. The fader switches in a few seconds with your song open; the others save and reopen the song.

## The chat assistant (AI tools package)

- It needs **your own Claude Code subscription**. Install Claude Code yourself and sign in with your own account. The pack stores no keys and ships none, and it counts against your own Claude plan limits.
- It starts Claude Code only when you act in the chat panel. If Claude Code is not installed, the chat shows a clear message and nothing else happens.
- The mixing tools need **Python 3.12 or newer**.
- Other programs that speak MCP can use the same bridge. It listens on your PC only, with an access token the pack makes on first start (see the AI tools README).
- The whole AI tools package is unticked at first, so tick it if you want the chat.

## Sound search

- To search by how sounds sound, get the sound model once: download `FSPX-sound-model.zip` (about 190 MB) from the release page, unzip it anywhere and double click `Install-Sound-Model.bat`. Everything else in Sound search works without it, including search by name, key and tempo.
- Tags, saved searches, watched folders and paste at the playhead are in the same panel.

### MIDI files and Play with song (one-time setup)

Sound search can play MIDI files (.mid) on one of your instrument tracks, in your song's key and tempo. The pack adds a MIDI input called **Sound Search** inside Studio Pro for this: there is no driver to install and nothing is added to your song. Studio Pro has to be told about it once:

1. In Studio Pro open **Options > External Devices**, press **Add**, then **New Keyboard**.
2. Set **Receive From** to **Sound Search**. Do not make it the Default Instrument Input: your own keyboard stays your default.
3. In the **Track** menu, switch on **Instrument Input follows Selection**.
4. Select an instrument track and set that track's input to **All Inputs**.

Now click a .mid file in Browser > Files. It plays on the selected track and loops, drums keep their pitch, and it pauses by itself while Studio Pro records. If you hear nothing, the track's input is probably one single device instead of All Inputs, or no instrument track is selected (the player says so under the file name).

**Play with song.** Loops and MIDI files you preview can start and stop with Studio Pro's transport and stay on the beat. The **Play with song** button sits next to Key and Tempo in the player and is on by default. For the tightest timing, set this up once, after steps 1 to 4:

5. Back in **Options > External Devices**, select the keyboard that receives from **Sound Search** and press **Edit**.
6. Set **Send To** to **Sound Search**, tick **Send MIDI Clock** and **Use MIDI Clock Start**, then press **OK**.

If a preview sounds a hair early or late, click the small **0 ms** box next to the button and move it **Earlier** or **Later** (up to 50 ms).

## Home page: Quick start

The Home page has a Quick start card (right column, under News) with your first four templates. Click one and a new song made from it opens at once. Fender's built-in templates and the **All** button open the New Song window. Each one-click song is saved in your Songs folder as *Untitled*.

## YouTube account tabs (optional)

The YouTube panel (package 07, not ticked at first) can show your own YouTube lists: Home, Subs, Liked, Later and History. Set it up once:

1. In Browser > Cloud open the **Account** tab and press **Connect**. The panel tries the browsers on your PC that you are signed in to YouTube with and uses the first one that works.
2. If Connect says "locked", close that browser completely (check the tray too) and press Connect again.
3. Chrome and Brave protect their sign-in, so Connect cannot read them. Use a cookies file instead: add the free extension **Get cookies.txt LOCALLY** to the browser you are signed in with, open youtube.com, press the extension and **Export** (youtube.com only), and save the file. In the panel press **Use a cookies file** and pick it. Keep that file private, and delete it when you are done.
4. Home, Subs, Liked, Later and History now show your own lists. **Sign out** in the panel deletes the saved sign-in.

Your sign-in never leaves your PC: it is stored encrypted for your Windows user, it is sent only to YouTube, and it is not part of this pack. Removing the YouTube panel package, or running the top-level Uninstall.bat, deletes it too. YouTube can limit or block accounts that tools use heavily, so use the account tabs lightly. The panel also needs yt-dlp, ffmpeg and a JavaScript runtime such as deno (`winget install yt-dlp.yt-dlp Gyan.FFmpeg DenoLand.Deno`). Live streams do not preview.

## Better stems

- **Better stems** is off by default. It is a large download, so the installer asks first.
- AI Lab is no longer part of the pack. If you installed it with 1.4.x, it is kept as it is on upgrade, and its uninstaller from 1.4.x still works.

## Back to stock

- **For one start:** hold **Shift** while Studio Pro starts. You get the stock look for that start.
- **Remove everything:** close Studio Pro and run the top-level **Uninstall.bat**. Studio Pro is exactly stock again; any file a package replaced comes back from the copy kept at install. It asks whether to keep the sound model (if you installed it) and your sound index (default: keep).
- **Remove one package:** run the **Uninstall.bat** in that package's folder. Packages that need it are named before anything changes (removing Skins also removes Home page, Track List, Plug-in browser, Sound search, YouTube panel and Chords panel).
- **Change a switch later:** run **Install.bat** again. It shows what is installed now. Untick the switch (or the package) and the summary lists it as "remove".
- Your settings and songs are never moved.

## If something goes wrong

- **Something looks or behaves oddly:** hold Shift at start. If that fixes it, a package or theme is the cause; untick packages one at a time with Install.bat.
- **After a Studio Pro update:** this version is made for build 8.2.0.114903. After an update Studio Pro still starts, stock-looking, with only the mods that do not depend on the build; `sandbox.log` names everything that was skipped.
- **A MIDI file is silent:** check the one-time setup above. The usual cause is a track whose input is one single device instead of All Inputs, or no instrument track selected.
- **Play with song is not quite in time:** do steps 5 and 6 above, then use the small ms box next to the button for the last bit.
- **The chat says Claude Code was not found:** install Claude Code and sign in, then try again. If it still cannot find it, the AI tools README says how to point the chat at it.
- **YouTube Connect says "locked" or "can't be read":** close that browser completely and try again, or use a cookies file. A YouTube preview that never starts is usually a live stream.
- **Still stuck:** share your Studio Pro build and the log `%LOCALAPPDATA%\StudioMods\studio-pro-x\sandbox.log`.

## Build your own

Want to build your own mods? Give the files in **For your AI agent** to Claude or another AI agent. Work in a copy of Studio Pro, never your only install.

*A free fan project. Not made or supported by Fender or PreSonus. Studio One and Studio Pro are trademarks of their owners.*

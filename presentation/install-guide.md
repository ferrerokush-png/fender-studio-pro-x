# Install Fender Studio Pro X in 3 steps

> **Heads up:** these mods are completely vibe-coded. I'm not a programmer. I built all of this with AI and I honestly don't know what I'm doing, so expect bugs and things that break. Back up your songs first, use it at your own risk, and if anything acts up, run the uninstaller to go back to stock.

<!-- DRAFT, version 1.3.0 (10 packages, 2026-10-17). -->

Version 1.3.0 comes in **10 packages**. Take only the ones you want; the installer adds anything a package needs.

## What you need

- Windows 10 or 11 (Windows 11 for the glass menus).
- Your own licensed **Fender Studio Pro 8**. Tested on 8.1.2 (build 8.1.2.113407).
- For the skins: **Python 3.11 or newer** from python.org, then once: `py -m pip install numpy pillow fonttools brotli`, and **ImageMagick 7** (`winget install ImageMagick.ImageMagick`). The installer checks both and tells you what is missing.
- Optional, only for the packages that use them: yt-dlp and ffmpeg (YouTube panel), an NVIDIA graphics card (AI Lab, Better stems), an MCP client or your own AI key (chat assistant).

## The 10 packages

| # | Package | What it does | Switches inside (default) | Ticked at first |
|---|---|---|---|---|
| 01 | **Skins** | Liquid Glass and Graphite, built on your PC from your own Studio Pro, with a Skins page in Options and live colour changes. | Skins page (on), Live colours (on) | yes |
| 02 | **Home page** | A calm Start page: your sessions, a Quick start card with your templates, music news, your artist card and your audio device. | Session library (on), News (on), Spotify panel (off) | yes |
| 03 | **Name and icon** | "Fender Studio Pro X" in the title bar and the X app icon. | Hide About and Check for Updates (off) | yes |
| 04 | **Track List** | A redesigned Track List with saved views, search and quick show/hide. | none | yes |
| 05 | **Plug-in browser** | Instruments and Effects as a modern panel: filters, favourites, folders and search. | none | yes |
| 06 | **Sound search** | Find samples by name, type, key and tempo in Browser > Files, play MIDI files on the selected track, and preview loops in time with your song. | none | yes |
| 07 | **YouTube panel** | Search YouTube audio in Browser > Cloud, preview it and bring it into your song (needs yt-dlp and ffmpeg). | none | no |
| 08 | **Look extras** | Smooth motion, glass pop-up menus, rounded ruler markers and loop. | Smooth motion (on), Glass pop-ups (on), Ruler markers and loop (on) | yes |
| 09 | **Pitch curves** | Draw and edit per-note pitch curves in the piano roll. | none | yes |
| 10 | **AI tools** | A chat assistant that can drive Studio Pro (any MCP client too), the AI Lab tab, and optional better stems. | Chat assistant & bridge (on), AI Lab (on), Better stems (off) | no |

**Added for you:**

- **Core** is always installed and never listed. It is the mod loader and the web-panel crash guard.
- **Skins** comes along with Home page, Track List, Plug-in browser, Sound search and YouTube panel, because they use its look. The installer says "Skins added: ... needs it".
- **Chat assistant & bridge** is locked on while AI Lab is on. AI Lab's "Add to song" button uses it.

## 1. Close and unzip

Close Studio Pro. Unzip the pack anywhere. Every package has its own folder under `packages` with a `README.md` that says what it does, how to use it and how to turn it off.

## 2. Run Install

Double-click **Install.bat**. A window lists the packages, with the switches of each one indented underneath. The recommended ones are ticked already.

- Tick what you want. If a package needs another one, that one is ticked for you and the window tells you why.
- **Before anything changes**, you get a summary: your packages and switches, what was added for you, what goes into the Studio Pro folder and which skins will be built. Press **Install**, or **Cancel** to change nothing.
- It asks for administrator rights once, because the small loader file (`version.dll`) goes next to `Studio Pro.exe`. Everything else goes to `%LOCALAPPDATA%\StudioMods\studio-pro-x`.

**Only one package?** Open its folder (for example `packages\05-plugin-browser\`) and double-click the `Install.bat` inside. Core and anything it needs are added automatically.

## 3. Build and pick

The skins are built on your PC from **your own** Studio Pro. That takes about 2 to 4 minutes and needs Python and ImageMagick (see above). Start Studio Pro, open **Options > Skins**, pick a skin and an accent, press **Restart now**. The same page has the **Glass tint** slider and the **Lighting** row: One light (a soft light in your accent colour, the default) or Cool zones (a quiet cool light in each panel). Tint and lighting show at the next start.

## MIDI files and Play with song in Sound search (one-time setup)

Sound search can play MIDI files (.mid) on one of your instrument tracks, in your song's key and tempo. The pack adds a MIDI input called **Sound Search** inside Studio Pro for this: there is no driver to install and nothing is added to your song. Studio Pro has to be told about it once:

1. In Studio Pro open **Options > External Devices**, press **Add**, then **New Keyboard**.
2. Set **Receive From** to **Sound Search**. Do not make it the Default Instrument Input: your own keyboard stays your default.
3. In the **Track** menu, switch on **Instrument Input follows Selection**.
4. Select an instrument track and set that track's input to **All Inputs**.

Now click a .mid file in Browser > Files. It plays on the selected track and loops, drums keep their pitch, and it pauses by itself while Studio Pro records. If you hear nothing, the track's input is probably one single device instead of All Inputs, or no instrument track is selected (the player says so under the file name).

### Play with song

Loops and MIDI files you preview can start and stop with Studio Pro's transport and stay on the beat. The **Play with song** button sits next to Key and Tempo in the player and is on by default. While your song plays, a loop or MIDI file you click starts in time; press Play while one previews and it joins in; press Stop and it stops. One-shots always play on their own. For the tightest timing, set this up once, after steps 1 to 4:

5. Back in **Options > External Devices**, select the keyboard that receives from **Sound Search** and press **Edit**.
6. Set **Send To** to **Sound Search**, tick **Send MIDI Clock** and **Use MIDI Clock Start**, then press **OK**.

The pack adds a matching **Sound Search** MIDI output for this (no driver). If a preview sounds a hair early or late, click the small **0 ms** box next to the button and move it **Earlier** or **Later** (up to 50 ms).

## Home page: Quick start

The Home page has a Quick start card (right column, under News) with your first four templates. Click one and a new song made from it opens at once. Fender's built-in templates and the **All** button open the New Song window. Each one-click song is saved in your Songs folder as *Untitled*.

## AI tools

- **AI Lab** always comes with the **Chat assistant & bridge**, because its "Add to song" button uses it.
- **Better stems** is off by default. It is a large download, so the installer asks first. Nothing downloads without your OK, and AI Lab's own models are optional downloads you pick one by one.
- The whole AI tools package is unticked at first, so tick it if you want it.

## Back to stock

- **For one start:** hold **Shift** while Studio Pro starts. You get the stock look for that start.
- **Remove everything:** close Studio Pro and run the top-level **Uninstall.bat**. Studio Pro is exactly stock again; any file a package replaced (the Moises tab, the SoundCloud panel) comes back from the copy kept at install.
- **Remove one package:** run the **Uninstall.bat** in that package's folder. Packages that need it go too (removing Skins also removes Home page, Track List, Plug-in browser, Sound search and YouTube panel); it tells you before it does anything.
- **Change a switch later:** run **Install.bat** again. It shows what is installed now. Untick the switch (or the package) and the summary lists it as "remove".
- Your settings and songs are never moved. Saved views, favourites and folders are kept.

## If something goes wrong

- **Something looks or behaves oddly:** hold Shift at start. If that fixes it, a package or skin is the cause; untick packages one at a time with Install.bat.
- **After a Studio Pro update:** run Install.bat again so the skins are rebuilt from the new version. Anything that needs an exact build switches itself off until it is updated (the Sound Search MIDI input is the exception: it patches nothing by address).
- **A MIDI file is silent:** check the one-time setup above. The usual cause is a track whose input is one single device instead of All Inputs, or no instrument track selected.
- **Play with song is not quite in time:** do steps 5 and 6 above (Send To: Sound Search, Send MIDI Clock, Use MIDI Clock Start), then use the small ms box next to the button for the last bit.
- **Skin missing on the Skins page:** it appears after its first successful build. Close Studio Pro and run Install.bat again.
- **Glass tint or lighting did not change:** both are applied when Studio Pro starts. Use **Restart now** on the Skins page, or close and start Studio Pro.
- **The skin build will not start:** close Studio Pro first, and check Python and ImageMagick.
- **Still stuck:** share your Studio Pro version and the log `%LOCALAPPDATA%\StudioMods\studio-pro-x\sandbox.log`.

## Build your own

Want to build your own mods? Give the files in **For your AI agent** to Claude or another AI agent.

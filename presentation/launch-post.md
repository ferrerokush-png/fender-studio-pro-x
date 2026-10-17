# Fender Studio Pro X: a free community expansion for Studio Pro 8

> **Heads up:** these mods are completely vibe-coded. I'm not a programmer. I built all of this with AI and I honestly don't know what I'm doing, so expect bugs and things that break. Back up your songs first, use it at your own risk, and if anything acts up, run the uninstaller to go back to stock.

<!-- DRAFT. Nothing here is published. The user decides what is shared and where. Matches deck, version 1.2.1 (10 packages, 2026-10-17). -->

I have been shaping Studio Pro 8 into the DAW I always wanted to work in, and I am sharing the result for free. **Fender Studio Pro X** is a set of skins and mods that sit on top of **your own, licensed** Studio Pro 8 and give you your own version of it.

It is free, it installs on the copy you already own, it never touches licensing, sign-in or activation, and one click takes you back to plain Studio Pro. No Fender files are shipped: the skins are built on your PC from your own copy of the program.

**Version 1.2.1 comes in 10 packages.** Tick the ones you want, with a few switches inside each. The installer adds what a package needs and shows you a summary before it changes anything.

**New in 1.2.1:** a cleaner Sound search look: calmer colours, a solid player, clearer rows.

**New in 1.2.0:** Sound search is much faster and now plays MIDI files on the instrument track you have selected (a one-time setup, written out in its README), and the Home page has a Quick start card: one click starts a new song from your own template.

Huge thanks to the Studio One team at Fender. This only exists because the program underneath is so good to work in.

![Fender Studio Pro X with the Liquid Glass skin](deck/assets/hero.jpg)

*Screenshots were taken before the final 1.1.0 polish, so a few details look slightly different.*

## The 10 packages

**1. Skins.** Liquid Glass and Graphite, built on your PC from your own Studio Pro. Liquid Glass is a dark, frosted look for the whole window with 8 accent colours and 4 presets (32 looks). One soft light in your accent colour comes from the top left, the Browser has its own glow from its bottom edge, and a Glass tint slider goes from no tint to strong glass. Prefer something cooler? The Lighting row offers Cool zones, a quiet cool light in each panel (the newest part, so the least tested). Smooth backgrounds, no film grain. The accent marks selection and sliders only; record, mute, solo and meters keep their usual colours. Track headers keep Studio Pro's own shapes, meter and colour effect, the transport stays clean whatever you show or hide, scrollbar handles stay visible at every zoom, and bypassed plug-ins are clearly dimmed in the mixer. A Skins page in Options lets you pick, with live colour changes. Want your own? One short text file: pick a base, list only what you want different, run one command. No coding. *Switches: Skins page, Live colours.*

**2. Home page.** A calm Start page: your sessions, a Quick start card (one click starts a song from your template), music news, your artist card and your audio device. Rename sessions right there too. *Switches: Session library, News, Spotify panel (off at first). Brings Skins with it.*

**3. Name and icon.** "Fender Studio Pro X" in the title bar and the X app icon. *Switch: Hide About and Check for Updates (off at first).*

**4. Track List.** A redesigned Track List: eye buttons, search, filters, folders, colours and saved views. *Brings Skins with it.*

**5. Plug-in browser.** Instruments and Effects as a modern panel: filters, favourites, your own folders, search and a type icon on every row. *Brings Skins with it.*

**6. Sound search.** Find samples by name, type, key and tempo in the Files tab, preview in time with the song, drag it in. MIDI files play too, on the track you have selected. *Brings Skins with it.*

**7. YouTube panel** (off at first). Search or paste a link in the Cloud tab, preview, and bring the audio into the song's folder. Uses yt-dlp and ffmpeg, which you install yourself. Only use audio you have the right to use. *Brings Skins with it.*

**8. Look extras.** Smooth motion (short fades, no bounce), glass pop-up menus, rounded ruler markers and loop. *Switches: Smooth motion, Glass pop-ups, Ruler markers and loop.*

**9. Pitch curves.** Draw and edit per-note pitch curves in the piano roll.

**10. AI tools** (off at first). A chat assistant that can drive Studio Pro, the AI Lab tab, and optional better stems.
- **Chat assistant & bridge.** Let the AI model of your choice (any MCP client, or the built-in chat panel with your own key) read the open song and make changes. It listens on your PC only, and every edit can be undone.
- **AI Lab.** A Browser tab for Stems, Voice, Choir and Generate that runs on your own PC. Default models are MIT or Apache licensed. Nothing is uploaded unless you add a key for an online service yourself. Its "Add to song" button uses the chat assistant & bridge, so the two always come together.
- **Better stems** (off at first). Studio Pro's own Audio > Separate Stems, same menu, dialog and undo, with open MIT-licensed models behind it. Large download, so it asks first.

Underneath it all sits **Core**, always installed: the mod loader and a crash guard for web panels.

## The promise

- Licensing stays untouched. No changes to sign-in, activation, subscriptions or the store.
- No Fender files are shipped. Skins are built on your PC from your own copy.
- Where a package replaces a stock file (the Moises tab, the SoundCloud panel), the original is kept and Uninstall puts it back.
- Hold **Shift** while Studio Pro starts for the stock look. Run the top-level **Uninstall.bat** and Studio Pro is exactly stock again. Each package also has its own Uninstall.bat.
- Our code is MIT licensed. Third-party models and tools keep their own licences.

## Honest limits

- Windows only. Tested on Studio Pro 8.1.2 (build 8.1.2.113407). On other builds, anything that needs an exact match switches itself off and says so in the log.
- Skins need Python 3.11+ and ImageMagick 7; the installer checks and tells you what is missing.
- Some parts of Studio Pro are drawn by the program itself and a skin cannot change them.
- Early software. Back up your projects as usual.

## Get it

Install steps: see the [install guide](install-guide.md). Questions, bugs and your own themes: see [community](community.md).
Download (free): **https://github.com/ferrerokush-png/fender-studio-pro-x** (Releases has the latest zip). Take the tour: https://ferrerokush-png.github.io/fender-studio-pro-x/

*Made by Roche. Free and not for sale. Studio One and Studio Pro are trademarks of their owners.*

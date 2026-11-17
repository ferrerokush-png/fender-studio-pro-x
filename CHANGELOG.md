# Fender Studio Pro X 2.0.0

## Fender Studio Pro X 2.0.0

Big update, for Studio Pro 8.2.0.114903. Built by a producer with AI coding tools: it is vibe-coded and tested on one setup, so expect bugs, back up your songs first, and keep the previous version handy. If your Studio Pro is a different build, the pack starts Studio Pro stock-looking and safe, and says what it turned off. 1.4.1 stays available for Studio Pro 8.1.2.

### What is new

- **Appearance window.** Pick a ready-made theme (Liquid Glass with seven accent colours and Original), press Apply, and your song is saved and reopens with the new look. Themes are installed ready-made, so there is no waiting for a build on your PC any more.
- **Skin options.** Change the track volume fader, mixer faders, meters (including a spectrum look), M/S/R buttons, track header text and the transport bar (Console, Island or Classic). The fader switches in a few seconds with your song open; the others save and reopen the song.
- **Chords panel** in the piano roll: see and change the chords of what you are playing, ideas that fit your key, preview through your own instrument, ghost notes drawn in the piano roll before you commit, and a play style (your own timing, strum or arpeggio). Chord statistics come from the Chordonomicon dataset (CC BY-NC 4.0, credit in the package): free use only.
- **Pitch curves** are now a full Expression mode: draw, edit and shape per-note pitch lines, one Undo per gesture.
- **Sound search:** find sounds by what they sound like (needs a small sound model you get once: download `FSPX-sound-model.zip`, about 190 MB, from the release page, unzip it anywhere and double click `Install-Sound-Model.bat`; everything else in Sound search works without it), tags, saved searches, folder watching, paste at the playhead; works on your graphics card or the processor.
- **Chat assistant:** talk to your song using your own Claude Code subscription (install Claude Code and sign in yourself; no keys are stored or shipped). It can read and change tracks, mixing, sends, plug-in settings, MIDI and automation through the same buttons you use. Off at first. Optional Python for the plug-in helper.
- **Small extras:** free-Hz arpeggiator rate, and a fix that makes some docked plug-in windows (for example Vovious) show up.

### Packages

12 packages now (was 10). New: **11 Chords panel** (adds Pitch curves and the AI tools bridge for you) and **12 Small extras**. Reworked: 01 Skins (Appearance window, ready-made themes, options), 06 Sound search, 09 Pitch curves, 10 AI tools (Claude Code based chat and optional better stems).

### Upgrading

Run Install.bat again; your settings are kept. Python and ImageMagick are no longer needed for themes. Python 3.11 or newer is only needed for the look options, and Python 3.12 or newer for the chat's mixing tools. An AI Lab installed with 1.4.x is kept as it is; AI Lab is no longer part of the pack.

### Good to know

Windows 10/11, your own licensed Studio Pro 8, administrator once for version.dll. Nothing touches sign-in, licence or activation. Uninstall.bat puts Studio Pro back to stock. The chat spends your own Claude plan limits.

A free fan project. Not made or supported by Fender or PreSonus. Thank you to the Studio One team.

<!-- Owner checks before release (not for the public text):
  - "Fix switches in a few seconds / rest reopen the song": measured 2026-11-13; re-check on the final build.
  - Promise wording "no Fender sign-in, licence or program code shipped" replaces the 1.4.1 line "no Fender files shipped" because themes now ship prebuilt (owner's 2026-11-06 risk call).
-->

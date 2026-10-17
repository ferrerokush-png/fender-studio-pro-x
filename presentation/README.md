# Fender Studio Pro X: presentation pack

> **Heads up:** these mods are completely vibe-coded. I'm not a programmer. I built all of this with AI and I honestly don't know what I'm doing, so expect bugs and things that break. Back up your songs first, use it at your own risk, and if anything acts up, run the uninstaller to go back to stock.

<!-- DRAFT. Local files only; nothing is published. The user decides what is shared, where and when. Version 1.2.1 (10 packages). -->

**Fender Studio Pro X** (version 1.2.1) is a free community expansion that gives you your own version of Fender Studio Pro 8. It comes in **10 packages**, each with a few switches inside: Skins (Liquid Glass and Graphite), Home page, Name and icon, Track List, Plug-in browser, Sound search, YouTube panel, Look extras, Pitch curves and AI tools (chat assistant and bridge, AI Lab, better stems). It installs on your own licensed copy, never touches licensing, ships no Fender files, and uninstalls in one click.

## What is in this folder

| File | What it is |
|---|---|
| `Fender Studio Pro X - Presentation.html` | The deck, one self-contained file. Open in Edge or Chrome; arrow keys or click to move, F for full screen. |
| `Fender Studio Pro X - Presentation.pdf` | The same 17 slides as a PDF (1920 x 1080 pages). |
| `Fender Studio Pro X - Feature Sheet.pdf` | One A4 page: the 10 packages, install steps, the promise. |
| `launch-post.md` | The announcement text. |
| `install-guide.md` | Install, the 10 packages, back to stock, troubleshooting. |
| `community.md` | Where people talk, house rules, how to report a bug. |
| `demo-shotlist.md` | Shot list for a 2:30 demo video, matched to the deck slides. |

## The deck, slide by slide

1. Cover  2. What it is (with the heads-up)  3. Before and after  4. Liquid Glass  5. Skins page and making your own  6. Home page  7. Track List and Plug-in browser  8. Sound search and YouTube panel  9. AI Lab  10. Assistant  11. The small things  12. New in 1.1.0  13. The 10 packages at a glance  14. Install  15. Back to stock  16. Our promise  17. Thanks and credits

Each slide names its package in the bottom corner. Every picture is a real screenshot of the running program (no mock-ups), taken before the 1.1.0 polish on slide 12 (no film grain, one light or Cool zones, stock-shaped track headers, dimmed bypassed plug-ins), so a few details look slightly different. The 1.2.1 change (a cleaner Sound search look) and the 1.2.0 changes (a faster Sound search that plays MIDI files, and the Home page Quick start card) are written up on slides 8, 12 and 13 and on the feature sheet; their pictures are not new. Personal branding and anything not in the public pack are cropped out.

## The 10 packages

| # | Package | Ticked at first |
|---|---|---|
| 01 | Skins | yes |
| 02 | Home page | yes |
| 03 | Name and icon | yes |
| 04 | Track List | yes |
| 05 | Plug-in browser | yes |
| 06 | Sound search | yes |
| 07 | YouTube panel | no |
| 08 | Look extras | yes |
| 09 | Pitch curves | yes |
| 10 | AI tools | no |

Core (the mod loader and the web-panel crash guard) is always installed. Home page, Track List, Plug-in browser, Sound search and YouTube panel bring Skins with them. See `install-guide.md` for the switches inside each package.

## Build your own

Want to build your own mods? Give the files in **For your AI agent** to Claude or another AI agent.

## Editing

The deck and the feature sheet are generated from editable source files (`deck.src.html`, `feature-sheet.src.html`) by a small build script that inlines fonts and pictures and prints both PDFs with Edge. This folder holds the finished files; the sources stay with the maintainer.

## Credits

Made by Roche, with thanks to the Studio One team at Fender. Open models and tools: Mel-Band RoFormer (Kim), HTDemucs (Meta), All-In-One, BTC chords, Whisper, Applio, VCTK starter voices (CC BY 4.0), ACE-Step 1.5, yt-dlp, ffmpeg, SQLite, Microsoft WebView2. Fonts: Inter and JetBrains Mono (SIL Open Font License). Our code: MIT.

*Free and not for sale. Studio One and Studio Pro are trademarks of their owners.*

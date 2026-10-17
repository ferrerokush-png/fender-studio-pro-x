# Fender Studio Pro 8 internals, for AI agents

A field guide for an AI coding agent that will build mods for Fender Studio Pro 8 (formerly PreSonus Studio One)
on Windows, on top of the "Fender Studio Pro X" mod pack. It condenses several weeks of reverse engineering into
facts you can act on: file formats, addresses, hooks that work, hooks that do not, and the traps that cost hours.

## How to use this file (read first)

- **Verified build:** every fact here was checked on **Studio Pro 8.1.2.113407, Windows x64** (Windows 11), on one
  machine. Read the running build from `Studio Pro.exe` (Properties > Details, or the FileVersion resource). The pack's
  loader logs it at every start: `app build 8.1.2.113407 matches [app] build=...`.
- **Addresses and offsets hold for that build only.** Formats (skin packages, script packages, skin XML, song files),
  window classes, exports, command names and the JavaScript API are much more stable, but still verify them.
  On another build, re-find every address with the routes in section 4 before patching anything, and make every
  patch check the bytes or pointer it expects first (log and do nothing on a mismatch).
- **Notation:** VAs are Ghidra defaults. `cclgui.dll` image base `0x180000000`, `Studio Pro.exe` image base
  `0x140000000`. At runtime: `address = module_base + (VA - image_base)`. Vtable "slot `+0x188`" = byte offset.
  CCL colours are `0xAABBGGRR` (R in the low byte).
- **Working rules (non-negotiable):**
  1. Work in a **copy** of the app folder (a sandbox), never in the real install under `C:\Program Files\Fender\Studio Pro 8`.
     Test mods there first.
  2. **Always keep a route back to stock**: every mod gets an `enabled=0` ini switch, honours **Shift held at start**
     (= off for that launch), and deleting its DLL (or the loader's `version.dll`) must return the app to stock.
     Never write to `Studio Pro.exe` or the app's DLLs on disk; patch in memory only.
  3. Mods **never touch licensing, activation, sign-in or subscription code or checks**, and never help anyone get
     around them. This guide deliberately contains nothing about those areas. If a task drifts there, stop.
  4. Run **one Studio Pro at a time per audio device** (two copies on one audio device freeze each other), unless you
     set up the instance switch in section 2.6 and a separate device.
  5. Verify with text before screenshots: `sandbox.log`, window lists, pixel sampling of saved captures,
     the bridge's `get_status`/`run_script` (section 12).
- **Disclaimer:** these notes were produced by a non-programmer working with AI agents. Facts were checked on one build
  on one machine; some may be wrong. Verify before you rely on them.

## Contents

1. The 60-second model
2. The mod pack's loader: layout, settings, log, safe mode, build guard
3. Writing a native mod (DLL) for the loader
4. Re-finding things on another build
5. Skins: packages, XML, colour tokens, the theme builder
6. Views, controls and painting (cclgui)
7. Animation engine
8. Windows, menus, pop-ups and dialogs
9. Web panels (WebView2 inside CCL views)
10. The command system
11. JavaScript edit tasks (read and change the open song)
12. The MCP bridge (llmbridge) and other mods' exports
13. Files and data formats (song, settings, recents, keys)
14. Exe internals worth knowing (undo journal, envelopes, ruler, dividers, startup)
15. Testing and tooling recipes
16. Address reference (8.1.2.113407)
17. What each pack mod does and which technique it uses
18. Gotchas, collected

---

## 1. The 60-second model

- **Framework:** Studio Pro is built on PreSonus' private C++ framework **CCL**, shipped as five DLLs next to the exe:
  `ccltext.dll`, `cclsystem.dll`, `cclsecurity.dll`, `cclnet.dll`, `cclgui.dll`. Almost all UI code is in
  **`cclgui.dll`**. There is almost no RTTI; classes are found through CCL's own type registry (section 4.2).
  The exe loads the five with `CCLModuleMain(h, 1)` in that order and unloads them in reverse.
- **Look:** 16 skin packages are embedded as `RCDATA` resources in `Studio Pro.exe` (`DEFAULT.SKIN`,
  `FENDERBASE.SKIN`, ...). Each is a CCL "PACKAGEF" archive of XML + images: colour tokens with Appearance-slider
  curves, Styles, vector Shapes, image atlases, Triggers/Animations, Perspectives (page layouts). `cclsystem.dll` reads
  them with `FindResourceW/SizeofResource/LoadResource/LockResource`, which is how the pack's loader serves rebuilt
  skins from disk (section 5.6).
- **Rendering:** Direct2D (`ID2D1DeviceContext`) on Direct3D 11, composited with DirectComposition. One shared device
  context paints every window on the UI thread. GPU animations are built with the Windows Animation Manager (WAM)
  and turned into DirectComposition curves.
- **Windows:** main window class `CCLWindowClass`; menus, drop-downs and context menus are **new layered
  `CCLDialogClass` windows every time**; dialogs are captioned `CCLDialogClass` windows; a hidden
  `CCLMessageWindowClass` window titled "Studio Pro" receives the single-instance hand-off.
- **Behaviour:** a **command table** (category + name, about 280 built in, ~1,500 with user macros) drives menus and
  keys; the song model is reachable from **JavaScript edit tasks** run by the app's own script engine (no reverse
  engineering of the document model needed for most song edits).
- **Web views:** CCL has a `WebBrowserView` that can host Microsoft Edge WebView2, but Studio Pro ships with WebView2
  switched off (it falls back to Internet Explorer). The pack's web-panel mods switch it on per view and host HTML
  panels in skin-defined slots.
- **Settings and data:** `%APPDATA%\Fender\Studio Pro 8\` (`Studio Pro.settings`, `RecentDocuments.settings`,
  `user.keyscheme`, `WindowState.settings`, `x64\Plugins-en.settings`...). Songs are zip files (`.song`).

---

## 2. The mod pack's loader: layout, settings, log, safe mode, build guard

### 2.1 What the pack installs

The pack ("Fender Studio Pro X") ships its own compiled DLLs, a skin builder (Python) that patches the user's own
stock skin, helper programs and installers. It ships **no** Fender/PreSonus files. On the user's PC:

| Where | What |
|---|---|
| `<app folder>\version.dll` | the mod loader (a proxy `version.dll`; Windows loads it from the exe folder first) |
| `<app folder>\Scripts\llmbridge.package`, `tracklist.package`, `pluginbrowser.package` | script packages for three mods (written by the installer, own code) |
| `%LOCALAPPDATA%\StudioMods\studio-pro-x\` = **`<root>`** | `sandbox.ini`, `mods\`, `skins\`, `skin-tools\`, `skin-src\` (extracted at install time from the user's own exe), `skin-work\`, `sandbox.log`, `uninstall\` |
| `<root>\WebView2\` | WebView2 user data for the web panels (the loader sets `WEBVIEW2_USER_DATA_FOLDER`, because a normal user cannot write inside Program Files) |

The root can be overridden with the environment variable **`STUDIO_PRO_X_ROOT`** (useful for a test sandbox).
No `sandbox.ini` in the root = the loader stays idle (it only forwards the real `version.dll` exports).
Uninstall = delete `version.dll` (plus the 3 packages); the app is then byte-for-byte stock.
The pack's mods are grouped into packages for the installer, but on disk every mod is still one `mods\<name>.dll`
with its own `<name>.ini` (and often a `mods\<name>\` folder for pages and data).

### 2.2 What the loader does at start

1. Forwards the 17 real `version.dll` exports (it loads `%SystemRoot%\System32\version.dll`).
2. Reads `<root>\sandbox.ini`.
3. In `Studio Pro.exe` (and `PlugInScanner.exe`): optional user-folder redirect (`[paths] redirect=1`, developer option:
   hooks cclsystem's `SHGetSpecialFolderPathW` for AppData, LocalAppData, Documents, Music, Desktop, Public Documents
   into `data=`; ProgramData and Common Files are never redirected) and an optional offline guard (`[network] offline=1`:
   cclnet/ucnet `getaddrinfo`, `connect`, `sendto` refuse non-local hosts except `allow=`). The pack installs with
   both **off**: the user's normal settings and songs, and the app stays online exactly as stock.
4. Only in `Studio Pro.exe`: instance suffix (`[instance] id=N`, section 2.6), **build guard** (2.4), **skin override**
   (5.6), optional fast quit (`[shutdown] fastquit=1`), then starts a thread that loads the mods.
5. The mod thread runs once the loader lock is free (all startup DLLs initialised). It first makes the import-address
   tables of the exe and every DLL in the app folder writable (a fix for a race, see 3.4), then
   `LoadLibraryW(<root>\mods\*.dll)` in `FindFirstFile` order (alphabetical on NTFS), logging
   `mod loaded: x.dll` or `ERROR mod failed: x.dll (error N)`.

### 2.3 `sandbox.ini` keys

```ini
[paths]
redirect=0        ; 1 = mods' own data folder (developer option)
data=             ; your %USERPROFILE%\AppData (written by the installer)
app=              ; the Studio Pro folder (written by the installer; mods read it to find <app>\Scripts)
[network]
offline=0
allow=
[skin]
enabled=0         ; 1 = serve skins\<dir>\*.SKIN instead of the exe's resources
dir=              ; e.g. skins\liquidglass
[shutdown]
fastquit=0
[app]
build=8.1.2.113407   ; build guard; "8.1.2" accepts any 8.1.2.x
force=0              ; 1 = load skin and all mods on another build (at your own risk)
portable=appicon,brandtitle,hidehelp,glasspopups   ; mods that may load on another build
[instance]
id=0              ; N>0 = run a second copy side by side (section 2.6)
```

`GetPrivateProfileString` strips a value's outer quotes: never store `"..." "..."` command lines in an ini.

### 2.4 Build guard

At start the loader reads the running exe's FileVersion straight from its mapped `VS_VERSIONINFO` (it cannot call
`GetFileVersionInfo`: it *is* version.dll). If it differs from `[app] build=`: WARNING in the log, **no skin override**
(and `SandboxRegisterSkinFilter` / `SandboxSkinFlush` refuse), and only mods named in `[app] portable=` load; every
other mod logs `mod skipped: x.dll (app build mismatch; add it to [app] portable= or set [app] force=1)`.
**Consequence for your mod:** if it patches by address, leave it out of `portable=`; if it only uses exports, window
messages, imports by name or resources by name, it may be added there.

### 2.5 Safe mode and off switches

- **Shift held while Studio Pro starts** = stock skin, and every well-behaved mod stays off for that launch
  (`GetAsyncKeyState(VK_SHIFT) & 0x8000` in the mod's setup).
- `<mod>.ini enabled=0`, or delete `mods\<mod>.dll` (close Studio Pro first: loaded DLLs are locked; a loaded DLL can be
  renamed, not overwritten).
- `[skin] enabled=0` = stock skin.

### 2.6 Running a second copy (test sandbox)

- The exe's single-instance check is a CCL "IPC semaphore": in `cclsystem.dll` a plain `CreateMutexA(NULL, 0, name)`
  with a fixed name (no `Global\`/`Local\` prefix, not path based). If it exists, the second process hands its command
  line to the first (section 10.5) and exits. `[instance] id=N` makes the loader hook `CreateMutexA/W` in cclsystem and
  the exe and append `_iN` to every named mutex (log `instance: id N ...`), so a copy runs side by side.
- Give the copy its own `sandbox.ini` root (`STUDIO_PRO_X_ROOT`), its own `data=` with `redirect=1`, its own
  llmbridge `port=`, its own `debug_port`s, and ideally its own audio device ("No Audio Device" exists in Audio Setup).
  Starting the installed exe while another copy runs hands off to the running one.
- Script packages contain an **absolute** exchange path (section 11.1): a copied sandbox must rewrite its packages, or
  the copy will execute the other copy's scripts.

### 2.7 The log

`<root>\sandbox.log`: every loader and mod line, prefixed `HH:MM:SS.mmm [thread id]`. Useful lines:
`==== mod loader start, pid ...`, `app build ... matches`, `skin: DEFAULT.SKIN replaced by ...`, `mod loaded: ...`,
`startfix: import tables of N app module(s) made writable`, plus each mod's own lines. Crash dumps are written by
Windows Error Reporting to `%LOCALAPPDATA%\CrashDumps\Studio Pro.exe.<pid>.dmp` (the loader cannot log a crash).

---

## 3. Writing a native mod (DLL) for the loader

### 3.1 How your mod plugs in

Put `mymod.dll` (+ `mymod.ini`, + optional `mymod\` folder) into `<root>\mods\` with Studio Pro closed. The loader
calls `LoadLibraryW` on it after startup; your `DllMain` runs; do the real work on your own thread. Nothing else to
register. Remove = delete the files. If it patches by address, it will be skipped automatically on another build
(2.4), which is what you want.

### 3.2 Loader exports (from `version.dll`)

```c
void  SandboxLog(const wchar_t *fmt, ...);                        // appends one line to sandbox.log (thread-safe)
void *PatchImport(HMODULE mod, const char *dll, const char *func, void *repl);
      // writes repl into mod's import slot for dll!func, returns the previous target (NULL = not found/failed).
      // dll is the import DLL name as written in the PE, e.g. "USER32.dll", "KERNEL32.dll".
int   SandboxRegisterSkinFilter(const wchar_t *pkgName, SkinFilterFn fn);   // 5.7
int   SandboxSkinFlush(const wchar_t *dir);   // switch the skin folder + forget cached .SKIN mappings (live reload, 5.9)
typedef const BYTE *(WINAPI *SkinFilterFn)(const wchar_t *name, const BYTE *in, DWORD inSize, DWORD *outSize);
```

Resolve them at run time so the DLL also loads without the loader:

```c
typedef void (*LogFn)(const wchar_t *, ...);
static LogFn Log;  static void NoLog(const wchar_t *f, ...) { (void)f; }

static DWORD WINAPI Setup(LPVOID self) {
    HMODULE ld = GetModuleHandleW(L"version.dll");
    Log = ld ? (LogFn)GetProcAddress(ld, "SandboxLog") : NULL;  if (!Log) Log = NoLog;
    wchar_t ini[MAX_PATH]; GetModuleFileNameW((HMODULE)self, ini, MAX_PATH);
    wcscpy(wcsrchr(ini, L'.'), L".ini");                         // mods\mymod.ini
    if (!GetPrivateProfileIntW(L"mymod", L"enabled", 1, ini)) { Log(L"mymod: off"); return 0; }
    if (GetAsyncKeyState(VK_SHIFT) & 0x8000) { Log(L"mymod: Shift held, off"); return 0; }
    /* validate the build / bytes you depend on, then patch; log what you did */
    return 0;
}
BOOL WINAPI DllMain(HINSTANCE self, DWORD reason, LPVOID r) {
    (void)r;
    if (reason == DLL_PROCESS_ATTACH) {
        DisableThreadLibraryCalls(self);
        HANDLE t = CreateThread(NULL, 0, Setup, self, 0, NULL); if (t) CloseHandle(t);
    }
    return TRUE;
}
```

### 3.3 Toolchain

Any x64 Windows C compiler works. The pack's mods are built with llvm-mingw clang:
`x86_64-w64-mingw32-clang -shared -O2 -Wall -o mymod.dll mymod.c -Wl,--no-insert-timestamp`
(add `-lws2_32 -lwinhttp -lole32 -luuid -ld2d1 -ld3d11 -ldcomp` as needed; `CoTaskMemFree` needs `<objbase.h>`).
`--no-insert-timestamp` makes builds byte-reproducible (handy for proving a source change changed nothing). The DLL's
own file name is stored in its export table: build it under its final name. mingw's C headers lack some COM macros
(`ID2D1Effect`, newer WebView2 interfaces): call by vtable slot number (slots are listed in this guide).

### 3.4 Patching techniques that work here

1. **Import (IAT) patch** with `PatchImport`, e.g. cclgui's `USER32.dll!SetWindowTextW` or `UpdateLayeredWindow`.
   Set your "real" function pointer from `GetProcAddress` **before** patching: the slot is live the moment it is
   written, and a call arriving in that gap would call your NULL pointer (a real crash seen in two mods).
   Use the export rather than your own `VirtualProtect`/write/restore: many mods patch slots on the same IAT page from
   their own threads, and one thread restoring read-only between another's unprotect and write crashed the app at
   start about 1 time in 10. The loader now pre-unlocks the app modules' IATs and writes slots with a lock and an
   interlocked exchange.
2. **Vtable slot patch** (vtables live in read-only `.rdata`: `VirtualProtect` RW, `InterlockedExchangePointer`,
   restore). Always check the slot holds the expected function first. Several mods may wrap the same slot (the
   web-view slots 26/27 are wrapped by five mods): **chain onto whatever the slot holds now**, and serialise with a
   named mutex (the pack uses `Local\StudioProWebViewSlotPatch` for the CCL web-view slots: use it too).
3. **Shared COM vtables** (Direct2D device context, UIAnimation transition library, DirectComposition, WebView2): the
   vtable sits in the system DLL and is shared by every instance. Create your own instance (e.g. D3D11 device ->
   `IDXGIDevice` -> `D2D1CreateDevice` -> `ID2D1Device::CreateDeviceContext`, slot 4) to read the vtable address,
   then patch the slots. Filter on the UI thread id inside the hook (one compare per call is cheap).
4. **`E8 rel32` call-site redirect:** the replacement must be reachable within +/-2 GB of the module. Allocate a
   stub `mov rax, imm64; jmp rax` near it by walking `VirtualQuery` for `MEM_FREE` regions around the module
   (blind 64 KB steps below cclgui failed). Validate the `E8` and its current target before rewriting the displacement.
5. **Inline detour** on a function prologue (14-byte absolute jump + trampoline). Only where the prologue is
   simple (e.g. three `mov [rsp+n], reg` = 15 bytes); copy the stolen bytes verbatim, check them first.
6. **In-memory resource patch:** `FindResourceW(GetModuleHandleW(NULL), L"MENUBAR.XML", RT_RCDATA)` +
   `LockResource` gives the mapped bytes; `VirtualProtect` RW (image pages are copy-on-write, nothing on disk
   changes), edit **same length** (e.g. blank an XML element into `<!--   -->`), restore. Do it before the app
   first parses the resource (a mod's setup thread is early enough for the menu bar). Count matches first; patch
   nothing unless each target occurs exactly once.
7. **Skin filter** (5.7): change skin XML in memory without touching files.

### 3.5 Getting onto the UI thread

- The UI thread is the process's oldest thread (toolhelp snapshot, earliest creation time); the main window is
  `CCLWindowClass` with no owner.
- **`SetWindowsHookEx(WH_GETMESSAGE, proc, hmod, uiThreadId)`** runs `proc` on the UI thread at its next message.
  Windows **removes the hook when the installing thread exits**: keep the installing thread alive (sleep/wait) until
  the hook has fired once, or install it from code that already runs on the UI thread. Guard the body with
  `InterlockedExchange` so it runs once.
- For work requested from other threads: in that hook create a **message-only window** (`HWND_MESSAGE`) on the UI
  thread and `SendMessageTimeout` jobs to it. All CCL objects, the command table, WebView2 calls
  (`PostWebMessageAsJson` etc.) and script dispatch must happen on the UI thread.

### 3.6 House rules that kept mods stable

- Byte guard before every patch; on mismatch log one line and do nothing. Log one "on" line with the build.
- Off switches: ini `enabled=0`, Shift at start, delete the DLL. Re-read the ini every second only if live tuning
  is useful (cheap), otherwise read once.
- No file or network I/O on the UI thread; no UI timers under ~100 ms unless animating; budget < 1 ms per frame
  for anything in a paint path. Keep your state in `mods\<name>\` or beside Studio Pro's settings with your own file name.
- Never call into the script engine from the UI thread synchronously from inside a paint or input handler.
- Shared names: one `debug_port` per process wins for all web panels (9.6); pick unique window class names,
  mutex names, ports.

### 3.7 Helper processes

For logic easier in Python or for external tools, the pattern used by the Skins page and the YouTube panel: the DLL
validates a page command, starts the helper hidden (`CreateProcess` with pipes, `CREATE_NO_WINDOW`), inside a **Job
object with kill-on-close** (cancel = `TerminateJobObject`, also kills grandchildren), forces UTF-8 in the child
(`PYTHONUTF8=1`, `PYTHONIOENCODING=utf-8`), reads JSON lines from stdout and posts each to the UI thread, which forwards
it to the page.

---

## 4. Re-finding things on another build

General rule: find a stable anchor (export name, string, constant, resource name, class name), then follow
references. Ghidra (headless works: `analyzeHeadless <projdir> <proj> -import "<app folder>\cclgui.dll"`, then
`-process cclgui.dll -noanalysis -postScript <script>` for each query; first analysis ~2-5 min per DLL) or
`llvm-objdump -d --start-address=` plus a 20-line Python PE section walker are enough.

### 4.1 Anchors that survive builds

| Need | Anchor |
|---|---|
| Command table | cclgui export **`CCLGetCommandTable`** |
| Theme manager (live skin reload) | cclgui export **`CCLGetThemeManager`** |
| Object table (`object://hostapp/...` URLs) | cclsystem export **`CCLGetObjectTable`** |
| Mutable strings | ccltext export **`CCLCreateMutableCString`** |
| Colour schemes | string `"ColorSchemeChanged"` (CCL constant; exe gets it via `CCLGetConstantCString`); the emitter is in cclgui |
| WebView2 switch | string `"CCL.Win32.WebView2 Enabled"` (config flag read/value bytes next to it) |
| Animation timing names | strings `linear`, `toggle`, `ease-in`, `ease-out`, `ease-in-out` in a `{char*, int64 FourCC}` table; FourCCs `linr togg eain eaou eaio` (search scalars) |
| View transition names | `fade, mvin, mvot, conc, revl, push, pusl, pusu, pusd, fall, lift, rise, sink, zmin, zmot` FourCC table |
| Single instance | import `CreateMutexA` in cclsystem; hand-off constant `0x436d646c` (`'Cmdl'`, WM_COPYDATA) |
| Fast-quit target | string `DspEffectAudioWorker` in `Plugins\dspshell.dll` |
| Menus | exe RCDATA `MENUBAR.XML`; commands: exe RCDATA `COMMANDS.XML`; CCL config `CCLGUI.CONFIG` |
| Script SDK | exe RCDATA `SDK/CCLAPP.JS`, `DEVICES.JS`, `ENGINE.JS`, `HELPIDS.JS`, `HOSTUTILS.JS`, `MEDIA.JS` |
| Script method signatures | strings in the exe beside method names, e.g. `importFile(url: Url, time: MediaTime, targetTrack: Track = 0, flags: int = 0)` |
| Piano-roll note controllers | setting string `Editing.NoteColoring.noteControllerIndicator` |
| Ruler / marker painters | skin keys `markerImage`, `markerWidth`, `lineWidth`, `loopgap`, `loopheight`, `markertolerance`, `startmarkerwidth` |
| Arrangement dividers | strings `trackNotesDivider`, `trackIconsDivider`, `faderDividerParamName` |
| Undo actions | class names `MovePointAction`, `InsertPointAction`, `RemovePointAction`, `TimeEventAction`; titles "Move Point", "Insert Point", "Delete Point" |
| App identity / startup | string `com.fender.studioapp`, user agent `Studio Pro/<version> Win x64` |
| Partner browser tabs | strings `Splice`, `Tonalic`, `Moises` near the tab registration |

### 4.2 Finding a class without RTTI (CCL type registry)

Each CCL class has a meta object in `.data`. In cclgui: `[-0x18]` meta vtable (`0x180543ec8` on 8.1.2), `[-0x10]`
parent meta, `[-0x08]` factory function, `[+0]` pointer to the class name string. Route: find the name string ->
find the 8-byte pointer to it -> read the factory -> decompile it -> the constructor -> `*this = &vtable`. Some names
appear in several tables; the parent field disambiguates. In the exe the meta objects look like
`{+0 meta vtable (0x1418ce140 on 8.1.2), +8 registry link, +0x10 factory or 0, +0x18 name}`, and `isA(meta)` is vtable
slot `+0x30` on any CCL object.

### 4.3 Finding a vtable slot from a function

Search `.rdata/.data` for the 8-byte absolute pointer to the function: that address is its slot; the vtable start is
found by walking back to the first non-code pointer. Diff two vtables (child vs parent) to find overrides.

### 4.4 Finding who uses a string or constant

Strings -> xrefs -> decompile the users. Constants (FourCCs, IIDs) -> scalar search. Imports (`FindWindowExW`,
`RegisterClassExW`, `CreateMutexA`) -> every caller. CCL constant names -> every function reading the global that
caches the constant. These five queries (strings, decompile+callers, scalar search, import users, global users) did
almost all the mapping in this guide.

### 4.5 Mapping the script API without source

Native script objects are not enumerable. What worked: (1) extract every camelCase / PascalCase identifier from
`Studio Pro.exe` (~125 k + ~48 k); (2) per object, one script does `typeof o[name]` for every candidate (fast, no
throws on most objects); (3) child nodes by URL: `Host.Objects.getObjectByUrl("object://hostapp/DocumentManager/ActiveDocument/<Name>")`
for every PascalCase name (null = absent, never throws); (4) `node.find(name)` the same way; (5)
`Object.getOwnPropertyNames(Native_<moduleBase>_<Class>)` on the `globalThis` class objects lists each native class's
**methods**. Caution: on some object nodes `typeof node[unknownName]` **throws** and every throw shows a modal (11.6);
use `find` / `findParameter` there.

---

## 5. Skins: packages, XML, colour tokens, the theme builder

### 5.1 Where the look lives

`Studio Pro.exe` embeds 16 skin packages as `RCDATA` (type 10):

| Resource | Contents |
|---|---|
| `DEFAULT.SKIN` (~749 files) | the app: `app/` (`colorscheme.xml` = main tokens, `shared.xml`, toolbar, `windowclasses.xml` = perspectives, `homepage.xml`, startup, `about.xml`, `options.xml`), `ccl/` (`browser.xml`, `overlays.xml` incl. the Options dialog form `UserOptionDialog`, `packages/`), `engine/` (track controls, editors, inspector, launcher, `eventtheme.xml`, `song.xml`), `devices/` (mixer, transport), `project/`, `show/`, `images/` |
| `FENDERBASE.SKIN` | base colours (`BaseColor00..100`, `PrimaryColor25..75`), `fonts.xml` (Nimbus Sans Novus = theme id StandardUI, Futura PT = FenderUI, Consolas = StandardUI Mono), Start-page glow image, `StartPageColorize`/`HomePageColorize` |
| `NEUTRALDESIGN.SKIN` | standard widgets: `ButtonShape`/`ButtonL/M/R`, checkbox/radio/scroll atlas `AtlasNeutralStandard`, knob atlas `AtlasNeutralKnobs`, shared colours |
| `UNIFIED`, `DAWSHARED` (mixer channel, inspector, device header), `FENDERSHARED`, `SHOWSHARED`, `MUSICSHARED` | shared chrome |
| `PLUGINS`, `NOTATION`, `SCOREFONTS`, `DEVICEICONS`, `FENDERICONS`, `FILEICONS`, `CHANNELICONS`, `DAWFILETYPES` | plug-in faces, notation, icon sets |

Packages import each other by name: `<Import url="@fenderbase"/>`. A package only sees **its own images** (declare
copies where needed). Plug-in DLLs (e.g. `spatialaudio.dll`, gadget DLLs like `soundcloud.dll`) carry their own skin
packages; a gadget's form resolves in the **plug-in's** skin, not the app skin.

### 5.2 Package format (CCL "PACKAGEF"; also used by script packages)

```
"PACKAGEF" | file data (each entry zlib-deflated) | directory | 64-byte trailer
trailer  = u64 dirOffset, u64 dirSize, 12 zero bytes, 16-byte format id 6276670fa7cdb040950a88b1bffcc6fb,
           u32 0, u32 1, u32 64, "PACKAGEF"          (constant: no checksum, no signature)
directory= "Root" u32 flags u32 count, then entries:
           "File" u32 flags, UTF-16Z name, 9-byte date, u64 offset, u64 storedSize, u64 size
           "Fold" u32 flags, UTF-16Z name, u32 count, <children>       (folders have no date)
```

- **Every entry must be deflated**, even when that grows it. The stock packer deflates everything; the reader shows a
  raw (stored) entry as "Image not loaded" and paints nothing, with no log line.
- The shipped `skin-tools\skinpack.py`: `python skinpack.py extract "<app folder>\Studio Pro.exe" <dir>` and
  `python skinpack.py build <dir>\<NAME> <out.SKIN>`; `read_pkg(bytes)` returns a tree of
  `('file', name, flags, date, raw, size)`; `unpack(raw, size)` inflates. Unchanged files keep their original bytes, so
  an untouched folder rebuilds byte-identical.

### 5.3 Skin XML: the elements that matter

- **Colour token:** `<ColorScheme name="Main"><ColorScheme.Color name="X" color="hsl(h,s,l[,a])" saturation="..."
  luminance="..." contrast="..." invertedvalue="..." huefixed slcombined invertible alphavalue invertedcolor/>`.
  Referenced as `@Main.X`. About 1,500-1,650 tokens across packages (`DEFAULT/app/colorscheme.xml` ~510). Live schemes on
  8.1.2: Main (~1,300 tokens), Arrangement, Plugins, Score, Transport, RenameTracks. Plain `<Color name= color=>`
  resources exist too (hsl/hex/rgb/named); colours may reference them as `$Name` or `$skin://hostapp/Name`
  (window and dialog backgrounds are such references to `BaseColorNN`/`PrimaryColorNN`).
- **Style:** named bundle of `<Color name="textcolor">`, `<Image name="background">`, `<Font>`, `<Metric>`;
  `inherit=` another style.
- **Shape:** vector drawing; states are child `<Shape name="normal|pressed|mouseover|normalOn|pressedOn|disabled|focus">`
  containing `<Rectangle size="x,y,w,h" style="fill stroke tiled" Brush.color= Pen.color= radius=>` and `<Line>`.
  Most modern buttons are flat Shapes.
- **ImagePart:** slice of an atlas PNG: `size=`, `margin=` (9-slice), `frames=` (state order, e.g.
  `normal pressed normalOn pressedOn mouseoverOn mouseover disabledOn disabled`), optional
  `<ImageFilter name="colorize|tint|lightadapt" color=>`. A kit's corner radius cannot exceed the part's margin.
  `ShapeImage` = an SVG used as an image; as a Toggle icon it is scaled to the toggle (SVG width/height ignored);
  `ShapeColorMapping` recolours an SVG keyed on its literal colours. SVGs are cached by URL across a live reload.
- **Triggers/Animations:** `<Trigger event="onAttached|onMouseDown|onTouch|..." property=...>` with
  `<StartAnimation><Animation property="..." from= to= duration= function= options=/>`. `function`: linear, toggle,
  ease-in, ease-out, ease-in-out (+ `spring` when the motion mod is loaded). Animatable properties are object
  properties (e.g. `vpos` on ScrollView, `phase` on Button filmstrips); views expose **no** size/opacity/offset
  property to skins.
- **Perspectives** (`app/windowclasses.xml`): Start, Song, Project, Show, ShowPerformance; `transition="fade"` etc.
  (none fade movein moveout conceal reveal push pushleft pushup pushdown fall lift rise sink zoom zoomin zoomout).
- **Forms, Variants, controllers:** `<Form name=...>` defines a panel; `<Variant name="../Browser/searchVisible">`
  switches children on a property; `<using controller="object://...">` scopes a controller; children of a
  `ColorBox` bind to the track controller. Layout attributes: `size="x,y,w,h"`, `attach="left right top bottom fill all"`,
  Horizontal/Vertical containers, `options=`.
- **Web view slot:** `<WebView name="X" title="X" size="0,0,w,h" attach="all" options="localonly"/>`: creates a CCL
  `WebBrowserView`; it never navigates by itself (a mod does, section 9).
- **Option tables found in cclgui:** Divider `outreachleft 8, outreachright 0x10, outreachtop 0x20, outreachbottom 0x40`
  (plain `outreach` reaches right/bottom only; `outreachleft` alone works, combinations disabled the grab);
  TextBox `multiline 1, fittext 2, nocontextmenu 4, scaletext 8, markup 0x10`; ValueBox `static 2, separatedigits 4,
  scaletext 8, nocontextmenu 0x10`. ListView style keys: `backcolor, backcolor2, separatorcolor, separatorBeneath,
  rowHeight, selectionColor, selectedtextcolor, selectionbarimage, hilitecolor, focusColor, marginH/marginV,
  itemInset, iconSize, textfont`, option `customBackgrounds` (no per-row image; column sets/widths are code).

### 5.4 Appearance sliders and colour rules

- The Appearance sliders (`user.colorscheme`: hueLevel, saturationLevel, luminanceLevel, contrastLevel, 0..1) drive
  per-token curves. Level is **signed around 0.5**: `saturation="id:c1/c2"` adds about `+c1` above 0.5 (c2 seemed
  ignored) and subtracts about `(1.7*id - 3)` below it. The full curve syntax is not decoded (`10`, `8:5`, `8:4/20`,
  `10:0/20/50/100`, `8-10`, `I` = inverted theme only): tune by measuring rendered pixels; scaling every number in a
  curve scales its effect roughly proportionally; keep integers.
- **First definition wins** across packages (not last). A plug-in's identical stock copy of a package loaded first can
  shadow your tokens (the loader handles this, 5.6).
- CCL **truncates** HSL to 8 bits; the 4th hsl value is alpha in percent; **decimals in hsl() silently lose the
  alpha**: use whole numbers.
- Literal hex in Styles is not touched by the sliders; tokens are.
- **Opaque-only paints:** a ValueBar `backcolor` and SegmentBox/TextBox `backcolor` ignore alpha (alpha 0 draws
  black). A SegmentBox ignores a `background` image.
- **A Style and an Image with the same name crash the app** when the form loads. Never reuse a name across the two.
- Changing font faces truncates labels in fixed layouts; check every area. Keep geometry identical where possible
  (recolour, don't delete or resize).
- Bevels = 1 px light strip (`...TopGapColor`) + body + shade strip (`...OnGapColor`, `...PressedGapColor`,
  `...BottomGapColor`); free-standing separators: `ZoneGap`, `PanelBackgroundGap`, `StandardDivider`,
  `InspectorSectionDivider`, `WindowAreaFrameOutline`.

### 5.5 How common controls are drawn (skin vs code)

- **Mixer fader** = 3 atlas parts in `DAWSHARED/AtlasMixerAssets`: groove, bargraph (normal/pressed), cap (variant per
  channel type in `audiochannel.styles.xml` `*Slider{Big,Narrow,SmallNarrow}`).
- **Mixer pan** = a horizontal Slider drawn from Shapes (`MixerPanSliderShape`, handle `MixerPanHandleShape`).
- **Knobs** (`StdKnobBase`, NEUTRALDESIGN `styles.xml`) are drawn by CCL code from metrics (`circle strokewidth
  range indicator indicatorwidth indicatormargin topoverlay`) + colours (`forecolor`, `hilitecolor`, `indicatorcolor`)
  over images from `AtlasNeutralKnobs.png`.
- **Transport** buttons: 34x34 Toggles with icon-only Styles (`Shared/PlayButton` = SVG states +
  `ShapeColorMapping` keyed on literal colours); hover/on plate `TransportBarButtonShape`.
- **Standard button** = `Neutral.Button` -> `ButtonShape` family (tokens `ButtonPenColor`, `MobileButton*Color`).
- **Arrangement track buttons** use `TCPlain{Mute,Solo,Record,Monitor,SoloSafe}Button`: 9-slice strips in an atlas,
  letters in a separate glyph atlas, and `TC*Mask` ImageParts over each group that paint rounded group corners.
- **Channel meter** is drawn by CCL from `ChannelScaleBase`/`ChannelMeterBase` (`audiochannel.styles.xml`); segments
  come from one tiny overlay part (`VUMeterOverlay`).
- **Engine-drawn shapes (no skin attribute can change the shape):** ruler marker labels and the loop range
  (`MarkerEventTheme`, `TimeRuler` painters in the exe, section 14.3), the arrangement playhead (DirectComposition
  sprites, 7.4). The skin only feeds their colours and metrics.

### 5.6 How the loader serves skins (`[skin] enabled=1 dir=skins\<id>`)

The loader hooks cclsystem's `FindResourceW/SizeofResource/LoadResource/LockResource`. For the exe's RCDATA `*.SKIN`
it memory-maps `<root>\<dir>\<NAME>.SKIN` (must start and end with `PACKAGEF`, else logged and the built-in one is
used) and returns a fake HRSRC (the address of its table slot). For a non-exe module asking for a same-named package,
it serves the override only when that module's copy is byte-identical to the exe's stock copy (a plug-in shipping a
stock duplicate, e.g. `DAWSHARED.SKIN` in `spatialaudio.dll`, would otherwise shadow ~150 tokens). Mapped files are
locked while the app runs: **you cannot rebuild the skin the running app uses**. Log: `skin: DEFAULT.SKIN replaced by ...`.

### 5.7 Skin filter API (patch skin XML in memory from a mod)

`SandboxRegisterSkinFilter(L"DEFAULT.SKIN", fn)`: each time the loader serves that package it passes the bytes it
would serve (override file, or the exe's stock resource) to `fn`, which may return a replacement PACKAGEF (copied by
the loader; checked structurally; a crashing or invalid filter is ignored and logged; return NULL / outSize 0 = no
change). Max 8 filters. Refused when the skin override is off. It applies the **next** time CCL opens the package:
register early in your setup thread (the first open is ~2 s after loader start; registering 0.2 s in is in time; a
late registration logs a WARNING). To edit, unpack the package (5.2), change the XML, repack with deflated entries.

### 5.8 The theme builder (shipped in `<root>\skin-tools`, Python 3.11+, ImageMagick 7)

- `skin-src\<NAME>\` = pristine extract of the user's own exe (never edit). One theme = `skin-tools\themes\<id>.py` or
  a folder `themes\<id>\theme.py` (+ `theme.toml`, `files\<PKG>\...` copied into the package, `web\`, `default-styles\`).
- `python skin-tools\skinbuild.py <id>` writes `skin-work\<id>\` and packs `skins\<id>\*.SKIN` + `skins\index.json`;
  `--activate <id>` sets `[skin] dir=`; `--list`; `--accent H`, `--surface H,T`, `--style family=pack`.
  It refuses to build the theme the running app is using.
- New theme without code: `python skin-tools\newtheme.py "Ocean" --accent #3AC7E0` (`--from liquidglass|graphite|stock`),
  then edit `themes\ocean\theme.toml` (sections `[theme] [surface] [accent] [glass] [shape] [semantics] [fonts]
  [styles] [faders]`; `base = "<id>"` makes the file sparse), build, switch to it. `skinbuild.py delete-theme <id>`.
  Validate one file: `python skin-tools\themefile.py <id>`. Full checks: `python skin-tools\check.py` (minutes).
- Theme module hooks (Python): `NAME, DESCRIPTION, SWATCHES, ORDER, retune(h,s,l), curve_scale(h,s), OVERRIDES
  {token: attrs}, FLATTEN_BEVELS, PERSPECTIVE_FADE, transform(path, text)` for any XML edit, `grey(l)`, `SLIDER_SCALES`,
  `FILES`, `HOST_REFS`, `IMAGES` + `image_rule(rel)` (PNG recolour via ImageMagick), `WEB` (web.css + fonts copied to
  `skins\<id>\`). Non-look features that every theme needs (web-panel slots, the Skins page, the chat panel slot) are
  rules in `skin-tools\features.py`.
- **Style packs** (control looks): `<root>\mods\stylepacks\<pack-id>\pack.toml` (`id, name, family = fader|knob|meter|msr|hvol|icons,
  style, module` or `base`, `preview`, data keys `shapes`/`svg`). Quick start: copy an existing pack, rename,
  `python skin-tools\stylepacks.py check`, pick it on the Skins page or `skinbuild.py --style knob=<pack-id> <theme>`.
- Web panels follow the skin: pages link `../../../skins/theme.css` (the Skins mod copies `skins\<loaded>\web.css`
  there at startup).

### 5.9 Live colour and live skin reload (no restart)

- **Colour internals (cclgui):** ColorSchemes singleton global `0x18064ee28` (getter `0x1801e5570`, scheme list `+0x40`).
  `ColorScheme` object (size 0xe8, vtable `0x180537e70`): `IColorScheme` interface at `+0x30` (vtable `0x180538080`:
  `+0x20 getLevel`, `+0x28 setLevel(name, value, mode)`, `+0x30 getColor(name)`, `+0x40 reset`), name `+0x50`,
  levels floats `+0x38` hue, `+0x3c` sat, `+0x40` lum, `+0x44` contrast; token list `+0x70`, items at `+0xb8`;
  token lookup by name `0x18034d550`. Token data (size 0x188): name `+0x30`, **resolved colour `+0x50`** (what drawing
  reads), base `+0x54`, inverted base `+0x58`.
- `setLevel` with mode 1 (or a changed level with mode -1) saves levels, recomputes every token (`0x18034d910`) and
  emits `ColorSchemeChanged` + view event `0xd` + full redraw on every window (`0x18034f680`).
- **Live change recipe (UI thread only):** write token base `+0x54` (and `+0x58`), then
  `setLevel(scheme+0x30, &emptyCString, 0, 1)` (a name matching no level + mode 1 = recompute + notify). The base still
  goes through the Appearance curves (write the same value the build would put in the XML). Views copy colours and
  re-read them only on event `0xd`.
- The pack's **Live colours** mod (`palette.dll`) wraps this: lines appended to `<root>\mods\palette\cmd.txt`:
  `dump`, `set <scheme> <token> #rrggbb`, `apply [file]` (from `skin-tools\palette_export.py`), `reload [dir]`.
- **Live skin reload:** `IThemeManager` = `CCLGetThemeManager()` (vtable `0x18053b9f8`: `+0x40 reloadTheme(theme)`,
  `+0x50 reloadAll(bool)`). `reloadAll` re-parses every theme package and drops cached visuals, but **does not rebuild
  forms** (layout stays) and views keep old colours until a `setLevel` refresh per scheme. The pack's `reload <dir>`
  does `SandboxSkinFlush(dir)` -> `reloadAll(mgr, 0)` -> `setLevel` refresh on every scheme -> `RedrawWindow`:
  1.7-3.3 s, works with a song open. Runtime colour overrides do not survive a reload (re-apply them).
  Layout changes need a form rebuild: CCL's workspace manager rebuilds frames when the perspective being left is named
  `cclspy:SkinRefresh` (select perspective `0x18025b440`, rebuild frames `0x180266630`); Studio Pro has no such
  perspective, so a mod would add one or call the rebuild. Studio Pro has no reload command, key or file watcher.

---

## 6. Views, controls and painting (cclgui)

### 6.1 Classes and vtables (8.1.2)

Chain: `View` -> `Control` -> `Button` -> {`ToolButton`, `Toggle` -> ... -> `RadioButton`}.

| Class | Main vtable | Notes |
|---|---|---|
| View | `0x1804cc900` | ctor `0x1803ab640`; rect `+0xc0` (4 ints l,t,r,b in parent coords); flags `+0x120` (bit 15 hidden; `0x400` needed for invalidate); child list head `+0xb0` (children linked at `child+0x30`, next = first qword of the link, child = link-0x30) |
| Control | `0x1804dd530` | ctor `0x18013cee0` |
| Button | `0x1804de830` | factory `0x180117cb0`, size 0x1a8 |
| ToolButton | `0x1804ddb58` | ctor `0x18011fdc0`; own draw `0x180120280`, mouse `0x180120ec0` |
| Toggle (Mute/Solo/Record, switches) | `0x1804dd000` | shares Button draw + mouse |
| Toggle subclass / RadioButton | `0x1804dd8a8` / `0x1804dc7f8` | |
| WebBrowserView | `0x180530448` | section 9 |

### 6.2 Vtable slots (byte offsets, all views)

| Slot | Meaning |
|---|---|
| `+0x08` / `+0x10` | retain / release (refcount int at object `+0x08`; release deletes at 0) |
| `+0x188` | `draw(UpdateRgn*)`. Control's (`0x18013da00`): renderer = `this->[+0x270]()`, `renderer->[+0x88](this, rgn)`, then `View::draw` (`0x1803ae100`, children loop) |
| `+0x190` | invalidate/redraw (`0x18013da50`) |
| `+0x1d8` | `createMouseHandler(event)`: called exactly on press; Button's `0x18011cb20` |
| `+0x1f8` | createTouchHandler |
| `+0x248` | accessibility provider |
| `+0x270` | getRenderer (Button: theme painter `[+0x80](10, style)`, cached at `+0x168`) |

`UpdateRgn`: `int rect[4]` at `+0x00` (view-local), `int offset[2]` at `+0x10` (the view's origin in the drawing
surface), region pointer at `+0x18`. No graphics object is passed; the renderer makes its own port from the view.
View strings: name `+0xe0`, title `+0xf0` (CCL String: vtable `+0x30` lock(&{const wchar_t*, int}) / `+0x38` unlock).

### 6.3 Painting = Direct2D

- One shared `ID2D1DeviceContext` paints all windows on the UI thread; Studio Pro calls `SetTransform` thousands of
  times per second. At the start of a button paint the transform is usually identity and the region offset gives the
  surface position.
- Main window target = the swap-chain back buffer (options TARGET|CANNOT_DRAW): copy from it with `CopyFromBitmap`,
  never draw it as a source. Some views paint through a full-width offscreen bitmap first; pop-ups have their own
  small targets; the playhead is a separate composition layer.
- `ID2D1RenderTarget`/`DeviceContext` slots used by the pack: SetTransform 30, GetTransform 31, BeginDraw 48, EndDraw 49,
  GetDpi 52, CreateBitmap 57, CreateEffect 63, GetTarget 75, DrawImage 83; primitives DrawLine 15, FillRectangle 17,
  DrawRoundedRectangle 18, FillRoundedRectangle 19, DrawGeometry 22, FillGeometry 23. Bitmap: GetPixelSize 5,
  GetPixelFormat 6, CopyFromBitmap 8, GetOptions 12. Effect: SetValue 9, SetInput 14, GetOutput 18. Struct returns
  use a hidden pointer after `this`.
- Exe-side CCL `IGraphics` slots (D2D underneath): `+0x18/+0x20` save/restore, `+0x38` setClip(rect), `+0x78`
  fillRect(int rect*, paint*), `+0x98` drawLine(p1*, p2*, paint*), `+0xc0` strokePath(path, paint*), `+0xc8`
  fillPath(path, paint*), `+0xe0` fillRectF, `+0x108` fillTriangle(int pts[3]*, paint*), `+0x128` drawText(rect*,
  string*, font*, paint*, flags*), `+0x188` drawImage(image, src*, dst*, 0), `+0x198` drawImage(image, pos*, 0).
  Paint struct = `{int 0; uint32 colour; float width; ptr}`. Paths from `CCLGetGraphicsHelper()->vt+0xb0(0)`:
  `+0x40` moveTo, `+0x58` lineTo, `+0x48` close; helper `+0x38` measureText(out rect*, string*, font*).
- **Technique "swap a primitive inside one painter":** wrap the painter's vtable slot only to raise a thread-local
  "span" flag around the original call; inside the span, hook the shared D2D slots and replace the primitive (e.g.
  `FillGeometry` of the marker flag -> `FillRoundedRectangle` with the same brush). Colours stay the app's brushes, so
  the skin still controls them. Cost outside spans: one thread-id compare per D2D call.
- **Backdrop blur in the main window works technically** (copy target rect + margin -> GaussianBlur effect ->
  DrawImage back, 0.2-1.0 ms), but chrome bars never overlap content, so blurring "behind" them blurs a flat fill.
  Gotcha: `D2D1_COMPOSITE_MODE_SOURCE_COPY` in `DrawImage` is unbounded: without a clip it clears the whole back buffer.

### 6.4 Button press feedback (example mod pattern)

Wrap `+0x188` and `+0x1d8` for Button and ToolButton only (validate each class by its expected mouse-handler
address). Press = non-NULL handler from `createMouseHandler`: retain the view, start an 8 ms UI timer. Release =
`WM_LBUTTONUP` seen by the `WH_GETMESSAGE` hook (fallback: physical button up). While animating, post-multiply every
`SetTransform` during that button's paint by a scale about its centre; return the unscaled matrix from `GetTransform`;
restore afterwards. Shipped off by default (press scale is optional in `springs.ini`).

---

## 7. Animation engine

### 7.1 Classes and timing

CCL classes: `BasicAnimation`, `TransformAnimation`, `CubicBezierTimingFunction`, `AnimationClock`, `AnimationManager`,
`ViewAnimator`, `StartAnimationAction`, `StopAnimationAction`. Animation object (ctor `0x18030af40`): `+0x68` duration
(double, seconds), `+0x70` timing id (FourCC), `+0x78..+0x90` custom bezier points (id `cubb`).

Timing name table `0x1804c4fb0` (`{const char* name; int64 id}`, NULL-terminated, terminator slot `0x1804c5000`):
`linear linr`, `toggle togg`, `ease-in eain`, `ease-out eaou`, `ease-in-out eaio`. Next table `0x1804c5010` = options
(backwards, forwards, autoreverse, forever, group).

**The one choke point:** `0x18030c570(id)` returns the shared preset curve interface (object + 0x30) for
eain/eaio/eaou, else NULL. Its only callers: `0x18009e94a` (GPU path), `0x18030cb14` (CPU evaluate), `0x18030c0cb`
(copy preset). Curve interface vtable: `+0x28 getControlPoints(double[4])`, `+0x30 evaluate(double t)` (t in XMM1).

- CPU path `0x18030cae0(anim, t)`: linr -> t; togg -> step; cubb -> own bezier; else preset->evaluate.
- GPU path `0x18009e830(engine, anim, scale)`: linr/togg -> `AddCubic` on an `IDCompositionAnimation`; else bezier
  points -> `IUIAnimationTransitionLibrary2::CreateCubicBezierLinearTransition` (slot 20, `+0xa0`) -> storyboard ->
  `GetCurve(IDCompositionAnimation)`. Engine: `[0x20]` IDCompositionDevice (CreateAnimation slot 25), `[0x22]`
  transition library.
- Layer `addAnimation` `0x1800a1d30(layer, propId, anim)`: prop 1 offsetX, 2 offsetY, 3 offset, 4 opacity, 8 transform.

### 7.2 View transitions

`ViewAnimator` ctor `0x1803b4c50`: defaults **0.25 s** and **`eaio`**, stored as immediates (`+0x30`, `+0x38`).
Factory `0x1803b4f30(view, transitionId)`, runner `0x1803b55c0`, reverse map `0x1803b5070` (rise<->sink,
zoomin<->zoomout). Used by perspective switches and page-switching views (`0x1801726c0`). Panels toggled with F3/F4/F5
do not animate at all.

### 7.3 How the motion mod (`springs.dll`) adds a curve

1. Redirects the 3 call sites of the preset lookup through a near stub: id `sprg` -> a static fake curve object
   (evaluate = damped spring; getControlPoints = a marker bezier `{0.2501, 1.2499, 0.5001, 1.0}`).
2. Writes `{"spring", 'sprg'}` into the table terminator slot -> skins can say `function="spring"`.
3. Wraps transition-library slot 20 (shared vtable in `UIAnimation.dll`, found by creating its own instance): marker
   points -> a custom `IUIAnimationInterpolator2` spring (32 Hermite segments, honours initial velocity).
4. Patches the ViewAnimator immediates (curve id + duration from the ini).
Spring model "duration + bounce": zeta = 1 - bounce, omega = 6.9078/zeta (settles to 0.1 % at t=1); bounce 0 =
critically damped. Shipped defaults are calm: transitions 0.15 s, bounce 0, pop-up fades only.

### 7.4 The arrangement playhead = DirectComposition sprites

Not painted by the editor. Per lane (ruler, marker/global lane, track area) it is one DComp visual (child of the
window's root visual) whose content is a 2 px wide surface, created when playback starts. While playing,
`SetOffsetX(float)` + `SetOffsetY(float)` are called on each ~31 times a second; while stopped nothing is called.
A visual added as a **child** of such a segment is accepted but never drawn; a **sibling** placed just below it
(`AddVisual(parent, v, FALSE, segment)`) is drawn. One DComp v1 device (`DCompositionCreateDevice`); surfaces are
slices of an atlas (BeginDraw with `IID_IDXGISurface` gives an offset into a 1024x1024 B8G8R8A8 texture). Real v1
visual slots (MSVC reverses overload order): 3 SetOffsetX(anim), 4 SetOffsetX(float), 5/6 Y, 7 SetTransform(obj),
8 SetTransform(matrix), 10 SetEffect, 13 SetClip(obj), 14 SetClip(rect), 15 SetContent, 16 AddVisual,
17 RemoveVisual, 18 RemoveAllVisuals; EffectGroup 3 SetOpacity(anim), 4 SetOpacity(float); Animation 3 Reset,
5 AddCubic, 8 End. Gotcha: `AddVisual` with a NULL reference and insertAbove FALSE puts the visual on **top**.
Cursor colour token: `CursorColor`.

---

## 8. Windows, menus, pop-ups and dialogs

### 8.1 Window kinds

| What | Class | Style / ex-style | Notes |
|---|---|---|---|
| Main window | `CCLWindowClass` | `9FCF0000` / `00040100` | maximised; normal OS caption (the title-bar icon = the window icon) |
| Menus, drop-downs, context menus, tooltips | `CCLDialogClass` | `96000004` / `00090000` (WS_POPUP, WS_VISIBLE, **WS_EX_LAYERED**) | a **new window every time**, created already visible by a path that bypasses cclgui's import table; content arrives via `UpdateLayeredWindow` (`ULW_ALPHA`, alpha 255, `AC_SRC_ALPHA`) from a 32-bit premultiplied DIBSECTION |
| Dialogs (Options, New Song...) | `CCLDialogClass` | `96CC0004` / `00010100` (caption, not layered) | Studio Pro sets only DWM attribute 35 (caption colour) |
| Single-instance receiver | `CCLMessageWindowClass` (HWND_MESSAGE) | | title "Studio Pro" |
| Script error box | `#32770` TaskDialog | | title "Studio Pro"; text is in a DirectUI pane (read with UI Automation, not GetWindowText) |

`cclgui.dll` is the only module importing `CreateWindowExW`, `ShowWindow`, `SetWindowPos`, `DeferWindowPos`,
`UpdateLayeredWindow`, `SetLayeredWindowAttributes`, `DwmSetWindowAttribute`. `TrackPopupMenu` is imported but menus
are CCL-drawn. Main window title: `Studio Pro` on the Start page, `Studio Pro - <song>` with a song open (a trailing
`*` = unsaved).

### 8.2 Hooking pop-ups

- The reliable trigger is the **first `UpdateLayeredWindow` with content** for an unseen layered `CCLDialogClass`
  popup (hooks on CreateWindowExW/ShowWindow/SetWindowPos do not see menus). Hook cclgui's `UpdateLayeredWindow` import.
- Click-opened vs hover/keyboard: track a click within 0.6 s via the `WH_GETMESSAGE` hook.
- Between cclgui's own frames you can send position/alpha-only updates:
  `UpdateLayeredWindow(h, NULL, &pos, NULL, NULL, NULL, 0, &blend, ULW_ALPHA)`.
- Frosted pop-ups: `SetWindowCompositionAttribute(h, {19, &ACCENT_POLICY{4 = acrylic, flags 2, tint AABBGGRR}})` works
  on these layered menus; blur shows where the DIB pixels are translucent. **But acrylic/blur-behind halves menu hover
  frame rate** (DWM re-renders the backdrop for every per-frame layered update; 59 -> 32 fps, 200-300 ms stalls), so the
  pack keeps accent 0 and only paints a ring into the DIB and asks DWM for rounded corners (attribute 33) and a border
  colour (34). Layered pop-ups get no DWM shadow.
- Dialogs are opaque in their client area: DWM acrylic is accepted but shows nothing.
- CCL reads the **real cursor** for menu hover: posted `WM_MOUSEMOVE` does not move the highlight; posted `VK_DOWN`
  to the pop-up hwnd does.

### 8.3 Menus are XML inside the exe

The menu bar is exe RCDATA `MENUBAR.XML` (~26.5 KB, plain ASCII, LF, loaded via the CCL string `"menubar.xml"`);
no skin file defines menus. Items look like `<MenuItem name="Check for Updates" follow="1"/>`,
`<MenuItem name="About" title="About $APPNAME" follow="1"/>`, `<MenuSeparator/>`, and conditional blocks
`<?defined checkupdates?>`, `<?platform mac?>`, `<?not:platform mac?>`. The "Hide help items" mod blanks two items
same-length in memory (3.4 item 6). Commands for menu items: `COMMANDS.XML` (section 10). Leave the command
definitions alone; hiding the menu entry is enough. `$APPNAME`/`$APPCOMPANY` come from the exe (the company string
`Fender` probably also builds the settings folder path: do not patch it; to rename the About text, filter text drawing
while that window is up instead).

### 8.4 Hooks die with their thread

A `WH_GETMESSAGE` hook installed from a thread that then exits is silently removed. Install from the UI thread (for
example from inside a D2D `SetTransform`, `ShowWindow` or `UpdateLayeredWindow` hook) and pass your DLL's HMODULE.

### 8.5 Window icon and title

- Icon: every 500 ms sweep this process's top-level `CCLWindowClass`/`CCLDialogClass` windows, `WM_SETICON` BIG + SMALL
  (sizes from `GetSystemMetricsForDpi(SM_CXICON/SM_CXSMICON, dpi)`) only when `WM_GETICON` differs (Studio Pro resets it).
- Title: patch cclgui's `SetWindowTextW` import and also poll with `WM_SETTEXT` (dialogs untouched). Handle both forms
  (`Studio Pro` and `Studio Pro - <song>`), keep a trailing `*`, leave already-changed titles alone. Other mods parse
  the title to detect the open song: keep `" - "` intact.

---

## 9. Web panels (WebView2 inside CCL views)

### 9.1 CCL's web view

- Skin `<WebView name=... title=... options="localonly"/>` (WebViewElement, createView `0x1803086a0`) creates a
  **WebBrowserView** (vtable `0x180530448`, size 0x160). The controller from `using controller=` sits at view `+0x150`;
  the view never navigates by itself.
- **attached** (vtable slot 26, `0x180312e80`) builds the platform object at view **`+0x158`** via `0x1800bcf30`:
  WebView2 (platform vtable `0x1804cf220`) only when the config flag `CCL.Win32.WebView2 Enabled` (read flag
  `0x180636bc0` / value `0x180636bc8`) is on **and** the shared WebView2 environment already exists (manager singleton
  `0x1800be0a0()`, environment at `+0x38`; the first call starts environment creation, which is async). Otherwise it
  falls back to **Internet Explorer** (`ieframe.dll`, vtable `0x1804cefb8`). Studio Pro ships with the flag **off**.
  **removed** = slot 27 (`0x1803130c0`).
- WebView2 platform object: `ICoreWebView2Controller*` at **`+0x110`** (ready flag `+0x120`), `ICoreWebView2*` at
  **`+0x118`**, IE-fallback check interface `+0x158`; the view pointer at platform `+0x48`. Navigate `0x1800bd760`
  (calls ICoreWebView2 slot 5).
- ICoreWebView2 slots confirmed: 3 get_Settings, 5 Navigate, 32 PostWebMessageAsJson, 33 PostWebMessageAsString,
  34/35 add/remove_WebMessageReceived. Settings: 6 IsWebMessageEnabled, 12 AreDevToolsEnabled, 14 ContextMenus,
  18 Zoom. WebMessageReceived args slot 5 = TryGetWebMessageAsString. Controller: `+0x30` put_Bounds; `get_CoreWebView2`
  slot 25.
- Switching from the Start page to a song destroys the Start page's web view (its browser process exits); returning
  re-attaches and reloads (~230 ms).
- Keys typed into a page go to the WebView2 browser process (`Chrome_RenderWidgetHostHWND` child), not to Studio Pro:
  single-key shortcuts (Space, tool keys) do not fire while a page has focus.

### 9.2 Hosting your own panel (the pattern every pack panel uses)

1. **Skin:** replace a form (or add a slot) with `<WebView name="MyPanel" title="MyPanel" size="0,0,w,h" attach="all"
   options="localonly"/>`, through a theme `transform` / `features.py` rule or the skin filter API (5.7).
2. **DLL:** under the named mutex `Local\StudioProWebViewSlotPatch`, wrap WebBrowserView vtable slots 26/27, chaining
   onto whatever they hold. In your attached wrapper, for views whose **title** is yours: force the WebView2 config byte
   on only around the original attach (restore it after), make sure the environment was started earlier (prewarm it
   from a one-shot `WH_GETMESSAGE` hook on the UI thread at startup, or the first view loses the race and gets IE; if IE
   appeared anyway, detach and re-attach).
3. Poll (100 ms, UI thread) until platform `+0x118` is non-NULL, QI-check it, set settings (web messages on, dev tools
   off unless testing, context menu off), add a WebMessageReceived handler, then Navigate to
   `file:///<root>/mods/<name>/web/index.html`.
4. **Protocol:** page -> DLL `window.chrome.webview.postMessage("cmd\npayload")`; DLL -> page `PostWebMessageAsJson`
   (UI thread only; from workers, post a window message to your message-only window first).
5. Track removal (slot 27) and drop your references; pages are reloaded on re-attach.

Ready-made examples in the pack: Session Library (Start page), Spotify panel (a gadget skin patched inside the plug-in
DLL), chat panel (Studio Assistant slot), Sound Search (Files tab), YouTube panel (Cloud tab), Track List, Plug-in
browser, Skins page (two web views inside the Options dialog), Home page, News card.

### 9.3 The WebView2 crash you must guard against (stock CCL bug)

When controller creation completes with an error (seen: `hr = 0x80004004 E_ABORT` because the host window went away
during the ~3 s creation right after a song close), cclgui stores the NULL controller and still runs the continuation
`0x1800be280`, which QIs and dereferences it -> access violation at `cclgui+0xbe340`. The pack's **web panel crash
guard** (`wv2guard.dll`, core) patches the continuation functor vtable slot `0x1804cf870` (validated == `0x1800be280`)
with a wrapper that runs the original only when a controller exists (log `wv2guard: WebView2 controller creation failed
for platform ... skipped`). Keep it installed whenever any mod creates WebView2 views.
The same mod also coalesces `put_Bounds` (platform vtable slot 19 calls it on every layout step; a lane-tab switch called
it 4 times per click and WebView2 showed one cut-off frame): it applies the last bounds once from a posted message.

### 9.4 Environment variables read at environment creation

- `WEBVIEW2_USER_DATA_FOLDER` (the pack's loader sets `<root>\WebView2`: the default `<exe folder>\Studio Pro.exe.WebView2`
  is not writable under Program Files, which broke every panel).
- `WEBVIEW2_DEFAULT_BACKGROUND_COLOR` (avoid a white flash).
- `WEBVIEW2_ADDITIONAL_BROWSER_ARGUMENTS` (`--remote-debugging-port=<n>` for tests).

### 9.5 Live theme for pages

Pages link `../../../skins/theme.css` (after their own CSS) and `skins/theme-live.js`. On a live colour change the
Live colours mod rewrites `skins\theme.css` and posts the plain string `theme-changed` (`PostWebMessageAsString`) to every
tracked web view; `theme-live.js` swaps the stylesheet link with a cache-busting query (no reload: form state kept).
**Your page's message handler must tolerate that plain string.** Don't define `--accent` in your page's `:root` if it
loads after theme.css (it would override the live accent). `skins\theme.css` restyles generic classes like `.row`,
`.title`, `.name`: prefix your class names.

### 9.6 Debug ports are process-wide

All mod pages share one WebView2 browser process. Any mod ini `debug_port>0` sets
`WEBVIEW2_ADDITIONAL_BROWSER_ARGUMENTS=--remote-debugging-port=<n>` for the whole process (the first one to set it
wins), which makes **every** panel (including the chat panel's API-key sheet) scriptable by any local process over the
Chrome DevTools Protocol. Use a port only while testing; ship `debug_port=0`.

### 9.7 Drag and drop out of a page

The page's mouse lives in another process, so the drag must start natively: page `pointerdown` + 6 px move sends
`drag\n<path>`; the DLL validates the path (extension, inside an allowed folder, exists), posts to its UI-thread
window, which calls `DoDragDrop` with a shell data object (`CF_HDROP`, via `IShellItem`) **only while
`GetAsyncKeyState(VK_LBUTTON)` is down**. Dropping a WAV on a track lane creates an audio event (copy effect).
Testing needs real input (`SendInput`); posted messages cannot start an OLE drag. Calling a CCL window's drop target
in-process via `GetPropW(hwnd, L"OleDropTargetInterface")` is not reliable (stale pointers, crashes): don't.

---

## 10. The command system

### 10.1 Command table (cclgui)

- Export **`CCLGetCommandTable`** (RVA `0xdf540`): lazy singleton (object 0x3f8 bytes, ctor `0x1800ff640`, global
  `0x18064e920`), returns **object + 0x30** (interface vtable `0x1804d9c98`).
- Slot `+0x50` = `0x1801016d0` **`interpretCommand(msg, char deferred, invoker)`**: deferred=0 -> `0x180101b70`
  runs it now; deferred=1 -> copies the msg into a queue (`0x180102050`) and returns 1 unless check-only.
  Slot `+0x68` = `0x180102a90` `findCommand(CString* cat, CString* name)`.
- **run-now** `0x180101b70(tableObj, msg, handler)`: a non-NULL `handler` gets `handler->vtbl[+0x20](msg)`
  (ICommandHandler::interpretCommand, IID `a90881cb-88dc-5241-ade0-c3ae489d75b5` at `0x1804c1078`) first; NULL ->
  the focused window -> its focus view (QI for the handler) -> window handlers -> parent chain -> the global handler list
  at table `+0x388`. `interpretCommand` always passes NULL.
- **CommandMsg:** `+0x00` category CString, `+0x18` name CString, `+0x30` invoker IUnknown*, `+0x38` flags (bit 0 =
  check only: "is it available now?"). **CString** = `{const char *text; IMutableCString *obj; ?}` (0x18 bytes). obj NULL
  = static literal (copies keep the raw pointer), so build real ones with ccltext `CCLCreateMutableCString(text)`
  (obj vtable `+0x20` text, `+0x08` retain, `+0x10` release).
- **Slot `+0x50` only sees check-only calls from the app**; menus, keys and the queue call run-now directly. To observe
  executed commands, redirect run-now's 6 `E8` call sites (`0x180275e62 0x1800fe350 0x1800fe420 0x1801017d2
  0x180259857 0x18027573e` on 8.1.2) through a near stub (3.4 item 4).
- To run a command safely from a mod: on the UI thread, first a check-only call, then a deferred call (or run-now when
  you need it synchronous). "Edit / Undo" through the deferred path applies a moment later.

### 10.2 Command names

`<Command category="..." name="..."/>` in exe RCDATA `COMMANDS.XML` (~278 with default keys; other
`<Commands name="Pro Tools|Cakewalk|Ableton Live|FL Studio">` blocks are keymap presets) plus
`%APPDATA%\Fender\Studio Pro 8\user.keyscheme` (~1,490 including the user's macros). Categories include Application,
Edit, Track, Transport, Console, Navigation, Event, Audio, View, Browser, Help, File, Musical Functions. Useful:
`View / Editor` (F2), `View / Console` (F3; both toggle and mark the song modified), `View / Start Page`,
`View / Studio Assistant`, `File / Save`, `File / Close`, `Edit / Undo`, `Edit / Select All`,
`Edit / Hide Selected Tracks`, `Browser / Show Instruments|Effects|Loops|Files|Pool|Home Page`. Every JavaScript edit
task is also a command (11.1).

### 10.3 From scripts

`Host.GUI.Commands.interpretCommand(category, name)`, and every object node has `interpretCommand(cat, name)`
dispatched through that node's own command handler (synchronous). Example used by the bridge to run a script in the
arrangement whatever has focus:
`Host.Objects.getObjectByUrl("object://hostapp/DocumentManager/ActiveDocument/EditEnvironment/MainEditor").interpretCommand("LLM Bridge", "Run Script")`.

### 10.4 Focus routing

Commands and edit tasks go to the editor that last had focus. Arrangement focused -> the real track list; Browser last
focused -> an editor with `numTracks = 0`. Route through `MainEditor` as above, or post a click into the arrangement.

### 10.5 Opening a document from code

Single-instance hand-off: send `WM_COPYDATA` with `dwData = 0x436d646c` ('Cmdl') and lpData = UTF-16 command line
`"<exe>" "<file>"` to the hidden `CCLMessageWindowClass` window titled "Studio Pro"
(`FindWindowEx(HWND_MESSAGE, ...)`, filter by process id). Works in-process from a worker thread. The main
`CCLWindowClass` window answers 1 and ignores it. Sender in cclgui: `0x1800da1a0`.

---

## 11. JavaScript edit tasks (read and change the open song)

Studio Pro's track/event/note functions are plain JavaScript run by the app's own engine. This is the cheapest way to
read or change the open song: no native addresses needed, every `functions.*` edit is undoable.

### 11.1 Script packages (`<app folder>\Scripts\*.package`)

- Same PACKAGEF format as skins (5.2). Inside: `metainfo.xml` (`Package:ID`), `classfactory.xml`, `*.js`, optional
  `translation.js`, optional `skin/skin.xml` (+ images) for task dialogs. **No signature check** for `Scripts\`;
  a new package is picked up at the next launch. Third-party `Extensions\*\scripts\*.package` are zips of encrypted
  `.j_` files (not readable). Read the shipped ones (trackedit, musicedit, audioedit, projectedit, toolset) with
  `skinpack.py` to learn the API: they are the best documentation.
- `classfactory.xml`: `<ScriptClass classID="{guid}" category="EditTask" subCategory="TrackEdit" name="..."
  sourceFile="x.js" functionName="createInstance">` with `<Attribute id= value=>` children:

| Attribute | Meaning |
|---|---|
| `commandCategory` | command category; default from subCategory: TrackEdit -> `Track`, EventEdit -> `Event`, MusicEdit -> `Musical Functions` |
| `menuPriority` | menu order; **-1 = hidden** from menus (still a command) |
| `menuGroup`, `menuFollow`, `TrackContextMenu`, `TrackSourceContextMenu` | where it shows |
| `arguments` | named args (`Name`, `Semitones`, ...) read via `context.getArguments()` |
| `useTrackSelection`, `useChannelSelection`, `useEventSelection`, `useTimeSelection`, `useLayerSegments` | what the context gets |
| `alwaysEnabled`, `allowEmptyParts`, `allowPatternParts`, `wantAudioParts`, `ignoreAudioEvents`, `ignoreEventLock`, `ignoreTrackLock`, `musicEditorOnly`, `supportsProject`, `supportsLauncher`, `hidden` | enable rules / scope |

  Other categories: `ExtensionHandler` (macro handler `MacroHandler.js`).
- **The package holds absolute paths if your JS does.** The pack's bridge packages contain
  `file:///<root>/mods/<x>/exchange/`; a package copied to another install runs the wrong folder's scripts.
  Generate packages per install (the pack ships `lbpkg.exe`, `tlpkg.exe`, `pbpkg.exe`:
  `<tool>.exe --build <out.package> <exchangeDir>`).

### 11.2 Task shape

```js
function createInstance() {
  return {
    prepareEdit: function (context) { return Host.Results.kResultOk; },   // optional, may show a dialog
    performEdit: function (context) {
      var f = context.functions;                     // EditFunctions, all undoable
      f.beginMultiple();                             // optional: group into ONE Edit > Undo step
      // ... edits ...
      f.endMultiple();
      return Host.Results.kResultOk;
    }
  };
}
```

One task run is one undo step (verified: a task adding 3 tracks = one Edit > Undo). Edits apply when the task ends.

### 11.3 Context, tracks, channels

- `context.mainTrackList`: `numTracks`, `getTrack(i)`, `numSelectedTracks`, `getSelectedTrack(i)`, `selectTrack(t)`,
  `getInsertPosition()`, `showTracks([tracks], visible)` (exactly those tracks, one undo step; never pass a 3rd arg),
  `unselectAll`, `isTrackSelected`, `makeTrackVisible` (scroll only), `expandAll`.
- `context.editor` (model, selection, environment, `cursorInfo`, `createSelectFunctions`), `context.iterator`
  (selected events), `context.getArguments()`.
- Track: `name, mediaType, channel, color, trackID, trackIndex (global), isFolder, isGlobalTrack, hidden, height,
  iconID, parentFolderID, layers, envelopes, startTime/endTime/lengthTime, createIterator() (done/next/first/last/previous),
  asEventList(), getTrack(), getRoot()`. A music (instrument) track has `channel === null`.
- Channel: `label` (display name), `name` (`Channel01`), `channelType` (`AudioTrack|AudioGroup|AudioEffect|AudioSynth|
  MusicTrack|AudioInput|AudioOutput`), `mute, solo, soloSafe, volume (1 = 0 dB), pan (0.5 centre), maxVolume, disabled,
  input {label, speakerType}, inserts, recordUnit, editor, openEditor(), focus(), connectTo()`, `findParameter(id)`.
  **Property writes work but are not undoable** (`channel.mute = 1`); a soloed track reads mute 0.
- Mixer lists: `functions.root.environment.find("MixerConsole").getChannelList(0 inputs | 1 tracks+buses | 2 outputs |
  3 masters)` -> `numChannels, getChannel(i), numSelectedChannels, getSelectedChannel(i)`.
- Inserts: `channel.inserts` (DeviceFolder: `insertDeviceClass("{GUID}")` = add a plug-in to the next free slot, one undo;
  `bypassAll`, `numChildren`); device slots are children named `FX01..FXnn` (allocation ids, not rack order):
  `slot.hasDevice, openEditor(), closeEditor(), find("Device")` (title = plug-in name), params `deviceName, deviceClass,
  bypass, disable, missing, softBypass`.
- Instrument insert: add an "Instrument" track (`addMediaTrack`), then
  `var mf = functions.root.createFunctions("MusicPartFunctions"); mf.executeImmediately = true;
  var slot = mf.addInstrument(classID); mf.connectTrackWithInstrument(track, slot, -1, false);` (as the shipped
  `Insert FX.js` does).

### 11.4 Edit functions (`context.functions`)

`newMediaTime, addEvent, removeEvent, transferEvent, moveEvent, resizeEvent, muteEvent, transposeEvent, renameEvent,
colorizeEvent, splitEvent, setLoopRange, enableLoop, setSyncPoint, enableSyncPoint, setEventAttribute,
makeIndependent, toBack, toFront, setEventIndex, addTrack, addMediaTrack, duplicateTrack, removeTrack, addEventOnTrack,
addEventOnParent, removeRange, moveToFolder, insertNewClip, setTrackDelay, setIconID, importFile, importFiles,
beginMultiple/endMultiple, setJournalEnabled/isJournalEnabled, executeImmediately`, plus `root`, `model`.
`functions.root.createFunctions("DeviceEditFunctions"|"AudioEffectFunctions"|"MusicPartFunctions")`.

Verified recipes and traps:
- **Add a track:** `functions.addMediaTrack(trackList.getInsertPosition(), "Name", Host.Engine.TrackFormats.findEqual("Mono"))`
  ("Stereo", "Instrument"). The new track is not visible to `numTracks`/`getTrack` until the task ends.
  `Engine.JS.*` constants are not in scope (they need `include_file`); using them throws.
- **Import audio:** `functions.importFile(Host.Url("file:///C:/dir with spaces/x.wav"), mediaTime, track, 0)` -> 1.
  Use the **raw** path in the URL (a `%20`-encoded path returns 0). **A missing time or a plain number as time crashes
  the app**: pass `event.startTime` or `context.editor.cursorInfo.cursorTime`. Flags (SDK ENGINE.JS): kMarkRecorded 1,
  kRemoveExisting 2, kAppendMode 4, kReportError 8.
- **Move:** `functions.moveEvent(ev, pos)` takes **quarter notes**: `functions.root.timeContext.secondsToPpq(sec)`.
  A MediaTime with `.seconds = s` lands at s beats.
- **Split:** `functions.splitEvent(ev, mediaTime)` returns the right part; split right-to-left on the original event
  (re-splitting a returned part inside the same task gives wrong offsets).
- `functions.transposeEvent(ev, semitones)` works; `ev.tune = x` and `setEventAttribute(ev, 'tune', x)` do nothing.
- `functions.muteEvent(track, true)` mutes the track's **events**, not the channel.
- `renameEvent(track, name)` renames a track (one undo step).
- Chords: on ChordTrack iterator events `e.chord.name` returns the chord name (`Am`, `F/A`, `Cmaj7`); `e.name` is empty.
- Inside `beginMultiple`/`endMultiple` a whole sequence is ONE undo; a `beginMultiple` in one task and `endMultiple` in a
  later task still grouped.

### 11.5 Song-level objects

- Object tree: `Host.Objects.getObjectByUrl("object://hostapp/DocumentManager/ActiveDocument")` = the song document.
  Nodes have only `find(name)`, `findParameter(id)`, `numChildren`, `name`, `title`, `parent`, `self`, `interpretCommand`
  (no child iterator). `object://studioapp/...` is an alias of `hostapp`. Known children: `TrackList` (`ArrangerEditComponent`,
  `ChordEditComponent`, `TimeSelection`, `TimeSnapper`, `Quantize`, `ScratchPad`, ...), `Environment` (`Transport`,
  `Metronome`, `AudioEngine`, `AudioMixer`/`Channels`, `MixerConsole`, `TransportPanel`, ...), `Editor`,
  `EditEnvironment/MainEditor` (the arrangement), `MediaPool`, `PoolBrowser`, `Notepad`, `Lyrics`, `Launcher`,
  `TempoEditor`, `SongSetup`, `Marker`, `Master`, `Inspector`, `EventInspector`, `ActionJournal`, `MusicEditorManager`.
- **Tempo and key, live:** `.../ActiveDocument/Environment/TransportPanel` has param `tempo` (BPM; also `loopStart/End`,
  `primaryTime`, `globalTransposition`) and children `KeySignature` (params `list` 0..26: 0..12 major Gb..F#, 13..25 minor
  Ebm..D#m, 26 none; `scale` 0 none 1 Major 2 Minor; `name` like "Em") and `TimeSignature`. Pitch class: major =
  `7*(list-6) mod 12`, minor = `(7*(list-19)+9) mod 12`. `findParameter(k).setValue(v, true)` writes the song.
  `Transport` itself exposes no tempo/position params.
- `functions.root.timeContext` = `{tempoMap, timeSignatureMap, ppqToSeconds(q), secondsToPpq(s), getTimeSignature(t) ->
  {numerator, denominator}, getBarStart(t), sampleRate}`. BPM at beat q = `60 / (ppqToSeconds(q+1) - ppqToSeconds(q))`.
- `context.editor.cursorInfo`: `cursorTime, loopStart, loopEnd, loopActive, playing, recording, setCursorTime,
  setEditCursorTime, setLoopRange, toggleLoop, beginEdit/endEdit`.
- MediaTime: `seconds, musical (quarter notes), samples, frames, string` + `convert, clone`.
- `functions.root`: `findTrackByClass("MarkerTrack"|"ArrangerTrack"|"ChordTrack"|"LyricsTrack"|"VideoTrack"|"FolderTrack")`,
  `findTrackByID(guid)`, `findFolderTrack(id)`. Marker events add `markerType` (2 Start, 3 End).
- Arrangement model `context.editor.model`: `markers`, `envelopes`, `arranger` (`addArrangerEvent`), `chords`
  (`addChordEvent`), `signatures`, `layers`, `patterns`, `folders`, `findEvent, findTrack, getTrackAt, editLoopRange,
  selectTimeSegment, setDocumentDirty`.
- **Selected track's inspector:** `doc.find("Inspector").find("TrackController")` params `name`, `timeFormat` (0 Seconds,
  1 Beats), `tempoFollow` (0/1/2 Timestretch), `stretchMethod`. `setValue(v, true)` = real undoable edit; `.value = n`
  only changes the display. It follows `selectTrack` one call late (select in one task, set in the next).
- Event inspector: `doc.find("EventInspector").find("EventInfo")` params `tune` (cents), `transpose`, `speed`,
  `timeLock`; it shows the event the **user** selected last (selecting from script does not move it).
- Active browser tab: `Host.Objects.getObjectByUrl("object://hostapp/Browser").findParameter("commandLinks")`.
- Metronome: `.../Environment` -> `find('Metronome').findParameter('clickOn')`; AudioEngine params `sampleRate,
  bufferSize, masterDeviceName, processLatency`.

### 11.6 Host globals and I/O

`Host.{Interfaces, Results, Classes, Objects, Services, Console, Signals, Locales, SystemInfo, IO, FileTypes, GUI,
Graphics, Engine, Settings}`. `Host.GUI`: `alert, ask, runDialog, openUrl, showFile, addIdleTask`.
`Host.IO`: `File, findFiles, openTextFile, createTextFile, loadJsonFile, XmlTree, openPackage, createPackage,
toBase64/fromBase64`. `Host.IO.createTextFile(Host.Url("file:///..."))` writes a **UTF-8 BOM** first (strip it when your
DLL reads the file; ASCII with `\uXXXX` escapes both ways avoids encoding issues). `Host.Engine`: `TrackFormats,
TrackColorPalette, TrackIcons, MediaClips, createFormatter`. `new Function` and `JSON` exist. Native objects are not
enumerable (`Object.keys` -> `[]`); `__ccl_accessNativeProperty(obj, name)` is the generic getter.

### 11.7 The Scripting Error trap

**Any thrown JS error shows a modal "Scripting Error" TaskDialog, even inside try/catch**, twice per throw (one per catch
frame), and blocks the UI thread until OK is pressed. Guard every probe with `typeof`, never dereference a
maybe-undefined object, never loop over calls that may throw (200 throws = ~100 s of modal churn). A watcher thread can
post `WM_COMMAND IDOK` to `#32770` windows titled "Studio Pro" on the UI thread while your script runs, but snapshot
the dialogs open before you start and leave those alone (they are the user's). Probing
`DocumentManager.activeDocument` deeply once stalled the UI for >10 s: keep probes small.

---

## 12. The MCP bridge (llmbridge) and other mods' exports

The pack's optional **LLM bridge** mod (`llmbridge.dll`) is the fastest way for you, the agent, to drive and inspect a
running Studio Pro with text only. It is an MCP server inside the app, bound to `127.0.0.1` only.

### 12.1 Connecting

- Endpoint: `POST http://127.0.0.1:<port>/mcp` (MCP Streamable HTTP, JSON responses; methods initialize, ping,
  tools/list, tools/call; notifications -> 202). `GET /health` is open. Port: `<root>\mods\llmbridge.ini port=` (default
  39765). Requests with a non-localhost `Origin` are rejected.
- **Token:** every `/mcp` request needs `Authorization: Bearer <64 hex>` = the contents of
  `<root>\mods\llmbridge\mcp.token` (made on first start, readable only by this Windows user; 401 otherwise). Each install
  makes its own token: never copy, commit or share it.
- Register with Claude Code:
  `claude mcp add --transport http studiopro http://127.0.0.1:39765/mcp --header "Authorization: Bearer <token>"`.
- Give calls a 45 s timeout (a stalled script can hold the UI up to 30 s).

### 12.2 Tools

| Tool | What |
|---|---|
| `get_status` | main window title (= which song, `*` = unsaved) |
| `list_commands {filter?, category?}` | the command table |
| `run_command {category, name, force?}` | runs a command on the UI thread (check-only first, then deferred; unknown names refused) |
| `recent_commands` | the last 64 commands the user executed (the mods' own timer tasks are filtered out) |
| `run_script {code}` | runs JavaScript as an edit task in the open song (below) |
| `get_project` | live track list |
| `get_tempo`, `get_key`, `get_markers`, `get_plugins`, `get_events {track, limit?}` | read song state |
| `add_track {name, format}`, `rename_track {track, name}`, `set_track_state {track, mute?, solo?}` | simple edits (track = index or exact name) |

### 12.3 `run_script` mechanics

The DLL writes `<root>\mods\llmbridge\exchange\request.js`, runs the hidden command `LLM Bridge / Run Script` immediately
on the UI thread, and reads `response.json` = `{ok, result | error, log}`. Your code is the body of
`new Function("context", "functions", "trackList", "log", "Host", code)`: use `return <value>` (JSON-serialisable) and
`log("...")`. By default the task re-dispatches itself to the arrangement (`MainEditor`, 10.3) so it sees the real
tracks whatever has focus; put `// @noroute` on the first line to run in the focused editor (piano roll etc.). A thrown
error is still answered (the bridge's guard dismisses the Scripting Error box and wraps the reply as
`{"dialog": ..., "response": ...}`), but see 11.7: avoid throws. Example:

```js
var n = trackList.numTracks, out = [];
for (var i = 0; i < n; i++) { var t = trackList.getTrack(i); out.push({ i: i, name: t.name, type: t.mediaType }); }
return out;
```

### 12.4 Using the bridge from your own mod (in-process)

`llmbridge.dll` exports (call from a worker thread, **never from the UI thread**):

```c
int LLMBridgeRunScript  (const char *code, char *out, size_t cap);             // 1 = a script ran; out = reply JSON
int LLMBridgeRunScriptEx(const char *code, char *out, size_t cap, int flags);  // flags: 1 = no error-box guard,
                                                                               //        2 = skip while a modal is open
```

`GetProcAddress(GetModuleHandleW(L"llmbridge.dll"), "LLMBridgeRunScriptEx")`; fall back gracefully when the bridge is
not installed. A polling mod should skip its poll while the user is busy (mouse button held, menu or move/size loop,
caret shown with input in the last 700 ms): a script dispatched mid-drag once replayed an edit on the wrong track.
Own script package instead of the bridge: write your own hidden edit task with its own exchange folder (the Track List
and Plug-in browser mods do this), generate the package per install, and expect ~80 ms per hop (first call after idle
~500 ms).

### 12.5 The chat panel

The bridge can also show an own-LLM chat panel in the Studio Assistant slot: the skin rule swaps that panel's content
form for `<WebView name="LLMChat" title="LLMChat" size="0,0,380,500" attach="all" options="localonly"/>`; only the form
is swapped (the mods leave licensing untouched). The page calls LLM HTTP APIs through the DLL (WinHTTP, so no CORS:
local Ollama / LM Studio work), runs a tool loop (<= 16 turns) over the same tools, and stores its settings DPAPI-encrypted
(`LLMChat.json` beside Studio Pro's settings; a copied file cannot be read by another Windows user). Gemini's OpenAI-compatible
endpoint rejects tools whose schema has empty `properties` (omit `parameters`).

---

## 13. Files and data formats

### 13.1 Settings folder `%APPDATA%\Fender\Studio Pro 8\`

| File | What |
|---|---|
| `Studio Pro.settings` | XML settings. Browser roots: `<BrowserState root="root/Volumes/C:/...">` (the Files tabs; written on quit). User Content folder: `<Section path="Content"><Attributes><Url x:id="location" type="2" url="file:///<your Documents>/Studio One/"/>` (%XX = UTF-8 bytes, XML entities, `file://host/` = UNC). Songs live in `<Content location>\Songs` by default. |
| `RecentDocuments.settings` | ordered URL list of recent documents (+ pinned), written on quit |
| `user.keyscheme` | key assignments and user macros (command names) |
| `user.colorscheme` | Appearance slider levels |
| `WindowState.settings`, `x64\Plugins-en.settings`, `Diagnostics.settings` | written at shutdown |
| `DataStore.db` | app-managed database: leave it alone |

The pack's mods keep their own small files there with their own names (`SessionLibrary.json`, `SpotifyPanel.json`,
`NewsPanel.json`, `LLMChat.json`, `SoundIndex.db`, ...). A session on a real profile rewrites several settings files
(tempo range, window state, recents): snapshot before tests and restore after if you test on the real profile.

### 13.2 Song files (`.song` = zip)

- Page layout is saved inside the song: `Workspace/perspective.xml`, `PerspectiveFrameState name="EditorZone"`:
  `visible=0` = Song page, `windowID="Editor1"` = Edit, `windowID="Console"` = Mix. A song re-opens on the same page.
- Note data: `Performances/<part>.musicx`, CCL binary: `{`/`}` object, `[`/`]` list, key = `i` + length + name,
  int = `i` byte, double = `D` + big-endian 8 bytes, string = `S` + ident. Part root keys `timeFormat`,
  `events[{start, pitch, noteId, length, velocity}]`, `envelopes[...]` (MIDI CC) and the per-note controller lists
  `noteControlPressure` / `noteControlPitch` / `noteControlTimbre`, each `[{pitch: <note pitch>, events: [{start, value}]}]`.
- Song folders hold `Media\`, `History\`, `Cache\`, `Mixdown\`, `Bounces\` and `(Autosaved)` files: skip those when you
  scan for songs.

### 13.3 Skin and script packages

PACKAGEF (5.2). Skins: exe RCDATA; scripts: `<app folder>\Scripts\*.package`.

---

## 14. Exe internals worth knowing (`Studio Pro.exe`, base 0x140000000)

### 14.1 Undo: the action journal

- `ActionJournal::perform` = `0x14141b3a0(journal, action, flags)`: executes (`0x141356340`, calls action `+0xb8`, drops
  no-op children), logs `exec`, pushes on the undo list (`journal+0x48`), merges with the previous action when < 0x1f5 ms
  apart and `prev->vt+0x98(new)` agrees (`mrge`), returns 0 and releases the action when execute fails (`ignr`). Then
  trims to the undo limit and clears the redo list (`journal+0xa8`). Undo `0x14141b820`, redo `0x14141bab0`
  (busy flag `+0x1a0`). List node = 0x18 bytes `{action*, older, newer}`; undo list top `+0x90`, oldest `+0x98`,
  count `+0x168`; append `0x1408f6680(list, action)`, pop `0x1408f66f0(list)`. Prologue of perform = 3 x
  `mov [rsp+n], reg` (15 bytes): detourable; calling undo/redo/append directly from the UI thread worked in a prototype.
- Action vtable layout (all actions): `+0x08` AddRef, `+0x30` isA, `+0x98` merge-with-next, `+0xb8` do-on-perform,
  `+0xc0` undo, `+0xc8` redo, `+0xd0` dispose. Base ctor `0x1413559c0(action, &titlePair)`; action `+0x30` title,
  `+0xe0` model-change notifier, `+0xe8` envelope, `+0xf0` point. `titlePair = {IString*, hash}`; `0x141d04b20` is a
  ready empty pair (the ctor then loads its own title).
- Edit context (stack object, 0x48 bytes): ctor `0x141357160(ctx, target)`, perform `vt+0x88` = `0x141357350`.
  Piano-roll edit target of a layer: `0x1410a2d30(layer)`.

### 14.2 Envelope points (used by the per-note pitch-curve mod)

- `MovePointAction` ctor `0x1412a1e50(action[0x108], env, pt, double time, double value, &titlePair)`, vtable
  `0x1418a1598`: stores the values from **before** the UI moved the point (the drag writes the point live); undo/redo
  swap stored and current. `InsertPointAction` ctor `0x1412a1620(action[0x100], env, newPt, 0, &titlePair)`, vtable
  `0x1418a1918`; `RemovePointAction` ctor `0x1412a1ce0`, vtable `0x1418a1308`.
- Envelope point `0x1404d8560(pt[0x60], time, value)`: `+0x50` time, `+0x58` value, `+0x30` next link, `+0x38` prev link
  (links point at `pt+0x30`). Envelope (`0x1409b2c30(region)`): `+0x48` first link, `+0x50` last, `+0x68/+0x70` min/max
  time, `vt+0xa0(pt)` index, `vt+0xb0(pt)` add.
- The piano roll's note-expression drawer is `0x1410a1590` (called at `0x1410a151b`); it paints envelopes only when the
  setting `Editing.NoteColoring.noteControllerIndicator` is on (piano roll Note Color > "Note Controller Indicator";
  Note Color = Part switches it off).

### 14.3 Ruler marker labels and the loop range (engine-drawn)

- `MarkerEventTheme` (style `engine/eventtheme.xml`, inherits `EventTheme`): vtable `0x141758ee0`, ctor `0x1409e68f0`
  (defaults markerWidth 5, lineWidth 2); loader reads `markerImage`, `markerWidth`, `lineWidth`; **drawEvent = slot
  `+0x100` (`0x1409e6e30`)**: one 6-point path (label box with a cut corner + stem) filled in the marker colour, a 1 px
  edge line, then the text. `markerImage` draws an extra bitmap at the stem foot; it does not replace the flag.
- `TimeRuler` painter (style `Shared/TimeRuler`): primary vtable `0x14186eef0`; `+0xf0` drawBackground, `+0xf8`
  grid/labels, **`+0x108` drawLoop (`0x1410515c0`)**, `+0x120` drawLoopMarker (triangles). Metrics keys `loopgap,
  loopheight, markergap, markertolerance, startmarkerwidth, loopdrawheight, hilite, nolines, nozoom, loopmarker`;
  colour keys `loophilite, loopdisabled, loopfore, loopdimmed, looprollfore, loopmarkerfore, startmarker`.
- Project-page markers use a different theme (`ProjectMarkerTheme`, vtable `0x141759078`).

### 14.4 Arrangement column dividers

Arrangement editor ctor `0x1408538a0` registers `trackNotesDivider` (id 0x3ed) and `trackIconsDivider` (0x3ee), int
44..190, default 84; main-part width = controlsWidth - notesDivider (if shown) - iconsDivider (if shown), kept >= 210.
Mixer strip divider param names: `faderDividerParamName`, `notesDividerParamName`, `iconsDividerParamName`.

### 14.5 Startup and shutdown order

`WinMain` (`0x1401e3370`) adds `<install dir>\3rd party` to the DLL search path -> `App_Startup` (`0x1401e3570`):
`CCLModuleMain(h, 1)` for ccltext -> cclsystem -> cclsecurity -> cclnet -> cclgui, GUI init, app object
(id `com.fender.studioapp`, settings registration, migration from older products' settings folders), single-instance
check -> `StudioApp_Startup` (app vtable `0x141707818` slot `+0xe0`, `0x1401b0dc0`): window manager, locale, browser
partner tabs (`0x140188620`: Splice, Tonalic, Moises), plug-in manager scan, scripting manager, object table, GUI,
services, plug-in categories (`UserOption`, `VSTUserOption`, `Gadget`, `ArtistProfile`) -> main run loop
(`CCLGetGUI()->[+0x40]`) -> `App_Shutdown` -> unload CCL modules in reverse. Skins are parsed early; a mod's setup
thread runs before the first skin open (~2 s after the loader starts) and before the menu bar is built.
Shutdown spends ~5 s in a 5 s timeout in `dspshell.dll`'s `DspEffectAudioWorker` stop (loader-lock deadlock, then
TerminateThread). The optional fast quit releases that pool early (destructor at `dspshell.dll` RVA `0x213c30`, global
RVA `0x2c80f0`, release = vtbl `+0x10`, guarded by 22 bytes): close 11.6 s -> 6.3 s. Settings files are written before
and after that wait: never kill the process to "speed up" a close.

---

## 15. Testing and tooling recipes

### 15.1 Set up a sandbox

1. Copy the whole app folder (`C:\Program Files\Fender\Studio Pro 8`, ~650 MB) to a writable folder you own. Keep a
   pristine copy too, to hash-compare against after uninstall tests.
2. Make a root folder with `sandbox.ini` (`redirect=1`, `data=<a copy of your AppData\Roaming\Fender profile, in its own
   folder structure>`, unique `[instance] id=`), `mods\`, `skins\`; put the pack's `version.dll` next to the copied
   `Studio Pro.exe`; start it with `STUDIO_PRO_X_ROOT=<root>` set in the environment.
3. Use a throwaway copy of a song for tests, never the user's real songs. A big song with plug-ins takes 60-80 s to load
   and ~11 GB of RAM; an empty song is far faster.
4. Check `sandbox.log` for `app build ... matches`, `mod loaded:` lines, no `ERROR`.

### 15.2 Driving the app without taking over the user's screen

- **Never send raw mouse input** (SetCursorPos/mouse_event/SendInput) unless the user agreed and is away; another window
  may be on top. Prefer, in order: the MCP bridge (`run_command`, `run_script`, `get_status`), posted window messages,
  CDP inside web pages, UI Automation, and only then guarded real input.
- Posted input: `PostMessage(WM_KEYDOWN/UP)` to the main window works for keys (F2/F3/F5, Space); posted
  `WM_LBUTTONDOWN/UP` (client px) reaches the arrangement but **not the piano roll** and not menu items. Close pop-ups by
  posting Escape to the pop-up's own hwnd. Menu items: open the menu with a posted click on its title, then post
  Up/Down/Enter to the pop-up hwnd. Posted mouse moves do not trigger tooltips or menu hover (CCL reads the real cursor).
- Web views inside the Options dialog are `Chrome_RenderWidgetHostHWND` children: a posted click on the dialog does
  nothing there.
- Real input, when allowed: before every `SendInput`, check `WindowFromPoint` belongs to Studio Pro's process (or its
  `msedgewebview2` children), keys only while the main window is foreground, restore the cursor afterwards.
- DPI: Studio Pro is per-monitor DPI aware; DPI-unaware tools see scaled coordinates. Call
  `SetProcessDpiAwarenessContext(-4)` (per-monitor v2) in your helper before `GetWindowRect`.
- Opening and closing: wait for the song to load (`get_status` title) before input. After anything that shows the Start
  page, wait until its web views report ready before the next input (9.3). Close with `WM_CLOSE` to the main window;
  a save prompt is a `#32770` TaskDialog: click "No" via `BM_CLICK` only for a throwaway song. Never kill the process
  (it skips settings writes and can trigger the "Studio Pro Safety" start screen next time).
- Start-up prompts (crash recovery, safe-mode questions, the Safety screen) appear in the first ~90 s: don't open dialogs
  in that window; dismiss them deliberately.

### 15.3 Seeing the app (text first)

- Window list (class, styles, rect) via `EnumWindows` filtered by process id: the cheapest "what is open" check.
- Screen capture: bring the window forward (AttachThreadInput trick) then CopyFromScreen; `PrintWindow` returns stale
  frames for the main window, but `PrintWindow(hwnd, hdc, PW_RENDERFULLCONTENT)` works for dialogs that may be covered.
  Capture only a region and measure pixels with a small script instead of looking at full screenshots.
- Find a drawn element without looking: give it a test colour (ini), capture, scan for that colour.
- One-frame glitches: GDI bursts miss them on high-refresh screens; record with ffmpeg Desktop Duplication
  (`ddagrab`, 120 fps) and diff frames (`tblend=all_mode=difference`).
- Text is ClearType-rendered (colour fringes): take a glyph's brightest pixel for text colour, the dominant colour for fills.

### 15.4 Web pages (CDP)

Set `debug_port=<n>` in one mod's ini (test only), then use the Chrome DevTools Protocol on `http://127.0.0.1:<n>/json`
to evaluate JS, click, read text and capture exact page images. All mod pages share one browser, so one port reaches
every page (match by URL). `Runtime.evaluate` with `awaitPromise` may not return here: store results into
`window.__x` and read them with a second call. Page edits hot-reload with `location.reload()`. Pages can also be
tested with no Studio Pro at all: serve the `web\` folder over http and stub `window.chrome.webview`.

### 15.5 Crashes and hangs

- Dumps: `%LOCALAPPDATA%\CrashDumps\Studio Pro.exe.<pid>.dmp`. A small minidump reader (exception record, registers as
  module+offset, return addresses into your modules) is enough to attribute a crash without a debugger.
- Hang triage: walk every thread's stack (dbghelp `StackWalk64`) of the live process; one seen hang was the UI thread
  waiting in `windowsmidi.dll` behind an audio-service call = environment (several copies on one audio device), not a mod.
- Known stock crashes (not yours): the WebView2 NULL-controller continuation (9.3); intermittent shutdown crashes in
  third-party plug-in teardown.

### 15.6 Ghidra queries that did the mapping

Write small headless scripts for: strings -> referencing functions (decompiled); decompile addresses + list callers;
dump pointers near a data address + decompile referencing code; class meta -> factory -> ctor -> vtable (4.2);
diff two vtables; functions using given constants; every user of a global; dump a pointer table with targets;
every caller of an import; instruction listing around an address (to see where arguments are saved before reading a
dump). Without Ghidra: `llvm-objdump -d --start-address=0x... --stop-address=...` on the DLL plus a Python PE walker
to read vtables and find qword references.

### 15.7 Shell gotchas (agents on Windows)

- Bash heredocs can mangle backslashes in Windows paths: write C/Python files with a file-writing tool, or use `/` or
  `0x5c` in paths.
- PowerShell variables are case-insensitive (`$out` overwrites `$Out`); `$pid` is read-only; splat arrays, not strings,
  for native arguments. `robocopy` without `/XO` copies older files over newer ones; `/MIR` deletes extras.
- A loaded DLL cannot be overwritten but can be renamed: build to `x.dll.new`, rename when Studio Pro is closed.

---

## 16. Address reference (8.1.2.113407)

Most addresses are given inline in their sections. The rest:

| Module | VA | What |
|---|---|---|
| cclgui | `0x180083660` / `0x180083670` | retain / release |
| cclgui | `0x1802f51b0` | SkinManager init (reads Documents `ccl-skin-development.properties`, `skins.properties`; singleton `0x18064ede8`) |
| cclgui | `0x18034f680` | emit `ColorSchemeChanged` + view event 0xd + redraw all windows |
| cclgui | `0x18034f460` / `0x18034d550` | scheme get-or-create / token by name |
| cclgui | `0x18037a900` / `0x1802fb7c0` | reload one theme / skin package parser (package URL -> skin.xml) |
| cclgui | `0x180379c80` | view cache reset walk (reloadAll) |
| cclgui | `0x1800bd200` | WebView2 platform: start controller creation (`env->CreateCoreWebView2Controller(hwnd, handler)`) |
| cclgui | `0x1800bdcc0` | controller-created completion `Invoke(hr, controller)` (no NULL check) |
| cclgui | `0x1800bcae0` | promise set-state -> runs continuation functor slot 3 |
| cclgui | `0x1804cf858` / `0x1804cf870` | continuation functor vtable / its slot 3 |
| cclgui | `0x1804c3dd0` / `0x1804c2d20` | Divider option table / TextBox option table |
| cclsystem | export `CCLGetObjectTable` (`0x180060f20`) | returns object+0x30; slot `+0x50(Url*)` resolves `object://hostapp/...` to a node |
| cclsystem | `0x1800cbd6f` | the single-instance `CreateMutexA` call |
| exe | `0x141d6ccc8` | global: CCL constant `"ColorSchemeChanged"` (exe side is a consumer; message handler `0x14025bb90`) |
| exe | `0x1412f9da0` / `0x1412faf50` | Appearance page `ColorSchemeOption` ctor / param handler -> `setLevel` |
| exe | `0x140a335e0` / `0x140a33790` | track icon image name (`Engine/TrackIcon:`) / 16x16 tinted icon bitmap for lists |
| exe | `0x1401d0ae0` | mixer strip divider param names |
| exe | resources | RCDATA `DEFAULT.SKIN` ... (16 skins), `COMMANDS.XML`, `MENUBAR.XML`, `CCLGUI.CONFIG`, `SDK/*.JS` |
| dspshell | RVA `0x213c30` / `0x2c80f0` | worker pool destructor / its global |

---

## 17. What each pack mod does and which technique it uses

Use these as templates: find the closest one, then read its INFO.txt and ini.

| Mod (DLL) | Visible effect | Technique |
|---|---|---|
| core loader (`version.dll`) | loads mods, serves skins | proxy DLL, IAT hooks in cclsystem (resources, folders, network, mutex), build guard |
| web panel crash guard (`wv2guard`) | no crash when a web panel closes while opening | vtable slot wrap on a continuation functor; `put_Bounds` coalescing |
| smooth motion (`springs`) | calm transitions and pop-up fades | `E8` call-site redirect, table write (`spring` timing), WAM interpolator, ViewAnimator immediates, ULW hook |
| glass pop-ups (`glasspopups`) | rounded, bordered menus | cclgui `UpdateLayeredWindow` IAT hook, DIB edit, DWM attributes 33/34 |
| ruler glass (`rulerglass`) | pill markers, rounded loop | exe vtable span flags + shared D2D primitive swap |
| per-note pitch curves (`mpe_pitch`) | edit note-expression pitch in the piano roll | call hook at the note-expression drawer, envelope actions through the action journal (one undo step per edit); hot-reloaded implementation DLL |
| title + icon (`brandtitle`, `appicon`) | custom title and window icon | `SetWindowTextW` IAT hook + polling; `WM_SETICON` sweep |
| hide help items (`hidehelp`) | two Help entries hidden | same-length in-memory edit of `MENUBAR.XML` |
| live colours (`palette`) | colour changes without restart | ColorScheme token writes + `setLevel` refresh; `reload` = `SandboxSkinFlush` + `reloadAll`; tracks web views to post `theme-changed` |
| skins page (`skinswitch`) | Options > Skins | skin rule adds two WebViews to `UserOptionDialog`; page commands run a Python helper (`skinpage.py`); restart helper |
| home page, session library, news card, Spotify panel | Start page panels | WebView slots (9.2); Spotify patches the SoundCloud gadget's own skin inside its DLL copy (original kept as `.orig`) |
| sound search (`soundsearch`) | sample search in the Files tab | WebView in a skin `Variant`; SQLite FTS5 index; WASAPI preview with pitch/tempo; native `DoDragDrop`; live tempo/key via the bridge |
| track list, plug-in browser | modern panels | skin filter API (in-memory `DEFAULT.SKIN` patch), own script packages + exchange folders, polling with user-busy guard |
| YouTube panel (`youtube`) | Cloud-tab audio search and download | WebView; helper processes in a Job object; native drag |
| LLM bridge (`llmbridge`) | MCP server + chat panel | command-table wrap, `E8` redirects of run-now, script package exchange, local HTTP server with a token |
| stem splitter engine (`stemhook`) | better stems behind Audio > Separate Stems | detours the native separation engine calls to an external open-model runner (8.1.2 only; see its INFO.txt) |

---

## 18. Gotchas, collected

1. Two Studio Pro copies on one audio device freeze each other. One app at a time unless you separate them (2.6).
2. The skin the running app uses is memory-mapped: close the app before rebuilding it (or use live reload, 5.9).
3. Every PACKAGEF entry must be deflated; a raw entry silently renders nothing.
4. A Style and an Image with the same name crash the app at form load.
5. hsl() with decimals loses its alpha; first token definition wins across packages; CCL truncates HSL to 8 bits.
6. `WH_GETMESSAGE` hooks vanish when the installing thread exits.
7. Set your real-function pointer before patching an import slot; use the loader's `PatchImport`.
8. Menus are new layered windows each time and bypass cclgui's window-creation imports; catch them at the first
   `UpdateLayeredWindow`. DWM acrylic on them halves hover frame rate.
9. WebView2 is off in stock CCL; force it per view, prewarm the environment, and keep the crash guard installed.
10. Any thrown JS error shows a modal, even inside try/catch. Probe with `typeof`.
11. `importFile` with a missing or numeric time crashes the app; URL paths must be raw, not %-encoded.
12. `moveEvent` positions are quarter notes; property writes (mute) are not undoable; `functions.*` are.
13. Commands and tasks go to the last-focused editor (Browser focused = zero tracks); route via `MainEditor`.
14. Script packages that embed absolute paths break when copied; generate per install.
15. `debug_port` in any mod ini opens every web panel to local CDP access; ship 0.
16. `Host.IO.createTextFile` writes a UTF-8 BOM.
17. Posted clicks do not reach the piano roll or menu items; posted mouse moves do not hover.
18. Never kill the process to close it; never patch files of the app on disk; keep an off switch on every mod.
19. On another build: the loader skips address-bound mods by design. Re-find, re-verify, then update `[app] build=`.
20. Keys typed in a web panel go to the browser process: app shortcuts do not fire while a page has focus.


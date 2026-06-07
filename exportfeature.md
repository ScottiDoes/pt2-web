# Feature: Export files to the user's computer (web build)

Status: **implemented** (Save-hook approach). Scope: **Module (.MOD)** and
**Samples**. (Whole-project `.zip` backup was considered but deferred.)

Implementation summary (web build only, all under `#ifdef __EMSCRIPTEN__`):
- Shared helper `pt2WebDownloadFile(vfsPath, mime)` in `src/pt2_diskop.c`
  (declared in `src/pt2_diskop.h`). Reads from the VFS and triggers a browser
  Save-As download. Relative paths resolve against the chdir'd `/persist` cwd.
- `modSave()` in `src/pt2_module_saver.c` calls it after a successful save
  (`application/octet-stream`).
- `saveSample()` in `src/pt2_sample_saver.c` calls it after save, with a MIME
  per format (WAV→`audio/wav`, IFF→`audio/x-aiff`, RAW→`application/octet-stream`).
- `pt2_mod2wav.c` refactored to use the shared helper (removed its inline EM_ASM).
- Filename sanitization/fallbacks were already handled by the existing savers
  (`sanitizeFilenameChar`, `untitled.mod`/`untitled` fallbacks); the download
  filename is `path.split('/').pop()` (basename only).

Verified: web build compiles; in-browser FS→Blob→anchor path confirmed working
(correct bytes + basename). The dedicated browser-side Export button (§4) was
not added — Save-hook covers the requirement.

This doc is self-contained so it can be picked up in a fresh session. See also
`../DEPLOY.md` (staged deploy) and the memory note `project-layout-and-deploy`.

---

## 1. Problem

In the web build the app's "Save" only writes into the **IndexedDB-backed
virtual filesystem** (`/persist/modules`, `/persist/samples`). Those files
persist across reloads but never reach the user's real computer, and can't be
moved to another browser/machine. Users expect "Save" in a tracker to give them
a file.

**Goal:** when the user saves a module or a sample, also deliver it as a browser
**download** (Save-As) to their real computer.

---

## 2. What already works (reuse this)

MOD2WAV (render song → WAV) **already downloads** to the host. It's the canonical
pattern — copy it. From `src/pt2_mod2wav.c` (~lines 341-364):

```c
#ifdef __EMSCRIPTEN__
    if (!editor.abortMod2Wav) {
        EM_ASM({
            var path = UTF8ToString($0);
            try {
                var data = FS.readFile(path);                       // Uint8Array from VFS
                var blob = new Blob([data.buffer], { type: 'audio/wav' });
                var url  = URL.createObjectURL(blob);
                var a = document.createElement('a');
                a.href = url; a.download = path.split('/').pop();    // filename only
                document.body.appendChild(a); a.click(); document.body.removeChild(a);
                setTimeout(function(){ URL.revokeObjectURL(url); }, 15000);
            } catch (e) { console.error('MOD2WAV download failed:', e); }
        }, lastFilename);   // lastFilename = full VFS path of the file just written
    }
#endif
```

Key facts:
- `FS` is an exported runtime method (`EXPORTED_RUNTIME_METHODS` in `build-web.sh`),
  so `FS.readFile(path)` works from `EM_ASM`.
- `FORCE_FILESYSTEM=1` is set, so the VFS is always present.
- The file must already be written to the VFS before the download call.

---

## 3. Design

### 3a. One reusable C helper (DRY)
Add a single download helper instead of duplicating the `EM_ASM` block. Put it
somewhere shared (e.g. `pt2_diskop.c`/`pt2_diskop.h`, or a small new
`pt2_web.c`):

```c
#ifdef __EMSCRIPTEN__
#include <emscripten.h>
// Read a file from the Emscripten VFS and hand it to the browser as a download.
// mime: e.g. "application/octet-stream" for .mod, "audio/wav" / "audio/x-aiff" etc.
void pt2WebDownloadFile(const char *vfsPath, const char *mime)
{
    EM_ASM({
        var path = UTF8ToString($0);
        var mime = UTF8ToString($1);
        try {
            var data = FS.readFile(path);
            var blob = new Blob([data.buffer], { type: mime || 'application/octet-stream' });
            var url  = URL.createObjectURL(blob);
            var a = document.createElement('a');
            a.href = url; a.download = path.split('/').pop();
            document.body.appendChild(a); a.click(); document.body.removeChild(a);
            setTimeout(function(){ URL.revokeObjectURL(url); }, 15000);
        } catch (e) { console.error('download failed for', path, e); }
    }, vfsPath, mime);
}
#endif
```
(Optionally refactor MOD2WAV to call this too, to remove its inline copy.)

### 3b. Hook the existing Save paths
Trigger the download right after the file is successfully written to the VFS, so
"Save" both persists (IDBFS) **and** downloads. No new in-canvas UI needed — this
maps the native "save to disk" onto the browser's download, which is the intuitive
behavior. Touchpoints:

- **Module:** `saveModule()` → `modSave()` in `src/pt2_module_saver.c`
  (`modSave()` @ line 15, `saveModule()` @ line 120). After a successful save,
  call `pt2WebDownloadFile(<full /persist/modules path>, "application/octet-stream")`.
  Capture the exact path/filename `saveModule()` built (mirror how MOD2WAV keeps
  `lastFilename`).
- **Sample:** `saveSample(bool checkIfFileExist, bool giveNewFreeFilename)` in
  `src/pt2_sample_saver.c` @ line 63. It already branches on format
  `DISKOP_SMP_WAV` (152), `DISKOP_SMP_IFF` (206), `DISKOP_SMP_RAW` (258). After
  the file is written, call the helper with a matching MIME:
  - WAV → `audio/wav`
  - IFF (8SVX) → `audio/x-aiff` (or `application/octet-stream`)
  - RAW → `application/octet-stream`

Guard every hook with `#ifdef __EMSCRIPTEN__` so the native build is unchanged
(it writes to the real OS filesystem and must NOT pop a browser download).

### 3c. Filenames
`a.download = path.split('/').pop()` already yields just the filename. The C side
builds names from the song/sample name; make sure the extension is correct
(`.mod`, `.wav`, `.iff`, `.raw`). Sanitize: strip path separators and control
chars; if the name is blank, fall back to `untitled.mod` / `sample.wav`.

---

## 4. Alternative trigger (if auto-download-on-save feels wrong)
Instead of (or in addition to) hooking Save, expose a small **browser-side
"Export" control** in `../web/shell.html` (like the existing install-prompt
button). It would `FS.readdir('/persist/modules')` / `/persist/samples`, let the
user pick a file, and download it via the same Blob pattern. Pros: explicit, no C
changes; lists everything in IDBFS. Cons: a second UI surface outside the canvas.
Decision deferred — owner to choose Save-hook vs. dedicated button (or both).

---

## 5. Edge cases / notes
- **Don't gate on `FS.syncfs`** — download reads the in-memory VFS directly; no
  need to wait for the IndexedDB sync.
- **Rapid multiple saves** each trigger their own anchor click; browsers handle
  sequential downloads fine. The 15s `revokeObjectURL` timeout is a safety margin.
- **Error feedback:** the MOD2WAV block only `console.error`s on failure. Consider
  surfacing a status (the app already has dialog/status text) so a failed export
  isn't silent.
- **iOS Safari** download UX is weaker (opens in a viewer / Files app); acceptable,
  but worth a manual check.
- **Native build:** behavior must be unchanged. All new code under `__EMSCRIPTEN__`.

---

## 6. Implementation checklist
1. Add `pt2WebDownloadFile(path, mime)` helper (shared, `#ifdef __EMSCRIPTEN__`).
2. Capture the saved full path in `saveModule()`; call the helper after success.
3. Same in `saveSample()` with per-format MIME.
4. (Optional) refactor `pt2_mod2wav.c` to use the shared helper.
5. (Optional) add a browser-side Export button to `web/shell.html`.
6. Filename sanitization + sensible fallbacks.
7. No new exported functions needed (uses `EM_ASM` + already-exported `FS`); if a
   JS-initiated export is added, expose a C entry via `EMSCRIPTEN_KEEPALIVE` and
   add it to `EXPORTED_FUNCTIONS` in `build-web.sh`.

## 7. Build / test / ship
- Build: `cd ~/Documents/ProTrackerWeb && ./build-web.sh` (single-file PWA).
- Test on the real server first, then promote (see `DEPLOY.md`):
  ```
  ./deploy.sh stage      # → app.protracker2.com/debug/
  ./deploy.sh test       # automated online+offline PWA check
  # manually: save a module + a sample in /debug/, confirm Save-As downloads
  ./deploy.sh promote    # byte-identical to root
  ```
- Commit the C changes to `pt2-clone` (`fork`/master). The `build-web.sh` /
  `shell.html` edits live in the parent (non-git) `ProTrackerWeb/`.

## 8. Key references
| Thing | Location |
|-------|----------|
| WAV download pattern (copy this) | `src/pt2_mod2wav.c:341-364` |
| Module save | `src/pt2_module_saver.c:15 modSave()`, `:120 saveModule()` |
| Sample save (+formats) | `src/pt2_sample_saver.c:63 saveSample()` (WAV 152 / IFF 206 / RAW 258) |
| VFS save paths (web) | `/persist/modules`, `/persist/samples` (`src/pt2_diskop.c:~485`) |
| FS exported to JS | `EXPORTED_RUNTIME_METHODS` in `build-web.sh`; `FORCE_FILESYSTEM=1` |
| Save button handler | `src/pt2_mouse.c:~3699` |
| Page shell / browser-side UI | `../web/shell.html` |

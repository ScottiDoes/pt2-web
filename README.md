# pt2-web

A **WebAssembly port of the ProTracker 2 clone** — run the classic tracker in your browser.

This is a fork of [8bitbubsy/pt2-clone](https://github.com/8bitbubsy/pt2-clone), modified to compile with Emscripten and run as a single-threaded web application.

## What's different from upstream

The original pt2-clone is a native SDL2 desktop app. This fork adapts it for the browser:

- **Main loop** — driven by `requestAnimationFrame` from JavaScript instead of a native `while` loop. Exported entry points: `pt2_loop_iteration()`, `pt2_load_file()`, `pt2_refresh_diskop()`.
- **No worker threads** — file listing, WAV rendering, and scope rendering run synchronously on the main thread. `SDL_CreateThread()` is not used.
- **Modal dialogs** — yield to the browser via `emscripten_sleep()` so the page stays responsive during blocking prompts.
- **Audio** — forward-progress guard prevents infinite audio callbacks if the tick counter stalls.
- **Virtual filesystem** — modules and samples are persisted via IDBFS (IndexedDB) to `/persist/modules` and `/persist/samples`. Survives page reloads.
- **File loading** — drag-and-drop or file picker in the browser. WAV renders are delivered as a direct browser download (Blob).
- **Mouse input** — canvas-relative coordinates (no desktop window position exists in the browser).
- **VSync** — delegated to the browser's `requestAnimationFrame`. SDL vsync is disabled to avoid errors.
- **Cross-thread guards** — spin-waits removed (deadlock-prone single-threaded) and drain guards added to prevent hangs on corrupt queues.

## Building

### Prerequisites

- [Emscripten SDK](https://emscripten.org/docs/getting_started/downloads.html)
- SDL2 compiled for Emscripten

### Build command

```bash
emcc src/*.c src/gfx/*.c src/smploaders/*.c src/modloaders/*.c \
  -O2 \
  -s USE_SDL=2 \
  -s ASYNCIFY \
  -s ALLOW_MEMORY_GROWTH=1 \
  -s EXPORTED_FUNCTIONS='["_pt2_loop_iteration","_pt2_load_file","_pt2_refresh_diskop","_main"]' \
  -s EXPORTED_RUNTIME_METHODS='["ccall","cwrap","FS","callMain"]' \
  -lidbfs.js \
  -I src \
  -o pt2.html
```

Notes:

- All four source directories must be included — omitting `src/modloaders/*.c` causes link errors.
- `-lidbfs.js` is required for the IDBFS (IndexedDB) persistence of `/persist/modules` and `/persist/samples`.
- `EXPORTED_RUNTIME_METHODS` exposes `ccall`/`cwrap`/`FS`/`callMain` so the host page can call the exported entry points and write picked files into the virtual filesystem.

*(Adjust flags and source files to match your actual build setup.)*

## Running

Serve the output directory with any static web server and open it in a browser. Requires WebAssembly and IndexedDB support.

## Credits

- **Original pt2-clone** — by [8bitbubsy](https://github.com/8bitbubsy) ([upstream repo](https://github.com/8bitbubsy/pt2-clone))
- **WebAssembly port** — by [ScottiDoes](https://github.com/ScottiDoes)

## License

See [LICENSE](LICENSE) — same as upstream.

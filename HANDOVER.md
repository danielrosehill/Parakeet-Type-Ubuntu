# Agent Handover Document

## Metadata
- **Created**: 2026-04-10T14:00:00+03:00
- **Creating agent**: Claude Opus 4.6 (1M context)
- **Repository**: parakeet-dictation
- **Branch**: main
- **Last commit**: a2f0d0f Update desktop entry with clearer local/offline description
- **Handover type**: Task list transition — expanding from single-engine to multi-engine dictation app

## Context

Daniel has been running Handy (a Tauri-based dictation app) for hotkey-triggered voice typing on KDE Plasma 6 / Wayland. It's broken: Tauri's global shortcuts use X11 (doesn't work on Wayland), its `handy_keys` evdev backend doesn't recognize the Pause key, and `wtype` (its typing method) only works on wlroots compositors, not KDE.

After researching alternatives (OpenWhispr, nerd-dictation, WhisperWriter, Voxtype, Speech Note, Dictee), none adequately combine: KDE Wayland cursor-typing + local models + optional cloud ASR. The best path forward is expanding Daniel's existing `parakeet-dictation` app into a multi-engine dictation tool.

Daniel has two working POCs:
1. **parakeet-dictation** (this repo) — local NeMo/Parakeet ASR via sherpa-onnx, GTK3 tray app, ~1935 lines
2. **voice-keyboard-linux** (`~/repos/github/my-repos/voice-keyboard-linux`) — Deepgram Flux cloud ASR via WebSocket, Rust, uinput virtual keyboard

The goal is to merge the Deepgram cloud engine concept into this Python app as an additional ASR backend, alongside the existing local sherpa-onnx engines.

## Task List

### Completed
- [x] Diagnosed Handy failure on KDE Wayland — X11 hotkey capture, wtype incompatibility, evdev key name mismatch
- [x] Researched all major Linux dictation tools for Wayland compatibility
- [x] Identified that `ydotool`/`dotool` (uinput-based) are the ONLY reliable text injection methods on KDE Wayland — `wtype` is wlroots-only
- [x] Read and understood both POC codebases
- [x] Read existing planning docs (roadmap.md, approaches-and-priorities.md)
- [x] Uninstalled OpenWhispr (was a notes app, not cursor-typing)

### In Progress
- [ ] **Refactor ASREngine into a pluggable backend system** — core architectural change
  - **Files touched**: none yet (planning phase)
  - **Current state**: Architecture understood, ready to implement
  - **Next action**: Create an `ASRBackend` abstract base class, refactor existing sherpa-onnx code into `SherpaOnnxBackend`, then add `DeepgramBackend`

### Not Started — Priority Order

- [ ] **P0: Fix KDE Wayland text injection** — Change default typer from `clipboard` to `ydotool` on KDE. Update the settings UI label from "ydotool (needs daemon+uinput)" to indicate it's the recommended KDE method. `wtype` label already correctly says "GNOME/Sway only". Daniel has ydotoold running and is in the `input` group already.

- [ ] **P0: Create ASRBackend abstraction** — Define a Python ABC with methods: `start()`, `stop()`, `pause()`, `is_running`, `is_paused`. Each backend receives audio and calls `on_text(final_text)` and `on_partial(partial_text)` callbacks. The existing `ASREngine` class does this but is tightly coupled to sherpa-onnx.

- [ ] **P0: Refactor existing sherpa-onnx code into `SherpaOnnxBackend`** — Extract the sherpa-onnx recognizer building, VAD detection, and offline/streaming run loops from `ASREngine` into a new backend class. Keep the same behavior, just cleaner separation.

- [ ] **P1: Add `DeepgramBackend`** — Port the Deepgram Flux WebSocket client from the Rust POC (`voice-keyboard-linux/app/src/stt_client.rs`) to Python. Use `websockets` library. The Rust code shows the protocol: connect to `wss://api.deepgram.com/v2/listen?model=flux-general-en&sample_rate=16000&encoding=linear16`, send PCM16 audio chunks, receive `TurnInfo` messages with transcript + partial/final events. Requires `DEEPGRAM_API_KEY` env var.

- [ ] **P1: Add engine selector to Settings UI** — New dropdown in Settings > General: "ASR Engine" with options like "Local: Parakeet TDT 0.6B", "Local: Canary 180M Flash", "Local: Nemotron Streaming", "Cloud: Deepgram Flux". The model profile dropdown should be contextual (only show relevant models for the selected engine type). Add an API key field that appears when a cloud engine is selected.

- [ ] **P1: Add engine selector to tray menu** — Quick-switch between engines from the tray right-click menu, similar to existing model switching.

- [ ] **P2: Rename project** — From "parakeet-dictation" to something engine-agnostic (e.g. `linux-voice-typing`, `voice-typing`, or `local-stt` — Daniel should choose). Update APP_NAME, APP_ID, config paths, .desktop file, README, debian packaging.

- [ ] **P2: Add faster-whisper backend** — Daniel has faster-whisper 1.2.1 installed. Add a `FasterWhisperBackend` using the same ABC. Lower priority since Parakeet is better for real-time dictation, but some users may prefer Whisper models.

- [ ] **P3: Investigate IBus engine for text injection** — The architecturally correct replacement for ydotool (full Unicode, no root, pre-edit support). See `planning/approaches-and-priorities.md` section 3.2 for details.

### Deferred / Out of Scope
- [ ] Hybrid local+cloud pipeline (local streaming + periodic Gemini cleanup) — P3 future direction per roadmap
- [ ] AMD GPU acceleration (ROCm via ONNX Runtime) — not a bottleneck yet
- [ ] Voxtral Mini / Qwen3-ASR integration — waiting for ONNX export support

## Current Repository State
- **Build status**: Working (installed as deb v1.3.0)
- **Uncommitted changes**: Yes — `dictation_app.py` has 49 insertions/138 deletions (looks like a cleanup pass)
- **Working tree clean**: No

## Key Decisions Made
- **ydotool is the correct text injection method for KDE Wayland** — wtype is wlroots-only, xdotool is X11-only, uinput via ydotool/dotool bypasses the compositor entirely
- **Expand this repo rather than merge two codebases** — parakeet-dictation is Python with the right GTK3 architecture; voice-keyboard-linux is Rust. Porting the Deepgram WebSocket protocol to Python is simpler than merging languages.
- **Keep sherpa-onnx as the local inference runtime** — it handles Parakeet, Canary, and Nemotron models efficiently with no PyTorch dependency
- **Backend abstraction over engine replacement** — don't rip out existing code, wrap it behind an interface and add new backends alongside

## Failed Approaches (Do Not Repeat)
- **Handy `handy_keys` evdev backend**: Switched keyboard_implementation to `handy_keys` — it started typing random characters by itself (reading raw evdev events and replaying them). Had to kill -9.
- **Handy `handy_keys` with "pause" binding**: Failed with "Unknown key: pause" — the evdev key parser doesn't recognize the Pause key name.
- **wtype on KDE Plasma**: Does NOT work. wtype uses `wlr-virtual-keyboard-unstable-v1` protocol which KDE doesn't implement. Only works on wlroots compositors (Sway, Hyprland, etc).
- **OpenWhispr**: Electron app, more of a notes/transcription tool than cursor-typing dictation. No evidence of wtype/ydotool integration for typing into focused windows.

## Files of Interest
| File | Why |
|------|-----|
| `dictation_app.py` | Entire app in one file (~1935 lines). Contains ASREngine, TextTyper, DictationController, HotkeyManager, GTK UI |
| `models.json` | Model profile definitions (download URLs, config params) |
| `planning/approaches-and-priorities.md` | Comprehensive analysis of the problem space, failed approaches, and architectural decisions |
| `planning/roadmap.md` | Feature roadmap and priorities |
| `~/.config/parakeet-dictation/config.json` | User config (current: typer=clipboard, model=desktop) |
| `~/repos/github/my-repos/voice-keyboard-linux/app/src/stt_client.rs` | Deepgram Flux WebSocket protocol implementation to port to Python |
| `~/repos/github/my-repos/voice-keyboard-linux/app/src/virtual_keyboard.rs` | uinput virtual keyboard impl (reference only, we'll use ydotool instead) |
| `~/.local/share/com.pais.handy/settings_store.json` | Handy config — was modified during debugging, keyboard_implementation reverted to "tauri", typing_tool changed to "wtype" (should revert to "ydotool" if Handy is ever used again) |

## Resumption Instructions

Start by reviewing the Task List above. The recommended order is:

1. **First**: Fix the KDE Wayland typing default (quick win, immediate value)
2. **Second**: Create the `ASRBackend` ABC and refactor existing sherpa-onnx into `SherpaOnnxBackend`
3. **Third**: Add `DeepgramBackend` — port the WebSocket protocol from the Rust reference
4. **Fourth**: Update Settings UI and tray menu for engine selection
5. **Last**: Rename the project once multi-engine support is working

Check the uncommitted changes in `dictation_app.py` first — there's a diff with 49 insertions / 138 deletions that may need to be committed or reverted before starting new work.

## Context the Next Agent Needs

- **OS**: Ubuntu 25.10, KDE Plasma 6, Wayland (XWayland available at :0)
- **ydotoold**: Running as PID on system, socket at `/tmp/.ydotool_socket`
- **User groups**: daniel is in `input` group (uinput access works)
- **Python**: System Python 3.x, repo uses `.venv` with uv
- **Installed deb**: parakeet-dictation v1.3.0 at `/opt/parakeet-dictation/`
- **Deepgram API key**: Should be in env as `DEEPGRAM_API_KEY` (check `~/.bashrc` or similar)
- **Models downloaded**: Parakeet TDT 0.6B v3 (desktop profile) confirmed in `~/.local/share/parakeet-dictation/models/`
- **Handy**: Still installed (`/usr/bin/handy`, deb v0.8.1) but not running. Its settings were modified during this session. Can be left as-is or uninstalled.

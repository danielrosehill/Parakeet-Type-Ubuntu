# Current Features (v0.1.0)

## Core
- On-device ASR via sherpa-onnx (CPU, no GPU required)
- VAD-segmented transcription (Silero VAD) — waits for speech pause, then transcribes
- True streaming transcription (Nemotron model) — text appears as you speak
- Punctuation and capitalization built into models
- Direct text entry via ydotool (Wayland) or xdotool (X11)
- Auto-detects Wayland vs X11 session

## Models (via sherpa-onnx + NeMo)
- **Parakeet TDT 0.6B v3 (int8)** — 639 MB, best accuracy, 25 European languages
- **Canary 180M Flash (int8)** — 198 MB, lightweight, EN/ES/DE/FR
- **Nemotron Streaming 0.6B (int8)** — 631 MB, real-time streaming, English only
- In-app model download manager

## UI / UX
- System tray (AyatanaAppIndicator) with status display
- Settings dialog with tabs: Models, Hotkeys, General, About
- Partial transcription status shown in tray menu

## Hotkeys (via pynput)
- Toggle mode: single key starts/stops dictation
- Start/Stop mode: separate keys for start and stop
- Pause hotkey: mutes mic without unloading model
- Hotkey capture widget in settings (click button, press combo)
- Default: Ctrl+0 toggle, Ctrl+9 start, Ctrl+8 stop, Ctrl+Alt+0 pause

## Configuration
- Per-user config at ~/.config/parakeet-dictation/config.json
- Configurable: model profile, CPU threads, VAD threshold, beep volume
- Night mode: suppress audio beeps during configurable hours
- Audio feedback: rising/falling tones for start/stop, double beep for pause

## Packaging
- .deb package installs to /opt/parakeet-dictation
- .desktop file for app launcher integration
- Post-install venv setup script

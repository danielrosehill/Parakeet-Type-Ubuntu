# Roadmap

## Project Objective

Lightweight, local-friendly dictation tool for Ubuntu based on NVIDIA Parakeet (via sherpa-onnx). Focus on direct text entry anywhere, at any text cursor. No cloud dependencies, optimized for lower-resource hardware (laptops without GPU).

---

## Planned Features

### Transcription Quality
- [ ] **Configurable segment duration** — Currently VAD max_speech_duration is 30s, min_silence_duration is 0.25s. Expose these as user settings. Longer chunks = better punctuation inference. Consider presets: "Fast" (short segments, lower latency), "Accurate" (longer segments, better punctuation), "Manual" (user controls when to flush).
- [ ] **Manual flush mode** — User presses a hotkey to signal "end of utterance" instead of relying on VAD silence detection. Gives full control over chunk boundaries for best punctuation quality.

### Output Modes
- [ ] **Clipboard mode** — Instead of typing directly via ydotool/xdotool, copy transcribed text to clipboard. User pastes when ready. Useful for applications that don't play well with simulated keystrokes.
- [ ] **Output target selector** — Toggle between direct typing, clipboard, or both (type + copy to clipboard as backup).

### Hotkeys / KDE Integration
- [ ] **KDE-friendly hotkey options** — Document laptop-friendly hotkey combos. Consider End key, PrintScreen, or other underused keys. Ensure pynput hotkeys don't conflict with KDE shortcuts.
- [ ] **D-Bus integration** — Register as a KDE/GNOME service so desktop shortcut settings can trigger dictation start/stop natively (instead of relying solely on pynput global hotkeys).
- [ ] **Flexible hotkey requirements** — Users can set any combination of hotkeys (toggle, start, stop, pause) or none at all (tray-only operation). No mandatory bindings.

### VAD Alternatives
- [ ] **Investigate AI Notepad's VAD** — User reports a non-Silero VAD in their AI Notepad app that works well and is lightweight. Evaluate as alternative or option.
- [ ] **VAD selection in settings** — If multiple VAD backends are supported, let user choose.

### Performance / Hardware
- [ ] **Benchmark on low-resource hardware** — Profile CPU usage, RAM, and transcription latency on Ryzen 7 5700U with Canary 180M model. Establish baseline metrics.
- [ ] **Thread auto-tuning** — Default thread count based on detected CPU cores rather than hardcoded 4.

### UX Polish
- [ ] **First-run wizard** — On first launch, prompt to download a model and set preferred hotkeys instead of requiring manual Settings navigation.
- [ ] **Transcription log/history** — Optional log of all transcribed text with timestamps, viewable from tray menu.
- [ ] **Notification integration** — Desktop notification on start/stop/error instead of (or in addition to) audio beeps.

---

## Non-Goals
- Cloud-based transcription (user has a separate tool for that)
- GPU acceleration (target is CPU-only laptops)
- Mobile/non-Linux platforms

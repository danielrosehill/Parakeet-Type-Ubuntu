# Approaches Tried & Priorities for Parakeet Dictation

**Daniel Rosehill | March 2026**

---

## 1. The Goal: The "Deepgram Feel" — Locally

The gold standard for dictation UX is what Deepgram achieves with Nova/Flux: a *semi-streaming* experience where there's a small deliberate delay between speech and text output. That delay isn't a bug — it's what allows the model to:

- **Infer sentence boundaries** and place punctuation correctly
- **Resolve hesitations** — drop "um", "uh", filler words
- **Fix ambiguities** by exposing the model to a bit of surrounding context before committing text
- **Occasionally rewrite prior text** when more context clarifies meaning

This sits in a sweet spot between two failure modes:

| Approach | Problem |
|----------|---------|
| **True real-time** (token-by-token, no delay) | Captures every "ehm" and "uhm". Visually distracting — text jumps and changes constantly. No time for the model to resolve ambiguity. |
| **Async / batch** (record then transcribe) | Feels sluggish. Breaks the flow of dictation. You have to wait before seeing any text. Not "typing" — more like "submitting". |
| **Semi-streaming** (small deliberate buffer) | The sweet spot. Text appears fast enough to feel real-time, but with enough context to be *clean*. Deepgram nails this. We want this locally. |

The partial-rewrite feature (where the model revises already-typed text as more context arrives) is currently considered **desirable** — it improves accuracy meaningfully. This may become a toggleable preference later if users find the visual rewriting distracting.

---

## 2. Projects & Approaches Tried

### 2.1 Real-Time-Text-Keyboard (Deepgram Flux, Rust)

**What it is:** A Rust application using Deepgram's Flux WebSocket API with direct `/dev/uinput` virtual keyboard input.

**What worked:**
- Genuinely real-time — text appears as you speak (~240ms update cadence)
- Smart incremental typing with backspace correction (diff-based, only modifies what changed)
- The Deepgram Flux API delivers exactly the semi-streaming feel we want
- "Enter" voice command support

**What didn't work:**
- Requires `sudo`/`pkexec` for uinput access — installation friction
- No reconnection logic — WebSocket failure is fatal
- No Unicode support (255 keycode limit via uinput)
- GUI tray icon disabled on KDE/Wayland
- Cloud-dependent — Deepgram Flux is early access with unclear long-term pricing/stability
- Cost: $1.76–2.08/day at heavy use (not prohibitive, but not free)

**Key takeaway:** The *UX model* is exactly right. The semi-streaming with partial revision is the target. The problem is that it's cloud-only and the input method (raw uinput) is too low-level for proper Linux desktop integration.

### 2.2 AI-Typer-V2 (Gemini Multimodal, Python)

**What it is:** A PyQt6 application that records audio then sends it to Gemini Flash-Lite for multimodal transcription (audio + formatting instruction).

**What worked:**
- Best text quality of any approach — the multimodal model transcribes AND cleans up simultaneously
- Handles filler words, broken sentences, spelling clarifications
- 15+ format presets (email, chat, formal, etc.)
- Extremely cheap (~$0.00001–0.00005 per transcription)
- ydotool-based input works on KDE Wayland

**What didn't work:**
- Not real-time — it's a record-then-transcribe workflow
- Latency scales with audio length (1.5–8 seconds depending on clip length)
- No streaming output at all

**Key takeaway:** Proves that multimodal instruction-following (audio + "format this as...") produces the highest quality output. The concept of periodic cloud cleanup passes informed the hybrid architecture idea. But this is a solved long-form dictation tool, not a real-time typing tool.

### 2.3 Parakeet Dictation — Current Project (Local NeMo, Python)

**What it is:** A GTK3 tray application using sherpa-onnx to run NVIDIA NeMo models (Parakeet TDT 0.6B, Canary 180M Flash, Nemotron Streaming 0.6B) with ydotool/xdotool text injection.

**What works:**
- 100% local, zero cost, no cloud dependency
- Native punctuation and capitalization (Parakeet TDT, Nemotron)
- Two modes: VAD-segmented (accurate, ~1–2s latency) and streaming (~100–200ms per frame)
- 25-language support (Parakeet TDT), explicit language selection (Canary)
- CPU-only inference is fast enough (~30x real-time on modern CPU)
- Model selector in GUI, persistent configuration

**What needs work:**
- The VAD-segmented mode has too much latency for real-time feel (1–2s after pause)
- The streaming mode (Nemotron) is English-only and the partial-revision typing isn't battle-tested
- The gap between these two modes is exactly where the "Deepgram feel" lives
- Text injection via ydotool is fragile on Wayland (permissions, no Unicode, no pre-edit)
- No AMD GPU acceleration (CPU-only via sherpa-onnx)

**Key takeaway:** The foundation is solid. The models are good. The architecture (sherpa-onnx + transducer models) is correct. The two hard problems remaining are: (1) getting the semi-streaming timing right, and (2) reliable text input on KDE Plasma / Wayland.

---

## 3. The Two Hard Problems

### 3.1 Finding the Semi-Streaming Sweet Spot

**Why Whisper is always wrong for this:**
Whisper is a seq2seq encoder-decoder. It processes 30-second windows then decodes. Every "real-time Whisper" implementation is a hack that re-runs the model on growing windows. It has no incremental output, no endpointing, no built-in way to emit text as speech arrives. It is architecturally incapable of the semi-streaming behavior we want.

**Why pure real-time is also wrong:**
Frame-by-frame streaming (like Nemotron in raw streaming mode) emits tokens as fast as they can be predicted. This captures every hesitation, filler word, and false start. The visual experience is distracting — text jumps around constantly. There's no buffer for the model to use context to resolve ambiguity.

**The sweet spot we're chasing:**
A transducer model (which processes frame-by-frame but has a language model predictor) combined with:
- Configurable endpoint detection (how long to wait before committing)
- Partial hypothesis revision (rewrite earlier text when more context helps)
- Silence/hesitation filtering (drop fillers before they reach the screen)
- A small deliberate buffer (~200–500ms) that gives the model enough context to be smart

The Parakeet TDT architecture supports this at the model level. The implementation challenge is in the orchestration layer: tuning the VAD thresholds, endpoint detection, and the partial-overwrite typing logic to produce a smooth experience.

**Models best positioned for this:**

| Model | Why it fits | Limitation |
|-------|------------|------------|
| Parakeet TDT 0.6B v3 | Best accuracy, 25 languages, native punctuation, transducer arch | Not natively streaming — relies on VAD segmentation |
| Nemotron Streaming 0.6B | True frame-by-frame streaming with endpoint detection and partial revision | English only |
| Voxtral Mini 4B | Configurable 80–2400ms delay — exactly the sweet spot concept | 4B params, CUDA-only, no ONNX. Can't run on our hardware. |
| Qwen3-ASR-0.6B | 52 languages with streaming | Requires vLLM (CUDA). No ONNX export. |

If Voxtral or Qwen3-ASR ever get ONNX export or ROCm support, they become top candidates.

### 3.2 Reliable Text Input on KDE Plasma / Wayland

This is the other half of the problem and arguably the more frustrating one. Wayland's security model intentionally prevents applications from injecting input into other windows — which is exactly what a dictation tool needs to do.

**Methods tried and their status:**

| Method | Status | Problem |
|--------|--------|---------|
| **ydotool** | Current, works | Needs uinput access (permissions friction), ASCII only, no Unicode, no pre-edit for partial results |
| **/dev/uinput direct** (Real-Time-Text-Keyboard) | Tested | Requires root, same ASCII limitation, too low-level |
| **xdotool** | Legacy | X11 only — doesn't work on Wayland windows at all |
| **wtype** | Investigated | wlroots-only — does NOT work on KDE Plasma |
| **wl-clipboard + paste** | Partial workaround | Works for Unicode but overwrites clipboard, no streaming |

**Architecturally correct solution: IBus engine**

Registering as an IBus input method engine is the right long-term approach:
- Full Unicode support
- No root/special permissions needed
- Native pre-edit display (show partial results in the text field's own UI)
- Proper Wayland integration (works through the toolkit, not around it)
- Existing implementations prove feasibility (IBus-Speech-To-Text, voice-typing-linux)

The trade-off is implementation complexity — going from "shell out to ydotool" to "implement an IBus engine" is a significant step up.

**The libei/EIS future:**

libei (Emulated Input) is the emerging Wayland-native standard for input injection. KDE Plasma already supports it. It would give us proper input injection without the IBus complexity. But it requires a portal dialog for authorization and the tooling/bindings are still maturing.

---

## 4. Priorities for Iteration

### P0 — Must Get Right (Foundation)

1. **Semi-streaming timing** — Tune the VAD/endpoint thresholds and partial-commit logic so the experience matches the Deepgram feel. This is the core UX differentiator. Without this, we're just another Whisper wrapper or a sluggish batch tool.

2. **Partial-overwrite typing reliability** — The backspace-and-retype mechanism for partial revision must be rock solid. Race conditions with user typing, flicker from ydotool latency, and aggressive revision all need careful handling. The toggle to disable this (type only on final endpoint) must work as a fallback.

3. **Model selection defaults** — Parakeet TDT 0.6B for multilingual/accuracy, Nemotron Streaming 0.6B for English real-time. Users should be guided to the right model for their use case without needing to understand transducer architectures.

### P1 — High Priority (Usability)

4. **KDE Wayland text input robustness** — Even if we stick with ydotool short-term, the permissions setup, error handling, and fallback behavior need to be bulletproof. Document the udev rules, input group membership, and ydotoold daemon requirements clearly.

5. **Filler word filtering** — Strip "um", "uh", "ehm" before they reach the screen. This is a major contributor to the "distracting real-time" problem. Can be done at the text layer even if the model emits them.

6. **Enter key safety** — Never inject Enter/Return via the typing layer. A stray newline in a chat window or terminal could send a message or execute a command. All `\n` and `\r` must be stripped unconditionally.

### P2 — Important (Quality of Life)

7. **IBus engine prototype** — Begin work on an IBus input method engine as the replacement for ydotool. This unlocks Unicode, removes root requirements, and enables proper pre-edit display for partial results. It's the architecturally correct path.

8. **AMD GPU acceleration investigation** — Current CPU performance (~30x real-time) is not a bottleneck. But as models grow or if we want to run larger models (Voxtral-class), GPU acceleration on AMD (ROCm or Vulkan via ONNX Runtime) becomes relevant. Track ONNX Runtime ROCm EP maturity for RDNA 3.

9. **Configurable delay/buffer** — Expose the semi-streaming buffer duration as a user setting. Some users will want tighter latency (accepting more revision churn), others will want a longer buffer (cleaner output, more delay). Presets: "Fast" / "Balanced" / "Accurate".

### P3 — Future Direction

10. **Hybrid architecture** — Local streaming ASR for instant text + periodic cloud cleanup (Gemini Flash-Lite) every 5–10 seconds for punctuation/grammar polish. Cost ~$0.05/day. Works offline in degraded mode. Best of both worlds.

11. **Multi-model pipeline** — Use a fast CTC model (Canary 180M) for instant partial display, then refine with the larger transducer (Parakeet TDT 0.6B) for final committed text.

12. **Monitor Voxtral / Qwen3-ASR** — If either gets ONNX export or AMD support, they become strong candidates for the primary model. Voxtral's configurable-delay streaming is architecturally ideal.

---

## 5. What "Done" Looks Like

The target experience:

1. User activates dictation (hotkey or tray click)
2. They speak naturally, including hesitations and filler words
3. Text appears in whatever application has focus, with ~200–500ms delay
4. Filler words are suppressed; sentence boundaries are inferred
5. Occasionally, previously-typed text is revised as more context clarifies meaning
6. When the user pauses, the final text is committed and revision stops
7. The text has proper punctuation and capitalization
8. No root permissions required, works on KDE Plasma / Wayland, supports Unicode
9. Runs entirely on-device with zero cloud dependency
10. CPU inference is fast enough; GPU is a bonus

We're not there yet, but the foundation (sherpa-onnx + transducer models + GTK tray app) is the right base to build on.

---

*Generated 25 March 2026 · Parakeet Dictation project planning*

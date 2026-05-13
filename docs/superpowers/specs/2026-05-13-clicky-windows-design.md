# Clicky Windows — Design Spec

**Date:** 2026-05-13  
**Status:** Approved  
**Goal:** Rebuild the Clicky macOS menu bar AI companion as a native Windows desktop app, feature-for-feature at MVP, with a clear path to enterprise deployment.

---

## 1. Overview

Clicky Windows is a system tray AI companion app for Windows. It mirrors the macOS Clicky app: push a global hotkey to speak, it captures your screen and transcribes your voice, sends both to Claude, then streams the response as text and audio. The architecture is identical in shape to the macOS original — only the OS integration layer changes.

**Stack:** Python + PySide6  
**Why Python:** User knows Python. PySide6 (Qt) is enterprise-grade, handles system tray, floating windows, and global hotkeys natively on Windows.  
**Why not Tauri/Rust or C#:** Tauri/Rust requires learning a new language; C# is the better long-term native choice but slower to prototype. Python lets us build and test each module quickly while learning.

---

## 2. MVP Scope

### In MVP
- System tray icon + floating control panel
- Global push-to-talk hotkey (Ctrl+Alt)
- Voice → AssemblyAI streaming transcription
- Screenshot of all connected monitors on key release
- Claude vision + SSE streaming response (via existing Cloudflare Worker)
- Floating transparent text overlay (response displayed on screen)
- ElevenLabs TTS audio playback (via existing Cloudflare Worker)

### Phase 2 (out of MVP)
- Blue cursor overlay + element pointing (`[POINT:x,y]` tags)
- PostHog analytics
- Packaged `.exe` installer (PyInstaller + NSIS)

---

## 3. Configuration

One setting in `.env`:

```
WORKER_URL=https://your-worker.workers.dev
```

The existing Cloudflare Worker proxy is reused unchanged. It already holds all API keys (Anthropic, ElevenLabs, AssemblyAI). The Windows app never calls any external API directly.

---

## 4. Project Structure

```
clicky-windows/
├── interfaces.py              # shared dataclasses — the module contract
├── requirements.txt
├── conftest.py                # shared pytest fixtures
├── .env                       # WORKER_URL only
├── modules/
│   ├── audio_capture.py       # mic → PCM16 stream
│   ├── hotkey_monitor.py      # global Ctrl+Alt press/release
│   ├── stt_client.py          # AssemblyAI websocket streaming
│   ├── screen_capture.py      # multi-monitor screenshots
│   ├── claude_client.py       # Claude SSE streaming
│   ├── tts_client.py          # ElevenLabs audio playback
│   ├── tray_panel.py          # system tray + floating panel
│   └── overlay.py             # transparent response overlay
├── tests/
│   ├── test_audio_capture.py
│   ├── test_hotkey_monitor.py
│   ├── test_stt_client.py
│   ├── test_screen_capture.py
│   ├── test_claude_client.py
│   ├── test_tts_client.py
│   └── test_overlay.py
├── scripts/                   # standalone run-and-see scripts
│   ├── test_audio.py          # record 5s, play back
│   ├── test_hotkey.py         # print press/release events
│   ├── test_stt.py            # speak, see transcript printed
│   ├── test_screenshot.py     # capture + open PNGs
│   ├── test_claude.py         # ask a question, stream answer
│   ├── test_tts.py            # type text, hear it spoken
│   └── test_overlay.py        # display sample streamed text
├── companion_manager.py       # module 9 — the orchestrator
└── main.py                    # entry point
```

---

## 5. Python Dependencies

```
sounddevice      # mic capture + audio playback
numpy            # PCM buffer manipulation
pynput           # global hotkey listener
keyboard         # fallback hotkey detection
httpx            # HTTP + SSE streaming
websockets       # AssemblyAI websocket stream
mss              # fast multi-monitor screen capture
Pillow           # image processing + base64 encoding
PySide6          # system tray, floating panel, overlay window
python-dotenv    # load .env
pytest           # test runner
pytest-asyncio   # async test support
```

---

## 6. interfaces.py — The Module Contract

Every module accepts and returns these types. No module reads another module's internals.

```python
from dataclasses import dataclass
from enum import Enum
from typing import Literal

@dataclass
class AudioChunk:
    pcm16_mono_bytes: bytes     # raw PCM16 little-endian mono audio
    sample_rate_hz: int         # always 16000 for AssemblyAI
    duration_seconds: float     # length of this chunk
    audio_level_rms: float      # 0.0–1.0, for waveform display

@dataclass
class Transcript:
    text: str                   # full transcript text
    is_final: bool              # True = finalized turn, False = partial
    confidence: float           # 0.0–1.0 from AssemblyAI

@dataclass
class CapturedScreen:
    monitor_index: int          # 1-based monitor number
    width_px: int
    height_px: int
    png_bytes: bytes            # raw PNG — claude_client base64-encodes this

ScreenCaptureResult = list[CapturedScreen]  # one per connected monitor

@dataclass
class ClaudeResponseToken:
    text_delta: str             # one streamed chunk of text
    is_final: bool              # True = stream complete
    full_text_so_far: str       # accumulated text up to this token

@dataclass
class HotkeyEvent:
    event_type: Literal["press", "release"]
    timestamp: float            # time.monotonic() at event time

class VoiceState(Enum):
    IDLE       = "idle"         # waiting for hotkey
    LISTENING  = "listening"    # hotkey held, capturing audio
    PROCESSING = "processing"   # key released, sending to Claude
    RESPONDING = "responding"   # streaming response + TTS playing
```

### Module Signatures

```python
AudioCapture.start(on_chunk: Callable[[AudioChunk], None]) → None
AudioCapture.stop() → None

HotkeyMonitor.start(on_event: Callable[[HotkeyEvent], None]) → None
HotkeyMonitor.stop() → None

STTClient.stream(chunks: AsyncIterator[AudioChunk]) → AsyncIterator[Transcript]

ScreenCapture.capture_all_monitors() → ScreenCaptureResult

ClaudeClient.ask(transcript: str, screens: ScreenCaptureResult, history: list[dict]) → AsyncIterator[ClaudeResponseToken]

TTSClient.speak(text: str) → None  # blocks until audio finishes

Overlay.show_streaming_text(tokens: AsyncIterator[ClaudeResponseToken]) → None
Overlay.clear() → None

TrayPanel.set_state(state: VoiceState) → None
```

---

## 7. State Machine

```
IDLE ──(hotkey press)──→ LISTENING ──(key release)──→ PROCESSING ──(first token)──→ RESPONDING ──(TTS done)──→ IDLE
```

**Interrupt:** hotkey pressed during RESPONDING → immediately stop TTS + clear overlay → LISTENING.

---

## 8. Full Pipeline — Step by Step

1. `HotkeyMonitor` fires `HotkeyEvent(type="press")` → transition to LISTENING, start `AudioCapture`, open `STTClient` websocket, show waveform in `TrayPanel`
2. Each `AudioChunk` → forwarded to `STTClient`. `Transcript(is_final=False)` partials shown live in `TrayPanel`
3. `HotkeyEvent(type="release")` → stop `AudioCapture`, close STT stream, receive final `Transcript`. Simultaneously call `ScreenCapture.capture_all_monitors()`. Transition to PROCESSING
4. Call `ClaudeClient.ask(transcript.text, screens, history)` → SSE stream opens through Cloudflare Worker (`screens` is a `ScreenCaptureResult`)
5. First `ClaudeResponseToken` arrives → transition to RESPONDING. Each token sent to `Overlay.show_streaming_text()` in real-time
6. SSE stream closes (`is_final=True`) → full response text sent to `TTSClient.speak()`. TTS audio plays. Overlay stays visible
7. TTS finishes → `Overlay.clear()`, `TrayPanel.set_state(IDLE)`. Conversation history retained for follow-up questions

---

## 9. Testing Strategy

Each module has two test surfaces:

**Standalone scripts** (`scripts/test_<module>.py`) — run directly with `python scripts/test_audio.py`. Designed for learning: you run it, something observable happens (sound plays, text prints, window opens). No pytest knowledge required.

**pytest tests** (`tests/test_<module>.py`) — regression safety. Run with `pytest`. Cover: correct output types, error handling, edge cases.

### Per-Module Test Plan

| Module | Standalone test | pytest covers |
|--------|----------------|---------------|
| `audio_capture` | Record 5s, play back | Correct PCM format, audio level 0–1 |
| `hotkey_monitor` | Print press/release events | Callback fires, no key-repeat duplicates |
| `stt_client` | Speak, see transcript printed | Token fetch, WS connect, partial+final |
| `screen_capture` | Capture + open PNGs | Returns list, correct monitor count |
| `claude_client` | Ask a question, stream answer | Streams tokens, error handling |
| `tts_client` | Type text, hear it spoken | Returns audio bytes, correct format |
| `overlay` | Display sample streamed text | Window appears, text updates |

---

## 10. Build Order & Parallel Development

### Sprint 0 — Sequential (½ day, done once in `main`)
Create repo, install dependencies, write `interfaces.py` with all dataclasses, write module stubs, set up `conftest.py` and `pytest.ini`.

### Sprint 1 — Parallel (all 8 worktrees simultaneously)

| Worktree | Module | Key library |
|----------|--------|-------------|
| 1 | `audio_capture` | `sounddevice` |
| 2 | `hotkey_monitor` | `pynput` |
| 3 | `stt_client` | `websockets` |
| 4 | `screen_capture` | `mss` |
| 5 | `claude_client` | `httpx` |
| 6 | `tts_client` | `httpx` + `sounddevice` |
| 7 | `tray_panel` | `PySide6` |
| 8 | `overlay` | `PySide6` |

Each worktree agent reads `interfaces.py` to know their input/output types. No agent reads another module's code.

### Sprint 2 — Sequential (after all PRs merged)
Build `companion_manager.py` — imports all 8 modules, wires the state machine, full end-to-end test.

---

## 11. CompanionManager Skeleton

```python
class CompanionManager:
    """Owns all modules. Drives the voice state machine."""

    def __init__(self, worker_url: str):
        self.audio   = AudioCapture()
        self.hotkey  = HotkeyMonitor()
        self.stt     = STTClient(worker_url)
        self.screens = ScreenCapture()
        self.claude  = ClaudeClient(worker_url)
        self.tts     = TTSClient(worker_url)
        self.overlay = Overlay()
        self.tray    = TrayPanel()
        self.state   = VoiceState.IDLE
        self.history: list[dict] = []  # conversation turns

    async def start(self): ...          # wire hotkey → _on_hotkey_event
    async def _on_hotkey_press(self): ...    # IDLE → LISTENING
    async def _on_hotkey_release(self): ... # LISTENING → PROCESSING → RESPONDING → IDLE
    def _set_state(self, state: VoiceState): ...  # updates tray + overlay
```

---

## 12. What Stays Unchanged from macOS

- `worker/src/index.ts` — Cloudflare Worker, zero changes
- All three API routes (`/chat`, `/tts`, `/transcribe-token`) — identical
- Anthropic, ElevenLabs, AssemblyAI API contracts — identical
- System prompt sent to Claude — identical
- Conversation history format — identical

# Privacy

JarvOS is local-first: there is no JarvOS server, no account, and no telemetry. This
document states exactly what data the app touches, where it goes, and where that's
enforced in code.

## What JarvOS touches

| Data | What happens to it | Where it lives |
| --- | --- | --- |
| **Microphone audio** | Captured only while listening (wake word / hotkey press). While idle, wake-word detection (`frontend/src/audio/wakeword.ts` → `src-tauri/src/wakeword.rs`) runs an on-device ONNX model against the mic stream directly in the Rust process. Once summoned, JarvOS's speech-to-text engine (`frontend/src/audio/stt.ts`) produces a text transcript — preferably the bundled on-device `whisper.cpp` engine (`src-tauri/src/stt.rs`), falling back to the webview's built-in recognizer only when that isn't installed. JarvOS's own code never uploads the raw audio anywhere. | Not persisted. The wake-word engine never writes audio to disk at all; the STT engine writes a temp WAV file for the `whisper.cpp` process to read and deletes it immediately after transcription, win or lose (`stt_transcribe`, `src-tauri/src/stt.rs`). |
| **Transcript** | The recognized text is sent to your local Ollama instance for intent resolution, and shown transiently in the orb overlay as a command echo. | Held **in memory only** for the duration of that turn. Never written to disk, never logged. Closing the app or moving to the next turn clears it. There is no chat log or history. |
| **Config** (wake word, hotkey, devices, language, model, allowlist/script policy, accessibility) | Read and written locally. | A single local JSON file (`config.json`) in the app's local data directory. |
| **Skills** (your registered `launch_app` / `open_url` / `run_script` actions) | Read and written locally. | A single local JSON file (`skills.json`) alongside `config.json`. |
| **Spoken replies (TTS)** | Synthesized via the OS/webview speech synthesizer, restricted to on-device voices only; falls back to an optional local Piper engine only when no offline OS voice exists at all (see below). | Not persisted. |

On Windows this local data directory is
`%APPDATA%\org.helveticlabs.jarvos` (see `src-tauri/src/storage.rs`, bundle identifier in
`src-tauri/tauri.conf.json`). Delete that folder to wipe all local JarvOS state.

## What JarvOS does not do

- No telemetry, analytics, or crash reporting.
- No chat log or interaction history — nothing is persisted beyond `config.json` and `skills.json`.
- No account, no auth, no cloud sync.
- No data is sold, shared, or transmitted to Anthropic, Helveticlabs, or any third party.

## The only network connection

The only outbound connection JarvOS's Rust backend will make is to a **local** Ollama
instance for LLM intent resolution. This is enforced in code, not just policy: every
Ollama request (`ollama_get` / `ollama_post` in `src-tauri/src/main.rs`) is routed
through `ensure_local`, which rejects any base URL that doesn't start with
`http://127.0.0.1`, `http://localhost`, or `http://[::1]` — the request is never sent
if the check fails.

```rust
// src-tauri/src/main.rs
fn ensure_local(base_url: &str) -> Result<(), String> {
    let ok = ["http://127.0.0.1", "http://localhost", "http://[::1]"]
        .iter()
        .any(|p| base_url.starts_with(p));
    if ok { Ok(()) } else { Err("JarvOS only talks to a local model (localhost).".into()) }
}
```

No other Tauri command makes a network call. OS actions (`dispatch_action`) launch
local apps, open URLs you configured, or run scripts you registered — JarvOS doesn't
call out on your behalf beyond what a skill explicitly does.

## A note on speech recognition

Speech-to-text tries two engines in order (`frontend/src/audio/stt.ts`'s `sttEngine`; see
`docs/STT.md`):

1. **A bundled `whisper.cpp` binary, run locally.** `src-tauri/src/stt.rs`'s `stt_transcribe`
   Tauri command is the only thing that touches the recording: it writes a temp WAV file,
   runs `whisper-cli` against it, and deletes that file immediately after — win or lose —
   before returning the transcript text to the webview. No network call is made anywhere in
   that path; this is JarvOS's own code, so it's a guarantee, not a property of the platform.
2. **The webview's built-in speech recognizer** (the standard browser `SpeechRecognition`
   API), used only when the local engine isn't installed. JarvOS never uploads audio itself
   here either, and no audio or transcript passes through `ensure_local` or any
   JarvOS-controlled endpoint — but whether the underlying OS/webview recognizer processes
   speech entirely on-device is a property of that platform's speech stack, not something
   JarvOS's code can enforce. Installing the local engine (`docs/STT.md`) removes this
   uncertainty.

Wake-word listening (`frontend/src/audio/wakeword.ts` → `src-tauri/src/wakeword.rs`) is
separate from both STT tiers above and isn't a webview API at all: it's an on-device ONNX
pipeline (openWakeWord-style — melspectrogram → embedding → classifier, run via `ort`)
reading the mic through `cpal` directly in the Rust process. This is a guarantee, not a
platform property, same as the local STT tier — see `docs/WAKEWORD.md`. It only recognizes
a fixed catalog of trained phrases, not arbitrary text; Settings → Activation reports when
the configured phrase isn't in that catalog or its model isn't installed. Settings → About
and Diagnostics report which STT tier is in use and whether wake word is active. Where
neither STT tier nor wake word is available, JarvOS falls back to the global hotkey and
typed input instead of voice.

## A note on spoken replies (TTS)

Unlike speech recognition, the browser's speech-synthesis API exposes a spec-defined
flag — `SpeechSynthesisVoice.localService` — that distinguishes an on-device voice from
one backed by a remote synthesis service. JarvOS enforces this in code, not just policy:
`listVoices()` and `speak()` in `frontend/src/audio/tts.ts` filter every voice to
`localService === true` before it can be listed in Settings or spoken from. A networked
voice (e.g. Chrome's "Google" voices, or Edge's cloud "Online (Natural)" voices, were the
app ever run there) is never selectable, and never used as an implicit default — every
utterance gets an explicit local voice assigned.

If no offline voice is installed at all (rare — most Windows installs ship at least one
SAPI/OneCore voice by default; stripped-down or Server-core images may not), JarvOS does
not fall back to any networked engine. Instead it tries one more, still-local tier: an
optional Piper engine (`src-tauri/src/tts.rs`, see `docs/TTS.md`) — not bundled in this
repository or the official installer (GPL-3.0, license-bearing, same non-bundled
convention as the `whisper.cpp` STT tier), used only when installed by hand. When used,
Rust runs the bundled `piper` binary against a temp WAV file, reads the audio back into
memory, and deletes the temp file immediately after — win or lose — before handing the
bytes to the webview to play; nothing is written anywhere else. Only if neither the OS
voice nor this local fallback is available does JarvOS disable spoken replies, show a
note in Settings pointing at *Settings → Time & Language → Speech* (or `docs/TTS.md`),
and continue to work fully via the on-screen caption. This keeps the guarantee absolute —
every spoken reply you hear from JarvOS ran entirely on your machine, and none of it
touches the network — at the cost of silence (not a network call) in that last edge case.

## Questions

Open an issue in this repository.

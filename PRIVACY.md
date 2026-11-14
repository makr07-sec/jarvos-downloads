# Privacy

JarvOS is local-first: there is no JarvOS server, no account, and no telemetry. This
document states exactly what data the app touches, where it goes, and where that's
enforced in code.

## What JarvOS touches

| Data | What happens to it | Where it lives |
| --- | --- | --- |
| **Microphone audio** | Captured only while listening (wake word / hotkey press). While idle, wake-word detection (`frontend/src/audio/wakeword.ts` → `src-tauri/src/wakeword.rs`) runs an on-device ONNX model against the mic stream directly in the Rust process. Once summoned, JarvOS's speech-to-text engine (`frontend/src/audio/stt.ts`) produces a text transcript — the on-device `whisper.cpp` engine (`src-tauri/src/stt.rs`) that the Windows installer includes, falling back to the webview's built-in recognizer only when that isn't installed (the macOS/Linux previews, or a damaged install). JarvOS's own code never uploads the raw audio anywhere. | Not persisted. The wake-word engine never writes audio to disk at all; the STT engine writes a temp WAV file for the `whisper.cpp` process to read and deletes it immediately after transcription, win or lose (`stt_transcribe`, `src-tauri/src/stt.rs`). |
| **Transcript** | The recognized text is sent to your local Ollama instance for intent resolution, and shown transiently in the orb overlay as a command echo. Only if you switch on Claude in Settings is it also sent to Anthropic (see [Claude](#claude-optional)). | Held **in memory only** for the duration of that turn. Never written to disk, never logged. Closing the app or moving to the next turn clears it. There is no chat log or history. |
| **Config** (wake word, hotkey, devices, language, model, allowlist/script policy, accessibility) | Read and written locally. | A single local JSON file (`config.json`) in the app's local data directory. |
| **SignalForge API key** (optional) | Only if you paste one in Settings. Used as the bearer token for the SignalForge inbox and notes search requests; never shown again, never logged, never part of backups. | A separate local file (`signalforge.json`) in the data directory, readable only by your user (mode 0600 on macOS/Linux; the per-user folder on Windows). |
| **Anthropic API key** (optional) | Only if you paste one in Settings → Claude. Sent as the API key of the Claude request; never shown again, never logged, never part of backups. | A separate local file (`anthropic.json`) in the data directory, readable only by your user (as above). |
| **Skills** (your registered `launch_app` / `open_url` / `run_script` actions) | Read and written locally. | A single local JSON file (`skills.json`) alongside `config.json`. |
| **Start at sign-in** (Windows, optional) | Only if you tick it under Settings → Activation: one entry in your Windows account's start-up list that starts JarvOS when you sign in. Nothing is sent anywhere. | The `JarvOS` value under `HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Run` (`src-tauri/src/autostart.rs`), removed when you untick it or uninstall JarvOS. |
| **Speech models** (whisper.cpp model, wake-word models) | Part of the app, not your data: they ship inside the installer and are read from the install folder. JarvOS downloads no models at runtime. | The install folder (`whisper\`, `wakeword\`), removed on uninstall. |
| **Spoken replies (TTS)** | Synthesized via the OS/webview speech synthesizer, restricted to on-device voices only; falls back to an optional local Piper engine only when no offline OS voice exists at all (see below). | Not persisted. |

On Windows this local data directory is
`%APPDATA%\org.helveticlabs.jarvos` (see `src-tauri/src/storage.rs`, bundle identifier in
`src-tauri/tauri.conf.json`). Delete that folder to wipe all local JarvOS state.

## What JarvOS does not do

- No telemetry, analytics, or crash reporting.
- No chat log or interaction history — nothing is persisted beyond `config.json` and `skills.json`, plus `signalforge.json` or `anthropic.json` if you save an API key (and `owner.json` or `connector.json` if you use the developer options).
- No account, no auth, no cloud sync.
- No data is sold or shared, and none is transmitted to Anthropic, Helveticlabs, or any third
  party, with these exceptions: the update check, which reveals your IP address to GitHub;
  the SignalForge inbox note or notes search you ask for with your own API key, which goes
  to your own account on SignalForge, also operated by HelveticLabs (see below); Claude,
  only if you switch it on (see [Claude](#claude-optional)); and the custom connection in the
  developer options, only to the address you save yourself (see
  [Custom connection](#custom-connection-developer-option)). The fallback
  speech recognizer is a separate case: see [A note on speech recognition](#a-note-on-speech-recognition).

## Network connections

Apart from the update check (see [Update check](#update-check)), the opt-in SignalForge inbox
and notes search, the opt-in Claude connection and the custom connection described below, the only outbound connection
JarvOS's Rust backend will make is to a **local** Ollama
instance for LLM intent resolution. This is enforced in code, not just policy: every
Ollama request (`ollama_get` / `ollama_post` in `src-tauri/src/main.rs`, `ollama_pull` in
`src-tauri/src/pull.rs`) is routed
through `ensure_local`, which rejects any base URL whose host isn't
`127.0.0.1`, `localhost` or `[::1]` over `http://` — the request is never sent
if the check fails.

```rust
// src-tauri/src/main.rs
fn ensure_local(base_url: &str) -> Result<(), String> {
    let local = reqwest::Url::parse(base_url).is_ok_and(|u| {
        u.scheme() == "http" && matches!(u.host_str(), Some("127.0.0.1" | "localhost" | "[::1]"))
    });
    if local { Ok(()) } else { Err("JarvOS only talks to a local model (localhost).".into()) }
}
```

**Downloading a model.** **Download Llama 3.2** in the first-start guide and in Settings → Model
asks your local Ollama to download that model; JarvOS sends it only the model name. Ollama then
fetches the model from its own model library on the internet, as `ollama pull` does; that
connection is Ollama's, under [Ollama's terms](https://ollama.com). **Get Ollama** opens
`https://ollama.com/download` in your browser.

Two things here are opt-in and local-first. The **rewrite-selection hotkey** copies
the text you selected, sends it to the same local Ollama model, pastes the result back and
restores your previous clipboard (text or image); JarvOS sends the selected text nowhere else
and never logs it. Copying it uses your system clipboard, so an OS or third-party clipboard
sync treats it like any other copy; what JarvOS itself puts there, including your restored
clipboard, is marked to skip clipboard history (and on Windows cloud sync). The **SignalForge inbox** sends data to a remote
host only when you have pasted a SignalForge API key into Settings and you say "remember
..." or "remind me ...": JarvOS then sends that sentence to `https://api.signalforge.it.com` as
a note in your own inbox (`src-tauri/src/inbox.rs`). With the same key, and only if you switch on
"Search and read out my notes" there, "what do my notes say about ..." sends only the topic words to the same address as a search of your own notes; the
titles and short excerpts of the best three matches come back, are read out and shown, and are
not stored. A note you locked with a PIN is never read out or named; JarvOS only says that
locked notes match. Without a key nothing is sent. The key
is kept in a separate owner-only file (`signalforge.json`) in the data folder, never in
`config.json`, so settings backups don't contain it; the interface can store or remove it
but cannot read it back, and it is never logged.

## Claude (optional)

Off by default. Under **Settings → Claude** you can let Claude handle requests the local model
can't (or every request) and answer questions. Claude gets no tools: it only answers with the
same structured intent the local model gives, and JarvOS still checks every action against your
skills and permissions.

While it is on, each request that goes to Claude sends to Anthropic, PBC (USA), at
`https://api.anthropic.com` (`src-tauri/src/claude.rs`):

- what you said, and for a follow-up the question JarvOS asked and your earlier sentence;
- your enabled skills' names, descriptions, aliases, action types and parameter names (not
  their targets: no paths, web addresses or scripts);
- the local date and time.

Nothing else goes there: no audio, no clipboard, no settings, no files. With "When the local
model can't help", a request goes to Claude only when the local model finds no match, has to
ask back, isn't sure, or isn't running. Anthropic bills your own API key, and handles the data
under its own terms and privacy policy. The key is kept like the SignalForge key: a separate
owner-only file (`anthropic.json`), never in `config.json` or backups, never shown again and
never logged.

The developer options (hidden in Settings → About) hold one more way to reach Claude, meant for
the owner of the install and usable only with the owner code: the same request goes through
the Claude Code program on this computer, signed in to the owner's own Claude account, instead
of an API key. Claude Code then sends it to Anthropic. For that, the rules and skill list are
written to `claude-code/system-prompt.md` in the data folder; what you said is passed directly
and not written to disk.

## Custom connection (developer option)

Off unless you save an address and a key under Settings → Developer options → Custom
connection. Then, when you ask for a briefing, for what waits for your decision, decide one of
those, or pass a task on, JarvOS sends that request to **the address you saved** (and only
there): for a task, the sentence you said; for a decision, the item's id and approve or
reject; always the language of the request. Nothing else goes there: no audio, no other
commands, no settings. Hiding the developer options turns it off; Settings → Reset to defaults
removes the address and key. You choose and run that service, so its own privacy terms apply. The
address and key are kept in a separate owner-only file (`connector.json`), never in
`config.json` or backups; the key is never shown again, never logged, and never sent to any
other address. See `docs/CONNECTOR.md`.

No other Tauri command makes a network call. OS actions (`dispatch_action`) launch
local apps, open URLs you configured, or run scripts you registered — JarvOS doesn't
call out on your behalf beyond what a skill explicitly does.

The ready-made web skills (Web search, Map, YouTube, Search clipboard) are such skills:
when you ask for one, JarvOS opens your browser at DuckDuckGo, Google Maps or YouTube with
your spoken words in the web address, and that website receives them like any search you
type. The skill packs you can add under Settings → Skills (Wikipedia, Google Translate,
Spotify, YouTube Music, Gmail, Google Calendar, Google Drive) work the same way, only when you
add them and ask for one. Exporting or importing a skill pack only writes or reads the file you
pick; nothing is uploaded. You can edit or remove them. A skill whose web address contains `{clipboard}` reads
the text on your clipboard only at the moment you ask for it and puts it into that address;
it never goes to the model. If you switched on the local audit log, the address is recorded
there like every other action, so the clipboard text is in that file on your computer. Nothing else in JarvOS reads the clipboard,
apart from the rewrite hotkey above.

## Update check

JarvOS checks for updates once each time it starts, and again when you press **Check for
updates** in Settings. The check at start cannot be turned off. Each check fetches one file,
`https://github.com/makr07-sec/jarvos-downloads/releases/latest/download/latest.json`.
If a newer version exists, JarvOS shows a notification (when OS notifications are on) and
installs nothing on its own. Only when you choose to install in Settings does it download the
matching installer from the same public
repository. Both are ordinary, unauthenticated downloads of static files. They send no
data about you, but like any web request they reveal your IP address to GitHub, which
hosts those files (release files are served from `github.com` and
`release-assets.githubusercontent.com`). The manifest's entries are signed, and JarvOS
refuses an update whose signature does not verify against the key built into the app.

## A note on speech recognition

Speech-to-text tries two engines in order (`frontend/src/audio/stt.ts`'s `sttEngine`; see
`docs/STT.md`):

1. **A bundled `whisper.cpp` binary, run locally.** The Windows installer includes the
   binary and its English model (`docs/STT.md`), so this is the engine a normal Windows
   install uses. `src-tauri/src/stt.rs`'s `stt_transcribe`
   Tauri command is the only thing that touches the recording: it writes a temp WAV file,
   runs `whisper-cli` against it, and deletes that file immediately after — win or lose —
   before returning the transcript text to the webview. No network call is made anywhere in
   that path; this is JarvOS's own code, so it's a guarantee, not a property of the platform.
2. **The webview's built-in speech recognizer** (the standard browser `SpeechRecognition`
   API), used only when the local engine isn't installed. JarvOS never uploads audio itself
   here either, and no audio or transcript passes through `ensure_local` or any
   JarvOS-controlled endpoint — but whether the underlying OS/webview recognizer processes
   speech entirely on-device is a property of that platform's speech stack, not something
   JarvOS's code can enforce. (WebView2's recognizer on Windows is an online service.)
   The macOS/Linux preview builds don't bundle the local engine yet; installing it by hand
   (`docs/STT.md`) removes this uncertainty there.

Wake-word listening (`frontend/src/audio/wakeword.ts` → `src-tauri/src/wakeword.rs`) is
separate from both STT tiers above and isn't a webview API at all: it's an on-device ONNX
pipeline (openWakeWord-style — melspectrogram → embedding → classifier, run via `ort`)
reading the mic through `cpal` directly in the Rust process. This is a guarantee, not a
platform property, same as the local STT tier — see `docs/WAKEWORD.md`. Its models ship in
every installer. It only recognizes
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

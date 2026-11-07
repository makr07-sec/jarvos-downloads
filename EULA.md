# End User License Agreement

This End User License Agreement ("EULA") covers the **compiled JarvOS desktop
application** (the installer/bundle you download or build, not the source code —
the source is MIT-licensed; see [LICENSE](LICENSE)).

By installing or running JarvOS, you agree to the following:

## 1. License grant

You are granted a free, non-exclusive, non-transferable license to install and run
JarvOS on devices you own or control, for any purpose, personal or commercial.

**Exception: third-party components.** The components included in the installer that
are listed in [NOTICE.md](NOTICE.md) are licensed to you under their own licenses; for
them, this EULA neither grants more rights nor restricts the rights those licenses give
you. In particular, the wake-word models (openWakeWord) are licensed under
[CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/) for
**non-commercial use only**, so the "commercial" in the grant above does not extend to
the wake word. For commercial use, turn the wake word off (Settings → Activation → Wake
word enabled) and summon JarvOS with the global hotkey; everything else stays available.

## 2. As-is, no warranty

JarvOS is provided **"AS IS", WITHOUT WARRANTY OF ANY KIND**, express or implied,
including but not limited to the warranties of merchantability, fitness for a
particular purpose, and noninfringement. In no event shall the authors or
copyright holders be liable for any claim, damages, or other liability arising
from, out of, or in connection with the software or its use — consistent with the
project [LICENSE](LICENSE).

## 3. You are responsible for what you register

JarvOS acts on **skills** you configure yourself — launching apps, opening URLs,
and running scripts you register in Settings. JarvOS only executes what you've
explicitly registered (or explicitly confirmed, depending on your allowlist and
script policy). You are solely responsible for the skills and scripts you add,
including any effect they have on your system or data. Review anything before you
register it, especially `run_script` entries.

## 4. Local-first, no data collection

JarvOS runs entirely on your device: no account, no telemetry, no chat log, and no
network egress beyond a local Ollama instance, except the update check and the optional
SignalForge inbox you switch on with your own API key. See [PRIVACY.md](PRIVACY.md) for the
full data model.

## 5. Termination

This license terminates automatically if you violate its terms. Nothing in this
EULA restricts the rights you have separately under the MIT [LICENSE](LICENSE) for
the source code.

## Questions

Open an issue in this repository.

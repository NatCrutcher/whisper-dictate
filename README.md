# voice — push-to-talk dictation for Linux (Wayland/KDE)

Press a key to start recording, press again to stop. Your speech is
transcribed by a local whisper.cpp server (GPU-accelerated) and pasted
into the active window.

Designed for dictating prose into terminal apps like Neovim and Claude
Code, where vim mode safety matters.

## Prerequisites

- **whisper.cpp** built from source with GPU support, including `whisper-server`
  (currently at `~/tools/whisper.cpp/`)
- A whisper model (currently `ggml-large-v3.bin`)
- **PipeWire** (for `pw-record`)
- **ydotool** — kernel-level input injection (`sudo apt install ydotool`)
- **wl-clipboard** — Wayland clipboard access (`wl-copy`)
- **jq** — JSON parsing
- **curl**
- **notify-send** (libnotify) — desktop notifications

## Setup

### 1. ydotool permissions

ydotool needs access to `/dev/uinput`. Create a udev rule and add
yourself to the `input` group:

```bash
echo 'KERNEL=="uinput", GROUP="input", MODE="0660"' | sudo tee /etc/udev/rules.d/80-uinput.rules
sudo usermod -aG input $USER
sudo udevadm control --reload-rules && sudo udevadm trigger
```

**Log out and back in** for the group change to take effect.

### 2. Whisper server (systemd user service)

Install and start the service that keeps the whisper model loaded on
the GPU:

```bash
cp ~/voice/whisper-dictation.service ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable --now whisper-dictation
```

The first start takes several seconds while the model loads into GPU
memory. Check status with:

```bash
systemctl --user status whisper-dictation
```

### 3. Keyboard shortcut

Bind a key to `~/voice/dictate` in your desktop environment.

**KDE Plasma:** System Settings → Shortcuts → Custom Shortcuts → Add
new → Command/URL. Set the trigger to your preferred key and the
command to `/home/nat/voice/dictate`.

## Usage

1. Press your dictation key — a "Recording..." notification appears
2. Speak
3. Press the key again — recording stops, a "Transcribing..." notification
   appears, and after ~1-2 seconds the text is pasted into whatever
   window has focus

## Theory of operation

### Architecture

```
┌──────────────┐    on login     ┌─────────────────────┐
│ systemd user │───────────────▶│ whisper-server       │
│ service      │                │ (model on GPU, idle) │
└──────────────┘                └──────┬──────────────┘
                                       │ localhost:8178
                                       │
┌──────────────┐   POST /inference     │
│ dictate      │──────────────────────▶│
│ (bash script)│◀──────────────────────│
└──────┬───────┘   JSON {text: "..."}
       │
       │  wl-copy + ydotool Ctrl+Shift+V
       ▼
┌──────────────┐
│ active window│
└──────────────┘
```

### Why a persistent server?

Loading a whisper model into GPU memory takes several seconds.
Inference on loaded models takes ~1-2 seconds for typical utterances.
The server pays the load cost once at login rather than on every
dictation.

### Toggle mechanism

The `dictate` script is a toggle. Each invocation checks for a PID
file:

- **No PID file (or stale PID):** Starts `pw-record` in the
  background recording 16kHz 16-bit mono WAV (Whisper's native
  format). Saves the PID and exits. The `pw-record` process is
  detached with `disown` so it survives the script exiting.

- **Valid PID file:** Sends SIGINT to `pw-record` (so it cleanly
  finalizes the WAV header), waits 200ms, then POSTs the audio to the
  whisper server. The server returns JSON with the transcribed text.

### Text injection and vim safety

The transcribed text is injected via the system clipboard rather than
simulated keystrokes:

1. `wl-copy` places the text on the Wayland clipboard
2. `ydotool` simulates Ctrl+Shift+V (terminal paste shortcut)

The terminal emulator wraps clipboard content in **bracketed paste**
escape sequences (`\e[200~…\e[201~`). Neovim and other vim-mode
applications recognize bracketed paste and handle it correctly
regardless of the current mode — text is inserted, not interpreted as
commands. This is critical for avoiding destructive accidents when
focus is in a vim normal-mode buffer.

`ydotool` was chosen over `wtype` because KDE Plasma's KWin does not
implement the `zwp_virtual_keyboard_v1` Wayland protocol that wtype
requires. ydotool bypasses Wayland entirely by injecting events at the
Linux `/dev/uinput` kernel level.

### Notifications

Desktop notifications provide feedback at each stage:

| Stage          | Timeout       |
|----------------|---------------|
| Recording...   | persistent    |
| Transcribing...| 10 seconds    |
| Done / Error   | 2 seconds     |

## Notes and issues

- **Clobbers clipboard.** The paste-based injection overwrites whatever
  is on the Wayland clipboard. This is an accepted trade-off for vim
  safety.

- **ydotool keyboard layout.** ydotool v0.1.8 assumes a QWERTY
  layout for `ydotool type`. This doesn't affect us since we only use
  `ydotool key` to press Ctrl+Shift+V — modifier + letter combos work
  regardless of layout.

- **Ctrl+Shift+V is terminal-specific.** This paste shortcut works in
  Konsole, Kitty, Alacritty, WezTerm, and most modern terminal
  emulators. It will not work correctly in non-terminal GUI apps that
  expect Ctrl+V. The script is currently optimized for the terminal
  dictation use case.

- **Server `--convert` flag.** The whisper server is started with
  `--convert`, which uses ffmpeg to normalize audio before inference.
  This is a safety net — `pw-record` already outputs the correct
  format (16kHz 16-bit mono WAV), but `--convert` handles edge cases
  like truncated WAV headers if `pw-record` is killed at an unlucky
  moment.

- **Temp files.** The WAV recording and PID file are stored in
  `$XDG_RUNTIME_DIR` (typically `/run/user/$UID/`), a RAM-backed tmpfs
  that is cleaned up on logout.

## Future enhancements

- **Save/restore clipboard.** Snapshot the clipboard before dictation
  and restore it after pasting, to avoid clobbering.

- **Detect active window type.** Use Ctrl+Shift+V for terminals and
  Ctrl+V for GUI apps. Could inspect the focused window class via
  `kdotool` or KWin scripting.

- **Neovim remote API.** When `$NVIM` is set, use Neovim's RPC socket
  to insert text directly via the API, bypassing the clipboard
  entirely. This would be the cleanest solution for Neovim
  specifically.

- **Streaming transcription.** For long dictations, send audio chunks
  incrementally rather than waiting for the full recording. whisper.cpp
  supports streaming, though the server endpoint does not currently
  expose it.

- **Model selection.** Allow switching between small/medium/large
  models via a flag or environment variable, trading accuracy for
  speed.

- **Prompt context.** The whisper server supports an `--prompt` flag
  for providing context that biases the decoder. Could be used to
  improve accuracy for domain-specific vocabulary.

- **Talon.** If latency or accuracy requirements grow, Talon provides
  real-time transcription with its own Conformer speech engine and a
  full voice command grammar system. It's a significant step up in
  complexity but also capability.

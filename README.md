# ODSS VS Code Extension

Records and replays VS Code and Cursor editing sessions, terminal interactions, diagnostics, and AI-assisted code changes using the Open Developer Session Stream (ODSS) container format.
<img width="1280" height="720" alt="VideoCode_demo_vsce_1_720p" src="https://github.com/user-attachments/assets/f1b7fc96-d483-4aee-9917-2d3fb4583c51" />

---

## Overview of ODSS: Structured Stream, Not Video

ODSS (Open Developer Session Stream) / ODSF is **not a video recording**. It does not capture pixels, encode screen frames into MP4/WebM files, or record display buffers.

Instead, it is a **deterministic, structured event log** of everything that occurred during a development session:
- **Text & Buffer State**: Exact character-level document edits, open/close lifecycles, and periodic content-addressed snapshots.
- **Developer Focus**: Cursor coordinates, multi-selection ranges, scroll bounds, and active editor tab changes.
<!-- - **Terminal Execution**: Raw PTY input and ANSI output streams (`terminal.output@1`) rather than terminal screen scrapes. -->
- **Tooling & Language Intelligence**: Language server diagnostics and debug session markers.
- **AI Attribution**: Explicit markers for AI-generated code proposals and whether each hunk was active, retained, or reverted.
<!-- - **Cryptographic Tamper-Evidence**: SHA-256 hash chaining and optional Ed25519 signatures validating log integrity. -->

### Why Structured Data Over Video?
1. **Interactive and Inspectable**: Because replay operates on real text documents and terminal buffers, you can pause at any sequence, select text, inspect syntax trees, search logs, and run diffs against live workspace files.
2. **Compact & Compressible**: Capturing semantic keystrokes and diffs takes megabytes per hour rather than gigabytes of video bitrate.
3. **Random-Access Determinism**: Instantaneous seeking to any event sequence without decoding intervening video frames.

### The Replay Experience
The ODSS playback engine drives VS Code's editor, virtual filesystem (`odss-fs://`), and pseudoterminals in real time or at accelerated speeds (0.5x to 4x). It renders paced typing, cursor drift, scroll framing, and terminal activity—producing a **video-like replay experience** directly inside a real editor environment.

<!-- ### Extension Process Model
To capture and replay this stream without impacting VS Code's UI responsiveness, the extension splits work across two processes:
- **Host Process (`src/host/`)**: Hooks into VS Code APIs (`vscode.workspace`, `vscode.window`, `vscode.languages`). During recording, it observes editor changes and forwards raw observations over IPC. During playback, it mounts the read-only in-memory `odss-fs://` filesystem and drives editors, decorations, and pseudoterminals.
- **Child Process (`src/child/`)**: Runs `RecorderCore`, the monotonic event sequencer, snapshot storage, cryptographic hashing, and shell execution (`node-pty` or piped fallback). This isolates CPU and disk I/O from VS Code's UI thread. -->

---

## Recording Features

### 1. Starting a Recording Session
<img width="1080" height="1350" alt="recording_ft" src="https://github.com/user-attachments/assets/dd8b0e60-843f-4c99-a1e3-f7e09bd4bef0" />

- **Command**: `Ctrl + Shift + P`, `ODSS: Start Recording` (`odss.startRecording`)  
- **Container Path Resolution**:
  - When a workspace folder is open, sessions are stored in `.odss-sessions/<timestamp>-<sessionId>/` at the workspace root.
  - When no workspace is open, the extension falls back to VS Code's global storage directory: `<globalStorage>/odss-sessions/<sessionId>/`.
- **Status Indicator**: An item appears in the status bar showing `$(sync~spin) ODSS...` while initializing, then switches to `$(record) ODSS Recording` once active.
- **Concurrency Guard**: Attempting to start a recording while one is already active triggers an explicit error dialog.

<!-- ### 2. Editor Activity Capture
The host attaches listeners via `wireEditorEventForwarding()` to capture editor activity across open documents:
- **Initial Document State**: Every document open at recording launch emits a `document.open` observation containing full text content.
- **Incremental Text Changes**: `onDidChangeTextDocument` emits `document.edit` with exact line/character ranges and replacement strings.
- **Lifecycle Events**: `document.close`, `document.save`, file renames (`document.rename`), and file deletions (`document.delete`).
- **Cursor and Selection Tracking**: Single and multi-cursor selections (`cursor.move`) record anchor and active positions.
- **Viewport Visibility**: `view.scroll` captures visible line spans (`[start, end]`).
- **Editor and Window Focus**: Window focus changes (`window.focus`) and active tab changes (`editor.focus`).
- **Diagnostics**: `onDidChangeDiagnostics` forwards language server errors, warnings, information, and hints for open buffers.
- **Debug Sessions**: Debug start/stop events (`command.invoked`).
- **URI Filtering**: System and noisy schemes (`git`, `gitlens`, `output`, `vscode-userdata`, `odss-fs`, etc.) are ignored. Virtual model schemes used by AI assistants (e.g., `chat-editing-text-model`, `vscode-chat-session`) are explicitly preserved so agent-driven edits are recorded. -->

<!-- ### 3. Recorded Terminals
- **Command**: `ODSS: Record on Terminal` (`odss.newRecordedTerminal`)
- **Dedicated Pseudoterminal**: Creates an interactive terminal titled `ODSS Recorded Terminal`.
- **PTY Resolution (`shell-factory-resolver.ts`)**:
  - **Native PTY (Preferred)**: Uses `node-pty` with ConPTY on Windows or POSIX PTY on macOS/Linux. Supports colored ANSI output, terminal resizing, cursor repositioning, and interactive curses/TUI tools.
  - **Piped Stdio Fallback**: If `node-pty` native prebuilds are unavailable, the extension automatically falls back to `child_process.spawn` with piped stdin/stdout/stderr so recording continues without crashing.
- **Ordered Event Emission**: Terminal output is serialized per-terminal in the child process to prevent out-of-order chunks when backpressure occurs. -->

### 2. Stopping a Recording Session
<img width="1080" height="1350" alt="stop_recording_ft" src="https://github.com/user-attachments/assets/5077bc9b-f4d4-444f-9ec2-4c2df8e354fe" />

- **Command**: `Ctrl + Shift + P`, `ODSS: Stop Recording` (`odss.stopRecording`)
- **Teardown Sequence**:
  1. Closes open terminal instances first to deliver `terminal.close` events cleanly over IPC while the child is alive.
  2. Disposes all editor event forwarders.
  3. Sends `stop` to the child process and awaits final flush of the append log and snapshot index.
  4. Terminates the child process and hides the status bar item.

---

## Replay Features
<img width="1920" height="1080" alt="open_replaying_ft-_2_" src="https://github.com/user-attachments/assets/27abdd4a-eb07-4218-bd9c-e3b593274a25" />

### 1. Opening and Mounting a Replay
- **Command**: `Ctrl + Shift + P`, `ODSS: Open Replay Session…` (`odss.openReplaySession`)
- **Session Picker (`session-picker.ts`)**:
  - Scans `.odss-sessions/` in the workspace root, parsing each session's `manifest.json`.
  - Displays relative start time, duration, and session ID in a QuickPick menu sorted with newest sessions first.
  - Includes a `$(folder-opened) Browse…` entry to open sessions stored elsewhere.
- **Launch Targets**:
  - **Open in New Window (Recommended)**: Spawns a dedicated VS Code window mounted to the replay workspace.
  - **Open in Current Window**: Replaces the current window's workspace with the replay filesystem.
  - **Add to Current Workspace**: Mounts the session folder as an additional workspace folder in the existing window.
- **Auto-Detection**: When VS Code opens with an `odss-fs` workspace URI, the extension detects the session authority and launches playback automatically.
- **Integrity Check**: Verifies event hash chains and cryptographic signatures on load.

### 2. Virtual File System (`odss-fs://`)
Playback runs entirely inside an in-memory `FileSystemProvider` registered under the `odss-fs` scheme:
- **Non-Destructive**: Replayed edits and file state transitions are isolated in memory. Local disk files are never overwritten.
- **Authority Format**: `odss-fs://session-<sessionId>/path/to/file.ts`.
- **Read-Only Guarantee**: Registered with `isReadonly: true` so the user cannot accidentally modify playback documents.
- **Clean Teardown**: Unmounting a session purges the in-memory tree for that session authority.

### 3. Playback Controls
<img width="1920" height="1080" alt="open_playback_fts" src="https://github.com/user-attachments/assets/62028f6a-dc35-4934-b2d5-b848ca9f6265" />

- **Play / Resume**: `ODSS: Resume Replay` (`odss.resumeReplay`)
- **Pause**: `ODSS: Pause Replay` (`odss.pauseReplay`)
- **Stop and Close**: `ODSS: Stop and Close Replay` (`odss.stopReplay`)
- **Playback Speed**: `ODSS: Set Replay Speed…` (`odss.setReplaySpeed`) allows selecting `0.5x`, `1x`, `2x`, or `4x`.
- **Seek to Sequence**: `ODSS: Seek Replay Event Number…` (`odss.seekReplay`) prompts for an event number (`0..total`) and jumps directly to that state.
- **Go to First Sequence**: `ODSS: Go to First Sequence` (`odss.goToFirstSeq`) jumps to sequence 0 (session start) while preserving the current playing or paused state.
- **Interactive Control Menu**: `ODSS: Replay Controls…` (`odss.replayControls`) opens a QuickPick menu with playback actions. This menu also opens whenever you click the replay status bar item.

### 4. Frame-by-Frame Keyboard Navigation
- **Command**: `Ctrl + Shift + P`, `ODSS: Toggle Replay Keyboard Navigation` (`odss.toggleKeyboardControl`)
- Activating arrow navigation automatically pauses playback to let you step through events manually. Exiting navigation resumes playback if it was playing before.
- **Keybindings** (active when keyboard navigation is enabled):
  | Key | Command | Description |
  | --- | --- | --- |
  | `Right Arrow` | `odss.stepForward` | Steps forward by one sequenced event. |
  | `Left Arrow` | `odss.stepBackward` | Steps backward by one event using the hybrid re-derive model. |
  | `Home` | `odss.goToFirstSeq` | Jumps back to sequence 0. |
  | `Escape` / `Enter` | `odss.toggleKeyboardControl` | Exits keyboard navigation mode. |

### 5. Editor Replay and Visual Feedback
<!-- - **Sequential Application**: All editor operations (document edits, cursor updates, scrolls) run through a serialized internal promise queue to guarantee FIFO order. -->
- **Document Sync Synchronization**: Edits written to `odss-fs` await VS Code's internal text document model synchronization (`waitForDocumentSync`) before decorations are calculated, eliminating race conditions.
- **Edit Highlights**:
  - Single-line edits show a blue left-border highlight for 1.5 seconds.
  - Multi-line or significant text insertions display a transient green hunk highlight for 2.5 seconds.
- **Cursor and Selection Playback**: Recreates single and multi-cursor selections in real time.
- **Adaptive Auto-Scrolling**:
  - Micro-drifts (within half a viewport) use `TextEditorRevealType.Default` to prevent view snapping.
  - Larger movements center the target range with `InCenterIfOutsideViewport`.
  - Large hunk insertions clamp the reveal range to `startLine + 5` so the viewport stays at the beginning of the edit rather than snapping to the bottom.
  - Can be toggled with `ODSS: Toggle Replay Auto-Scrolling` (`odss.toggleScroll`) or configured via settings.
- **Diagnostics Replay**: Language server diagnostics (errors, warnings, info, hints) are restored into VS Code's Problems panel and inline squiggles via a dedicated `DiagnosticCollection`.

### 6. AI Diff Attribution and Historical Views
The extension provides visual tracking for code changes generated by AI assistants:
- **Change Status Line Decorations (`change-status-decorations.ts`)**:
  - Highlights lines based on change lifecycle using native VS Code theme colors:
    - **Active** (`↻`): Blue selection background tint (`editor.selectionBackground`) and blue gutter marker.
    - **Retained** (`✓`): Green diff insertion tint (`diffEditor.insertedTextBackground`) and green gutter marker.
    - **Reverted** (`✗`): Red diff removal tint (`diffEditor.removedTextBackground`) and red gutter marker.
- **Persistent Deleted Code Windows (`persistent-diff-decorations.ts`)**:
  - Renders deleted pre-edit code using VS Code's native Comments API (`CommentThread`).
  - Displays a clean, dedented code block anchored directly at the edit line showing what code existed prior to the AI change.
  - Displays a status title (e.g., `🔴 Deleted — original code before this AI edit was accepted`).
  <!-- - Line-keyed and bounded: threads update in place on subsequent edits and are managed with a FIFO cap (maximum 200 active threads) to keep editor performance steady. -->
  <!-- - Drift-aware: adjusts comment anchors across document edits via line-delta tracking. -->

<!-- ### 7. Terminal Replay
- **Pseudoterminal Driver (`replay-terminal.ts`)**:
  - Creates an `ODSS Replay: <terminalId>` terminal for each recorded terminal session.
  - Output streams into the terminal during normal playback.
  - Seeking or stepping triggers an ANSI clear-and-home sequence (`\x1b[2J\x1b[H`) followed by a full buffer write to reflect the exact state at that sequence.
  - Pressing `Space` inside a replay terminal toggles pause/resume. -->

### 8. Status Bar Heads-Up Display
The replay status bar item (`formatReplayStatusBarText`) renders dynamic state:
```
$(play) ODSS Replay [document.edit] [█████░░░░░] 240/480 (Arrow Nav)
```
- **State Icon**: `$(play)` for playing, `$(debug-pause)` for paused, `$(sync~spin)` for loading.
- **Active Event Type**: Displays the current event without version tags (e.g., `[document.edit]`, `[terminal.output]`, `[cursor.move]`).
- **Visual Progress Bar**: 10-segment block progress (`█████░░░░░`).
- **Sequence Counter**: Current sequence vs. total sequence count (`240/480`).
- **Mode Indicator**: Appends `(Arrow Nav)` when keyboard navigation is engaged.
- **Clickable**: Clicking the status item opens the `ODSS: Replay Controls…` menu.

<!-- ---

## Configuration Settings

| Setting | Type | Default | Description |
| --- | --- | --- | --- |
| `odss.replayer.enableScroll` | `boolean` | `true` | Enable or disable automatic viewport scrolling and reveal during playback. | -->

---

## Command Reference

| Command ID | Title | Keybinding | Context / Description |
| --- | --- | --- | --- |
| `odss.startRecording` | ODSS: Start Recording | — | Starts a new recording session. |
| `odss.stopRecording` | ODSS: Stop Recording | — | Stops and saves the active recording session. |
| `odss.newRecordedTerminal` | ODSS: Record on Terminal | — | Opens a recorded terminal within the active session. |
| `odss.openReplaySession` | ODSS: Open Replay Session… | — | Opens the session picker to load and replay a session. |
| `odss.pauseReplay` | ODSS: Pause Replay | — | Pauses active playback. |
| `odss.resumeReplay` | ODSS: Resume Replay | — | Resumes paused playback. |
| `odss.stopReplay` | ODSS: Stop and Close Replay | — | Stops playback and unmounts the virtual workspace. |
| `odss.seekReplay` | ODSS: Seek Replay Event Number… | — | Prompts for an event sequence number to seek to. |
| `odss.goToFirstSeq` | ODSS: Go to First Sequence | `Home` *(Arrow Nav)* | Seeks back to sequence 0 (session start). |
| `odss.setReplaySpeed` | ODSS: Set Replay Speed… | — | Sets playback rate (`0.5x`, `1x`, `2x`, `4x`). |
| `odss.replayControls` | ODSS: Replay Controls… | — | Shows the QuickPick menu of replay actions. |
| `odss.toggleKeyboardControl` | ODSS: Toggle Replay Keyboard Navigation | `Escape` / `Enter` *(Arrow Nav)* | Toggles frame-by-frame arrow navigation mode. |
| `odss.stepForward` | ODSS: Step Replay Sequence Forward | `Right` *(Arrow Nav)* | Steps forward by one sequence event. |
| `odss.stepBackward` | ODSS: Step Replay Sequence Backward | `Left` *(Arrow Nav)* | Steps backward by one sequence event. |
| `odss.toggleScroll` | ODSS: Toggle Replay Auto-Scrolling | — | Toggles the `odss.replayer.enableScroll` setting. |

---

<!-- ## Development & Build

From the root or within `apps/vscode-extension`:

```bash
# Build both host and child bundles
pnpm --filter odss-vscode-extension compile

# Watch mode during development
pnpm --filter odss-vscode-extension watch

# Run TypeScript typechecks
pnpm --filter odss-vscode-extension typecheck

# Run test suite
pnpm --filter odss-vscode-extension test
``` -->

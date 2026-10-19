# Implementation Plan: `nanotap` — Local AI & MCP Traffic Inspector

## Problem & Approach

Build **nanotap**, a single-binary Local AI & MCP Traffic Inspector in Zig 0.17 on `justrach/nanoui`. `nanotap` operates in two modes from a single executable:
1. **GUI (`nanotap`)**: Opens a native `nanoui` window listening on a user-private (`0700`) Unix domain socket watched via `nanoui.fdwatch` (event-driven, zero polling).
2. **Stdio Tap (`nanotap --tap [label] [--out <file.jsonl>] -- <cmd> [args...]`)**: Spawns `<cmd>`, forwards `stdin` ↔ `stdout` byte-for-byte cleanly for JSON-RPC 2.0 (MCP / ACP), correlates request/response pairs by `id` with microsecond latency, supports holding/modifying `tools/call` responses when GUI interception is active, and streams framed events to the GUI socket and optional JSONL file.

## File Structure (Every File ≤ 600 Lines)

- `build.zig` & `build.zig.zon`: Build configuration depending on `justrach/nanoui`, forwarding `-Dportable`, supporting cross-compilation (`aarch64-linux-gnu`, `x86_64-windows-gnu`), `test`, `test-bin`, and `lines` steps.
- `AGENTS.md`: Project rules and architecture overview (≤ 600 lines).
- `tools/lines.zig`: Enforces the 600-line limit across all source and build files.
- `src/main.zig`: Entry point dispatching `--tap` CLI mode vs native `nanoui` GUI window (with demo/seeded state support for headless `--nanoui-snapshot` verification).
- `src/model.zig`: Core data structures (`Direction`, `Frame`, `Session`, `Filter`, `Model`), JSON-RPC 2.0 line parser (`initialize`, `tools/list`, `tools/call`, `session/*`), `id` request/response correlation with microsecond latency, throughput/latency history series, JSON pretty-printer, and 100k-row filtering index.
- `src/tap.zig`: Byte-transparent stdio passthrough engine (`Tap`), line framing for `C→S` and `S→C`, malformed-line resilience, JSONL file recorder (`--out`), socket event streaming, and response hold/release interception.
- `src/proc.zig`: Cross-platform child process pipe spawning (`posix_spawnp` on POSIX, clean compile stub on Windows) and non-blocking pipe helpers.
- `src/host.zig`: Event-driven Unix domain socket server (`0700` directory) using `nanoui.fdwatch`, ingesting frames from `--tap` processes into `Model` and sending intercept/forward/drop commands back to connected taps.
- `src/theme.zig`: Codegraff warm dark/light palette and `nanoui.ui.Theme` setup (`Geist` + `JetBrains Mono`).
- `src/view.zig`: Immediate-mode root view, keyboard shortcuts, `.glass(.{})` top bar (live throughput/latency cell sparkline, session selector, method/error filter badges, search `ui.textField`, **Hold (`intercept`)** toggle), and left-pane virtualized `ui.list` frame table (`ui.key` per row).
- `src/view_detail.zig`: Right-pane formatted JSON payload inspector with terminal-style text selection (`.selectable = true`), **Copy** button, and intercepted `tools/call` response editor (`ui.textField` + **Forward** / **Drop** buttons).
- `src/tests.zig`: Unit tests (passthrough, latency pairing, malformed lines, JSONL recording), socket bridge integration test (`nanotap --tap test -- <mock-mcp-server>`), and offscreen `ui.Tester` UI tests (filtering, row selection, payload copy, holding/forwarding/dropping responses, 100k-row virtualization, and `/tmp/nanotap.png` snapshot).

## Implementation Steps

1. **Scaffold & Protocol Model (`src/model.zig`, `src/tap.zig`)**
   - Parse JSON-RPC 2.0 lines (`C→S` and `S→C`), pairing responses to requests by `id` to compute duration (µs), byte size, method (`initialize`, `tools/list`, `tools/call`, `session/*`), tool name, and error state.
   - Verify byte-identical stdio passthrough, latency pairing, malformed-line resilience, and JSONL recording without spawning a window.
2. **Socket Bridge (`src/host.zig`, `src/proc.zig`)**
   - Stream captured frames from `--tap` child processes to the GUI host over a `0700` Unix socket watched via `nanoui.fdwatch`.
   - Verify via integration test spawning `nanotap --tap test -- <mock-mcp-server>`, exchanging `initialize` + `tools/call`, and asserting `Host` receives all correlated frames in order.
3. **Native Inspector UI (`src/view.zig`, `src/view_detail.zig`)**
   - Top bar (`.glass(.{})`): live throughput/latency cell sparkline, session selector, method/error filter badges, search `ui.textField`, and **Hold (`intercept`)** toggle.
   - Left pane (`ui.list` with stable `ui.key`): virtualized 100k-row frame table showing direction, badge (`tools/call`, `error`), method/tool name, bytes, and latency.
   - Right pane: formatted JSON payload inspector with terminal-style text selection, copy button, and editable response buffer with **Forward** / **Drop** buttons when a `tools/call` response is held.
   - Verify with offscreen `ui.Tester` tests in `src/tests.zig` and save headless snapshot via `--nanoui-snapshot=/tmp/nanotap.png`.
4. **Verification Gate**
   - `zig build test` and `zig build test -Dportable=true`
   - `zig build -Dtarget=aarch64-linux-gnu` and `zig build -Dtarget=x86_64-windows-gnu`
   - `zig build lines` (every file ≤ 600 lines)

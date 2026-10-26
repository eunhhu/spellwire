# Spellwire

[한국어](README.ko.md)

Spellwire is a stateful realtime input-automation runtime for Bun and TypeScript: hotkeys, key remaps, and overlay logic are written in TypeScript, compiled ahead of time into bounded native bytecode, and executed by a Rust VM on the input thread instead of calling JavaScript for every event.

**Status:** Early alpha. The compiler, native VM, consuming hotkeys/remaps, state gates, delay scheduler, Bun FFI host, platform backends, and retained overlay are implemented. macOS live loopback and suppression are verified; Windows interactive loopback, injection, reload, and overlay lifecycle are verified, but physical suppression and visual transparency checks remain; Linux target verification remains. No npm release has been published yet.

## Features

- Portable hotkey strings (`"Ctrl+Shift+K"`) with consume/pass-through, repeat control, and release triggers
- Key remaps (`rt.remap("CapsLock", "Escape")`) that emit paired down/up transitions automatically
- Persistent native state from module-scope integer and boolean `let` declarations
- `when` gates that switch actions and pass-through on the same condition
- Typed fixed-payload effects published through a preallocated SPSC lane
- Keyboard/mouse output (`tapKey`, `keyDown`, `clickMouse`, `moveMouse`, `wheelMouse`) and non-blocking delays (`sleep.ms`, `sleep.seconds`)
- State-driven, auto-layout native overlay (`ui.row`, `ui.column`, `ui.panel`, `ui.text`, `ui.bind`, `ui.when`) with configurable window policy
- Authenticated local RPC (`SpellwireRpcServer` / `SpellwireRpcClient`) for Electron or sidecar processes
- Deterministic inspector/simulator and native dispatch benchmark
- Watch mode with native hot reload and automatic platform permission setup

## API at a glance

| Task | API |
| --- | --- |
| Consuming hotkey | `rt.hotkey("Ctrl+Shift+K", handler)` |
| Key remap | `rt.remap("CapsLock", "Escape")` |
| Persistent state | module-scope `let enabled = true` |
| Transient typed event | `const changed = effect("changed", schema)`, then `changed.emit(payload)` |
| Keyboard/mouse output | `tapKey`, `keyDown`, `clickMouse`, `moveMouse`, `wheelMouse` |
| Delay | `sleep.ms(250)`, `sleep.seconds(2)` |
| Start input + watch + UI | `Spellwire.start(options)` |
| Overlay layout and content | `ui.row`, `ui.column`, `ui.panel`, `ui.stack`, `ui.text`, `ui.badge`, `ui.divider` |
| State-bound UI | `overlay: state => ...`, `ui.bind`, `ui.when` |
| Overlay window | `overlayOptions.window` (`alwaysOnTop`, `transparent`, `focusable`, `clickThrough`) |
| Electron/sidecar bridge | `SpellwireRpcServer`, `SpellwireRpcClient` |

See the [API reference](docs/api.md) for signatures, defaults, option tables, and platform notes.

## How it works

- `packages/spellwire` contains the public SDK, the TypeScript AOT compiler, and the `spellwire` CLI. The compiler extracts only the bounded realtime handlers (conditions, loops, arithmetic, helper functions, held-input queries, delays) and writes a versioned binary plus a JSON state manifest. Ordinary Bun code outside those handlers stays unrestricted.
- `crates/spellwire-core` decodes that binary, keeps the trigger table and persistent-state VM, and runs the delay scheduler. It produces fixed output batches without allocating on the hot path.
- `crates/spellwire-native` exposes a stable C ABI and an owned host that wires global observers and injectors: Windows hooks/`SendInput`, macOS event tap/`CGEventPost`, Linux evdev/uinput.
- The Bun side loads the native library over FFI, drives watch/reload, and receives only actual state changes and effects over the SPSC lane, so the overlay refreshes on change instead of polling.
- `crates/spellwire-overlay` is a separate retained renderer process (winit/wgpu) that draws the state-bound UI tree.

## Tech stack

- Runtime/Language: Rust 2021 (MSRV 1.81), TypeScript on Bun 1.4
- Native: Cargo workspace of five crates; C ABI consumed through Bun FFI
- Overlay: winit + wgpu renderer process
- Infra/Tooling: GitHub Actions (`ci.yml`, `publish.yml`), `cargo fmt`/`clippy` with warnings denied, `bun test`, `scripts/check-docs.ts`

## Getting started

### Requirements

- Bun 1.4+
- Rust toolchain 1.81+ with Cargo
- Platform input permissions (Accessibility/Input Monitoring on macOS, udev rule in `packaging/linux` on Linux); see [Troubleshooting](docs/troubleshooting.md)

### Install

Once published, the package will be available as `bun add spellwire` or `bun create spellwire my-automation`. From a source checkout:

```bash
git clone https://github.com/eunhhu/spellwire.git
cd spellwire
bun run setup   # bun install --frozen-lockfile, typecheck, cargo build --release
```

### Run

Deterministic inspector/simulator (no OS permissions required):

```bash
bun run compile:example
bun run inspect:example
bun run simulate:example
```

Live global input with hot reload from the source checkout:

```bash
bun run build:native
bun packages/spellwire/src/cli.ts watch examples/stateful.spellwire.ts
```

`Ctrl+C` stops the runtime and releases any synthetic inputs that remain held. Other useful scripts:

```bash
bun run check            # typecheck, TS tests, doc checks, cargo fmt/test/clippy
bun run bench            # native dispatch percentile benchmark
bun run bench:platform   # platform injector benchmark
bunx spellwire compile src/main.ts   # compile a single file directly
```

Example of a stateful handler:

```ts
import { Key, rt, tapKey } from "spellwire";

let enabled = true;

rt.hotkey("Ctrl+Shift+K", () => {
  tapKey(Key.Enter);
}, { when: () => enabled });

rt.hotkey("F8", () => {
  enabled = !enabled;
}, { consume: false });

rt.remap("CapsLock", "Escape", { when: () => enabled });
```

## Project structure

```
crates/
  spellwire-core/      bytecode decoder, trigger table, persistent-state VM, scheduler
  spellwire-native/    C ABI, owned host, global observers, native injectors, platform-bench
  spellwire-overlay/   retained winit/wgpu overlay renderer process
  spellwire-cli/       native inspector and deterministic simulator
  spellwire-bench/     native dispatch percentile benchmark
packages/
  spellwire/           public SDK, AOT compiler, TypeScript CLI
  create-spellwire/    `bun create spellwire` project initializer
docs/                  English and Korean (.ko.md) documentation
examples/              sample realtime handlers and overlay programs
scripts/               doc checks, native staging, platform smoke tests, benchmarks
packaging/linux/       udev rule for evdev/uinput access
.github/workflows/     CI and publish pipelines
```

## Configuration

Environment variables read by the Bun host:

- `SPELLWIRE_NATIVE_LIBRARY` — path to a prebuilt `spellwire-native` shared library (default: the artifact from `bun run build:native`)
- `SPELLWIRE_OVERLAY_EXECUTABLE` — path to a prebuilt `spellwire-overlay` binary

Runtime behavior (hotkeys, state, overlay window policy such as `alwaysOnTop`, `transparent`, `clickThrough`) is configured in code via `Spellwire.start(options)`; see the [API reference](docs/api.md).

## Documentation

- [API reference](docs/api.md) — create/run/build, hotkeys, state, output, lifecycle, overlay API
- [Quick Start](docs/quick-start.md) and [Troubleshooting](docs/troubleshooting.md)
- [Platform Verification](docs/platform-verification.md) — copyable macOS, Windows, and Linux checks
- [Effects and RPC](docs/effects-rpc.md), [Realtime compiler subset](docs/typescript-runtime.md), [Live native host](docs/live-host.md)
- [Architecture](docs/architecture.md), [Native C ABI](docs/native-abi.md), [Platforms](docs/platforms.md), [Overlay design](docs/overlay.md)
- [Automation semantics and AutoHotkey migration](docs/automation.md)
- [Publishing](docs/publishing.md), [Implementation status](docs/status.md), [Verification](docs/runtime-verification.md), [Documentation index](docs/index.md)

Performance note: Spellwire separates framework-boundary latency from switch debounce, USB polling, OS scheduling, compositor behavior, and target-application polling. Physical end-to-end latency has not been measured; claims require platform-specific percentile and jitter measurements.

## License

[MIT](LICENSE)

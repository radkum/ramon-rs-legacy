# RAMON - Windows minifilter in Rust

**Edition 1 of 2 - hand-written, no AI (2025).**

This is the original RAMON sensor: a Windows file-system minifilter plus a user-mode client,
written by hand in Rust. The 2026 edition - rebuilt, extended and lab-verified with an AI
assistant - is [`radkum/ramon-rs`](https://github.com/radkum/ramon-rs).

The driver is a `no_std` kernel component (`ramon.sys`) that registers with the filter manager,
watches file creates and three kernel notify classes, and streams structured events to a console
client over a minifilter communication port. It is a research prototype, not a product - see
[Status](#status).

## What it does

Kernel driver (`ramon-km`):

- registers a minifilter with `FltRegisterFilter` / `FltStartFiltering` at altitude `345102`
  (`ramon.inf`)
- post-create callback on `IRP_MJ_CREATE` reports file creates
- process-create (`PsSetCreateProcessNotifyRoutineEx`), image-load
  (`PsSetLoadImageNotifyRoutine`) and registry set-value (`CmRegisterCallbackEx`, altitude
  `7657.124`) notifications
- one communication port, `\RAMON.KM2UM.Port`, created with `FltCreateCommunicationPort`, with
  connect / disconnect / message callbacks; events leave the driver through `FltSendMessage`
- an explicit teardown path that unregisters the filter, closes the port and removes every notify
  routine (the `Cleaner` struct keeps the callback pointers and the registry cookie)
- WDK build wired through `build.rs`; the INF installs the service and the instance altitude

User-mode client (`ramon-client`):

- connects to the port and reads events with `FilterGetMessage`
- subcommands: `all`, `registry`, `modules`, `files`, `processes`, `threads`

Shared crate (`common`):

- event structs and the wire format: `EventHeader { event_class: u32, event_size: u32 }` followed
  by the body, little-endian
- event classes `CRE ` (file create), `PRO ` (process create), `DLL ` (image load),
  `REG ` (registry set value)
- fallible serialization for kernel use (`try_reserve_exact` before every allocation), YAML forms
  for tests, and hashing helpers

## Layout

| Crate | What it is |
|---|---|
| `ramon-km` | the driver (`cdylib`, `no_std`); minifilter, notifiers, comm port, teardown |
| `common` | event structs, serialization and the port command format; builds for kernel and host |
| `ramon-client` | console client: connects to the port and prints events |
| `xtask` | build helper for the driver/client artifacts |

## Build

Requirements: Windows, Visual Studio Build Tools, WDK 10 and a Rust nightly toolchain.

```
cd ramon-km
cargo build
```

The output is `target\x86_64-pc-windows-msvc\debug\ramon.dll`; rename it to `ramon.sys`.
`ramon-km/Makefile.toml` has `cargo-make` tasks that build, rename, sign with a self-signed
certificate, copy and run `inf2cat` (`cargo make all`). The host crates build and test with the
default toolchain:

```
cargo build -p ramon-client
cargo test -p common
```

## Trying it

The driver is unsigned; load it only on a test machine. Enable test signing once
(`bcdedit /set testsigning on`, reboot), then either install the INF or load the service directly:

```
fltmc load ramon
.\ramon-client.exe all
fltmc unload ramon
```

## Status

This repository is preserved as it was written in 2025: about 2.4k lines of Rust across the
driver, the shared protocol and the client. It is a prototype, and the code says so:

- global `static mut` state and a single filter/client instance (the 2026 edition replaces this
  with `Locked<T>` / `SpinLocked<T>`)
- only the `IRP_MJ_CREATE` post-op is registered; `IRP_MJ_SET_INFORMATION` is stubbed out
- no policy engine, no self-protection, no signing, no host tests for the driver
- x64 only

It is useful as the reference for the callbacks, the port protocol and the teardown order, not as
a production component.

## Editions

| Edition | Year | How it was written | Repository |
|---|---|---|---|
| 1 | 2025 | By hand, no AI | this repository |
| 2 | 2026 | AI-assisted rewrite and extension | [`radkum/ramon-rs`](https://github.com/radkum/ramon-rs) |

Edition 2 keeps the same idea and the same event protocol, and adds process lineage (`ariadne`),
MOTA file marking, Mirage decoy redirection, a guardrail for protected paths, memory-mapped write
tracking, WFP egress flow telemetry, `static mut`-free state, a 32 KiB event queue with a single
sender thread, a keyed port handshake, a Driver Verifier run, CI and host tests.

## Provenance

Built on [`windows-kernel-rs`](https://github.com/radkum/windows-kernel-rs) (a public example of a
Rust driver and minifilter) and on a fork of
[`winapi-rs`](https://github.com/Trantect/winapi-rs), branch `feature/km`.

## License

Dual-licensed under either of

- Apache License, Version 2.0 ([LICENSE-APACHE](LICENSE-APACHE))
- MIT license ([LICENSE-MIT](LICENSE-MIT))

at your option. Unless you explicitly state otherwise, any contribution intentionally submitted for
inclusion in this project by you shall be dual-licensed as above, without additional terms.

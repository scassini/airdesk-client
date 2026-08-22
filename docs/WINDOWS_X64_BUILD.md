# Build Windows x64 (airdesk)

Runbook distilled from the first successful x64 build (2026-08-16). Follow in
order; each gotcha below was hit for real and cost significant time.

## 1. Toolchain (fresh machine)

Install in this order. All commands assume `C:\dev` as the clone root.

```powershell
winget install --id Git.Git -e --source winget --silent --accept-package-agreements --accept-source-agreements
winget install --id Python.Python.3.12 -e --source winget --silent --accept-package-agreements --accept-source-agreements
winget install --id Microsoft.VisualStudio.2022.BuildTools -e --source winget --silent --accept-package-agreements --accept-source-agreements --override "--quiet --wait --add Microsoft.VisualStudio.Workload.VCTools --add Microsoft.VisualStudio.Component.VC.Tools.x86.x64 --add Microsoft.VisualStudio.Component.VC.CMake.Project --add Microsoft.VisualStudio.Component.Windows11SDK.22621"
```

**Always pass `--source winget`** on install/uninstall — without it, winget
prompts for `msstore` terms-of-service acceptance and hangs forever in a
non-interactive session.

**`Microsoft.VisualStudio.Component.VC.CMake.Project` is not optional.** The
VCTools workload alone does *not* pull in CMake, and `flutter build windows`
fails at the CMake-configure step with "Unable to find suitable Visual
Studio toolchain" if it's missing. Symptom is easy to misread as a generic
VS problem — it's specifically the missing CMake component.

Rust (don't use winget for this — install via rustup directly):
```powershell
curl.exe -o C:\dev\rustup-init.exe https://static.rust-lang.org/rustup/dist/x86_64-pc-windows-msvc/rustup-init.exe
C:\dev\rustup-init.exe -y --default-toolchain stable --profile default
```

LLVM (needed for `bindgen`, used by `magnum-opus`'s build script — without
it, `cargo build` fails with "Unable to find libclang"):
```powershell
curl.exe -L -o C:\dev\LLVM-installer.exe https://github.com/llvm/llvm-project/releases/download/llvmorg-18.1.8/LLVM-18.1.8-win64.exe
C:\dev\LLVM-installer.exe /S
```
Set `LIBCLANG_PATH=C:\Program Files\LLVM\bin` for every build invocation.

vcpkg:
```powershell
git clone https://github.com/microsoft/vcpkg C:\vcpkg
C:\vcpkg\bootstrap-vcpkg.bat
set VCPKG_MAX_CONCURRENCY=8   REM only use 1 if the machine has flaky RAM — see below
C:\vcpkg\vcpkg.exe install aom:x64-windows-static ffmpeg:x64-windows-static libjpeg-turbo:x64-windows-static libsodium:x64-windows-static libvpx:x64-windows-static libyuv:x64-windows-static mfx-dispatch:x64-windows-static opus:x64-windows-static
```

Flutter SDK: download+extract `flutter_windows_3.24.5-stable.zip` to
`C:\src\flutter`. Use `curl.exe`, not `Invoke-WebRequest` (10-50x faster).

Windows Defender exclusions (big speedup, add manually — automation of this
gets blocked by security tooling):
```powershell
Add-MpPreference -ExclusionPath "C:\dev","C:\src","C:\vcpkg"
```

## 2. Clone + rebrand

```powershell
git clone https://github.com/rustdesk/rustdesk C:\dev\airdesk-client
cd C:\dev\airdesk-client
git submodule update --init --recursive
```

Branding edits (see `libs/hbb_common/src/config.rs`, `flutter/windows/runner/Runner.rc`)
— already applied in this repo's history, re-check they're intact after a
fresh clone from upstream `rustdesk/rustdesk` (they are NOT — a fresh clone
resets to stock RustDesk branding).

## 3. Generate the FFI bridge (mandatory, not automated by build.py)

`src/bridge_generated.rs` and `flutter/lib/generated_bridge.dart` are
**gitignored** and must be generated locally before `cargo build` will work.
`build.py`'s Docker helper does this automatically; the bare-metal path does
**not** — this is the single most confusing gap in the stock build script.

The codegen tool version **must exactly match** the `flutter_rust_bridge`
crate version pinned in the root `Cargo.toml` (currently `=1.80`), or the
generated code has trait-bound mismatches (`IntoIntoDart` errors) that look
like a Rust compiler bug but aren't:

```powershell
cargo install flutter_rust_bridge_codegen --version 1.80.1 --locked
```

Do **not** use the fork/version referenced in `build.py`'s embedded Docker
script (`SoLongAndThanksForAllThePizza/flutter_rust_bridge`, currently
resolves to 1.75.0) — it's stale and mismatched with the pinned crate
version.

`flutter/pubspec.yaml`'s `ffigen` version constraint must match what your
frb_codegen build expects — frb_codegen 1.80.1 wants `ffigen >=8.0.0 <9.0.0`.
If `flutter pub get` fails with an ffigen version error, adjust the
constraint, don't downgrade frb_codegen.

```powershell
cd flutter && flutter pub get && cd ..
flutter_rust_bridge_codegen --rust-input ./src/flutter_ffi.rs --dart-output ./flutter/lib/generated_bridge.dart
```

**Known follow-up bug**: the generated `generated_bridge.dart` may declare
FFI `Opaque`/`Struct` subclasses (`_Dart_Handle`, `DartCObject`, `Display`,
`wire_uint_8_list`, `wire_int_32_list`, `wire_StringList`, `XRectangle`)
without the `base` keyword, which Dart 3.x's class-modifier system rejects
at compile time (`must be 'base', 'final' or 'sealed'`). If this happens,
patch each `class X extends ffi.Opaque/Struct` to `base class X extends ...`.
Check first — future frb_codegen point releases may fix this upstream.

## 4. Build

```powershell
set PATH=C:\src\flutter\bin;%PATH%
set VCPKG_ROOT=C:\vcpkg
set LIBCLANG_PATH=C:\Program Files\LLVM\bin
set CARGO_BUILD_JOBS=8
set CARGO_PROFILE_RELEASE_CODEGEN_UNITS=4
python build.py --flutter
```

`build.py` invokes `python3`/`pip3` internally for the packaging step —
on a fresh Windows install these resolve to the broken Microsoft Store app
alias stub ("Accès refusé" / "Python est introuvable" with no useful error).
Only `python`/`pip` (no trailing digit) work. If `build.py` fails at the
`generate.py`/portable-packaging step for this reason, run the remaining
steps manually with `python` explicitly (see §5).

Output: `flutter/build/windows/x64/runner/Release/airdesk.exe` (+ DLLs).

## 5. Package the portable installer

```powershell
cd libs\portable
python -m pip install -r requirements.txt
python ./generate.py -f ../../flutter/build/windows/x64/runner/Release/ -o . -e ../../flutter/build/windows/x64/runner/Release/airdesk.exe
cargo build --locked --release
cd ..\..
```

Output: `target\release\airdesk-portable-packer.exe` — single self-extracting
distributable, embeds `data.bin` (compressed app payload). `build.py`
renames this to `airdesk-qs.exe` (see §7.1) — the `-qs.exe` suffix is
required, not cosmetic.

## 6. Icons — read this before touching anything icon-related

There are **four independent icon/logo locations**. All four must be updated
together or the app will show a mix of old and new branding depending on
which UI surface you're looking at:

| File | Controls | Consumed by |
|---|---|---|
| `flutter/windows/runner/resources/app_icon.ico` | .exe file icon, window icon, taskbar icon | `Runner.rc` → `IDI_APP_ICON`, loaded via `LoadIcon` in `win32_window.cpp` |
| `res/icon.ico` | portable-packer wrapper .exe's own icon | `libs/portable/build.rs` → `winres` |
| `res/tray-icon.ico` | system tray icon | compiled into `librustdesk.dll` via `include_bytes!` in `src/tray.rs` |
| `flutter/assets/icon.png` (+ `icon.svg` fallback) | **in-app logo widget** shown in the app UI itself (home/connection screen) | `loadIcon()` in `flutter/lib/common.dart` — this is a Flutter asset, has nothing to do with the three `.ico` files above |

Easy to fix three of four and still see "the wrong icon everywhere" — the
in-app logo is the one people forget, since it's not a Windows resource at
all.

**ICO file format gotcha**: if you generate the `.ico` with Pillow
(`Image.save(..., format='ICO', sizes=[...])`), it encodes **every** frame
as PNG, including the small 16/32/48/64px ones. Windows Explorer's shell
icon extraction (`SHGetFileInfo`) and `System.Drawing.Icon` render this as
garbage/noise instead of falling back gracefully — the icon effectively
fails to load and Explorer silently shows a cached/default icon instead. No
error is raised anywhere in the build. Fix: build the `.ico` with raw
32bpp BGRA DIB frames for sizes ≤64px and PNG only for 128/256px (the
standard convention). A working generator snippet exists in this session's
history — do not use Pillow's default ICO writer for anything meant to
ship.

**Verifying an icon actually works**: don't trust visual inspection of the
source PNG. Verify what Windows will actually render:
```powershell
Add-Type -AssemblyName System.Drawing
$icon = [System.Drawing.Icon]::ExtractAssociatedIcon("path\to\built.exe")
$ms = New-Object System.IO.MemoryStream
$icon.ToBitmap().Save($ms, [System.Drawing.Imaging.ImageFormat]::Png)
[IO.File]::WriteAllBytes("check.png", $ms.ToArray())
```
If this renders as noise, the ICO format is the problem, not caching —
check the format before assuming it's a Windows icon-cache issue.

## 7. Renaming the executable (if rebranding away from "rustdesk")

Four places, must change together:
- `flutter/windows/CMakeLists.txt`: `set(BINARY_NAME "airdesk")`
- `libs/portable/src/main.rs`: `const APP_PREFIX: &str = "airdesk";` — also
  changes the runtime extraction folder from `%LocalAppData%\rustdesk` to
  `%LocalAppData%\airdesk`
- `libs/portable/Cargo.toml`: `name = "airdesk-portable-packer"`
- `build.py`: `generate.py -e .../airdesk.exe` invocation + output-rename
  line (`airdesk_portable.exe` → `airdesk-qs.exe`)

### 7.1 Final filename must end in `-qs.exe`

`core_main.rs::is_quick_support_exe()` checks the running exe's filename for
`-qs-`, `-qs.exe`, or `_qs.exe`. When it matches (and the app isn't already
installed), RustDesk auto-enables "Quick Support" mode, which starts the
portable service and requests admin elevation automatically — no separate
installer step, no manual "Run as administrator". `build.py`'s final rename
step must produce `airdesk-qs.exe`, not a versioned name like
`airdesk-{version}-install.exe` — renaming it after the fact defeats the
purpose since the check runs against the filename at launch.

Renaming a workspace member's `Cargo.toml` package name **breaks
`--locked`** for the whole workspace until `Cargo.lock` is updated. Don't
regenerate the lockfile from scratch (`cargo generate-lockfile`) — it
re-resolves git dependencies to their current HEAD, which can break
unrelated pinned versions (hit this with `tray-icon`, whose HEAD had moved
past the `^0.21.3` requirement in `Cargo.toml`). Instead, surgically edit
the one `name = "..."` line for that package inside `Cargo.lock`.

## 8. Known-safe build settings

- `VCPKG_MAX_CONCURRENCY=8`, `CARGO_BUILD_JOBS=8`,
  `CARGO_PROFILE_RELEASE_CODEGEN_UNITS=4` — fine on a healthy 8-core/16GB+
  machine (cut the vcpkg build for all 8 packages, incl. ffmpeg, from 5+
  hours down to ~15 minutes vs. concurrency=1).
- Only drop to `VCPKG_MAX_CONCURRENCY=1` if you're seeing
  `STATUS_HEAP_CORRUPTION` / `STATUS_ACCESS_VIOLATION` crashes during
  compilation — that's a hardware (RAM) symptom, not a build config one.
  Run Windows Memory Diagnostic before assuming it's software.

## 9. Multi-user-profile gotcha

If Rust/Cargo were installed under one Windows user account and you're
building from a different one (e.g. the machine got renamed/re-profiled
mid-project), `cargo`/`rustup` need `RUSTUP_HOME` and `CARGO_HOME` pointed
at the original install (`C:\Users\<original-user>\.rustup` /
`.cargo`) — otherwise `rustup` errors with "could not choose a version of
cargo to run" even though `cargo.exe` itself is on PATH.

## 10. Running builds unattended over SSH

`flutter.bat` (and some other batch-wrapped tools) can terminate their own
parent `cmd.exe` process when run non-interactively — a multi-step batch
script that calls `flutter pub get` then continues with more commands may
silently stop after the flutter call with no error. Split flutter
invocations into their own isolated script/process rather than chaining
them with unrelated steps.

For long builds that must survive an SSH/API disconnection: launch via WMI
(`Invoke-CimMethod -ClassName Win32_Process -MethodName Create`), not
`Start-Process` (which can silently no-op in a detached SSH session) and
not Scheduled Tasks (registers fine but process creation can fail silently
depending on the machine's security posture, with no useful error).

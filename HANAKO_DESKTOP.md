# Hanako Desktop

Desktop sibling of [Hanako for Android](https://github.com/guincalo/Hanako).
Branch `hanako` = AyuGram Desktop `dev` (full history) + Hanako branding + CI.
Branch `dev` = untouched mirror of `AyuGram/AyuGramDesktop@dev` at fork time.

## Base choice: AyuGram Desktop

Checked live on 2026-10-20:

| | AyuGram/AyuGramDesktop | forkgram/tdesktop |
|---|---|---|
| Latest release | v7.2.9 (2026-10-05) | v7.3.1 (2026-10-10) |
| Upstream base | merged tag v7.2.9; upstream `dev` is now at v7.3.1+ (1171 commits ahead of AyuGram `dev`) | v7.3.1, tracks upstream within days |
| Own patches vs upstream tag | 774 commits; upstream sources touched: 316 files, +6980/−1489 lines; plus a self-contained `Telegram/SourceFiles/ayu/` tree (~147 files, mostly vendored sqlite/sqlite_orm/json) | 866 commits, mostly CI/packaging; no client features we need |
| Windows/Linux GitHub Actions | **none** (removed in "chore: remove workflows", 2025-05-20; only the upstream canary pipelines remain, which need Telegram's own secrets) | working Windows/Linux release workflows (green on 2026-10-19) |
| Ghost mode, deleted + edit history, message filters, streamer mode, message shot | yes | no |

Decision: **AyuGram**. It already covers the top of Hanako's feature list
(ghost mode, deleted/edit history, filters, streamer mode, message shot), which
would be thousands of lines of C++ to write on forkgram. It is one minor version
behind upstream (7.2.9 vs 7.3.1), which is acceptable. Its missing CI is the cost:
the workflows here are adapted from upstream tdesktop's `win.yml`/`linux.yml` at
tag v7.2.9 (AyuGram's `Telegram/build/prepare` and `Telegram/build/docker` are
byte-identical to upstream v7.2.9, so upstream's recipe applies unchanged), switched
to Release builds and split into "libraries" and "build" jobs so each fits the
6-hour hosted-runner limit. Forkgram stays useful as a reference for release CI.

## Branding done in this pass

- Binary `Hanako` / `Hanako.exe` (CMake `output_name`), app name "Hanako Desktop",
  short name "Hanako", Qt application name `HanakoDesktop` (so its data dir does not
  collide with an AyuGram install), window title, tray strings, crash window,
  Windows version resource, Start-menu shortcut names.
- Linux `.desktop` entry: name/Exec/TryExec/WMClass → Hanako. The desktop-file id
  is still `com.ayugram.desktop` (it is wired through the source in several places;
  renaming it is a follow-up).
- Inno Setup script: Hanako names, new AppId GUID, signing removed, installs `Hanako.exe`.
- Icons and in-app AyuGram logos/strings (settings page, about box) are unchanged.
- Updater: CI passes `-D DESKTOP_APP_DISABLE_AUTOUPDATE=ON` (AyuGram's own updater
  is also disabled in its source). Crash reports off.

## API credentials

Hanako Android hardcodes Goregram's public `api_id 105810` in its workflow. The same
values are stored here as repository secrets `TDESKTOP_API_ID` / `TDESKTOP_API_HASH`
and passed as `-D TDESKTOP_API_ID/HASH`. With the secrets unset (forks/PRs), CI falls
back to `-D TDESKTOP_API_TEST=ON` (Telegram's test credentials).

## CI

- `.github/workflows/win.yml` — Windows x64 Release.
  - `libraries`: `Telegram\build\prepare\win.bat silent` (Debug+Release libs, Qt default),
    cached as `win64-libs-complete-<hash of prepare.py + SDK>`; a timed-out run saves a
    `partial` cache and the next run resumes.
  - `build`: `configure.bat x64 … -D CMAKE_CONFIGURATION_TYPES=Release`, then
    artifact `Hanako-win64-portable` (zip with `Hanako.exe` + `modules/`) and,
    best-effort, `Hanako-win64-installer` (`hanakosetup-x64.<ver>.exe`).
- `.github/workflows/linux.yml` — Linux x86_64 Release.
  - `env`: official Rocky Linux 8 library image from `Telegram/build/docker/centos_env`
    (generated with `DEBUG= LTO=`), pushed to `ghcr.io/guincalo/hanakodesktop-env:centos-<hash>`
    with a registry layer cache; skipped when that tag exists.
  - `build`: `build.sh` in that image with `CONFIG=Release`, ccache in the Actions cache;
    artifact `Hanako-linux-x86_64` (`Hanako/Hanako` in a `.tar.xz`).
- Both trigger on push to `hanako` and on manual dispatch. AyuGram's canary workflows
  were removed (they need Telegram's signing infrastructure).

## Building locally

Same as upstream tdesktop/AyuGram (see `docs/building-win.md`, `docs/building-linux.md`);
the binary is called `Hanako`. Don't build on the 1.9 GiB VPS.

Windows (VS 2022/2026 with the v14.44 toolset, SDK 10.0.26100.0):

    git clone --recursive -b hanako https://github.com/guincalo/HanakoDesktop.git tdesktop
    tdesktop\Telegram\build\prepare\win.bat
    cd tdesktop\Telegram
    configure.bat x64 -D TDESKTOP_API_ID=... -D TDESKTOP_API_HASH=... -D DESKTOP_APP_DISABLE_AUTOUPDATE=ON
    cmake --build ..\out --config Release

Linux (docker):

    cd Telegram/build/docker/centos_env && D=$(DEBUG= LTO= python3 gen_dockerfile.py) && echo "$D" > Dockerfile
    docker build -t hanako-env .
    cd ../../../.. && docker run --rm -u $(id -u) -v $PWD:/usr/src/tdesktop -e CONFIG=Release hanako-env \
      /usr/src/tdesktop/Telegram/build/docker/centos_env/build.sh -D TDESKTOP_API_ID=... -D TDESKTOP_API_HASH=... -D DESKTOP_APP_DISABLE_AUTOUPDATE=ON

## Feature gap vs Hanako Android

| Hanako Android feature | Desktop base status | Work needed |
|---|---|---|
| Ghost mode | **AyuGram has it** (no read/online/typing/upload/story-read packets, per-account or global, drawer/tray toggles) | Hanako naming/UX only |
| Deleted-message history | **AyuGram has it** (local sqlite `ayu/data`) | none for parity; later: import/export with Android's format |
| Edit history | **AyuGram has it** (`saveMessagesHistory`, `ayu/ui/message_history`) | none |
| Message filters | **AyuGram has it** (regex filters, import box, per-chat) | check rule format compatibility with Android |
| Streamer mode | **AyuGram has it** (`features/streamer_mode`, hides windows from capture) | none |
| Message shot | **AyuGram has it** (`features/message_shot`) | none |
| Settings search | **upstream has it** (`settings/settings_search.cpp`) | index AyuGram/Hanako settings pages so they are searchable (C++) |
| Export | **upstream has it** (`export/`, the reference implementation) | none; Android should mirror it |
| Chat lock (per-chat passcode/biometric) | missing (only global local passcode) | C++: lock state per peer, overlay over history widget, Windows Hello via WinRT optional |
| Peek (open chat without marking read) | partial (ghost mode can suppress read receipts globally; no one-shot peek) | C++: context-menu "Peek" that opens history with read packets suppressed for that peer only |
| Session backup | missing | C++: export/import of `tdata` keys for an account (encrypted file), plus restore flow |
| Activity log | missing | C++: local log (sqlite in AyuGram's DB) of online/status/profile changes + viewer UI |
| Parakeet STT | missing (Telegram Premium transcription only) | C++ + bundled model runtime (sherpa-onnx / onnxruntime), model download on demand; largest item |

## Proposed port order

1. Branding finish: icons, desktop-file id `com.hanako.desktop`, in-app AyuGram strings/logos, About box.
2. Settings search over AyuGram/Hanako pages (small, touches only settings, makes everything else discoverable).
3. Peek (small; reuses ghost-mode plumbing).
4. Chat lock.
5. Activity log (reuses AyuGram's sqlite layer).
6. Session backup.
7. Parakeet STT (biggest: native runtime + model management + Windows/Linux packaging).
8. Rebase onto newer AyuGram releases as they land (they follow upstream tdesktop with a lag of ~1–3 weeks).

# winget WiX/.msi installer: notes (fork-only)

Goal: give fzf's winget manifest a real per-machine `.msi` alongside the
portable zip, as Starship and PowerShell ship, for parity across the four
shell-integrated tools (Atuin, fzf, Yazi, zoxide).

## The problem

winget's portable install puts a symlink in `%LOCALAPPDATA%\Microsoft\WinGet\Links`,
created by whoever runs winget, normally a non-elevated terminal. Windows will
not let an elevated process follow a link that a less-privileged process
created (error 448, "The path cannot be traversed because it contains an
untrusted mount point"). An administrator's SSH session is elevated, so fzf
fails there; a standard user is unaffected.

## The change (commit "Add WiX .msi installer alongside portable zip for Windows")

- `wix/main.wxs`: WiX v4/v5 source; per-machine install to
  `Program Files\fzf\bin`, machine PATH entry, LICENSE.
- `.github/workflows/release.yml`: the macOS release job uploads goreleaser's
  windows amd64/arm64 `fzf.exe` as an artifact; a new `msi` job on
  `windows-latest` builds both MSIs (WiX only runs on Windows), in both
  workflow modes, and uploads them to the release on a tag push. WiX is
  pinned to 5.0.2, the last release without the Open Source Maintenance Fee
  EULA requirement.
- `.github/workflows/winget.yml`: installers-regex also matches `.msi`.

## Verified

- Fork smoke run (branch `winget-wix-smoke`, `msi-smoke.yml`):
  https://github.com/meop/fzf/actions/runs/37306258749. Builds both MSIs as
  release.yml does; on `windows-latest` and `windows-11-arm`, installs a real
  `fzf.exe` of the right architecture (0x8664, 0xAA64), adds it to the machine
  PATH, runs it locally and over SSH, uninstalls cleanly.
- Real machine (glass, Windows 11 26H2 x64), with the zoxide fork's
  `windows-msi-test/repro-ssh.ps1`: winget portable install from the desktop
  session, then run from an elevated SSH session: error 448. The MSI install
  runs from the same session. 2026-10-05.

## Open

- Whether winget-releaser sets `Scope: machine` for the `.msi` (it happens
  inside that action); check on the first real release.
- fzf's PR template: "We do not accept pull requests generated primarily by
  AI without genuine understanding or real-world usage context", with an
  acknowledgement checkbox. The PR description should be the user's own
  words, from hitting this over SSH; the evidence above backs it.

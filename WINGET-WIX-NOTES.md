# winget WiX/.msi installer — WIP notes

Goal: give fzf's winget manifest a real per-machine `.msi` installer
alongside the existing portable/zip one, matching starship's setup.
Motivation: winget's portable installers land as a symlink in the WinGet
Links folder, and Windows blocks following symlinks over SSH by default.
fzf is the weakest-but-plausible candidate of this batch: official
`fzf --bash`/`fzf --zsh` shell-integration flags are sourced from shell
profiles for keybindings (Ctrl+R/Ctrl+T), similar in spirit to
starship/zoxide/atuin's init-hook pattern, though less first-party for
pwsh/nu specifically.

**Key difference from the other 3 repos in this batch**: fzf is **Go**, not
Rust — `cargo wix` doesn't apply. This uses the WiX v4/v5 dotnet-tool CLI
instead.

Reference implementation studied conceptually: starship/starship (WiX
structure — `Product`/`Package`, `UpgradeCode`, per-machine install,
PATH-registering `Environment` component). Not the literal `cargo wix`
tooling, since that's Rust-specific.

## What's done (commit e1d098c1, branch `winget-wix-installer`)
- `wix/main.wxs` (new — used `wix/` not `install/`, since `install` already
  exists as fzf's shell install script, a file not a dir) — WiX v4/v5
  template, fresh UpgradeCode GUID, per-machine install to
  `Program Files\fzf\bin`, PATH-registering `Environment` component, ships
  `LICENSE`. No icon/banner (none exist in-repo).
- `.github/workflows/release.yml` — added 3 steps after the existing
  `goreleaser` step (same `macos-latest` job): install the WiX dotnet-tool
  CLI (`dotnet tool install --global wix`), build `.msi` for amd64/arm64 by
  locating the goreleaser-built `fzf.exe` under `dist/` via `find`, upload
  via `gh release upload --clobber`. Runs cross-platform on macOS since WiX
  v4/v5 is pure .NET tooling — no Windows runner needed. No `.msi` for the
  armv7 build (WiX v4's `-arch` flag doesn't support it; zip-only there,
  matching what winget already ships for that arch).
- `.github/workflows/winget.yml` — widened `installers-regex` to also match
  `.msi`.

## Needs verification (none of this ran in real CI)
- [ ] Confirm GitHub's `macos-latest` runner has a compatible .NET SDK
      preinstalled for `dotnet tool install --global wix` (or add a
      `setup-dotnet` step if not).
- [ ] The `find ... -path "*windows_${goarch}*"` glob used to locate
      goreleaser's built `fzf.exe` is a reasonable guess at goreleaser's
      `dist/` naming convention, not confirmed against a real
      `goreleaser build --snapshot` run — verify the path actually resolves.
- [ ] Confirm `wingetcreate`/`winget-releaser` correctly infers `Scope:
      machine` for the `.msi` vs. leaving the `.zip` unscoped, the way
      starship's manifest does explicitly (that inference happens inside
      the third-party action, not in this repo).
- [ ] Confirm the `.msi` actually builds, installs, and lands `fzf.exe` on
      PATH.

## Not in scope here
Submitting the actual winget-pkgs manifest update — separate PR to
`microsoft/winget-pkgs` once the `.msi` is a real release asset.

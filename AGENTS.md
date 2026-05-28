# PROJECT KNOWLEDGE BASE

**Generated:** 2026-05-28
**Commit:** 00c5951
**Branch:** kix

## OVERVIEW
OpenWrt feed-style package repo for daed + its LuCI UI. Clone it as `package/dae` inside an OpenWrt tree; this is not a standalone app workspace.

## STRUCTURE
```
./
├── daed/              # backend OpenWrt package; builds /usr/bin/daed from upstream Git repos
├── luci-app-daed/     # LuCI app package; legacy luasrc/controller + CBI + view layout
├── patchset/          # standalone upstream patch stash; not wired into package Makefiles
├── PIC/               # README screenshots only
└── .github/workflows/ # release/autoupdate automation for branch kix
```

No child `AGENTS.md` files are warranted: the repo has ~23 non-image files, and root guidance covers the two package boundaries without duplication.

## WHERE TO LOOK
| Task | Location | Notes |
|------|----------|-------|
| Backend package pins/build | `daed/Makefile` | Coupled upstream SHAs, Node/pnpm web build, Go/BPF generation |
| Runtime daemon config | `daed/files/daed.config` | Installed as `/etc/config/daed`; keep LuCI UCI names aligned |
| procd service | `daed/files/daed.init` | Starts `/usr/bin/daed run`; owns `DAE_LOCATION_ASSET` and `/etc/daed` setup |
| LuCI package metadata | `luci-app-daed/Makefile` | Depends on `+daed +zoneinfo-asia +luci-compat`, `LUCI_PKGARCH:=all` |
| LuCI routes | `luci-app-daed/luasrc/controller/daed.lua` | Menu hidden unless `/etc/config/daed` exists |
| Settings UI | `luci-app-daed/luasrc/model/cbi/daed/basic.lua` | Applies UCI and restarts `/etc/init.d/daed` |
| Dashboard iframe | `luci-app-daed/luasrc/view/daed/daed.htm` | Uses `dashboard_port`, else parses port from `listen_addr` |
| LuCI-side service hook | `luci-app-daed/root/etc/init.d/luci_daed` | Toggles `dhcp.@dnsmasq[0].localuse`; no longer edits `/etc/init.d/daed` |
| Network hotplug | `luci-app-daed/root/etc/hotplug.d/iface/98-daed` | Restarts daed 60s after ethernet `ifup`, guarded by lock file |
| CI package build | `.github/workflows/build-packages.yml` | Builds `PACKAGES: luci-app-daed` with OpenWrt SDK action |
| Upstream bump automation | `.github/workflows/autoupdate.yml` | Rewrites `daed/Makefile` pins; chmods init/hotplug scripts; runs dos2unix |

## CODE MAP
| Symbol / entry | Type | Location | Role |
|----------------|------|----------|------|
| `Package/daed` | OpenWrt package stanza | `daed/Makefile` | Backend deps/install metadata |
| `Build/Prepare` | OpenWrt build hook | `daed/Makefile` | Clones `dae-wing` + `dae`, builds web UI assets |
| `Build/Compile` | OpenWrt build hook | `daed/Makefile` | Runs wing and dae-core `go generate`, then Go package compile |
| `start_service` | procd init function | `daed/files/daed.init` | Reads UCI, sets env, starts daemon with log/config flags |
| `index()` | LuCI controller | `luci-app-daed/luasrc/controller/daed.lua` | Registers settings/dashboard/log/status routes |
| `Map("daed")` | CBI model | `luci-app-daed/luasrc/model/cbi/daed/basic.lua` | Main UCI form and restart-on-apply |

## CONVENTIONS
- Treat this as OpenWrt packaging, not a standalone JS/Go/Lua app.
- `luci-app-daed/` uses legacy LuCI Lua/CBI; do not convert assumptions to `htdocs/luci-static/resources/view` JS layout.
- UCI option names must stay synchronized across `daed/files/daed.config`, `basic.lua`, `daed.init`, and `daed.htm`.
- `PKG_SOURCE_VERSION` is the full `daeuniverse/daed` commit; `DAED_VERSION`, `WING_VERSION`, and `CORE_VERSION` hold short hashes used by `Build/Prepare`.
- Init scripts and hotplug files must remain executable; CI explicitly `chmod 755`s them during autoupdate.

## ANTI-PATTERNS (THIS PROJECT)
- Do not treat `patchset/` as an OpenWrt package `patches/` directory; package Makefiles do not apply it.
- Do not add repo-local npm/go test commands unless manifests/configs are added; none exist now.
- Do not reintroduce runtime `sed` edits from `luci_daed` into `/etc/init.d/daed`; daemon env belongs in `daed/files/daed.init`.
- Do not change `listen_addr` semantics without updating dashboard port derivation in `luci-app-daed/luasrc/view/daed/daed.htm`.
- Do not delete daemon logs on stop; `daed.init` intentionally preserves `/var/log/daed/daed.log` for debugging.

## COMMANDS
```bash
# From the parent OpenWrt tree, after cloning this repo as package/dae:
make menuconfig              # LUCI -> Applications -> luci-app-daed
make package/dae/luci-app-daed/compile V=s

# Lightweight local syntax checks for edited init scripts:
sh -n daed/files/daed.init
sh -n luci-app-daed/root/etc/init.d/luci_daed
sh -n luci-app-daed/root/etc/hotplug.d/iface/98-daed
```

## BUILD / CI NOTES
- Build hosts need recent `clang`/`llvm`; README also lists `npm` + global `pnpm` for build preparation.
- `daed/Makefile` downloads Node `v24.12.0`, installs `pnpm`, runs `pnpm install`, and `pnpm build --filter daed` inside the upstream daed source checkout.
- `Build/Compile` exports BPF variables and runs `go generate control/control.go` plus `go generate trace/trace.go` inside `dae-core`.
- CI matrix builds OpenWrt 24.10 (`ipk`) and 25.12 (`apk`) on `aarch64_generic` and `x86_64` using `sbwml/openwrt-gh-action-sdk@go1.26`.
- `autoupdate.yml` can continue after commit/push failures (`continue-on-error: true`); verify generated `daed/Makefile` diffs manually.

## TESTING REALITY
- No repo-local `package.json`, lockfile, `go.mod`, Taskfile, Justfile, test directory, or configured lint/typecheck command exists.
- Practical verification is OpenWrt package compile/CI; for script-only edits, run `sh -n` on the touched rc/hotplug files.

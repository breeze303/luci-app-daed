# AGENTS.md

This is an OpenWrt feed-style package repo for `daed` plus its LuCI UI. Clone/use it as `package/dae` inside an OpenWrt tree; there is no standalone Go/Node/Lua workspace here.

## Package boundaries

- `daed/`: backend OpenWrt package. Its `Makefile` fetches `daeuniverse/daed`, separately clones `dae-wing` and `dae` at pinned short SHAs, builds the web UI, generates BPF bindings, and installs the binary as `/usr/bin/daed`.
- `luci-app-daed/`: LuCI package using legacy Lua controller + CBI models + template views, not the newer JS `htdocs/luci-static/resources/view` app layout.
- `patchset/`: standalone patch stash only. Package Makefiles do not apply it as OpenWrt `patches/`.
- `PIC/`: README screenshots.

## Commands agents usually guess wrong

Run package builds from the parent OpenWrt tree after this repo is checked out as `package/dae`:

```bash
make menuconfig              # LUCI -> Applications -> luci-app-daed
make package/dae/luci-app-daed/compile V=s
```

There is no repo-local `go test`, `npm test`, lint, typecheck, or lockfile workflow. For script-only edits, use syntax checks on touched files:

```bash
sh -n daed/files/daed.init
sh -n luci-app-daed/root/etc/init.d/luci_daed
sh -n luci-app-daed/root/etc/hotplug.d/iface/98-daed
```

## Build and runtime wiring

- `daed/Makefile` pins `PKG_SOURCE_VERSION` (full daed SHA), plus `DAED_VERSION`, `WING_VERSION`, and `CORE_VERSION` short hashes; keep these coupled.
- `Build/Prepare` downloads Node `v24.12.0`, installs `pnpm`, runs `pnpm install` and `pnpm build --filter daed`, then copies `apps/web/dist` into `wing/webrender/web`.
- `Build/Compile` runs `go generate ./...` in wing and `go generate control/control.go` + `go generate trace/trace.go` in `dae-core` with `BPF_TARGET="bpfel,bpfeb"`; do not hand-edit generated BPF outputs in fetched sources.
- Build hosts need recent `clang`/`llvm`, `npm`, and global `pnpm`; OpenWrt config must support eBPF/BTF (`CONFIG_KERNEL_DEBUG_INFO_BTF=y` for kernel BTF, or the vmlinux-btf package path from README).
- `daed/files/daed.init` is the procd entrypoint: starts `/usr/bin/daed run --config /etc/daed/ --listen <listen_addr>`, sets `DAE_LOCATION_ASSET=/usr/share/v2ray`, and preserves `/var/log/daed/daed.log` on stop.

## LuCI/UCI coupling

Keep UCI option names synchronized across:

- `daed/files/daed.config`
- `daed/files/daed.init`
- `luci-app-daed/luasrc/model/cbi/daed/basic.lua`
- `luci-app-daed/luasrc/view/daed/daed.htm`

Non-obvious current mismatch: `dashboard_port` exists in the LuCI model/view but not in `daed/files/daed.config`; changing dashboard/listen behavior must address both sides. The dashboard iframe uses `dashboard_port` when set, otherwise parses the port from `listen_addr`.

## LuCI-side operational side effects

- `luci-app-daed/luasrc/controller/daed.lua` hides the menu unless `/etc/config/daed` exists and exposes status/log fetch/clear endpoints.
- `luci-app-daed/root/etc/init.d/luci_daed` is not just UI glue: when daed is disabled/stopped it sets `dhcp.@dnsmasq[0].localuse=1`; when enabled it sets `localuse=0`, restarts dnsmasq, and rewrites `/etc/resolv.conf` from WAN DNS or fallback public DNS.
- `luci-app-daed/root/etc/hotplug.d/iface/98-daed` restarts daed 60 seconds after ethernet `ifup`, guarded by a mkdir lock under `/tmp/lock`.
- `luci-app-daed/root/usr/share/rpcd/acl.d/luci-app-daed.json` grants LuCI UCI read/write access only for `daed`.

## CI and release automation

- `.github/workflows/build-packages.yml` builds only `PACKAGES: luci-app-daed` via `sbwml/openwrt-gh-action-sdk@go1.26` for OpenWrt `24.10` (`ipk`) and `25.12` (`apk`) on `aarch64_generic` and `x86_64`.
- `.github/workflows/autoupdate.yml` polls `daeuniverse/daed`, `daeuniverse/dae-wing`, and `olicesx/dae`, rewrites the pin fields in `daed/Makefile`, runs `dos2unix`, and chmods init/hotplug scripts. Commit/push steps are `continue-on-error`; verify resulting diffs manually.
- Init scripts and hotplug files must stay executable; CI explicitly `chmod 755`s them.

## Avoid

- Do not add standalone npm/go commands unless manifests/configs are introduced.
- Do not treat `patchset/` as automatically applied package patches.
- Do not delete daemon logs in `daed.init` stop handling; the current behavior intentionally keeps logs for debugging.
- Do not convert the LuCI app to JS views unless explicitly requested; current package depends on `luci-compat` and legacy Lua/CBI files.

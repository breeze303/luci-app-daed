# SPDX-License-Identifier: GPL-2.0-only
#
# Copyright (C) 2023 ImmortalWrt.org
#

include $(TOPDIR)/rules.mk

PKG_NAME:=daed
PKG_VERSION:=2026.02.27
PKG_RELEASE:=1

PKG_SOURCE:=$(PKG_NAME)-$(PKG_VERSION).tar.gz
PKG_SOURCE_URL:=https://github.com/breeze303/daed-ci/releases/download/daed-source-latest/
PKG_HASH:=skip

PKG_LICENSE:=AGPL-3.0-only MIT
PKG_LICENSE_FILES:=LICENSE wing/LICENSE
PKG_MAINTAINER:=Tianling Shen <cnsztl@immortalwrt.org>

# 解压后的主工程目录
PKG_BUILD_DIR:=$(BUILD_DIR)/$(PKG_NAME)-$(PKG_VERSION)/wing
PKG_BUILD_DEPENDS:=golang/host bpf-headers
PKG_BUILD_PARALLEL:=1
PKG_BUILD_FLAGS:=no-mips16

GO_PKG:=github.com/daeuniverse/dae-wing
GO_PKG_LDFLAGS:= \
	-s -w \
	-X '$(GO_PKG)/db.AppDescription=$(PKG_NAME) is a integration solution of dae, API and UI.'
GO_PKG_LDFLAGS_X:= \
	$(GO_PKG)/db.AppName=$(PKG_NAME) \
	$(GO_PKG)/db.AppVersion=$(PKG_VERSION)-r$(PKG_RELEASE)
GO_PKG_TAGS:=embedallowed,trace

include $(INCLUDE_DIR)/package.mk
include $(INCLUDE_DIR)/bpf.mk
include $(TOPDIR)/feeds/packages/lang/golang/golang-package.mk

GO_PKG_TARGET_VARS+= \
	CGO_LDFLAGS="$(TARGET_LDFLAGS) -static" \
	GOEXPERIMENT=newinliner,simd

GO_PKG_DEFAULT_LDFLAGS:=-w -s -linkmode external -extldflags '-static'

define Package/daed/Default
  SECTION:=net
  CATEGORY:=Network
  SUBMENU:=Web Servers/Proxies
  URL:=https://github.com/daeuniverse/daed
endef

define Package/daed
  $(call Package/daed/Default)
  TITLE:=A Modern Dashboard For dae
  DEPENDS:=$(GO_ARCH_DEPENDS) $(BPF_DEPENDS) \
	+ca-bundle +kmod-sched-core +kmod-sched-bpf +kmod-xdp-sockets-diag \
	+kmod-veth +v2ray-geoip +v2ray-geosite
endef

define Package/daed/description
daed is a backend of dae, provides a method to bundle arbitrary
frontend, dae and geodata into one binary.
endef

define Package/daed/conffiles
/etc/daed/wing.db
/etc/config/daed
endef

define Build/Prepare
	mkdir -p $(PKG_BUILD_DIR)/..
	$(TAR) -C $(PKG_BUILD_DIR)/.. -xzf $(DL_DIR)/$(PKG_SOURCE)
endef

DAE_CFLAGS:= \
	-O2 -Wall -Werror \
	-DMAX_MATCH_SET_LEN=1024 \
	-I$(BPF_HEADERS_DIR)/tools/lib \
	-I$(BPF_HEADERS_DIR)/arch/$(BPF_KARCH)/include/asm/mach-generic

define Build/Compile
	( \
		set -e; \
		case "$$(uname -m)" in \
			x86_64) host_goarch=amd64 ;; \
			aarch64|arm64) host_goarch=arm64 ;; \
			i?86) host_goarch=386 ;; \
			*) echo "Unsupported host arch: $$(uname -m)" >&2; exit 1 ;; \
		esac; \
		host_goos="$$(uname -s | tr '[:upper:]' '[:lower:]')"; \
		host_go_generate() { \
			$(GO_PKG_VARS) \
			GOOS="$$host_goos" GOARCH="$$host_goarch" \
			GO386= GOAMD64= GOARM= GOARM64= GOMIPS= GOMIPS64= GOPPC64= \
			CGO_ENABLED=0 \
			CC= CXX= \
			CGO_CFLAGS= CGO_CPPFLAGS= CGO_CXXFLAGS= CGO_LDFLAGS= \
			go generate "$$@"; \
		}; \
		cd $(PKG_BUILD_DIR); \
		host_go_generate ./...; \
		cd ../dae-core; \
		export BPF_CLANG="$(CLANG)"; \
		export BPF_STRIP_FLAG="-strip=$(LLVM_STRIP)"; \
		export BPF_CFLAGS="$(DAE_CFLAGS)"; \
		export BPF_TARGET="bpfel,bpfeb"; \
		export BPF_TRACE_TARGET="$(GO_ARCH)"; \
		host_go_generate ./control/control.go; \
		host_go_generate ./trace/trace.go; \
	)
	$(call GoPackage/Build/Compile)
endef

define Package/daed/install
	$(call GoPackage/Package/Install/Bin,$(PKG_INSTALL_DIR))

	$(INSTALL_DIR) $(1)/usr/bin
	$(INSTALL_BIN) $(PKG_INSTALL_DIR)/usr/bin/dae-wing $(1)/usr/bin/daed

	$(INSTALL_DIR) $(1)/etc/config
	$(INSTALL_CONF) $(CURDIR)/files/daed.config $(1)/etc/config/daed

	$(INSTALL_DIR) $(1)/etc/init.d
	$(INSTALL_BIN) $(CURDIR)/files/daed.init $(1)/etc/init.d/daed
endef

$(eval $(call GoBinPackage,daed))
$(eval $(call BuildPackage,daed))

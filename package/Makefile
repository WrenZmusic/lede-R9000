# SPDX-License-Identifier: GPL-2.0-only
#
# Copyright (C) 2007 OpenWrt.org

TOPDIR:=${CURDIR}
LC_ALL:=C
LANG:=C
TZ:=UTC
export TOPDIR LC_ALL LANG TZ

empty:=
space:= $(empty) $(empty)
$(if $(findstring $(space),$(TOPDIR)),$(error ERROR: Build directory path must not contain spaces))

world:

DISTRO_PKG_CONFIG:=$(shell $(TOPDIR)/scripts/command_all.sh pkg-config | grep -e '/usr' -e '/nix/store' -m 1)

export ORIG_PATH:=$(if $(ORIG_PATH),$(ORIG_PATH),$(PATH))
export PATH:=$(if $(STAGING_DIR),$(abspath $(STAGING_DIR)/../host/bin),$(TOPDIR)/staging_dir/host/bin):$(PATH)

ifneq ($(OPENWRT_BUILD),1)
  _SINGLE=export MAKEFLAGS=$(space);

  override OPENWRT_BUILD=1
  export OPENWRT_BUILD
  GREP_OPTIONS=
  export GREP_OPTIONS
  CDPATH=
  export CDPATH
  include $(TOPDIR)/include/debug.mk
  include $(TOPDIR)/include/depends.mk
  include $(TOPDIR)/include/toplevel.mk
else
  include rules.mk
  include $(INCLUDE_DIR)/depends.mk
  include $(INCLUDE_DIR)/subdir.mk
  include target/Makefile
  include package/Makefile
  include tools/Makefile
  include toolchain/Makefile

# Include test suite if available
-include tests/Makefile

# Dependency chaining
$(toolchain/stamp-compile): $(tools/stamp-compile) $(if $(CONFIG_BUILDBOT),toolchain_rebuild_check)
$(target/stamp-compile): $(toolchain/stamp-compile) $(tools/stamp-compile) $(BUILD_DIR)/.prepared
$(package/stamp-compile): $(target/stamp-compile) $(package/stamp-cleanup)
$(package/stamp-install): $(package/stamp-compile)
$(target/stamp-install): $(package/stamp-compile) $(package/stamp-install)

# Validation targets
check: $(tools/stamp-check) $(toolchain/stamp-check) $(package/stamp-check)
printdb: ; @true

# Core build targets
prepare: $(target/stamp-compile)

# Clean operations
_clean: FORCE
	rm -rf $(BUILD_DIR) $(STAGING_DIR) $(BIN_DIR) \
	       $(OUTPUT_DIR)/packages/$(ARCH_PACKAGES) \
	       $(TOPDIR)/staging_dir/packages

clean: _clean
	rm -rf $(BUILD_LOG_DIR)

targetclean: _clean
	rm -rf $(TOOLCHAIN_DIR) $(BUILD_DIR_BASE)/hostpkg $(BUILD_DIR_TOOLCHAIN)

dirclean: targetclean clean
	rm -rf $(STAGING_DIR_HOST) $(STAGING_DIR_HOSTPKG) $(BUILD_DIR_BASE)/host
	rm -rf $(TMP_DIR)
	$(MAKE) -C $(TOPDIR)/scripts/config clean

toolchain_rebuild_check:
	$(SCRIPT_DIR)/check-toolchain-clean.sh

cacheclean:
ifneq ($(CONFIG_CCACHE),)
	$(STAGING_DIR_HOST)/bin/ccache -C
endif

# Prerequisite handling
ifndef DUMP_TARGET_DB
$(BUILD_DIR)/.prepared: Makefile
	@mkdir -p $$(dirname $@)
	@touch $@

tmp/.prereq_packages: .config
	unset ERROR; \
	for package in $(sort $(prereq-y) $(prereq-m)); do \
		$(_SINGLE)$(NO_TRACE_MAKE) -s -r -C package/$$package prereq || ERROR=1; \
	done; \
	if [ -n "$$ERROR" ]; then \
		echo "Package prerequisite check failed."; \
		false; \
	fi
	touch $@
endif

prereq: $(target/stamp-prereq) tmp/.prereq_packages
	@if [ ! -f "$(INCLUDE_DIR)/site/$(ARCH)" ]; then \
		echo 'ERROR: Missing architecture configuration for "$(ARCH)"'; \
		echo '       Required file: $(INCLUDE_DIR)/site/$(ARCH)'; \
		exit 1; \
	fi

# Build metadata generation
$(BIN_DIR)/profiles.json: FORCE
	$(if $(CONFIG_JSON_OVERVIEW_IMAGE_INFO), \
		WORK_DIR=$(BUILD_DIR)/json_info_files \
			$(SCRIPT_DIR)/json_overview_image_info.py $@ \
	)

json_overview_image_info: $(BIN_DIR)/profiles.json

checksum: FORCE
	$(call sha256sums,$(BIN_DIR),$(CONFIG_BUILDBOT))

buildversion: FORCE
	$(SCRIPT_DIR)/getver.sh > $(BIN_DIR)/version.buildinfo

feedsversion: FORCE
	$(SCRIPT_DIR)/feeds list -fs > $(BIN_DIR)/feeds.buildinfo

diffconfig: FORCE
	mkdir -p $(BIN_DIR)
	$(SCRIPT_DIR)/diffconfig.sh > $(BIN_DIR)/config.buildinfo

buildinfo: FORCE
	$(_SINGLE)$(SUBMAKE) -r diffconfig buildversion feedsversion

# Full build sequence
prepare: .config $(tools/stamp-compile) $(toolchain/stamp-compile)
	$(_SINGLE)$(SUBMAKE) -r buildinfo

world: prepare $(target/stamp-compile) $(package/stamp-compile) \
       $(package/stamp-install) $(target/stamp-install) FORCE
	$(_SINGLE)$(SUBMAKE) -r package/index
	$(_SINGLE)$(SUBMAKE) -r json_overview_image_info
	$(_SINGLE)$(SUBMAKE) -r checksum
ifneq ($(CONFIG_CCACHE),)
	$(STAGING_DIR_HOST)/bin/ccache -s
endif

.PHONY: clean dirclean prereq prepare world package/symlinks \
        package/symlinks-install package/symlinks-clean \
        check printdb _clean targetclean cacheclean \
        json_overview_image_info checksum buildversion feedsversion \
        diffconfig buildinfo FORCE

endif

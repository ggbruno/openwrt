#
# Copyright (C) 2006-2010 OpenWrt.org
#
# This is free software, licensed under the GNU General Public License v2.
# See /LICENSE for more information.
#
include $(TOPDIR)/rules.mk
include $(INCLUDE_DIR)/image.mk

DEVICE_VARS += IMAGE_SIZE CVIMG_KERNEL_BURN_ADDR CVIMG_LOADER_START_ADDR CVIMG_ROOTFS_BURN_ADDR DEV_PROFILE


define Build/loader-common
  rm -rf $@.src
  $(MAKE) -C lzma-loader \
     PKG_BUILD_DIR="$@.src" \
     TARGET_DIR="$(dir $@)" LOADER_NAME="$(notdir $@)" \
     BOARD="$(DEV_PROFILE)" \
     SUBTARGET="$(SUBTARGET)" \
     LZMA_TEXT_START=0x81000000 LOADADDR=0x80000000 \
     $(1) compile loader.bin
  mv "$@.bin" "$@"
  rm -rf $@.src
endef

define Build/loader-cmdline-compile
  $(call Build/loader-common,LOADER_DATA="$@" KERNEL_CMDLINE="$(CMDLINE)")
endef

define Image/Prepare/Profile/$(1)/squashfs
	$(call Image/BuildLoader,loader-$(SUBTARGET)-$(1),bin,$(call mkcmdline,$(1),$(2),$(3),$(6)),$(4),vmlinux.bin.lzma)
endef

define Image/Prepare
	lzma e $(KDIR)/vmlinux -lc1 -lp2 -pb2 $(KDIR)/vmlinux.bin.lzma
	$(call Image/Prepare/Profile/$(PROFILE)/squashfs)
	$(if $(CONFIG_TARGET_ROOTFS_INITRAMFS), $(call Image/Prepare/Profile/$(PROFILE)/initramfs))
	$(if $(CONFIG_RTK_SDCARD_IMG), $(call Image/Prepare/Profile/$(PROFILE)/sdcard))
endef

define Build/cvimg
        $(STAGING_DIR_HOST)/bin/cvimg \
                -i $@ \
                -o $@.new \
                $(1) && mv $@.new $@ || rm -f $@
endef

define Build/cvimg-kernel
        $(call Build/cvimg, \
                -e $(CVIMG_LOADER_START_ADDR) \
                -b $(CVIMG_KERNEL_BURN_ADDR) \
                -c new \
                -t kernel \
                $(1))
endef

define Build/cvimg-fw
        $(call Build/cvimg, \
                -e $(CVIMG_LOADER_START_ADDR) \
                -b $(CVIMG_KERNEL_BURN_ADDR) \
                -c new \
                -t fw \
                $(1))
endef

define Build/cvimg-custom
        $(call Build/cvimg, \
                -e $(CVIMG_LOADER_START_ADDR) \
                -b $(CVIMG_KERNEL_BURN_ADDR) \
                -s $(CVIMG_CUSTOM_SIGNATURE) \
                $(1))
endef

define Build/cvimg-rootfs
        $(call Build/cvimg, \
                -e $(CVIMG_ROOTFS_BURN_ADDR) \
                -b $(CVIMG_ROOTFS_BURN_ADDR) \
                -c new \
                -t rootfs \
                $(1))
endef

define Build/append-cvimg-rootfs
        cp $(IMAGE_ROOTFS) $(IMAGE_ROOTFS).new && \
        $(STAGING_DIR_HOST)/bin/padjffs2 $(IMAGE_ROOTFS).new \
                $(if $(BLOCKSIZE),$(BLOCKSIZE:%k=%),4 8 16 64 128 256) && \
        $(STAGING_DIR_HOST)/bin/cvimg \
                -i $(IMAGE_ROOTFS).new \
                -o $(IMAGE_ROOTFS).new \
                -e $(CVIMG_ROOTFS_BURN_ADDR) \
                -b $(CVIMG_ROOTFS_BURN_ADDR) \
                -c new \
                -t rootfs \
                $(1) && (dd if=$(IMAGE_ROOTFS).new >> $@ && \
                        rm -f $(IMAGE_ROOTFS).new) || rm -f $@ $(IMAGE_ROOTFS).new
endef

define Build/append-fake-rootfs
        $(call Build/cvimg, \
                -a $(BLOCKSIZE) \
                -f)
endef

#define Image/Build/Profile/$(1)/squashfs
#	cvimg-rtl8197f linux $(KDIR)/loader-$(SUBTARGET)-$(1).bin $(BIN_DIR)/$(IMG_PREFIX)-$(1)-linux.bin $(4) $(5)
#	dd if=$(BIN_DIR)/$(IMG_PREFIX)-$(1)-linux.bin of=$(BIN_DIR)/$(IMG_PREFIX)-$(1)-linux_4k.bin bs=$(BlockSize) conv=sync
#	cat $(BIN_DIR)/$(IMG_PREFIX)-$(1)-linux_4k.bin $(KDIR)/root.squashfs-4k > $(BIN_DIR)/$(IMG_PREFIX)-$(1)-fw_4k_cat.bin
#	cvimg-rtl8197f fix_chksum  $(BIN_DIR)/$(IMG_PREFIX)-$(1)-fw_4k_cat.bin $(BIN_DIR)/$(IMG_PREFIX)-$(1)-squashfs-sysupgrade.bin
#	rm -rf $(BIN_DIR)/$(IMG_PREFIX)-$(1)-linux_4k.bin $(BIN_DIR)/$(IMG_PREFIX)-$(1)-fw_4k_cat.bin $(BIN_DIR)/$(IMG_PREFIX)-$(1)-linux.bin
#endef

define Build/cvimg-pad-rootfs
        $(call Build/cvimg, \
                -b $(CVIMG_KERNEL_BURN_ADDR) \
                -c new \
                -a $(BLOCKSIZE) \
                -t rootfs \
                -j)
endef

define Build/tbs_dlink_big
  $(STAGING_DIR_HOST)/bin/tbs_dlink -b $(1) $@
endef

define Device/Default
  BLOCKSIZE := 4k
  PROFILES := Default $$(DEV_PROFILE)
  CVIMG_ROOTFS_BURN_ADDR := 0
endef

define Device/RealtekDTS
  PROFILES = Default $$(DTS)
  DEVICE_DTS_DIR := ../dts
  KERNEL := kernel-bin | append-dtb | lzma
endef



ifeq ($(SUBTARGET),rtl8197f)
define Device/RE708
  $(Device/RealtekDTS)
  BLOCKSIZE := 64k
  DEVICE_DTS := RE708
  DEVICE_TITLE := Multilaser RE708 V1
  SUPPORTED_DEVICES := re708v1 re708-v1
  IMAGE_SIZE := 7936k
  DEV_PROFILE := RE708
  CMDLINE := console=ttyS0,115200
  KERNEL := kernel-bin | append-dtb | lzma | loader-cmdline-compile
  CVIMG_KERNEL_BURN_ADDR := 0x80000
  CVIMG_LOADER_START_ADDR := 0x81000000
  IMAGES := factory.bin sysupgrade.bin
  IMAGE/factory.bin = append-kernel | append-fake-rootfs | append-rootfs | cvimg-kernel | cvimg-pad-rootfs
  IMAGE/sysupgrade.bin = append-kernel | append-fake-rootfs | append-rootfs | cvimg-kernel | pad-rootfs
  $(eval $(call SingleProfile,RE708,ttyS0,115200,0x84000000,80000,20000))
endef
TARGET_DEVICES += RE708

endif


define Image/BuildLoader
	$(LOADER_MAKE) LOADER=$(1).$(2) KERNEL_CMDLINE="$(3)" \
		LZMA_TEXT_START=$(4) LOADADDR=0x80000000 \
		LOADER_DATA="$(KDIR)/$(5)" BOARD="$(1)" \
		compile loader.$(2)
endef

define Image/Build/Initramfs
        $(call Image/Build/Profile/$(PROFILE),initramfs)
endef

#define Image/Build
#	dd if=$(KDIR)/root.squashfs of=$(KDIR)/root.squashfs-4k bs=$(BlockSize) conv=sync
#	$(call add_jffs2_mark,$(KDIR)/root.squashfs-4k)
#	$(call Image/Build/Profile/$(PROFILE)/squashfs)
#ifeq ($(CONFIG_RTK_SDCARD_IMG),y)
#	$(call Image/Build/Profile/$(PROFILE)/sdcard)
#endif
#endef

$(eval $(call BuildImage))

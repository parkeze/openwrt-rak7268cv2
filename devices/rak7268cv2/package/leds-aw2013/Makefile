#
# The AW2013 driver from Linux 6.12 with hardware breathing (hw_pattern)
# added, built out of tree. Built in tree it would put CONFIG_LEDS_AW2013 in
# the kernel config, which changes the vermagic and stops every release kmod
# from installing; out of tree it changes nothing. Upstream OpenWrt gets it as
# an ordinary kernel package, so this goes once a release has that.
#

include $(TOPDIR)/rules.mk
include $(INCLUDE_DIR)/kernel.mk

PKG_NAME:=leds-aw2013
PKG_RELEASE:=1
PKG_LICENSE:=GPL-2.0-or-later

include $(INCLUDE_DIR)/package.mk

define KernelPackage/leds-aw2013
  SUBMENU:=LED modules
  TITLE:=Awinic AW2013 LED support
  DEPENDS:=+kmod-i2c-core +kmod-regmap-i2c
  FILES:=$(PKG_BUILD_DIR)/leds-aw2013.ko
  AUTOLOAD:=$(call AutoProbe,leds-aw2013)
endef

define KernelPackage/leds-aw2013/description
  The Awinic AW2013 three-channel I2C LED driver, with the pattern
  trigger's hw_pattern driving the chip's own breathing.
endef

define Build/Compile
	$(KERNEL_MAKE) M="$(PKG_BUILD_DIR)" modules
endef

$(eval $(call KernelPackage,leds-aw2013))

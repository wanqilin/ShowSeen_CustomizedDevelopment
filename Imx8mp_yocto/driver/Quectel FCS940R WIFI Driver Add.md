# Quectel FCS940R Wifi Driver Porting

## 一、Development environment

Hardware Environment：i.MX8MP

Software Environment：yocto 4.0.22 kirkstone kernel 5.15

driver code：fcs940r-master.zip

## 二、Porting step

1.  Step 1: In this path (**/sources/meta-boundary/recipes-bsp**) create a new folder **fcs940r** According to the following directory to put the driver code.



    **tree -L 3**


    ```bash
         .
    ├── fcs940r-driver.bb
    └── files
        ├── code
        │   ├── clean
        │   ├── core
        │   ├── hal
        │   ├── ifcfg-wlan0
        │   ├── include
        │   ├── Kconfig
        │   ├── Makefile
        │   ├── os_dep
        │   ├── platform
        │   ├── runwpa
        │   └── wlan0dhcp
        └── wifi
            └── txpower

    ```

2.  step 2、Makefile Modify


    ```shell
    ifeq ($(CONFIG_PLATFORM_ARM_RK3188), y)
    EXTRA_CFLAGS += -DCONFIG_LITTLE_ENDIAN -DCONFIG_PLATFORM_ANDROID -DCONFIG_PLATFORM_ROCKCHIPS
    # default setting for Android 4.1, 4.2, 4.3, 4.4
    EXTRA_CFLAGS += -DCONFIG_IOCTL_CFG80211 -DRTW_USE_CFG80211_STA_EVENT
    EXTRA_CFLAGS += -DCONFIG_CONCURRENT_MODE
    EXTRA_CFLAGS += -DCONFIG_RTW_IOCTL_SET_COUNTRY
    EXTRA_CFLAGS += -DCONFIG_RTW_HOSTAPD_ACS
    # default setting for Power control
    #EXTRA_CFLAGS += -DRTW_ENABLE_WIFI_CONTROL_FUNC
    ifeq ($(CONFIG_SDIO_HCI), y)
    EXTRA_CFLAGS += -DRTW_SUPPORT_PLATFORM_SHUTDOWN
    endif
    # default setting for Special function
    ARCH := arm64   // * Added Notice The 64-bit toolchain must be changed to 64-bit
    #CROSS_COMPILE := /home/android_sdk/Rockchip/Rk3188/prebuilts/gcc/linux-x86/arm/arm-eabi-4.6/bin/arm-eabi-
    KSRC := /mnt/disk2/nxp/boundarydeviceEx/build/tmp/work-shared/nitrogen8mp/kernel-source  // *Modify kernel source code path
    MODULE_NAME := 8723ds    // *Add module FCS940R module: 8723ds
    endif
    ```

3.  step 3、Create the fcs940r-driver.bb configuration file

    &#x20;     Note: Package the files under txpower to /lib/firmware.

    &#x20;                1. Add source path SRC\_URI ="file://wifi/txpower"

    &#x20;                2.Add `FILES:${PN} += "{nonarch_base_libdir}/firmware"`  Pack it into Image

    &#x20;                &#x20;
    ```shell
    SUMMARY = "Example of how to build an external Linux kernel module"
    LICENSE = "CLOSED"

    inherit module

    SRC_URI = "file://code \
               file://wifi/txpower "

    SRCREV = "${AUTOREV}"

    S = "${WORKDIR}"
    PARALLEL_MAKE = "-j 8"

    INSANE_SKIP:${PN} += "installed-vs-shipped"
    do_configure () {
    bbnote skip do_configure
    }
    do_compile() {
    oe_runmake -C ${WORKDIR}/code
    }

    do_install() {
    	install -d ${D}${base_sbindir}
    	install -d ${D}${nonarch_base_libdir}/modules
    	install -d ${D}${nonarch_base_libdir}/modules/extra/
    	install -d ${D}${nonarch_base_libdir}/firmware/
    	install -m 755 ${WORKDIR}/code/*.ko ${D}${nonarch_base_libdir}/modules/extra/
    	install -m 755 ${WORKDIR}/wifi/txpower/PHY_REG_PG.txt ${D}${nonarch_base_libdir}/firmware/
    	install -m 755 ${WORKDIR}/wifi/txpower/TXPWR_LMT.txt ${D}${nonarch_base_libdir}/firmware/

    }
    # The inherit of module.bbclass will automatically name module packages with
    # "kernel-module-" prefix as required by the oe-core build environment.
    RPROVIDES_${PN} += "kernel-module-fcs940r-driver"

    FILES:${PN} += "${nonarch_base_libdir}/firmware"
    ```

4.  step 4、Add module imx-image-multimedia.bb 中

    &#x20;  `IMAGE_INSTALL += "fcs940r-driver "`&#x20;


5.  step 5、modify kernel .bb config。

    1.git add  // modify option

    2.git commit -m "xxx" 

    3.git format-patch HEAD^ 
    ```shell
    From 99b541093284bd22a3a10ccd0b13491268431709 Mon Sep 17 00:00:00 2001
    From: wanqilin <ddswanqilin@163.com>
    Date: Fri, 25 Oct 2024 00:06:12 +0800
    Subject: [PATCH] add FCS940R wifi

    ---
     arch/arm64/configs/imx_v8_defconfig | 7 ++++++-
     1 file changed, 6 insertions(+), 1 deletion(-)

    diff --git a/arch/arm64/configs/imx_v8_defconfig b/arch/arm64/configs/imx_v8_defconfig
    index 97dc633d7800..6de528e31006 100644
    --- a/arch/arm64/configs/imx_v8_defconfig
    +++ b/arch/arm64/configs/imx_v8_defconfig
    @@ -170,7 +170,7 @@ CONFIG_BT_HCIUART_3WIRE=y
     CONFIG_BT_HCIUART_BCM=y
     CONFIG_BT_HCIUART_QCA=y
     CONFIG_BT_HCIVHCI=y
    -CONFIG_CFG80211=y
    +CONFIG_CFG80211=m
     CONFIG_NL80211_TESTMODE=y
     CONFIG_CFG80211_WEXT=y
     CONFIG_MAC80211=y
    @@ -1029,6 +1029,11 @@ CONFIG_CRYPTO_USER_API_HASH=m
     CONFIG_CRYPTO_USER_API_SKCIPHER=m
     CONFIG_CRYPTO_USER_API_RNG=m
     CONFIG_CRYPTO_USER_API_AEAD=m
    +CONFIG_ASYMMETRIC_KEY_TYPE=y
    +CONFIG_ASYMMETRIC_PUBLIC_KEY_SUBTYPE=y
    +CONFIG_X509_CERTIFICATE_PARSER=y
    +CONFIG_PKCS7_MESSAGE_PARSER=y
    +CONFIG_SYSTEM_DATA_VERIFICATION=y
     CONFIG_CRYPTO_DEV_FSL_CAAM_SECVIO=m
     CONFIG_CRYPTO_DEV_FSL_CAAM=m
     CONFIG_CRYPTO_DEV_FSL_CAAM_SM_TEST=m
    -- 
    2.34.1
    ```

6.  step 6 、Add kernel patch

    &#x20; 1.copy the kernel patch generated in the previous step to "/source/meta-boundary/recipes-kernel/linux/linux-boundary"

    &#x20; 2.Add or modify under this path "/source/meta-boundary/recipes-kernel/linux/linux-boundary\_%.bbappend"

```shell
COMPATIBLE_MACHINE = "(nitrogen6x|nitrogen6x-lite|nitrogen6sx|nitrogen7|nitrogen8m|nitrogen8mm|nitrogen8mn|nitrogen8mp|nitrogen8ulp)"

# In case of 8mp, kernel-module-isp-vvcam will build and cause the following error:
# The recipe linux-boundary is trying to install files into a shared area when those files already exist (kernel-module-imx219)
# So we need to remove config from kernel to avoid error.
EXTRA_OEMAKE:append:mx8mp-nxp-bsp = " CONFIG_VIDEO_IMX219=n"

FILESEXTRAPATHS:prepend := "${THISDIR}/${PN}:"

SRC_URI += "file://0001-add-FCS940R-wifi.patch"
```




# Quectel FCS940R BT Driver Porting

## 一、开发环境

硬件平台：i.MX8MP

软件平台：yocto 4.0.22 kirkstone kernel 5.15

驱动：fcs940r-master

## 二、移植步骤

1.  第一步、在这个路径下(**/sources/meta-boundary/recipes-bsp**)新建一个文件夹 **bt-fcs940r**按下面的目录放驱动相关代码。

    **tree -L 3**

    ```bash
    /sources/meta-boundary/recipes-bsp$ tree -L 3
    .
    ├── bt-fcs940r
    │   ├── bt-fcs940r-driver.bb
    │   └── files
    │       ├── driver
    │       └── FW


    ```

2.  第二步、\bt-fcs940r\files\driver\uart\bluetooth\_uart\_driver\Makefile修改

3.

        #ifneq ($(KERNELRELEASE),)
              obj-m		:= hci_uart.o
              hci_uart-y	:= hci_ldisc.o hci_h4.o hci_rtk_h5.o rtk_coex.o
              #EXTRA_CFLAGS += -DDEBUG
        #else
              PWD := (shell uname -r)
              KVER := $(shell uname -r)
              # KDIR := /lib/modules/$(KVER)/build
              KDIR := $(KBUILD_OUTPUT) #yocto imx.8 support
        all:
              echo $(KVER)
              $(MAKE) -C $(KDIR) M=$(PWD) modules
        clean:
              rm -rf *.o *.mod.c *.mod.o *.ko *.symvers *.order *.a
        #endif

&#x20;   \files\driver\uart\rtk\_hciattach\Makefile

    CFLAGS := -Wall -g
    #CC := $(CROSS_COMPILE)gcc
    all: rtk_hciattach
    OBJS := hciattach.o hciattach_rtk.o hciattach_h4.o rtb_fwc.o

    rtk_hciattach: $(OBJS)
    	$(CC) -o rtk_hciattach $(OBJS)

    %.o: %.c
    	$(CC) -c $< -o $@ $(CFLAGS)

    clean:
    	rm -f $(OBJS)  rtk_hciattach

    tags: FORCE
    	ctags -R
    	find ./ -name "*.h" -o -name "*.c" -o -name "*.cc" -o -name "*.cpp" > cscope.files
    	cscope -bkq -i cscope.files
    PHONY += FORCE
    FORCE:	

1.  第三步、创建\sources\meta-boundary\recipes-bsp\bt-fcs940r\bt-fcs940r-driver.bb 配置文件。

    &#x20;                &#x20;

    ```shell
    SUMMARY = "Example of how to build an external Linux kernel module"
    LICENSE = "CLOSED"

    inherit module

    SRC_URI = "file://driver \
               file://FW"

    SRCREV ?= "${AUTOREV}"

    S = "${WORKDIR}/driver/uart/bluetooth_uart_driver"
    S1 = "${WORKDIR}/driver/uart/rtk_hciattach"

    EXTRA_OEMAKE += 'KDIR:=${KBUILD_OUTPUT}'
    EXTRA_OEMAKE += 'KVER="${KERNEL_VERSION}"'

    TARGET_CC_ARCH += "${LDFLAGS}"
    INSANE_SKIP_${PN} = "ldflags"
    INSANE_SKIP_${PN}-dev = "ldflags"
    PARALLEL_MAKE = "-j 8"

    FILESEXTRAPATHS:prepend := "${THISDIR}/${BPN}:"

    do_compile() {
        cd ${S}
        oe_runmake

        cd ${S1}
        oe_runmake
    }

    do_install () {
        install -d ${D}${nonarch_base_libdir}/fcs940r_modules
        cp --no-preserve=ownership ${S}/hci_uart.ko ${D}${nonarch_base_libdir}/fcs940r_modules
        install -m 0644 ${S}/hci_uart.ko ${D}${nonarch_base_libdir}/fcs940r_modules/

        install -d ${D}${sbindir}
        install -m 0755 ${S1}/rtk_hciattach ${D}${sbindir}

        install -d ${D}${nonarch_base_libdir}/firmware/rtlbt
        install -m 755 ${WORKDIR}/FW/rtl8723d_config ${D}${nonarch_base_libdir}/firmware/rtlbt
        install -m 755 ${WORKDIR}/FW/rtl8723d_fw ${D}${nonarch_base_libdir}/firmware/rtlbt
    }
    # The inherit of module.bbclass will automatically name module packages with
    # "kernel-module-" prefix as required by the oe-core build environment.

    # RPROVIDES_${PN} += "kernel-module-hci-uart"

    FILES:${PN} += "${sbindir}"
    FILES:${PN} += "${nonarch_base_libdir}/fcs940r_modules/*"
    FILES:${PN} += "${nonarch_base_libdir}/firmware/rtlbt"

    ```

2.

3.  第四步、修改kernel配置文件。

    1.git add 修改项

    2.git commit -m "xxx" 添加注释

    3.git format-patch HEAD^ 导出差异包

    ```shell
    From 81c868f651695433d84a4f4405bb72af61ff3c97 Mon Sep 17 00:00:00 2001
    From: wanqilin <ddswanqilin@163.com>
    Date: Wed, 6 Nov 2024 09:43:58 +0800
    Subject: [PATCH] add bt fcs940r

    ---
     arch/arm64/configs/imx_v8_defconfig | 8 ++++----
     1 file changed, 4 insertions(+), 4 deletions(-)

    diff --git a/arch/arm64/configs/imx_v8_defconfig b/arch/arm64/configs/imx_v8_defconfig
    index 6de528e31006..782a4ba2f226 100644
    --- a/arch/arm64/configs/imx_v8_defconfig
    +++ b/arch/arm64/configs/imx_v8_defconfig
    @@ -162,10 +162,10 @@ CONFIG_BT_HIDP=y
     CONFIG_BT_LEDS=y
     # CONFIG_BT_DEBUGFS is not set
     CONFIG_BT_HCIBTUSB=m
    -CONFIG_BT_HCIUART=y
    -CONFIG_BT_HCIUART_BCSP=y
    -CONFIG_BT_HCIUART_ATH3K=y
    -CONFIG_BT_HCIUART_LL=y
    +#CONFIG_BT_HCIUART=y
    +#CONFIG_BT_HCIUART_BCSP=y
    +#CONFIG_BT_HCIUART_ATH3K=y
    +#CONFIG_BT_HCIUART_LL=y
     CONFIG_BT_HCIUART_3WIRE=y
     CONFIG_BT_HCIUART_BCM=y
     CONFIG_BT_HCIUART_QCA=y
    -- 
    2.34.1
    ```

4.  第五步、添加内核补丁

    &#x20; 1.把上一步生成内核补丁copy到 /source/meta-boundary/recipes-kernel/linux/linux-boundary

    &#x20; 2.在这个路径下添加或修改/source/meta-boundary/recipes-kernel/linux/linux-boundary\_%.bbappend



        COMPATIBLE_MACHINE = "(nitrogen6x|nitrogen6x-lite|nitrogen6sx|nitrogen7|nitrogen8m|nitrogen8mm|nitrogen8mn|nitrogen8mp|nitrogen8ulp)"

        # In case of 8mp, kernel-module-isp-vvcam will build and cause the following error:
        # The recipe linux-boundary is trying to install files into a shared area when those files already exist (kernel-module-imx219)
        # So we need to remove config from kernel to avoid error.
        EXTRA_OEMAKE:append:mx8mp-nxp-bsp = " CONFIG_VIDEO_IMX219=n"

        FILESEXTRAPATHS:prepend := "{PN}:"
        SRC_URI += "file://0001-add-FCS940R-wifi.patch
                    file://0001-add-bt-fcs940r.patch"     

&#x20;  第六步 添加映象recipes-boundary/images/boundary-image-multimedia-full.bb   &#x20;

         IMAGE_INSTALL:append =  “bt-fcs940r-driver”


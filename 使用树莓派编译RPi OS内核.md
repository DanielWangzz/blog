# 使用树莓派编译RPi OS内核
##  前期准备
1. 准备一块安装有最新版Raspberry Pi OS的树莓派，开机并联网
2. 安装Git并建立依赖
`sudo apt install git bc bison flex libssl-dev make`
3. 获取linux源码
`git clone --depth=1 https://github.com/raspberrypi/linux`
## 交叉编译
### 所需的依赖和工具链
#### 安装依赖
`sudo apt install git bc bison flex libssl-dev make libc6-dev libncurses5-dev`
#### 为32位内核安装32位工具链
`sudo apt install crossbuild-essential-armhf`
#### 为64位内核安装64位工具链
`sudo apt install crossbuild-essential-arm64`
## 内核配置
### 应用默认配置
+ Raspberry Pi 1, Pi Zero, Pi Zero W, and Compute Module

```
cd linux
KERNEL=kernel
make bcmrpi_defconfig
```
+ Raspberry Pi 2, Pi 3, Pi 3+, and Compute Module 3 
```
cd linux
KERNEL=kernel7
make bcm2709_defconfig
```
+ Raspberry Pi 4 default build 
```
cd linux
KERNEL=kernel7l
make bcm2711_defconfig
```
### 使用LOCALVERSION自定义内核版本
除了更改内核配置外，您可能还希望调整`LOCALVERSION`以确保新内核不会收到与上游内核相同的版本字符串。这既使您在`uname`的输出中运行自己的内核，又确保`/lib/modules`中的现有模块不会被覆盖。
`sudo nano .config`

修改`CONFIG_LOCALVERSION="-v7l-MY_CUSTOM_KERNEL"`
### MENUCONFIG图形化配置内核
#### 安装ncurses开发包
`sudo apt install libncurses5-dev`
#### 使用menuconfig
`make menuconfig`
+ 交叉编译32位内核
`make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- menuconfig`
+ 交叉编译64位内核
`make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- menuconfig`
### 修改.config配置内核配置
`sudo nano .config`
### 编译defconfig
#### 32-bit configs
+ For Pi 1, Pi Zero, Pi Zero W, or Compute Module:
```
cd linux
KERNEL=kernel
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- bcmrpi_defconfig
```
+ For Pi 2, Pi 3, Pi 3+, or Compute Module 3:
```
cd linux
KERNEL=kernel7
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- bcm2709_defconfig
```
+ For Raspberry Pi 4:
```
cd linux
KERNEL=kernel7l
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- bcm2711_defconfig
```
#### 64-bit configs
+ For Pi 3, Pi 3+ or Compute Module 3:
```
cd linux
KERNEL=kernel8
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- bcmrpi3_defconfig
```
+ For Raspberry Pi 4:
```
cd linux
KERNEL=kernel8
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- bcm2711_defconfig
```
## 编译Kernel, Modules, and DTBs
编译kernel, modules, and DTBs;根据使用的Pi型号，此步骤可能需要很长时间：
```
make -j4 zImage modules dtbs
sudo make modules_install
sudo cp arch/arm/boot/dts/*.dtb /boot/
sudo cp arch/arm/boot/dts/overlays/*.dtb* /boot/overlays/
sudo cp arch/arm/boot/dts/overlays/README /boot/overlays/
sudo cp arch/arm/boot/zImage /boot/$KERNEL.img
```
## 安装 Modules
首先，在插入SD卡之前和之后使用`lsblk`进行识别。应该看到以下内容：
```
sdb
   sdb1
   sdb2
```
sdb1是FAT（引导）分区，而sdb2是ext4文件系统（根）分区。
如果是NOOBS卡，则应该看到以下内容：
```
sdb
  sdb1
  sdb2
  sdb5
  sdb6
  sdb7
```
sdb6是FAT（引导）分区，而sdb7是ext4文件系统（根）分区。
先挂载他们，并根据需要调整NOOBS卡的分区号：
```
mkdir mnt
mkdir mnt/fat32
mkdir mnt/ext4
sudo mount /dev/sdb6 mnt/fat32
sudo mount /dev/sdb7 mnt/ext4
```
接下来，将内核模块安装到SD卡上：
+ For 32-bit
`sudo env PATH=$PATH make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- INSTALL_MOD_PATH=mnt/ext4 modules_install`
+ For 64-bit
`sudo env PATH=$PATH make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- INSTALL_MOD_PATH=mnt/ext4 modules_install`
最后，将内核和设备树DTBs复制到SD卡上，确保备份旧内核。
+ For 32-bit
```
sudo cp mnt/fat32/$KERNEL.img mnt/fat32/$KERNEL-backup.img
sudo cp arch/arm/boot/zImage mnt/fat32/$KERNEL.img
sudo cp arch/arm/boot/dts/*.dtb mnt/fat32/
sudo cp arch/arm/boot/dts/overlays/*.dtb* mnt/fat32/overlays/
sudo cp arch/arm/boot/dts/overlays/README mnt/fat32/overlays/
sudo umount mnt/fat32
sudo umount mnt/ext4
```
+ For 64-bit
```
sudo cp mnt/fat32/$KERNEL.img mnt/fat32/$KERNEL-backup.img
sudo cp arch/arm64/boot/Image mnt/fat32/$KERNEL.img
sudo cp arch/arm64/boot/dts/broadcom/*.dtb mnt/fat32/
sudo cp arch/arm64/boot/dts/overlays/*.dtb* mnt/fat32/overlays/
sudo cp arch/arm64/boot/dts/overlays/README mnt/fat32/overlays/
sudo umount mnt/fat32
sudo umount mnt/ext4
```
另一种选择是将内核复制到同一位置，但是使用不同的文件名（例如kernel-myconfig.img），而不是覆盖kernel.img文件。 然后，您可以编辑config.txt文件选择想让Pi启动进入的内核：
`kernel=kernel-myconfig.img`
这样做的好处是可以使内核与系统和任何自动更新工具管理的内核映像分开，并允许您在内核无法启动的情况下轻松地恢复到普通内核。 最后，将卡插入Pi并启动！
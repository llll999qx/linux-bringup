
之前查到的 SD 卡第一个分区从扇区 `225280` 开始：

```
225280 × 512 字节 = 110 MiB
```
也就是说，当时 SD 卡的 **0～110 MiB 都在第一个分区之前**。我们计划使用的区域只到 42 MiB，因此**按照之前那次的分区表，暂时不需要重新分区**；原有的三个分区可以保留。
```bash
lsblk -o NAME,SIZE,MODEL,TYPE,MOUNTPOINTS
```

### 第一步：重新查看 SD 卡

把 SD 卡连接到 Ubuntu 虚拟机，在 **Ubuntu 终端**运行

**1.查看磁盘和挂载情况**
```bash
lsblk -o NAME,SIZE,MODEL,TYPE,MOUNTPOINTS
```
lsblk       列出系统识别到的块设备，例如硬盘、SD 卡
-o          指定要显示哪些列
NAME        设备名称
SIZE        大小
MODEL       设备型号
TYPE        类型，例如磁盘 disk、分区 part
MOUNTPOINTS 挂载到哪里

**2.fdisk -l：查看分区的具体位置**
```bash
sudo fdisk -l /dev/sdb
```

sudo       以管理员权限执行
fdisk      磁盘分区工具
-l         列出信息，这里不会修改分区
**/dev/sdb**   指定查看整张 SD 卡

| 设备        | 启动  | 起点扇区    | 末尾扇区    | 扇区数量    | 大小   | Id  | 类型              |
| --------- | --- | ------- | ------- | ------- | ---- | --- | --------------- |
| /dev/sdb1 |     | 225280  | 245759  | 20480   | 10M  | c   | W95 FAT32 (LBA) |
| /dev/sdb2 |     | 245760  | 1269759 | 1024000 | 500M | c   | W95 FAT32 (LBA) |
| /dev/sdb3 |     | 2097152 | 7744510 | 5647359 | 2.7G | c   | W95 FAT32 (LBA) |
起点 225280 扇区 × 512 字节 = 115343360 字节 = 110 MiB（磁盘**前面先空**出来 110MiB，**从这个位置才开始 sdb1**）

### 第二步：准备写入的三个文件有多大

1.
```bash
stat -c '%n：%s 字节' \
  ~/imx6ull/build-kernel/arch/arm/boot/zImage \
  ~/imx6ull/build-kernel/arch/arm/boot/dts/nxp/imx/imx6ull-14x14-evk-emmc.dtb \
  ~/imx6ull/rootfs.cpio.gz
```
xq999@xq999-virtual-machine:~$ stat -c '%n：%s 字节' \
  ~/imx6ull/build-kernel/arch/arm/boot/zImage \
  ~/imx6ull/build-kernel/arch/arm/boot/dts/nxp/imx/imx6ull-14x14-evk-emmc.dtb \
  ~/imx6ull/rootfs.cpio.gz
  
/home/xq999/imx6ull/build-kernel/arch/arm/boot/zImage：9684416 字节
/home/xq999/imx6ull/build-kernel/arch/arm/boot/dts/nxp/imx/imx6ull-14x14-evk-emmc.dtb：45946 字节=45kb
/home/xq999/imx6ull/rootfs.cpio.gz：3181928 字节=3.03Mib

1 MiB = 1024 * 1024 = 1048576 字节
9684416 / 1048576 = 9.24Mib

2.查看配置项
```bash
grep -E '^(CONFIG_ENV_(IS_IN|OFFSET|SIZE|SECT_SIZE|MMC)|CONFIG_SYS_MMC_ENV)' ~/imx6ull/build-uboot-imx/.config
```

| 配置项 | 作用 |
| ---- | ---- |
| CONFIG_ENV_IS_IN_MMC | 环境变量存放在 MMC（SD/eMMC） |
| CONFIG_ENV_OFFSET | env 在 MMC 里的偏移地址 |
| CONFIG_ENV_SIZE | env 分区占用多大 |
| CONFIG_ENV_SECT_SIZE | flash 扇区大小 |
| CONFIG_ENV_MMC、CONFIG_SYS_MMC_ENV_DEV | 指定用哪一个 MMC 设备 |
CONFIG_ENV_SIZE=0x2000         大小 8 KiB
CONFIG_ENV_OFFSET=0xE0000  偏移 896 KiB
CONFIG_ENV_IS_IN_MMC=y      
CONFIG_ENV_MMC_DEVICE_INDEX=1   配置默认指向 MMC 设备 1，而你此前确认 SD 卡是设备 0
CONFIG_ENV_MMC_EMMC_HW_PARTITION=0  指定 U-Boot 环境变量**存放在 eMMC 哪一个硬件分区 (HW Partition)**

**1.先做备份**
SD 卡 /dev/sdb
│
├─ 0～511 字节        MBR（第 0 扇区）
├─ 512～1023 字节     第 1 扇区
├─ 1024 字节起        之前烧入的 u-boot.imx
│                     ……
└─ 到 42 MiB 为止
       │
       └── 把这一整段复制到 Ubuntu 中的一个文件
    1.
```bash
mkdir -p ~/imx6ull/sd-backup（创建备份文件夹在Ubuntu）
```

	2.
```bash
backup_file="$HOME/imx6ull/sd-backup/sd-first-42MiB-$(date +%Y%m%d-%H%M%S).img"（加时间是为了方便区分每次备份）
```

	3.从 SD 卡读取数据，保存为备份文件
```bash
sudo dd if=/dev/sdb of="$backup_file" bs=1M count=42 status=progress conv=fsync
```

|参数|意思|
|---|---|
|sudo|使用管理员权限读取磁盘|
|if=/dev/sdb|输入来源是 SD 卡|
|of="$backup_file"|输出到刚才命名的 Ubuntu 文件|
|bs=1M|每块按 1 MiB 处理|
|count=42|读取 42 块，即前 42 MiB|
|status=progress|显示复制进度|
|conv=fsync|结束前确保备份数据写入存储设备|
```
SD 卡前 42 MiB
包含 MBR、已有 U-Boot，以及后面准备使用的区域
        ↓ 复制
Ubuntu 中的 .img 备份文件
```

	4.检查备份内容是否与 SD 卡一致
```bash
sudo cmp -n 44040192 /dev/sdb "$backup_file" && echo "前 42 MiB 备份校验通过"
```

**2.写入设备树、内核、initramfs**
1.写入设备树
```bash
sudo dd \
  if="$HOME/imx6ull/build-kernel/arch/arm/boot/dts/nxp/imx/imx6ull-14x14-evk-emmc.dtb" \
  of=/dev/sdb bs=1M seek=1 conv=notrunc,fsync status=progress
```
- `conv=notrunc`：**不要截断 SD 卡**（不会清空卡后面已有的数据，很关键）
- `conv=fsync`：写完强制刷缓存，保证数据落盘
- `status=progress`：显示写入进度
**设备树对比**（ `-n 45946`：只对比 45946 字节（就是这个 dtb 文件的大小）
- `-i 0:1048576`：第一个文件从偏移 0 开始读，SD 卡 `/dev/sdb` 从偏移 1048576 字节（1MiB）开始读）
```bash
sudo cmp -n 45946 -i 0:1048576 \
  "$HOME/imx6ull/build-kernel/arch/arm/boot/dts/nxp/imx/imx6ull-14x14-evk-emmc.dtb" \
  /dev/sdb && echo "设备树校验通过"
```
2.写入内核
```bash
sudo dd \
  if="$HOME/imx6ull/build-kernel/arch/arm/boot/zImage" \
  of=/dev/sdb bs=1M seek=2 conv=notrunc,fsync status=progress
```
3.写入initramfs
```bash
sudo dd \
  if="$HOME/imx6ull/rootfs.cpio.gz" \
  of=/dev/sdb bs=1M seek=34 conv=notrunc,fsync status=progress
```

**3.在 Ubuntu 中安全断开 SD 卡**
```bash
sync（把系统缓存里还没写到 SD 卡的数据，强制全部刷到磁盘上，防止数据丢失）
sudo udisksctl power-off -b /dev/sdb（安全卸载 `/dev/sdb` 这张 SD 卡，断电断开，之后就可以直接拔卡）
```

1.启动uboot
=> mmc list
FSL_SDHC: 0 (SD)
FSL_SDHC: 1
查看有多少个设备

2.=>`mmc dev 0`：选择第 0 号 MMC 设备（SD 卡 /eMMC）作为当前操作的设备
=>`mmc rescan`：重新扫描 MMC 总线，识别卡、刷新分区信息，卡插拔后常用
=>`mmc info`：打印当前选中 MMC 设备信息，比如容量、版本、速度模式
(Device: FSL_SDHC
Manufacturer ID: 3
OEM: 5344
Name: SU04G
Bus Speed: 49500000
Mode: SD High Speed (50MHz)
Rd Block Len: 512
SD version 3.0
High Capacity: Yes
Capacity: 3.7 GiB
Bus Width: 4-bit
Erase Group Size: 512 Bytes)

```
芯片开始执行内部固化的 Boot ROM
↓
根据启动拨码选择 SD 卡
↓
在 SD 卡规定的位置读取 IVT 等启动信息
↓
根据镜像里的 DCD 配置初始化 DDR
↓
根据镜像里的加载地址、长度，把 U-Boot 加载到 DDR
↓
跳转到镜像指定的入口，执行 U-Boot
↓
U-Boot 初始化时，再把自己重定位到 DDR 的高地址区域
```
芯片手册规定“哪些地址是 DDR”，U-Boot 的板级配置决定“具体放在 DDR 的哪里”。
芯片和硬件：规定可以使用哪一块地方
U-Boot 板级配置：指定在这块地方的哪个位置放程序
Boot ROM：负责搬过去并开始执行

3.mmc read  DDR目标地址  SD卡起始扇区  读取扇区数量
```bash
mmc read 0x82000000 0x1000 0x49e3
```

```bash
mmc read 0x82000000 0x1000 0x49e3
```

```bash
mmc read 0x82000000 0x1000 0x49e3
```

| 内容        | SD 卡位置 | 起始扇区    | 读取扇区数  | DDR 地址     |
| --------- | ------ | ------- | ------ | ---------- |
| 内核        | 2 MiB  | 0x1000  | 0x49e3 | 0x82000000 |
| 设备树       | 1 MiB  | 0x800   | 0x5a   | 0x83000000 |
| initramfs | 34 MiB | 0x11000 | 0x1847 | 0x84000000 |
4.`bootargs` 是**传给 Linux 内核的启动参数**，内核启动时读取这串参数，决定：用哪个串口打印、怎么挂载根文件系统、启动日志、ramdisk 启动方式等。

```bash
setenv bootargs 'console=ttymxc0,115200 earlycon=ec_imx6q,0x02020000 rdinit=/init loglevel=8 ignore_loglevel'
```

| 参数                           | 说明                                                      |
| ---------------------------- | ------------------------------------------------------- |
| console=ttymxc0,115200       | 控制台串口：i.MX6ULL UART0，波特率 115200，内核打印、shell 走这个串口        |
| earlycon=ec_imx6q,0x02020000 | 早期串口，内核正式串口驱动还没跑起来的时候就输出 log，调试用，产品可以删掉                 |
| rdinit=/init                 | 当前 initramfs/cpio.gz 启动必须：内核解压完 ramfs 之后，第一个运行程序是 /init |
| loglevel=8                   | 内核日志级别，数字越大打印越多，8 是大量调试信息                               |
| ignore_loglevel              | 无视 loglevel，强制全部内核消息打印，调试用，产品可以删掉                       |

```bash
bootz 0x82000000 0x84000000:0x308d68 0x83000000
```
`0x308d68` 是 initramfs 的实际长度，即 **3181928 字节**





**从 eMMC 启动 Linux**

1.
```bash
mmc list
mmc dev 1
```

=> mmc list
FSL_SDHC: 0 (SD)
FSL_SDHC: 1
=> mmc dev 1
Card did not respond to voltage select! : -110
(mmc list 能列出设备 1
        ↓
说明 U-Boot 已经识别到对应的 MMC 控制器
        ↓
mmc dev 1 尝试初始化连接的存储设备
        ↓
等待设备响应超时，返回 -110)

2.
```bash
diff -u \
  configs/mx6ull_14x14_evk_defconfig \
  configs/mx6ull_14x14_evk_emmc_defconfig
```

| 配置            | 普通版本              | eMMC 版本                    |
| ------------- | ----------------- | -------------------------- |
| U-Boot 使用的设备树 | imx6ull-14x14-evk | imx6ull-14x14-evk-emmc     |
| eMMC 专用启动功能   | 没有显式开启            | CONFIG_SUPPORT_EMMC_BOOT=y |

**编译build-uboot-emmc-test为u-boot.imx**（`0x400` 指的是字节偏移，不是扇区编号。当时是把 `u-boot.imx` 写到整张 SD 卡距离开头 **1024 字节**的位置。）
1.
```bash
cd ~/imx6ull/imx6ull/uboot-imx
```
2.
```bash
export PATH="/usr/bin:/bin:$PATH"
export M4=/usr/bin/m4
hash -r
export CROSS_COMPILE="$HOME/toolchains/arm-gnu-toolchain-11.3.rel1-x86_64-arm-none-linux-gnueabihf/bin/arm-none-linux-gnueabihf-"
```
3.在一个新的输出目录中生成配置：
```bash
make ARCH=arm CROSS_COMPILE="$CROSS_COMPILE" \
  O="$HOME/imx6ull/build-uboot-emmc-test" \
  mx6ull_14x14_evk_emmc_defconfig
```
4.编译：
```bash
make ARCH=arm CROSS_COMPILE="$CROSS_COMPILE" \
  O="$HOME/imx6ull/build-uboot-emmc-test" \
  -j4 u-boot.imx
```
5.先在 Ubuntu 检查新镜像：
```bash
stat -c '%n：%s 字节' \
  ~/imx6ull/build-uboot-emmc-test/u-boot.imx
```
xq999@xq999-virtual-machine:~/imx6ull/imx6ull/uboot-imx$ stat -c '%n：%s 字节' \ 
  ~/imx6ull/build-uboot-emmc-test/u-boot.imx
/home/xq999/imx6ull/build-uboot-emmc-test/u-boot.imx：617472 字节
617472 ÷ 1024 = 603 KiB
起点：1 KiB（1024（0x400）÷ 1024 = 1 ）
结束边界：1 + 603 = 604 KiB
```
0 字节          MBR
1 KiB           新 u-boot.imx 开始
604 KiB         新镜像结束
                空闲间隔
896～904 KiB    预留的 U-Boot 环境变量位置
1 MiB           已写入的设备树
```

6.读取 `u-boot.imx` 最开头 4 个字节，以单字节十六进制打印出来**校验 IVT 头部的 magic 标识（0x402000D1）**
```bash
od -An -tx1 -N4 \
  ~/imx6ull/build-uboot-emmc-test/u-boot.imx
```
~/imx6ull/build-uboot-emmc-test/u-boot.imx  
d1 00 20 40（表示“以十六进制显示文件开头的 4 个字节”。结果符合这种 i.MX 启动镜像的 IVT 头格式，说明 **IVT 位于这个文件的开头**。）

7.在 Ubuntu 写入新镜像
```bash
sudo dd \
  if="$HOME/imx6ull/build-uboot-emmc-test/u-boot.imx" \
  of=/dev/sdb bs=1K seek=1 conv=notrunc,fsync status=progress
```
这会从 SD 卡的 **1 KiB（`0x400`）** 位置写入
```bash
sync
sudo udisksctl power-off -b /dev/sdb
```

8.（补充）
```bash
cd ~/imx6ull/imx6ull/uboot-imx
```

```
cp board/nxp/mx6ullevk/mx6ullevk.c \
   board/nxp/mx6ullevk/mx6ullevk.c.before-emmc-gpio
```
备份
```bash
nano board/nxp/mx6ullevk/mx6ullevk.c
```
进入
```
int board_init(void)
{
    int ret;

    /* Address of boot parameters */
    gd->bd->bi_boot_params = PHYS_SDRAM + 0x100;

    /* Configure NAND_ALE as GPIO4_IO10 for the eMMC test. */
    imx_iomux_v3_setup_pad(
        MX6_PAD_NAND_ALE__GPIO4_IO10 | MUX_PAD_CTRL(NO_PAD_CTRL));

    ret = gpio_request(IMX_GPIO_NR(4, 10), "emmc-control");
    if (ret) {
        printf("eMMC GPIO request failed: %d\n", ret);
        return ret;
    }

    ret = gpio_direction_output(IMX_GPIO_NR(4, 10), 1);
    if (ret) {
        printf("eMMC GPIO output setup failed: %d\n", ret);
        return ret;
    }
```
- `imx_iomux_v3_setup_pad(...)`：明确把 `NAND_ALE` 引脚复用成 GPIO4_IO10。
- `gpio_request(...)`：向 U-Boot 申请使用这根 GPIO。
- `gpio_direction_output(..., 1)`：设置为**输出高电平**，相当于之前手动执行的 `gpio set GPIO4_10`。

**1.然后重新编译**
**2.mmc dev 1**（选中 mmc0）
mmc info
**3.mmc part** （这条命令：查看当前选中的 mmc0 的分区表）
fatls mmc 1:1
(fatls     列出 FAT 文件系统中的文件
mmc 1:1   MMC 设备 1 的第 1 个分区)
**4.mmc dev 1**
mmc part
当前选择的是 eMMC

|分区|起始扇区|扇区数量|大小|类型|
|---|---|---|---|---|
|1|20480|262144|128 MiB|`0c`：FAT32 类型|
|2|282624|14987264|约 7.15 GiB|`83`：Linux 分区类型|

每个扇区是 **512 字节**。例如第一个分区的大小:
262144 × 512 ÷ 1024 ÷ 1024 = 128 MiB
它从 eMMC 的 **10 MiB** 位置开始:
20480 × 512 ÷ 1024 ÷ 1024 = 10 MiB

**5.=> fatls mmc 1:1**
CMD_SEND:16
                ARG                      0x00000200
                MMC_RSP_R1,5,6,7         0x00000900
                
CMD_SEND:17
                ARG                      0x00000000
                MMC_RSP_R1,5,6,7         0x00000900
**U-Boot：请读取 0x5000 位置的数据块。**
**eMMC：收到，我返回当前状态。**
( - `CMD_SEND:17`：发送读取单个数据块的命令。
- `ARG 0x00005000`：告诉 eMMC 要从哪个块位置读取。
- `MMC_RSP_R1,5,6,7`：调试程序用这个标签显示响应类型。**不表示同时收到了四份响应**
- `0x00000900`：eMMC 返回的状态值。它是按二进制位记录的，不是一段直接能读懂的文字。)
  6792136   zImage(`zImage` 是 **Linux 内核**，大小约 6.48 MiB(6792136 ÷ 1024 ÷ 1024 ≈ **6.48 MB**))
    40381   imx6ull-14x14-emmc-4.3-480x272-c.dtb
    40381   imx6ull-14x14-emmc-4.3-800x480-c.dtb
    40381   imx6ull-14x14-emmc-7-800x480-c.dtb
CMD_SEND:16
                ARG                      0x00000200
                MMC_RSP_R1,5,6,7         0x00000900
CMD_SEND:18
                ARG                      0x00005086
                MMC_RSP_R1,5,6,7         0x00000900
CMD_SEND:12
                ARG                      0x00000000
                MMC_RSP_R1,5,6,7         0x00000b00
    40381   imx6ull-14x14-emmc-7-1024x600-c.dtb
    40381   imx6ull-14x14-emmc-10.1-1280x800-c.dtb
    41217   imx6ull-14x14-emmc-hdmi.dtb
    41125   imx6ull-14x14-emmc-vga.dtb
8 file(s), 0 dir(s)

| 日志                                | 这次操作中的作用                                                   |
| --------------------------------- | ---------------------------------------------------------- |
| `CMD_SEND:16`                     | 设置后续数据传输的块长度                                               |
| `CMD_SEND:17`                     | 读取一个数据块                                                    |
| `CMD_SEND:18`                     | 连续读取多个数据块                                                  |
| `CMD_SEND:12`                     | 停止多块读取                                                     |
| eMMC 第 1 分区从 10 MiB 开始，大小 128 MiB | U-Boot：`mmc dev 1` 后执行 `mmc part`                          |
| 第 1 分区有原厂 `zImage` 和 `.dtb`       | U-Boot：`fatls mmc 1:1`                                     |
| 第 2 分区约 7.15 GiB，里面挂载了原厂根文件系统     | Linux：`cat /proc/partitions`、`cat /proc/cmdline` 和 `dmesg` |
| eMMC 前 10 MiB 没有普通分区              | 根据第 1 分区的起始扇区 `20480` 计算                                   |

6.选择 SD 卡
fatinfo mmc 0:2(查看 **SD 卡第 2 个分区**的 FAT 文件系统信息)
fatls mmc 0:2(列出该分区现有文件)
help fatwrite(查看当前 U-Boot 支持的写文件命令格式)

7.mkdir -p ~/imx6ull/sdcard-check
sudo mount -o ro /dev/sdb2 ~/imx6ull/sdcard-check(通过这个目录看到 SD 卡分区里的文件)
df -h ~/imx6ull/sdcard-check

发现sd卡不够备份之前的emmc 
用Ubuntu备份

sudo ip addr add 192.168.50.10/24 dev ens37（给直连开发板的网卡设置 `192.168.50.10`，子网掩码为 `/24`。）
sudo ip link set ens37 up（启用这张网卡）
ip -br addr show ens37（检查地址是否设置成功）
建立备份目录mkdir -p ~/imx6ull/emmc-backup

检查能否让 U-Boot 把 eMMC 通过 USB 暴露给 Ubuntu
uboot 执行
version
mmc dev 1
mmc info
help ums（检查是否支持 USB 大容量存储功能，以及具体命令格式）
想用usb把emmc的数据放进我的Ubuntu里面
```bash
gsettings set org.gnome.desktop.media-handling automount false（插入 U 盘 / SD 卡，系统不会自动挂载）
gsettings set org.gnome.desktop.media-handling automount-open false（就算自动挂载开着，挂载完也不会自动弹出文件管理器窗口）
```

（
1.从 SD 卡重新加载镜像到 DDR
mmc dev 0
mmc read 0x83000000 0x800 0x5a
mmc read 0x82000000 0x1000 0x49e3
mmc read 0x84000000 0x11000 0x1847
2.设置这次启动要用的参数
setenv bootargs 'console=ttymxc0,115200 earlycon=ec_imx6q,0x02020000 rdinit=/init loglevel=8 ignore_loglevel'
printenv bootargs（- `console`：指定正常串口控制台。
- `earlycon`：显示内核早期日志。
- `rdinit=/init`：运行 initramfs 中的 `/init`。）
3.bootz 0x82000000 0x84000000:0x308d68 0x83000000
）

**eMMC 用户区备份**

1.数据传输路径
	开发板 eMMC
	/dev/mmcblk1
	    ↓ Linux 的 MMC 驱动读取
	开发板 nc 发送
	    ↓ TCP，经以太网线
	Ubuntu nc 接收
	    ↓ 写入 Ubuntu 硬盘
	~/imx6ull/emmc-backup/emmc-user.img
	
2.给两端设置网络地址
Ubuntu 终端：
	sudo ip link set ens37 up（启用虚拟机中桥接到有线网卡的 ens37）
	sudo ip addr replace **192.168.50.10/24** dev ens37（给 ens37 网卡设置静态 IP 192.168.50.10，掩码 24 位）
开发板串口：
	ip link set eth1 up
	ip addr add **192.168.50.20/24** dev eth1    （前三段相同 → 同一网段  最后一段不同 → 区分两台设备）

3.确认链路和通信
开发板：
	cat /sys/class/net/eth1/carrier  
	ping -c 4 192.168.50.10
	
4.确认要读取哪个存储设备
	cat /proc/partitions（查看“有哪些存储设备和分区）
	dmesg | grep -i mmc（驱动识别存储设备的过程）
	cat /proc/mounts（哪些文件系统正在被使用）
	
5.Ubuntu 先准备接收
Ubuntu 执行:
	mkdir -p ~/imx6ull/emmc-backup
	nc -l 192.168.50.10 9000 > ~/imx6ull/emmc-backup/emmc-user.img

| 部分                | 意思                          |
| ----------------- | --------------------------- |
| `nc`              | 网络数据传输工具                    |
| `-l`              | 作为接收端，监听连接                  |
| `192.168.50.10`   | 监听 Ubuntu 的这个本地地址           |
| `9000`            | 本次使用的 TCP 端口                |
| `>`               | Shell 把收到的数据写到文件，已有同名文件会被覆盖 |
| `…/emmc-user.img` | 备份文件的保存位置                   |
6.开发板再开始发送
开发板执行:
	nc -w 3 192.168.50.10 9000 < /dev/mmcblk1(CP 发给 `192.168.50.10:9000`；如果空闲 3 秒，自动结束)
	
7.查看进度
Ubuntu执行：
	watch -n 2 'ls -lh ~/imx6ull/emmc-backup/emmc-user.img'（持续盯着这个 `emmc-user.img` 文件，**每 2 秒刷新一次**，看这个文件的大小有没有变化）

8.传完后检查大小和内容
Ubuntu执行：
	sync
	stat -c '%n：%s 字节' ~/imx6ull/emmc-backup/emmc-user.img
	sha256sum ~/imx6ull/emmc-backup/emmc-user.img
- `sync`：要求系统把待写入的数据同步到存储。
- `stat`：查看准确的文件字节数。
- `sha256sum`：计算文件的内容摘要。
开发板执行：
	cat /sys/class/block/mmcblk1/size
	sha256sum /dev/mmcblk1

9.其他文件传输
Ubuntu：先监听，指定保存文件
```
nc -l 192.168.50.10 9000 > 备份文件路径
```
开发板：再连接，指定读取设备
```
nc -w 3 192.168.50.10 9000 < 源设备路径
```

|路径部分|含义|
|---|---|
|`/sys`|内核向用户程序提供设备、驱动等信息的入口|
|`/sys/class`|按设备类别组织的信息|
|`/sys/class/net`|网络接口这一类|
|`/sys/class/net/eth1`|`eth1` 这个网络接口|
|`carrier`|该接口的链路状态属性|




**开始转移**
1.在开发板运行的Linux系统里面创建文件夹 并挂载emmc的文件 查看emmc-boot也就是查看/dev/mmcblk1p1
mkdir -p /mnt/emmc-boot
mount -o ro /dev/mmcblk1p1 /mnt/emmc-boot
ls -lh /mnt/emmc-boot（查看这个目录里**有哪些文件、文件大小**）
df -h /mnt/emmc-boot（查看这个分区**总容量、已用、剩余空间、使用率**）

| 区域                   | 大小         | 当前内容                                                         |
| -------------------- | ---------- | ------------------------------------------------------------ |
| 用户区前部，首个分区之前         | 10 MiB     | MBR 分区表、其他原始数据或空闲区域；具体启动数据位置尚未确认                             |
| 第 1 分区 `mmcblk1p1`   | 128 MiB    | 原厂 `zImage`、7 个不同屏幕配置的 DTB，以及 `System Volume Information` 目录 |
| 第 2 分区 `mmcblk1p2`   | 约 7.15 GiB | 原厂 Linux 根文件系统，包括程序、库和配置等                                    |
| 独立启动区 `mmcblk1boot0` | 4 MiB      | 已备份；具体内容尚未解析                                                 |
| 独立启动区 `mmcblk1boot1` | 4 MiB      | 具体内容尚未解析                                                     |
2.command -v mmc（检查当前环境里有没有 mmc 这个程序，并且打印它的路径）
`mmc partconf 1` → 查询 mmc1 (eMMC) 现在的启动分区配置

3.
**boot0、boot1 这两个区域是 eMMC 芯片自带的**

|东西|谁提供或决定|
|---|---|
|boot0、boot1 两个抽屉|eMMC 芯片提供|
|抽屉里装什么|原厂或你通过烧录写入|
|启动时选哪个抽屉|eMMC 启动配置决定|
|U-Boot|可以放进去的一份启动程序|
=> mmc partconf 1  
EXT_CSD[179], PARTITION_CONFIG:  
BOOT_ACK: 0x1  
BOOT_PARTITION_ENABLE: 0x1 (boot0)  （存放boot.imx）
PARTITION_ACCESS: 0x0 (user)

| 输出                                   | 含义                    |
| ------------------------------------ | --------------------- |
| `BOOT_ACK: 0x1`                      | 启用了 eMMC 启动确认响应       |
| `BOOT_PARTITION_ENABLE: 0x1 (boot0)` | eMMC 启动区域选为 **boot0** |
| `PARTITION_ACCESS: 0x0 (user)`       | 当前普通读写访问的是**用户区**     |

4.检查已备份的 boot0 和 U-Boot 配置
```
od -Ax -tx1 -N 32 ~/imx6ull/emmc-backup/emmc-boot0.img
```
**从 boot0 镜像偏移 1024 字节处，读取 32 字节原始数据，带十六进制偏移打印出来**。
```
od -Ax -tx1 -j 1024 -N 32 ~/imx6ull/emmc-backup/emmc-boot0.img
```
```
grep -E '^CONFIG_(ENV_|SYS_MMC_ENV)' \
  ~/imx6ull/build-uboot-emmc-test/.config
```
查看我们编译的 U-Boot 将环境变量存在哪里，避免布局冲突

U-Boot 程序的位置
→ Boot ROM 去哪里加载 U-Boot
环境变量的位置
→ U-Boot 运行后，去哪里读取保存过的设置

(1)board/nxp/mx6ullevk/mx6ullevk.c（放这个板子专属的初始化代码：板级 GPIO、电源、引脚、eMMC/SD、外设、板子硬件相关配置。）
(2)arch/arm/mach-imx/mx6/soc.c（是芯片本身的初始化：时钟、DDR、USDHC 控制器、芯片寄存器、SOC 基础能力。）
(3)bootcmd：上电后自动执行哪些命令？
(4)bootargs：给 Linux 传哪些启动参数？

| 命令            | 选择的区域        |
| ------------- | ------------ |
| `mmc dev 1 0` | eMMC 用户区     |
| `mmc dev 1 1` | eMMC 的 boot0 |
| `mmc dev 1 2` | eMMC 的 boot1 |
- `mmc read`：**MMC → DDR**（读：存储 → 内存）
- `mmc write`：**DDR → MMC**（写：内存 → 存储）

SD 卡中的三个镜像
    ↓ mmc dev 0 + mmc read
DDR 内存
    ↓ mmc dev 1 0 + mmc write
eMMC 用户区
列如

mmc dev 0
mmc read 0x83000000 0x800 0x5a 把sd卡的内容先放入 ddr
mmc dev 1 0
mmc write 0x83000000 0x800 0x5a 再把ddr的内容放入 emmc（user）

mmc read 0x85000000 0x800 0x5a 先放入0x85000000然后做对比
cmp.b 0x83000000 0x85000000 0xb400

**从 eMMC 加载并启动**
mmc dev 1 0

mmc read 0x83000000 0x800 0x5a
mmc read 0x82000000 0x1000 0x49e3
mmc read 0x84000000 0x11000 0x1847

setenv bootargs 'console=ttymxc0,115200 earlycon=ec_imx6q,0x02020000 rdinit=/init loglevel=8 ignore_loglevel'

bootz 0x82000000 0x84000000:0x308d68 0x83000000

| 内容        | 两者采用的起点 | 起始扇区      | 预留空间   | DDR 加载地址     |
| --------- | ------- | --------- | ------ | ------------ |
| 设备树       | 1 MiB   | `0x800`   | 1 MiB  | `0x83000000` |
| 内核        | 2 MiB   | `0x1000`  | 32 MiB | `0x82000000` |
| initramfs | 34 MiB  | `0x11000` | 8 MiB  | `0x84000000` |
`bootargs`
console=ttymxc0,115200          Linux 串口控制台
earlycon=ec_imx6q,0x02020000    早期串口日志
rdinit=/init                   执行 initramfs 中的 /init
loglevel=8 ignore_loglevel     显示详细日志

`/proc/partitions`
是内核提供的一份“块设备和分区清单,它显示有哪些设备、多大，不存放你的固件内容。

eMMC 用户区
0～1 MiB      保留
1～2 MiB      设备树预留区
2～34 MiB     内核预留区
34～42 MiB    initramfs 预留区
42 MiB 以后   尚未安排的数据空间

=> mmc part
```
CMD_SEND:16
ARG 0x00000200
MMC_RSP_R1,5,6,7 0x00000900
CMD_SEND:17
ARG 0x00000000
MMC_RSP_R1,5,6,7 0x00000900
```
1. **CMD16：CPU 跟 eMMC 说：后面咱们一次读 512 字节（一个扇区）** eMMC 回复：收到，我准备好了。
2. **CMD17：CPU 跟 eMMC 说：把 0 号扇区发给我** eMMC 回复：收到，我准备好了。

md.b 0x85000000 0x20
- `md` = memory display，看内存
- `.b` 代表按**1 字节**看
- `0x85000000`：DDR 内存地址
- `0x20`：一共读 32 个字节 一句话：**去 DDR 地址 0x85000000，打印后面 32 个字节的数据**

**1.把imx放入emmc**
**（1）先核对镜像大小，在 Ubuntu 执行**
stat -c '%s 字节' ~/imx6ull/build-uboot-emmc-test/u-boot.imx  (617472 字节)
617472 ÷ 512 = 1206 个扇区 = 0x4b6

**（2）在开发板 `=>`，从 SD 卡读取 U-Boot**
mmc dev 0
mmc info

mmc read 0x86000000 0x2 0x4b6
从 SD 卡第 2 扇区，即 1024 字节处，读取完整 U-Boot 到 DDR `0x86000000`
读取开头
md.b 0x86000000 0x10
d1 00 20 40（vit）

**（3）切换到 eMMC boot0，写入**
mmc dev 1 1  （这里的硬件区域 `1` 对应 Linux 的 `mmcblk1boot0`）
mmc info

mmc write 0x86000000 0x2 0x4b6 （把ddr的值放入boot0里面）

**（4）验证**
mmc read 0x87000000 0x2 0x4b6
cmp.b 0x86000000 0x87000000 0x96c00

**（5）校验通过后，测试从 eMMC 启动 U-Boot （改板子上面的boot_cfg启动）**

**卡住了 

1.**问题**
=> mmc list（`mmc list` 能看到控制器，并不代表控制器已经和 eMMC 通信成功）
FSL_SDHC: 0 (SD)
FSL_SDHC: 1
=> mmc dev 1
**Card did not respond to voltage select! : -110**

| 项目       | 说明                                                       |
| -------- | -------------------------------------------------------- |
| USDHC    | NXP 的 SD/MMC 控制器，全称 Ultra Secure Digital Host Controller |
| USDHC2   | i.MX6ULL 里第 2 个 SD/MMC 硬件控制器                             |
| 用途       | 一般用来接 **eMMC**（你这个 emmc 版本就是用它）；有的板子复用接 SD 卡             |
| 对应设备树    | `&usdhc2`                                                |
| U-Boot 里 | `mmc 1` 通常就是 USDHC2（mmc0 是 usdhc1）                       |
2.**不确定**
- 是不是 eMMC 芯片损坏；
- 是不是 U-Boot 没有驱动；
- 是不是引脚配置错误；
- 是不是供电、复位或电压协商出了问题。
**（1）先检查使用的是哪份配置**
```bash
grep -E '^CONFIG_(DEFAULT_DEVICE_TREE|DEFAULT_FDT_FILE|OF_CONTROL|OF_UPSTREAM|DM_MMC|SYS_BOARD|SYS_CONFIG_NAME|TARGET_MX6ULL.*)=' \
  ~/imx6ull/build-uboot-imx/.config（`.config`：本次构建实际使用的 U-Boot 配置文件）
```
CONFIG_SYS_BOARD="mx6ullevk"
CONFIG_TARGET_MX6ULL_14X14_EVK=y
CONFIG_DEFAULT_DEVICE_TREE="imx6ull-14x14-evk"
CONFIG_DM_MMC=y（开启驱动模型（DM）框架下的 MMC 驱动，支持 SD/USDHC）
**（2）接下来为什么比较配置**
源码里同时有普通 EVK 和 eMMC EVK 配置
```bash
diff -u \
  configs/mx6ull_14x14_evk_defconfig \
  configs/mx6ull_14x14_evk_emmc_defconfig
```
CONFIG_DEFAULT_DEVICE_TREE="imx6ull-14x14-evk-emmc"
CONFIG_SUPPORT_EMMC_BOOT=y
编译了 eMMC 版本，放到 SD 卡启动。结果依然是 `mmc dev 1` 超时
**（3）检查最终设备树有没有用上**
```bash
find ~/imx6ull/build-uboot-emmc-test -type f -name '*.dtb'
```
- `find`：查找文件；
- `-type f`：只找普通文件；
- `-name '*.dtb'`：找名称以 `.dtb` 结尾的文件。
**反编译设备树**
```bash
dtc -I dtb -O dts \
  -o "$HOME/imx6ull/build-uboot-emmc-test/inspect-uboot.dts" \
  "$HOME/imx6ull/build-uboot-emmc-test/dts/dt.dtb"
```
- `dtc`：设备树编译工具；
- `-I dtb`：输入是二进制设备树；
- `-O dts`：输出为可读文本；
- `-o`：指定输出文件。
**搜索 USDHC2**
```bash
grep -n -A 40 \
  -E '(mmc|usdhc)@2194000|usdhc2grp' \
  ~/imx6ull/build-uboot-emmc-test/inspect-uboot.dts
```
- `-n`：显示行号；
- `-A 40`：匹配到的内容之后，再显示 40 行；
- `-E`：允许使用扩展匹配。
得到
bus-width = <0x08>;
status = "okay";
pinctrl-0 = <0x2d>;
non-removable;
最终构建出的设备树确实启用了 USDHC2，并配置了 8 位引脚组。
**（4）看似相关的线索，要确认作用阶段**
正点原子文档提到，在 Linux 设备树中添加：
```
no-1-8-v;
```
检查u-boot里面
```bash
grep -R -n -A 12 -B 12 \
  'no-1-8-v' drivers/mmc
```
- `-R`：递归搜索目录；
- `-B 12`：同时显示匹配行前面 12 行；
- `-A 12`：显示后面 12 行。
	if (dev_read_bool(dev, "no-1-8-v")) {
	    cfg->host_caps &= ~(UHS_CAPS | MMC_MODE_HS200 |
	                       MMC_MODE_HS400 | MMC_MODE_HS400_ES);（把这几个高速能力位全部清除掉）
	}
说明 U-Boot 通用 MMC 代码认识这个属性，它会限制部分高速模式。
现在的失败是在高速模式切换时发生的，还是更早就发生了？

搜索报错字符串：
```bash
grep -R -n -A 20 -B 25 \
  'Card did not respond to voltage select' drivers/mmc
```
`drivers/mmc/mmc.c`：程序先尝试 SD，再尝试 MMC/eMMC 的初始识别

**不要只看错误文字里的“voltage”就猜测电压切换失败，要去源码里找到打印错误的位置。**

**（5）打开 MMC 跟踪，观察设备到底回应了什么**
```bash
grep -R -n -A 12 -B 3 \
  -E '^config MMC_TRACE|^config MMC_VERBOSE' drivers/mmc
```
找到CONFIG_MMC_TRACE（目的打开 MMC 调试打印跟踪）
```bash
./scripts/config \
  --file "$HOME/imx6ull/build-uboot-emmc-test/.config" \
  --enable MMC_TRACE
```
`./scripts/config` 是内核 / U-Boot 源码自带的脚本工具，用来**命令行直接修改 .config 配置，不用打开 menuconfig**
编译配置：
```bash
make ARCH=arm CROSS_COMPILE="$CROSS_COMPILE" \
  O="$HOME/imx6ull/build-uboot-emmc-test" olddefconfig
```
```bash
make ARCH=arm CROSS_COMPILE="$CROSS_COMPILE" \
  O="$HOME/imx6ull/build-uboot-emmc-test" -j4 u-boot.imx
```
检查是否开启
```bash
grep '^CONFIG_MMC_TRACE=' ~/imx6ull/build-uboot-emmc-test/.config
```
得到
CMD_SEND:0
MMC_RSP_NONE

CMD_SEND:8
RET -110

CMD_SEND:55
RET -110

CMD_SEND:1
RET -110
- `CMD0` 不要求响应，因此看到 `MMC_RSP_NONE` 不能说明 eMMC 已经正常回应。
- `CMD8`、`CMD55` 在这里属于前面的 SD 探测。
- **`CMD1` 是后面尝试识别 MMC/eMMC 的关键命令；它也超时了。**
**（6）找一套能工作的系统做对照**
```bash
cat /proc/cmdline
cat /proc/partitions
dmesg | grep -iE 'mmc|sdhci|usdhc'
```
原来的Linux查看
mmcblk1: mmc1:0001 8GTF4R 7.28 GiB
EXT3-fs (mmcblk1p2): mounted filesystem
原厂的 u-boot执行可以查出mmc dev 1
**（7）对照原厂源码，找到gpio线索**
源码：
	#define USDHC2_PWR_GPIO IMX_GPIO_NR(4, 10)
	gpio_direction_output(USDHC2_PWR_GPIO, 0);
	udelay(500);
	gpio_direction_output(USDHC2_PWR_GPIO, 1);
原厂程序在初始化 USDHC2 前，会让 GPIO4_IO10 **先低、等待 500 微秒、再高**
**（8）检查新版 U-Boot 的引脚实际状态**
先在源码中查到引脚复用定义
```bash
grep -n -E 'NAND_ALE__(GPIO4_IO10|RAWNAND_ALE)' \
  dts/upstream/src/arm/nxp/imx/imx6ul-pinfunc.h
```
NAND_ALE__RAWNAND_ALE  ... 0 ...
NAND_ALE__GPIO4_IO10   ... 5 ...

在自己下的u-boot读取引脚复用寄存器
md.l 0x020e01a0 1（得到020e01a0: 00000005）
再读取 GPIO4 的数据、方向和实际电平
md.l 0x020a8000 3（得到020a8000: 00007000 00000000 00007000）
- `0x020a8000`：GPIO4_DR（数据寄存器）
- `0x020a8004`：GPIO4_GDIR（方向寄存器）
- `0x020a8008`：GPIO4_PSR（状态寄存器，当前引脚真实电平）

gpio status -a（GPIO4_10: input: 0 [ ]）
原厂 U-Boot：
GPIO4_IO10 → 输出，最终为高电平
新版 U-Boot：
GPIO4_IO10 → 输入，读取为低电平
找到了问题
**（9）gpio set GPIO4_10（把 GPIO4_IO10 设置为输出高电平）**




